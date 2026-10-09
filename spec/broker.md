# curator-credential-broker: specification

- **Status:** draft v0.2 (2026-10-09). Nothing is implemented.
- **Normative language:** MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.
- **Companions:** [curator-host-helper](https://github.com/relux-works/curator-host-helper) (per-agent OS users, the launcher that starts processes under them, and per-UID firewall rules), [curator-network-profiles](https://github.com/relux-works/curator-network-profiles) (egress profiles), curator-spec CIP-0010 and CIP-0011 (the Curator side: credential sources, the protected credential binding, the executor capability, the launch-plan extension).

### Revision 0.2

An architecture review on 2026-10-09 kept the design and asked for tighter contracts. This revision:

- treats an authenticated socket as a capability with a bounded lifetime and states the rules for privileged clients (§4.1);
- keys every managed principal by UID, generation and active state, and names agent leaves by their account generation instead of a reusable label (§4.2, §6.3, §7);
- freezes a non-circular grant format with domain-separated hashing and signing, verifies every delegation edge including the leaf, and closes the v0 grammar (§6);
- gives local revocation a publication point and fail-closed reads (§6.4);
- binds every lease to its owner and repeats authorisation on renewal (§8);
- implements CIP-0010's protected credential binding for broker mode (§8.2, §12.2);
- removes the per-turn token-only Codex home: Codex personal plans are served only through app-server external tokens (§8.4, §9);
- completes the external-token contract (account identity, initialisation, deadlines) and the auth owner's enrolment ceremony and failure states (§9);
- keeps networking explicitly cooperative until the helper publishes trusted applied state (§5.5);
- reorders delivery so that the first slice is one protected Claude launch (§16).

## 1. Purpose

Agent systems run many harness processes (Claude Code, Codex, Muse, and others) on behalf of a few people. Each harness needs the credential of a person's subscription or API account. Today every managed home keeps its own login, which costs a browser login per home and, for refresh-token logins, lets parallel copies invalidate each other.

The broker is a small service that:

1. holds each account's credential once, under its own OS user, in its own store;
2. hands a credential only to a launch that is authorised for it, identifying the launching process by its OS user through the kernel;
3. decides authority from signed grants that chain to a configured root key, so that orchestrators can delegate narrower rights to the agents they start;
4. serialises refresh for accounts whose credentials rotate (Codex personal plans), so that exactly one process refreshes and launches receive short-lived access tokens;
5. can require that an account is used only through a named network profile.

### 1.1 Non-goals

- The broker does not run harnesses, compose launch plans or manage homes (Curator does).
- It does not create OS users or start processes under them (curator-host-helper does) and does not decide who may start an agent (the dispatcher does).
- It does not authenticate to model vendors on its own behalf, bypass a vendor's login flow, or intercept model traffic. Enrolment always uses the vendor's own flow, completed by a person.
- It does not make a credential non-extractable. A credential delivered into a harness process is readable by that process (§12.1).
- It does not multiply one person's subscription across people. One account belongs to one person.

## 2. Terms

| Term | Meaning |
|---|---|
| account | One credential of one person for one harness family, for example a Claude subscription token or a Codex ChatGPT login. Identified by an account id such as `ivan/claude/personal`. |
| owner | The person an account belongs to, identified by the OS account that enrolled it (§4.2). |
| principal | A participant the broker authorises: an operator, a dispatcher, an account owner, or an agent. |
| generation | The never-reused identifier that curator-host-helper assigns when it creates an OS account (its ledger, helper §3.4). |
| grant | A signed statement that a key or an agent generation may lease named accounts for named harnesses and profiles in a time window (§6). |
| binding | The broker's record that an agent OS account (UID and generation) currently acts as one agent, with the grant chain it uses (§7). |
| dispatcher | relux-works/curator-dispatcher, running under its own service account: it asks curator-host-helper for an OS account, binds it in the broker, and starts processes under it through the helper's launcher. Curator's `agent-user` commands are its clients (CIP-0011). |
| lease | One authorised use of an account's credential by one binding for one launch (§8). |
| auth owner | The broker-side component that alone refreshes a rotating account (§9). |
| executor | The final executor of a launch: the trusted process that runs under the agent's OS account, resolves the protected credential binding, requests the lease and execs the harness. |

## 3. Components

| Component | Runs as | Job |
|---|---|---|
| broker daemon | its own unprivileged service user (`cur-s-broker` by default) | listens on the broker socket; keeps accounts, grants, revocations, bindings and leases; runs auth owners; writes the audit log |
| account store | files owned by the broker user | account records and their secret material (§5.3) |
| grant store | files owned by the broker user | signed grants and the revocation state (§6.4) |
| auth owners | inside the daemon, one per rotating account | refresh (§9) |
| client library (`pkg/client`) | inside executors, dispatchers and the CLI | the socket protocol, delivery of material to a harness, renewal |
| CLI | the calling person or component | `curator broker …` in Curator, or the standalone `curator-broker` binary (§14) |

The broker opens no network listener. Its only interface is a Unix-domain socket.

## 4. Identities and trust

### 4.1 Peer identity and connections

On every connection the broker MUST obtain the peer's effective UID from the kernel (`getpeereid` or `LOCAL_PEERCRED` on macOS and BSD, `SO_PEERCRED` on Linux) and MUST refuse the connection if the platform cannot provide it. A UID supplied in a request is never trusted.

The kernel reports the credentials of the process that **connected**, not of whoever writes later on that descriptor. An authenticated connection is therefore a capability, and the following rules apply:

1. Clients MUST open broker connections close-on-exec and MUST NOT pass them to another process (`SCM_RIGHTS`) or let them be inherited. A trusted caller that deliberately passes a descriptor delegates its own authority; this specification treats that as the caller's act.
2. Operators and dispatchers SHOULD use one connection per operation. A process that changes credentials (for example a dispatcher starting an agent) MUST close its broker connections before the transition; the agent's executor opens a fresh connection afterwards.
3. The broker associates each connection with the principal it authenticated at connect time (§4.2) and closes every connection of a UID when that UID's binding is removed or its account generation is retired.
4. Clients MUST check that the server's peer UID is the configured broker service UID before sending anything (`server_identity_mismatch`).
5. Lease methods accept only bound agent principals; administrative methods accept only operators, dispatchers and owners (§13.2). A connection never changes role.

The socket lives in its own directory outside the broker's data tree (§15.2). Qualification covers a connection opened before a credential change, inherited and passed descriptors, and substitution of the socket path (§16).

### 4.2 Principals

The broker's configuration (owned by the broker user, §15) names operators and dispatchers. Each principal is recorded as follows:

- **Managed principals** (accounts created by curator-host-helper: agents, and dispatchers or services that run under helper-created service accounts) are keyed by `(uid, generation)` and are valid only while the helper ledger shows that generation as `active` for that UID. The broker reads the ledger file directly (helper §3.4); a missing, ambiguous, retired, malformed or unreadable entry refuses (`principal_ledger_mismatch`). The check is repeated on bind, lease and renew.
- **Unmanaged principals** (people's own OS accounts acting as operators or owners) are pinned at configuration by UID plus the account's directory identity (on macOS the account's `GeneratedUID`; on Linux the user name and home path). A change refuses until an operator re-pins the principal (`principal_identity_changed`).
- Every principal that enrols an account becomes that account's **owner**.

### 4.3 Trust root

Grants are verified against one or more configured **root public keys** (Ed25519). In v0 the root is a local software key of the operator; `curator broker grant …` signs with it. v0 is therefore a local broker authorisation format with a software trust root.

A later key keeper (platform architecture §7.2) changes more than the configured root: keeper-derived keys need a root-signed identity certificate, roles need a mapping, old grants need reissue, and revocations must survive. Migration is an explicit procedure: certify or replace keys, map roles, run old and new roots side by side for a transition window, reissue grants and replace bindings under the new root, preserve the revocation state, then retire the old root. The broker gains a versioned verification adapter for the keeper's formats; this specification does not promise that the v0 verifier stays unchanged.

## 5. Accounts

### 5.1 Record

```
account/1
{ id:           "ivan/claude/personal",       // owner-chosen, unique in the broker
  harness:      "claude_code",                // claude_code | codex_cli | muse | gemini | qwen | …
  kind:         "claude-oauth-token",          // §5.2
  owner:        principal reference,           // §4.2
  vendor_account: string | null,               // the vendor's account identifier where one exists (Codex: chatgpt_account_id)
  enrolled_at:  "2026-10-09T10:00:00Z",
  issued_at:    "2026-10-09T09:58:00Z" | null, // only when the owner stated it
  expires_at:   "2027-10-09T09:58:00Z" | null, // computed from issued_at and the kind's lifetime, else null
  network:      { profile_ref, profile_digest, assurance: "cooperative" | "enforced" } | null,   // §5.5
  state:        "active" | "refresh_failed_transient" | "reauth_required" | "owner_unavailable" | "revoked",
  last_refusal: optional { at, code } }
```

The record never contains secret material.

### 5.2 Kinds

| Kind | Material | Refresh | Delivery channels (§8.4) |
|---|---|---|---|
| `claude-oauth-token` | the one-year token printed by `claude setup-token` | none: a new token and a restart | environment `CLAUDE_CODE_OAUTH_TOKEN` |
| `codex-chatgpt` | a Codex ChatGPT login held by the auth owner | the auth owner only (§9) | Codex app-server external tokens only |
| `codex-access-token` | a Business or Enterprise access token | none in the broker (admin-issued) | environment `CODEX_ACCESS_TOKEN` |
| `api-key` | a vendor API key (`ANTHROPIC_API_KEY`, `CODEX_API_KEY`, `META_API_KEY`, `GEMINI_API_KEY`, …) | none | environment, or stdin where the harness supports it |
| `muse-account` | reserved: Muse subscription login | to be researched (§10.2) | reserved |

### 5.3 Secret store

Material is kept behind a secret-store interface (`put`, `get`, `delete`, `describe`). **v0 ships one backend: a file with mode 0600 in a 0700 directory owned by the broker user**, on macOS and Linux alike, written by atomic replacement in its own directory. Keychain or Secret Service backends MAY be added later; a backend that can prompt a person MUST NOT be used for launches that cannot answer a prompt.

### 5.4 Enrolment

Enrolment is always performed by a person through the vendor's own flow:

- **Claude:** the person runs `claude setup-token` and pipes the printed token to `curator broker enrol claude_code --account <id> --token-stdin [--issued-at <date>]`.
- **Codex personal plan:** a ceremony run by the broker under its own user (§9.2): the broker starts the vendor's device login in a fresh broker-owned Codex home and relays the verification address and code to the person's CLI; the person completes the login in a browser. No existing login is imported, no other user's home is adopted, and no keyring item is exported.
- **Business access tokens and API keys:** `--access-token-stdin` or `--api-key-stdin`.

Material travels over the socket once and is written to the secret store. Enrolling a deliberately supplied token is not permission to extract an existing native login, and the broker never reads another user's harness stores.

### 5.5 Network requirement

An account MAY require a network profile: `{ profile_ref, profile_digest, assurance }`, using the identifiers of curator-network-profiles.

- **Cooperative (v0).** The executor resolves the launch's network binding with curator-network-profiles as the destination-local process owner and passes the resulting record in the lease request. The broker checks `profile_ref` and `profile_digest` against the account's requirement and records the result as **declared**. The record proves only what the requesting process asserted; network profiles in this version set proxy settings that a process can ignore.
- **Enforced (later).** An account that requires `assurance: "enforced"` is refused (`lease_network_enforcement_unavailable`) until curator-host-helper v1 publishes trusted applied state for the agent's `(uid, generation)`: the profile digest, the proxy listener identity and the firewall generation. The broker then matches leases and renewals against that state, not against the request, and refuses drift.

### 5.6 One person, one subscription

An account belongs to the person who enrolled it. The broker MUST NOT issue a lease for an account unless the grant chain's root authorises that account (§6). Bounds on concurrent use are not part of v0; grants that state them are refused rather than ignored (§6.2).

## 6. Grants

### 6.1 Format

A grant is an envelope with a body, the body's identifier and a signature:

```
grant envelope
{ body: {
    type:       "grant/1",
    issuer:     "ed25519:<base64url, 32 bytes, unpadded>",
    subject:    "ed25519:<…>"           // a delegating key
              | "agent:<generation>",    // a leaf only: one agent account generation (§6.3)
    actions:    ["credential.lease"],
    accounts:   ["ivan/claude/personal"],   // exact account ids
    harnesses:  ["claude_code"],
    profiles:   ["dev", "review"],          // Curator profiles a lease may serve
    not_before: "2026-10-09T10:00:00Z",
    expires_at: "2026-10-10T10:00:00Z",
    depth:      0,                          // remaining delegation depth
    parent:     "sha256:<hex>" | null },
  id:   "sha256:<hex>",
  sig:  "<base64url, 64 bytes, unpadded>" }
```

- The preimage is the domain tag `curator-broker grant/1` followed by a newline and the CCJ-1 canonical bytes of `body`. `id = "sha256:" + lowercase hex of SHA-256(preimage)`; `sig` is the Ed25519 signature of the issuer over the same preimage. Neither `id` nor `sig` is part of the body, so there is no circular preimage.
- A verifier recomputes `id` from `body` and refuses a mismatch (`grant_id_mismatch`).
- CCJ-1's rejection rules apply: duplicate keys, invalid Unicode, floating-point numbers, integers outside the safe range, negative zero and unknown members refuse (`grant_malformed`). Times are RFC 3339 UTC with a `Z` suffix.
- The repository publishes canonical byte vectors and negative vectors for every rule.

### 6.2 Chain verification

A request carries the full chain as envelopes, leaf first. The broker accepts it only when all of the following hold:

1. The chain is between 1 and 4 envelopes long, contains no repeated id and no record that is not part of the chain (`grant_chain_malformed`).
2. Every envelope is well formed and its signature verifies (`grant_malformed`, `grant_id_mismatch`, `grant_signature_invalid`).
3. For **every** adjacent pair `(child, parent)`, starting with the leaf and its parent: `child.parent == parent.id`; `child.issuer == parent.subject`, which therefore must be a key; and the child narrows the parent (`grant_chain_broken`, `grant_not_narrowing`).
4. The last envelope is the root: `parent` is null and its `issuer` is a configured root key (`grant_root_unknown`).
5. Narrowing means: actions, accounts, harnesses and profiles are subsets of the parent's; `[not_before, expires_at]` lies within the parent's window; `child.depth < parent.depth`.
6. Only the leaf may have an `agent:` subject, and a leaf with an `agent:` subject has `depth: 0` (`grant_subject_invalid`). Role subjects, wildcards and other conditions are not part of v0 and refuse (`grant_unsupported`).
7. No envelope, issuer key or subject key is revoked (§6.4), and the current time is inside every window (`grant_revoked`, `grant_expired`, `grant_not_yet_valid`).

The verifier's test suite includes a valid chain and mutants that skip each rule, including one that skips only the leaf edge.

### 6.3 Agent leaves

An agent is named in a grant by its helper account generation, `agent:<generation>`. Generations are never reused, so a grant cannot be reused by a later agent that happens to get the same label, user name or UID. Display labels are kept in the binding for people to read and play no part in authorisation.

### 6.4 Revocation

A revocation is an envelope of the same construction with the domain tag `curator-broker revocation/1`:

```
revocation body
{ type: "revocation/1",
  revokes: { grant: "sha256:<hex>" } | { key: "ed25519:<…>" },
  by:   "ed25519:<…>",
  at:   time,
  reason: optional string }
```

- A grant may be revoked by its issuer, by a root key or by an operator's key. A key may be revoked only by a root key or an operator's key; a revoked key invalidates every envelope it issued and every envelope whose subject it is.
- **Publication.** In v0 the broker keeps one authoritative local revocation state `{ seq, records }`. A revocation is admitted under the broker's state lock, which also serialises lease issuance and renewal: the broker writes the new state to a temporary file, flushes it, renames it over the old one, flushes the directory, updates its memory, and only then acknowledges. Every lease and renewal checks the chain against the state current under the same lock immediately before releasing material.
- **Fail closed.** If the revocation state cannot be read or does not parse, the broker refuses every lease and renewal (`revocation_state_unavailable`) until an operator repairs it; it never starts from an empty set after a failed read.
- A revocation stops new leases and renewals. It cannot erase an access token that a harness already holds; that ends with the token's expiry or with revocation at the vendor (§12.1).

### 6.5 Registry

Where grants and revocations live beyond one machine is open: a git journal, a registry service with signed snapshots, carrier room state, or a hybrid with local verified caches. Because chains travel with requests, the choice mainly affects revocation and discovery. Any remote form MUST provide authenticated freshness and rollback protection for revocation snapshots before a broker relies on it.

### 6.6 Relation to the trust design

curator-trust (capabilities, delegation verification) and waggle (signed envelopes and keys) define the platform's long-term forms. Broker lease grants are not board capabilities. Reconciliation goes through a versioned adapter or a shared policy layer before 1.0 (§4.3); where the platform forms differ, they win and this format gains a version.

## 7. Bindings

### 7.1 Record

```
binding/1
{ binding_id:   random 128-bit identifier, never reused,
  uid:          612,
  generation:   "g-0193",                 // helper ledger, §4.2
  label:        "dev-7f3",                // for people; not used for authorisation
  profiles:     ["dev"],
  chain:        [leaf id, …, root id],    // leaf subject = "agent:g-0193"
  created_by:   principal reference of the dispatcher,
  request_id:   the dispatcher's request id (for reconciliation),
  created_at, expires_at,
  state:        "active" | "removed" }
```

### 7.2 Creating and removing

- `bind` MUST come from a dispatcher principal (§4.2) and MUST name a UID whose ledger entry is an active **agent** account of that generation, created by the same dispatcher (or by an operator). The chain MUST verify (§6.2) with leaf subject `agent:<generation>`.
- A bind with a `request_id` already used by the same dispatcher returns the existing binding if the arguments are equal and refuses otherwise (`bind_request_conflict`).
- `unbind` comes from the dispatcher that created the binding or from an operator. It ends the binding's leases and closes the UID's connections. Expired bindings are removed by the broker.
- A UID with no active binding receives `lease_unbound` for every lease request.

### 7.3 Ephemeral agents

A binding's `expires_at` is the end of the agent's task. The dispatcher removes it when the task ends and before the OS account is retired. The broker never extends a binding on its own.

### 7.4 Account reuse

UIDs are recycled when OS accounts are deleted, and labels can repeat. Neither carries authority: bindings, grants and leases are keyed by the generation, and every lease and renewal re-reads the ledger (§4.2). A generation that the ledger shows as retiring or removed is refused (`principal_ledger_mismatch`).

## 8. Leases

### 8.1 Request

```
lease.request/1
{ harness:   "claude_code",
  profile:   "dev",                       // the Curator profile of the launch
  account:   "ivan/claude/personal",      // the account the user-owned configuration names
  channel:   "env" | "stdin" | "external-token",
  credential_binding: {                   // CIP-0010 C3.4, resolved by the executor before exec
    digest, harness, profile, account, channel,
    endpoint, executable_sha256, policy_generation },
  network:   { profile_ref, profile_digest, assurance } | null,
  launch_id: string,                      // correlation only (plan digest or run id)
  executor:  { capabilities: ["credential-injection/1"], version } }
```

### 8.2 Checks, in order

1. The peer is the agent principal of an active binding (`lease_unbound`, `principal_ledger_mismatch`).
2. The binding's chain verifies now against the current revocation state (§6.2, §6.4).
3. The leaf covers `credential.lease` on the account, the harness and the profile, and the profile is one of the binding's profiles (`lease_not_authorised`).
4. The account exists, its harness equals the requested harness, and it is `active` (`lease_account_unknown`, `lease_harness_mismatch`, `lease_account_revoked`, `lease_reauth_required`, `lease_owner_unavailable`).
5. The channel is valid for the kind (`lease_channel_unsupported`), and the harness release is qualified for this kind and channel (`lease_harness_unqualified`).
6. The credential binding's fields equal the request's, its endpoint is one the broker's configuration approves for the kind, and its executable digest is on the broker's approved list for the harness (`lease_binding_mismatch`).
7. The account's network requirement is met as described in §5.5 (`lease_network_mismatch`, `lease_network_enforcement_unavailable`).

Checks 2 to 7 run under the state lock immediately before the material is released.

### 8.3 Response and lease record

```
lease/1
{ lease_id,
  account, harness, channel,
  payload: { channel: "env", variable: "CLAUDE_CODE_OAUTH_TOKEN", value }
         | { channel: "stdin", value }
         | { channel: "external-token", access_token, chatgpt_account_id, plan_type? },
  token_expires_at: time | null,        // the vendor token's expiry when known
  authorized_until: time,               // the broker's own limit: earliest of grant, binding and account expiry
  renewable: bool }
```

The payload appears only in this response and in renewal responses, never in logs. The broker persists the lease without material:

`(lease_id, uid, generation, binding_id, account, harness, profile, channel, credential_binding digest, launch_id, policy generation, authorized_until, state: active | released | expired | revoked)`.

### 8.4 Delivery channels

The client library delivers material only into the process the executor starts:

- `env`: the variable is set in the harness process environment at exec, never in the parent, argv, files or logs.
- `stdin`: for harnesses that read a key from stdin.
- `external-token`: for a Codex app-server in external-token mode (§9.3).

Conflicting sources are refused, never replaced (§12.2). There is no channel that writes a credential file for a launch.

### 8.5 Renewal, release and reports

- `lease.renew`, `lease.release` and `event.report` name a `lease_id` and are accepted only from the lease's own `(uid, generation, binding_id)` (`lease_not_owner`).
- Renewal is refused after `authorized_until` or once the lease is released, expired or revoked. It repeats checks 1 to 7 before releasing new material, including after an asynchronous refresh (§9).
- Release is idempotent and ends renewal.
- A report affects only the reporting lease's record. A client report never changes an account's state; account states change by the owner's command or by the auth owner's own refresh results (§9.4).
- Leases are not tied to connections; a disconnected executor can renew from a new connection of the same principal.

## 9. Codex auth owner

### 9.1 Why

A Codex ChatGPT login rotates its refresh token on every refresh. Several processes refreshing one login race and can log the subscription out. One component per account must refresh.

### 9.2 Enrolment ceremony

1. A person runs `curator broker enrol codex_cli --account <id>`.
2. The broker creates a fresh Codex home for the account inside its own store, configured for file credential storage, and runs the approved Codex executable (pinned path and digest) with a sanitised environment to start the vendor's device login.
3. The broker relays the verification address and code to the person's CLI. The person completes the login in a browser.
4. The broker reads the vendor account identifier from the new login and refuses if another broker account already maps to it (`account_duplicate`).
5. The auth owner starts and the account becomes `active`.

The broker never adopts an existing home, imports another login or exports a keyring item.

### 9.3 External-token mode

- The executor starts `codex app-server`, initialises it with the experimental capability that external-token login requires, and logs in with `account/login/start { type: "chatgptAuthTokens", accessToken, chatgptAccountId }` from the lease payload. A failed initialisation refuses the launch (`external_token_init_failed`).
- When the app-server receives a 401 it asks its client with `account/chatgptAuthTokens/refresh`. The executor calls `lease.renew { lease_id, rejected_sha256 }`, verifies that the renewed payload names the same `chatgpt_account_id`, and answers with the new token and account id.
- The app-server waits only a bounded time for that answer (about ten seconds in the vendor's documentation at the time of writing). The broker answers `lease.renew` within a configured deadline below that bound (default 8 seconds) or returns `renew_timeout`; the executor then answers the app-server with an error and the turn fails. No late or uncorrelated answer is sent, and nothing falls back to another login. Proactive refresh (§9.4) makes a 401-triggered refresh rare; it does not guarantee that every turn continues.
- The launch home contains no `auth.json`.

### 9.4 Refresh and states

- The auth owner holds a stable lock file beside the account's Codex home for the whole read, refresh and reload sequence. The vendor's own processes do not honour this lock, which is why no other consumer may use the home (§9.5).
- When the current access token expires within the refresh window (48 hours in the reference implementation), the owner refreshes by running the approved Codex executable once; it never reimplements the vendor's token exchange. Requests queue for the owner with a bound; a full queue returns `renew_timeout`.
- `lease.renew { rejected_sha256 }` refreshes only if the rejected token is still the current one; N launches rejected with the same token cause one refresh.
- States: a transient failure (network, vendor error, timeout) sets `refresh_failed_transient` and retries with backoff while the current token is still valid; a rejected refresh token sets `reauth_required` and leases are refused until the person re-enrols; a stopped owner sets `owner_unavailable` and leases are refused.
- The refresh token never leaves the broker's store. The broker never falls back to a harness's own login.

### 9.5 Other consumers

Every consumer of an enrolled Codex account on the machine uses the broker. A person who also runs Codex by hand with the same vendor account on the same machine is outside the owner's guarantee; `status` reports the account's single-owner rule. Using one vendor account on several machines needs a separate ownership design and is not supported.

### 9.6 Qualification

The mechanism is measured in relux-works/remote-worker-harness (`internal/authowner`: eight concurrent callers near expiry cause one refresh; five rejected tenants cause one renewal) and was live-accepted on Codex 0.155.1. Codex 0.158 added a fetch of an application network policy; the reference tests run only on 0.155.x. A Codex release is qualified only when the real login, policy fetch, 401 renewal, renewal failure, owner contention and slow renewal pass with the approved executable and configuration; a skipped test is not a pass. Until then the broker refuses `codex-chatgpt` leases for that release (`lease_harness_unqualified`).

## 10. Other harnesses

### 10.1 Claude Code

A `claude-oauth-token` has no refresh. When a harness reports an authentication failure, the executor reports it on its lease (`credential_error`, §11). The account moves to `reauth_required` only on the owner's command. Recovery is re-enrolment and a restart; Claude Code resumes its transcript with `--resume`. Vendor-side revocation happens only in the vendor's web interface.

### 10.2 Muse

A Muse subscription login is required, not only API keys. Its store, refresh behaviour and concurrency are researched in parallel with the first slices; if it rotates like Codex, it gets an auth owner of its own. Until then only `api-key` with `META_API_KEY` is supported for Muse.

## 11. Events

| Event | When | v0 surface |
|---|---|---|
| `lease_expiring` | a token with a known expiry expires within 5 minutes, or a Claude token with a stated issue date within 3 days | `status`, audit log |
| `lease_expired` | the expiry passed | same |
| `credential_error { class: auth \| permission \| quota \| unknown }` | an executor reports a harness failure on its own lease; `unknown` when a 401 or 403 cannot be told apart | same |
| `reauth_required` | the owner marked the account, or the auth owner's refresh was rejected | same |

Events carry typed metadata only. Later versions deliver them to the session host and the message bridge, so that a session is parked and a person is asked to re-enrol from their phone.

## 12. Security

### 12.1 What a compromised agent can do

It can read the credential leased to it and copy it within its OS account; this trusted-process posture is a scoped, owner-approved departure from the stronger secret-free agent model of curator-trust. It cannot obtain accounts outside its grants, cannot bind itself, cannot forge its UID, cannot read the broker's store, and cannot refresh a Codex account. A revocation stops new leases; a token already copied lives until it expires or is revoked at the vendor. Stronger non-extraction (injection at an egress proxy, or a credential the process cannot read) is a separate design.

### 12.2 Protected binding and conflicting sources

Broker mode implements CIP-0010's protected credential binding (`credential_binding/1`). The executor, running under the agent's account before exec, resolves the tuple destination-locally: peer identity, profile, harness, account, channel, vendor endpoint, approved executable digest and policy generation. It checks the tuple before requesting the lease and again immediately before exec, and refuses on any change.

The executor MUST refuse to exec when the inherited environment, a harness settings or configuration file the launch would read, a native credential store selector, or a configured helper already supplies a credential for the same harness (`credential_source_conflict`). It never replaces a conflicting source silently.

Only an executor whose harness tuple has been qualified to keep the credential out of child processes it does not control (MCP servers, hooks, tool subprocesses), and to use the leased source as the effective source, may declare `credential-injection/1`. A clean environment at exec does not by itself control what a harness passes to its children, so this is part of each harness release's qualification.

The broker's binding check (§8.2 step 6) keeps configuration consistent; it is not a boundary against a compromised agent, which can already read its own lease (§12.1).

### 12.3 The broker itself

The broker holds every enrolled credential, so it stays minimal: one service user, no network listener, no plugins, no shell-outs except the approved Codex executable for enrolment and refresh, an append-only audit log, and rate limits per principal. The audit log records who, which account, which launch and which outcome as typed metadata; it never records material, raw protocol frames, secret-bearing errors or token hashes.

### 12.4 Files

All broker files are owned by the broker user; directories are 0700. Every state file (configuration, accounts, grants, revocations, leases) is replaced atomically within its own directory. The broker refuses to start if its store or configuration, or any parent directory, is writable by another user.

## 13. Socket protocol `broker/1`

### 13.1 Transport

A Unix-domain stream socket. Frames are a 4-byte big-endian length followed by a CCJ-1 canonical JSON object; the broker rejects frames over 64 KiB before reading them. The first exchange is `hello { versions: [1] }`; an unknown version closes the connection. Each request carries a client-chosen `request_id`; every response echoes it. Requests have deadlines: 2 seconds for administrative reads, 8 seconds for `lease.renew` (§9.3), 30 seconds for lease requests that wait for the auth owner.

### 13.2 Methods

| Method | Accepted from | Purpose |
|---|---|---|
| `hello` | any authenticated peer | version negotiation; returns the peer's principal |
| `account.enrol`, `account.describe`, `account.revoke`, `account.mark` | the account's owner | §5, §9.2 |
| `account.list` | owners (their own), operators | §5 |
| `grant.put`, `grant.list`, `revocation.put` | operators, and key holders for records they signed | §6 |
| `bind`, `unbind` | dispatchers (their own bindings), operators | §7 |
| `binding.list` | dispatchers (their own), operators | §7 |
| `lease.request` | the agent principal of an active binding | §8 |
| `lease.renew`, `lease.release`, `event.report` | the lease's own principal and binding | §8.5 |
| `status` | owners (their accounts), operators | accounts, owners, leases, events |

Every refusal is a typed error `{ code, message, details }`; codes are listed in Appendix A. Errors never contain material.

## 14. Command line

The same operations are available as `curator-broker <verb>` and, through Curator's provider mechanism, as `curator broker <verb>`:

```
curator broker enrol claude_code --account ivan/claude/personal --token-stdin [--issued-at 2026-10-09]
curator broker enrol codex_cli   --account ivan/codex/personal
curator broker accounts
curator broker grant --to-key <dispatcher key> --account ivan/claude/personal --profile dev --harness claude_code --depth 1 --until 2026-10-10T00:00Z
curator broker revoke <grant id | key>
curator broker bindings
curator broker status
```

Agent leaves (`agent:<generation>`) are issued by the dispatcher when it binds an agent (CIP-0011).

## 15. Installation and lifecycle

### 15.1 Service user

The broker runs as a dedicated unprivileged service user created by curator-host-helper (`user.create { kind: "service" }` from the installer principal). The platform installer installs the binary, the service user and the service unit together.

### 15.2 Paths

| What | macOS | Linux |
|---|---|---|
| data (store, grants, revocations, leases, config, audit) | the service user's home, 0700 | the service user's home, 0700 |
| socket | a dedicated directory outside the data tree, owned by the broker user, mode 0755, whose every ancestor is root-owned and not writable by others; socket mode 0666 (authorisation is by peer principal, §4) | same |
| service | a LaunchDaemon with `UserName` set to the broker user | a systemd system unit with `User=` |

### 15.3 Removal

Uninstall stops the service, ends leases and bindings, and leaves the store unless the operator asks to destroy it.

## 16. Delivery order

1. **Formats.** `grant/1`, `revocation/1` and the `broker/1` frames are frozen with canonical and negative vectors; the verifier ships with its mutant suite (§6.2).
2. **Slice 0: one protected Claude launch.** curator-host-helper v0 and its launcher; the broker daemon with the file store, local grants and revocation state, bind and unbind, lease request and release, the `env` channel, the protected binding and conflict refusal, the audit log; the broker client in Curator's first executor (CIP-0011). Acceptance on hosted runners only: a dispatcher stand-in creates an agent account, binds it and starts the executor under it through the launcher; the executor receives a lease and runs Claude; typed refusals for another UID, a retired generation, another account, another profile, an expired grant, a revoked grant, a revocation state that cannot be read, a conflicting source and an unqualified harness; the token appears in no file, argv, log or child process the qualification covers.
3. **Slice 1: Codex.** The enrolment ceremony, the auth owner, external tokens with renewal and deadlines, and qualification on the supported Codex release (§9.6).
4. **Slice 2: enforced networking.** Helper v1 firewall rules and trusted applied state; accounts that require enforced profiles become serviceable (§5.5).

## 17. Later

- a key keeper as the trust root and agent identities as keys (§4.3);
- grants and revocations from a shared registry (§6.5);
- roles, wildcards and further grant conditions, including bounds on concurrent use per account;
- events delivered to the session host and a person's phone (approve a lease, re-enrol);
- a Muse auth owner, if Muse rotates;
- Windows.

## 18. Open questions

1. Registry for grants and revocations beyond the local state (§6.5).
2. Whether Claude tokens should also be issued per launch by an owner once vendors offer short-lived subscription tokens.
3. Ownership of one Codex vendor account across several machines (§9.5).

## Appendix A. Refusal codes

`peer_identity_unavailable`, `server_identity_mismatch`, `principal_ledger_mismatch`, `principal_identity_changed`, `role_not_permitted`, `frame_too_large`, `version_unsupported`, `request_deadline_exceeded`, `account_exists`, `account_unknown`, `account_duplicate`, `account_store_keyring_refused`, `enrol_material_invalid`, `grant_malformed`, `grant_id_mismatch`, `grant_signature_invalid`, `grant_chain_malformed`, `grant_chain_broken`, `grant_not_narrowing`, `grant_root_unknown`, `grant_subject_invalid`, `grant_unsupported`, `grant_revoked`, `grant_expired`, `grant_not_yet_valid`, `revocation_not_authorised`, `revocation_state_unavailable`, `bind_not_dispatcher`, `bind_target_invalid`, `bind_request_conflict`, `lease_unbound`, `lease_not_authorised`, `lease_account_unknown`, `lease_harness_mismatch`, `lease_account_revoked`, `lease_reauth_required`, `lease_owner_unavailable`, `lease_channel_unsupported`, `lease_harness_unqualified`, `lease_binding_mismatch`, `lease_network_mismatch`, `lease_network_enforcement_unavailable`, `lease_not_owner`, `lease_not_active`, `renew_not_current`, `renew_timeout`, `renew_failed`, `external_token_init_failed`, `credential_source_conflict`.

## Appendix B. Diagrams

PlantUML sources in [`diagrams/`](../diagrams/): `enrol.puml`, `agent-registration.puml`, `lease-at-launch.puml`, `codex-renewal.puml`, `revoke-and-unbind.puml`.
