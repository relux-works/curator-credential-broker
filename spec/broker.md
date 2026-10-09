# curator-credential-broker: specification

- **Status:** draft v0.1 (2026-10-09). Nothing is implemented.
- **Normative language:** MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.
- **Companions:** [curator-host-helper](https://github.com/relux-works/curator-host-helper) (per-agent OS users and per-UID firewall rules), [curator-network-profiles](https://github.com/relux-works/curator-network-profiles) (egress profiles), curator-spec CIP-0010 and CIP-0011 (the Curator side: credential sources, executor capability, launch-plan member).

## 1. Purpose

Agent systems run many harness processes (Claude Code, Codex, Muse, and others) on behalf of a few people. Each harness needs the credential of a person's subscription or API account. Today every managed home keeps its own login, which costs a browser login per home and, for refresh-token logins, lets parallel copies invalidate each other.

The broker is a small service that:

1. holds each account's credential once, under its own OS user, in its own store;
2. hands a credential only to a launch that is authorised for it, identifying the launching process by its OS user through the kernel;
3. decides authority from signed grants that chain to a root key, so that orchestrators can delegate narrower rights to the agents they start;
4. serialises refresh for accounts whose credentials rotate (Codex personal plans), so that exactly one process refreshes and launches receive short-lived access tokens;
5. can require that an account's traffic leaves only through a named network profile.

### 1.1 Non-goals

- The broker does not run harnesses, compose launch plans or manage homes (Curator does).
- It does not create OS users (curator-host-helper does) and does not decide who may start an agent (the dispatcher does).
- It does not authenticate to model vendors on its own behalf, bypass a vendor's login flow, or intercept model traffic. Enrolment always uses the vendor's own flow, run by a person.
- It does not make a credential non-extractable. A credential delivered into a harness process is readable by that process (§12.1).
- It does not multiply one person's subscription across people. One account belongs to one person.

## 2. Terms

| Term | Meaning |
|---|---|
| account | One credential of one person for one harness family, for example a Claude subscription token or a Codex ChatGPT login. Identified by an account id such as `ivan/claude/personal`. |
| owner | The person an account belongs to; recorded at enrolment by the OS user that enrolled it. |
| identity | A signing key that names a participant (person, orchestrator, agent). Until a key keeper exists, persons use a local signing key and agents may be named by a label. |
| grant | A signed statement that an identity or role may perform an action on a target under conditions until a time (§6). |
| binding | The broker's record that an OS user (UID) currently acts for an identity, with the grants it may use (§7). |
| dispatcher | The component that starts agents: it asks curator-host-helper for an OS user and then binds that user in the broker. Until a dispatcher exists, a Curator command plays this role (CIP-0011). |
| lease | One issued use of an account's credential for one launch (§8). |
| auth owner | The broker-side process that alone refreshes a rotating account and mints access tokens for launches (§9). |
| launcher | The final executor of a launch: the process that execs the harness and asks the broker for a lease. |

## 3. Components

| Component | Runs as | Job |
|---|---|---|
| broker daemon | its own unprivileged service user (`cur-s-broker` by default) | listens on the broker socket; keeps accounts, grants, revocations and bindings; issues leases; runs auth owners; writes the audit log |
| account store | files owned by the broker user | account records and their secret material (§5.3) |
| grant store | files owned by the broker user | signed grants and revocation records (§6.5) |
| auth owners | inside the daemon, one per rotating account | refresh and mint (§9) |
| client library (`pkg/client`) | inside each launcher | requests leases, delivers material to the harness, reports renewal needs |
| CLI | the calling person or component | `curator broker …` in Curator, or the standalone `curator-broker` binary (§14) |

The broker opens no network listener. Its only interface is a Unix-domain socket.

## 4. Identities and trust

### 4.1 Peer identity

On every connection the broker MUST obtain the peer's effective UID from the kernel (`getpeereid` or `LOCAL_PEERCRED` on macOS and BSD, `SO_PEERCRED` on Linux) and MUST refuse the connection if the platform cannot provide it. A UID supplied in a request is never trusted.

### 4.2 Local roles

The broker's configuration (owned by the broker user, §15) names:

- **operators:** UIDs that may administer the broker (configure dispatchers, import trust roots, read the audit log);
- **dispatchers:** UIDs that may create and remove bindings (§7);
- every UID that enrols an account becomes that account's **owner UID**.

Launchers need no configuration: a launcher is any UID with a live binding.

### 4.3 Trust root

Grants are verified against one or more configured **root public keys** (Ed25519).

- **Before a key keeper exists (v0):** the operator generates a local signing key and configures its public key as the root. `curator broker grant …` signs grants with it.
- **With a key keeper:** the keeper's published root becomes the root. The operator's key is either re-issued as a keeper-derived key or certified by a root-signed record naming it a delegate. The grant format and the verification code do not change; only the configured root does (§6.6).

## 5. Accounts

### 5.1 Record

```
account/1
{ id:            "ivan/claude/personal",       // owner-chosen, unique in the broker
  harness:       "claude_code",                // claude_code | codex_cli | muse | gemini | qwen | …
  kind:          "claude-oauth-token",          // see 5.2
  owner_uid:     501,                           // UID that enrolled it
  owner_identity: optional public key,          // when the owner has a signing key
  enrolled_at:   "2026-10-09T10:00:00Z",
  issued_at:     "2026-10-09T09:58:00Z" | null, // only when the owner stated it
  expires_at:    "2027-10-09T09:58:00Z" | null, // computed from issued_at and the kind's lifetime, else null
  network:       { profile: "egress-a", digest: optional } | null,   // §5.5
  state:         "active" | "revoked" | "reauth_required",
  last_refusal:  optional { at, code } }
```

The record never contains secret material.

### 5.2 Kinds

| Kind | Material | Refresh | Delivery channels (§8.4) |
|---|---|---|---|
| `claude-oauth-token` | the one-year token printed by `claude setup-token` | none: a new token and a restart | environment `CLAUDE_CODE_OAUTH_TOKEN` |
| `codex-chatgpt` | a Codex ChatGPT login (`auth.json`) held by the auth owner | the auth owner only (§9) | app-server external token; per-turn minted token-only home |
| `codex-access-token` | a Business or Enterprise access token | none in the broker (admin-issued) | environment `CODEX_ACCESS_TOKEN` |
| `api-key` | a vendor API key (`ANTHROPIC_API_KEY`, `CODEX_API_KEY`, `META_API_KEY`, `GEMINI_API_KEY`, …) | none | environment, or stdin where the harness supports it |
| `muse-account` | reserved: Muse subscription login | to be researched (§10.2) | reserved |

### 5.3 Secret store

Material is kept behind a secret-store interface (`put`, `get`, `delete`, `describe`). **v0 ships one backend: a file with mode 0600 in a 0700 directory owned by the broker user**, on macOS and Linux alike. Keychain or Secret Service backends MAY be added later; a backend that can prompt a person MUST NOT be used for launches that cannot answer a prompt.

### 5.4 Enrolment

Enrolment is always performed by a person through the vendor's own flow:

- Claude: the person runs `claude setup-token` and pipes the printed token to `curator broker enrol claude_code --account <id> --token-stdin [--issued-at <date>]`.
- Codex personal plan: the person logs in with `codex login` in a home that the broker's auth owner will own, then `curator broker enrol codex_cli --account <id> --chatgpt` hands that home to the broker (§9.4). The broker refuses a keyring-backed Codex store, because it never exports a keyring item.
- Business access tokens and API keys: `--access-token-stdin` or `--api-key-stdin`.

The material travels over the socket once and is written to the secret store. Enrolling a deliberately supplied token is not permission to extract an existing native login, and the broker never reads another user's harness stores.

### 5.5 Network requirement

An account MAY require a network profile. A lease for such an account is issued only when the launch's network binding (a `relux-network-binding-record-v1` produced by curator-network-profiles for that launch) names the same profile, and, when the account pins a digest, the same profile digest. Otherwise the lease is refused with `lease_network_mismatch`.

Today network profiles are cooperative: they set proxy variables for the launched process. With curator-host-helper's per-UID firewall rules (helper v1) the agent's OS user can reach the network only through its proxy, which makes the requirement enforced rather than declared.

### 5.6 One person, one subscription

An account belongs to the person who enrolled it. The broker MUST NOT issue a lease for an account to a binding whose grant chain does not lead back to that owner or to the root on the owner's behalf. A later version bounds concurrent leases per account to one person's volume (§17).

## 6. Grants

### 6.1 Record

```
grant/1
{ id:          content hash of the canonical record without sig,
  issuer:      public key of the signer,
  subject:     public key | "role:<name>" | "label:<agent label>",
  actions:     ["credential.lease"],
  targets:     ["account:ivan/claude/personal"],      // account ids; "account:ivan/*" only in grants, never in requests
  conditions:  { lang: "caveats/v1",
                 profiles:  ["dev", "review"],         // Curator profiles the lease may serve
                 harnesses: ["claude_code"],
                 not_before?: time, hours?: "09-21 Europe/Yerevan", max_concurrent?: n },
  delegable:   { depth: 0 | n },
  parent:      grant id | null,
  issued_at, expires_at,
  sig:         Ed25519 over the canonical record without sig }
```

The canonical form is the same canonical JSON that Curator uses for its signed registry records (CCJ-1). The vocabulary (`credential.lease`, `account:` targets) is registered with the platform's action registry when one exists.

### 6.2 Chain verification

A request carries the full chain, leaf first (`grant_refs` plus the records). The broker accepts the chain when every link:

1. has a valid signature by its `issuer`;
2. after the first, is issued by the `subject` of its parent;
3. narrows its parent: actions, targets and every condition are subsets, `expires_at` is not later, and the remaining delegation depth allows it;
4. is unexpired and not revoked;

and the root link is issued by a configured root key (§4.3).

### 6.3 Leaf subject

The leaf subject must match the binding's identity (§7). In v0, when agents do not yet have keys, a binding MAY use a `label:` identity; grants to that label are then issued by a dispatcher or operator key, and the label exists only for the binding's lifetime.

### 6.4 Revocation

A revocation is a signed record `revocation/1 { revokes: grant id | public key, by, at, sig }` by the grant's issuer, any ancestor issuer, or a root. Revocations are kept in the grant store; a **revocation snapshot** `{ issued_at, records }` is what the broker consults, and a deployment with a remote registry MUST bound the snapshot's age.

### 6.5 Registry

Where grants and revocations live is pluggable. **v0: a local file of signed records in the broker's grant store**, written by `curator broker grant` and `revoke`. Candidates for later (a git journal, a registry service with signed snapshots, carrier room state, or a hybrid with local verified caches) are an open platform question; because chains travel with requests, the choice mainly affects revocation and discovery, not verification.

### 6.6 Relation to the trust design

curator-trust (capabilities, delegation verification) and waggle (signed envelopes and keys) define the platform's long-term forms. This specification uses the minimal subset needed for leases and MUST be reconciled with those forms before 1.0; where they differ, they win and this record gains a version.

## 7. Bindings

### 7.1 Record

```
binding/1
{ uid:           612,
  uid_generation: helper ledger id of the account creation,   // §7.4
  identity:      public key | "label:dev-7f3",
  profiles:      ["dev"],
  grant_refs:    [leaf … root],
  created_by:    dispatcher UID,
  created_at, expires_at }
```

### 7.2 Creating and removing

- `bind` MUST come from a dispatcher UID (§4.2) and MUST carry a chain that verifies (§6.2) with the binding's identity as leaf subject.
- In a platform with a key keeper, the dispatcher additionally proves that it may assign these grants (its own `spawn` grant); the broker verifies that chain as well.
- `unbind` comes from the dispatcher that created the binding or from an operator. Expired bindings are removed by the broker.
- A UID with no live binding receives `lease_unbound` for every lease request.

### 7.3 Ephemeral agents

A binding's `expires_at` is the end of the agent's task. The dispatcher removes it when the task ends and before the OS user is removed. The broker never extends a binding on its own.

### 7.4 UID reuse

UIDs are recycled when OS users are deleted. A binding MUST carry the helper ledger's creation identifier for that account (`uid_generation`); the broker re-checks it against the helper's read-only ledger (curator-host-helper `user.describe`) when it issues a lease, and refuses a mismatch with `lease_uid_generation_mismatch`.

## 8. Leases

### 8.1 Request

```
lease.request/1
{ harness:   "claude_code",
  profile:   "dev",                       // the Curator profile of the launch
  account:   "ivan/claude/personal",      // the account the launch configuration names
  network:   relux-network-binding-record-v1 | null,
  channel:   "env" | "stdin" | "external-token" | "token-home",
  launch_id: string,                      // the launch plan digest or run id, for audit
  executor:  { capabilities: ["credential-injection/1"], version } }
```

### 8.2 Checks, in order

1. Peer UID has a live binding (`lease_unbound`); the UID generation matches (§7.4).
2. The binding's chain verifies now (`lease_grant_invalid`, `lease_grant_revoked`, `lease_grant_expired`).
3. The leaf grant covers `credential.lease` on `account:<account>`, the `profile` and the `harness` (`lease_not_authorised`).
4. The account exists and is `active` (`lease_account_unknown`, `lease_account_revoked`, `lease_reauth_required`).
5. The account's network requirement is met (`lease_network_mismatch`).
6. The requested channel is valid for the kind (`lease_channel_unsupported`).
7. Conditions hold (time window, concurrency) (`lease_condition_failed`).

### 8.3 Response

```
lease/1
{ lease_id, account, harness, channel,
  material: string,                        // only in this response, never logged
  expires_at: time | null,                 // access-token expiry for Codex; account expiry for Claude, if known
  renew: { method: "lease.renew" } | null } // present for codex-chatgpt
```

### 8.4 Delivery channels

The client library delivers material only into the process it starts:

- `env`: the variable is set in the harness process environment at exec, never in the parent shell, argv, files or logs; the client removes conflicting variables (§12.2).
- `stdin`: for harnesses that read a key from stdin.
- `external-token`: for a Codex app-server in external-token mode (§9.3).
- `token-home`: a per-turn token-only Codex home (0600, removed afterwards) for `codex exec`.

### 8.5 Release

The launcher releases a lease when the harness exits (`lease.release`). Unreleased leases expire with the binding.

## 9. Codex auth owner

### 9.1 Why

A Codex ChatGPT login rotates its refresh token on every refresh. Several processes refreshing one `auth.json` race and can log the subscription out. One process per account must refresh.

### 9.2 Behaviour

- The auth owner for an account holds the account's Codex home under the broker user and an exclusive lock on its `auth.json`.
- On a lease request it returns the current access token and the ChatGPT account id. If the token expires within the refresh window (48 hours in the reference implementation), it first refreshes by running Codex itself once (a one-shot `codex app-server` reading the account); it never reimplements the OAuth exchange.
- `lease.renew { lease_id, rejected_sha256 }` refreshes only if the rejected token is still the current one; N launches rejected with the same token cause one refresh.
- The refresh token never leaves the auth owner.

### 9.3 External-token mode

A launcher that drives `codex app-server` logs in with `account/login/start { type: "chatgptAuthTokens", accessToken, chatgptAccountId }`. When the app-server receives a 401 it asks its client with `account/chatgptAuthTokens/refresh`; the client calls `lease.renew` and answers with the new token, and the turn continues without a restart. The launch home contains no `auth.json`.

### 9.4 Qualification

This mechanism is measured in relux-works/remote-worker-harness (`internal/authowner`: eight concurrent callers near expiry cause one refresh; five rejected tenants cause one renewal) and was live-accepted on Codex 0.155.1. Codex 0.158 added a fetch of an application network policy from the ChatGPT backend; the external-token path MUST be re-qualified on each Codex release the broker supports before leases of kind `codex-chatgpt` are issued for it (`lease_harness_unqualified` otherwise).

### 9.5 The person's own Codex

The account's native Codex home belongs to the auth owner after enrolment. A person who also runs Codex by hand for the same account MUST NOT do so concurrently with the owner's refresh; `status` says so. A later version may hand the person a token the same way.

## 10. Other harnesses

### 10.1 Claude Code

A `claude-oauth-token` has no refresh. When a harness reports an authentication failure, the launcher reports it (`credential_error`, §11); the account moves to `reauth_required` only on an owner's command or a repeated, classified failure. Recovery is re-enrolment and a restart; Claude Code resumes its transcript with `--resume`. Vendor-side revocation happens only in the vendor's web interface.

### 10.2 Muse

A Muse subscription login is required, not only API keys. Its store (Keychain first, `auth.json` fallback), refresh behaviour and concurrency are to be researched; if it rotates like Codex, it gets an auth owner of its own. Until then only `api-key` with `META_API_KEY` is supported for Muse.

## 11. Events

| Event | When | v0 surface |
|---|---|---|
| `lease_expiring` | an account or access token expires within 5 minutes (tokens with known expiry) or within 3 days (Claude tokens with a stated issue date) | `status`, audit log |
| `lease_expired` | the expiry passed | same |
| `credential_error { class: auth | permission | quota | unknown }` | a launcher reports a harness failure it could classify; `unknown` when a 401/403 cannot be told apart from a permission or quota cause | same |
| `reauth_required` | an owner marked the account, or failures were classified as authentication | same |

Later versions deliver events to the session host and the message bridge, so that a session is parked and a person is asked to re-enrol from their phone.

## 12. Security

### 12.1 What a compromised agent can do

It can read the credential leased to it and copy it within its sandbox. It cannot obtain other accounts, cannot bind itself, cannot forge its UID, cannot read the broker's store, and cannot refresh a Codex account. Removing per-home copies limits the blast radius to one credential with one revocation point; it does not make that credential non-extractable. Stronger non-extraction (injection at an egress proxy, or a credential the process cannot read) is a separate design and is out of scope.

### 12.2 Conflicting sources

The client library MUST refuse to exec when the inherited environment, a harness settings file the launch would read, or a configured helper already supplies a credential for the same harness (`credential_source_conflict`). Lease variables are not passed to MCP server children.

### 12.3 The broker itself

The broker holds every enrolled credential, so it stays minimal: one service user, no network listener, no plugins, no shell-outs except Codex for refresh, an append-only audit log (who, which account, which launch, which outcome; never material), and rate limits per UID.

### 12.4 Files

All broker files are owned by the broker user; directories are 0700 except the socket directory (§15.2). The broker refuses to start if its store or configuration is writable by any other user.

## 13. Socket protocol `broker/1`

### 13.1 Transport

A Unix-domain stream socket. Frames are a 4-byte big-endian length followed by a canonical JSON object; the broker rejects frames over 64 KiB before reading them. The first exchange is `hello { versions: [1] }`; an unknown version closes the connection.

### 13.2 Methods

| Method | Caller | Purpose |
|---|---|---|
| `hello` | any | version negotiation; returns the peer's role |
| `account.enrol`, `account.list`, `account.describe`, `account.revoke`, `account.mark` | owner (own accounts), operator (list) | §5 |
| `grant.put`, `grant.list`, `revocation.put` | issuer's key holder (signed records only), operator | §6 |
| `bind`, `unbind`, `binding.list` | dispatcher, operator | §7 |
| `lease.request`, `lease.renew`, `lease.release` | bound UID | §8, §9 |
| `event.report` | bound UID | a launcher reports `credential_error` |
| `status` | owner, operator | accounts, owners, leases, events |

Every refusal is a typed error `{ code, message, details }`; codes are listed in Appendix A.

## 14. Command line

The same operations are available as `curator-broker <verb>` and, through Curator's provider mechanism, as `curator broker <verb>`:

```
curator broker enrol claude_code --account ivan/claude/personal --token-stdin [--issued-at 2026-10-09]
curator broker enrol codex_cli   --account ivan/codex/personal  --chatgpt
curator broker accounts
curator broker grant --to label:dev-7f3 --account ivan/claude/personal --profile dev --harness claude_code --until 2026-10-10T00:00Z
curator broker revoke <grant-id>
curator broker bindings
curator broker status
```

## 15. Installation and lifecycle

### 15.1 Service user

The broker runs as a dedicated unprivileged service user. curator-host-helper creates it (`user.create { kind: "service", name: "broker" }`); in v0 an operator MAY create it once by hand. The platform installer installs the binary, the service user and the service unit together.

### 15.2 Paths

| What | macOS | Linux |
|---|---|---|
| data (store, grants, config, audit) | the service user's home, 0700 | the service user's home, 0700 |
| socket | a directory owned by the broker user, mode 0755, socket mode 0666 (authorisation is by peer UID, §4.1) | same |
| service | a LaunchDaemon with `UserName` set to the broker user | a systemd system unit with `User=` |

### 15.3 Removal

Uninstall stops the service, removes leases and bindings, and leaves the store unless the operator asks to destroy it.

## 16. Slice 0

Slice 0 is the first implementation and proves one chain end to end on one machine:

- broker daemon, file secret store, local grant file, peer-UID authentication;
- accounts of kinds `claude-oauth-token` and `codex-chatgpt` (with the auth owner and external-token renewal re-qualified on the supported Codex release);
- `bind`/`unbind` from a configured dispatcher UID, with `label:` identities and operator-signed grants;
- `lease.request`, `lease.renew`, `lease.release`, the network check, typed refusals, the audit log;
- the client library used by Curator's final executor (CIP-0011);
- acceptance: a person enrols a Claude token and a Codex login; a dispatcher stand-in creates an agent OS user with curator-host-helper and binds it; a launch under that user receives a lease and runs Claude and Codex; a launch from another UID, for another account, with an expired grant, with a revoked grant or with the wrong network profile is refused; eight concurrent Codex launches cause one refresh; the token appears in no file, argv, log or MCP child.

Tests that create OS users or run harnesses run on hosted CI runners only.

## 17. Later

- concurrency bounds per account (one person's volume) and lease durations;
- a key keeper as the trust root and agent identities as keys;
- grants from a shared registry with signed revocation snapshots;
- events delivered to the session host and a person's phone (approve a lease, re-enrol);
- a Muse auth owner, if Muse rotates;
- Windows.

## 18. Open questions

1. Registry for grants and revocations beyond the local file (§6.5).
2. Whether Claude tokens should also be minted per launch by an owner process once vendors offer short-lived subscription tokens.
3. Whether the broker or a separate process should hold Codex homes when several machines share one account (today: one machine per account).

## Appendix A. Refusal codes

`peer_identity_unavailable`, `role_not_permitted`, `frame_too_large`, `version_unsupported`, `account_exists`, `account_unknown`, `account_store_keyring_refused`, `enrol_material_invalid`, `grant_signature_invalid`, `grant_chain_broken`, `grant_not_narrowing`, `grant_root_unknown`, `bind_not_dispatcher`, `bind_chain_invalid`, `lease_unbound`, `lease_uid_generation_mismatch`, `lease_grant_invalid`, `lease_grant_revoked`, `lease_grant_expired`, `lease_not_authorised`, `lease_account_unknown`, `lease_account_revoked`, `lease_reauth_required`, `lease_network_mismatch`, `lease_channel_unsupported`, `lease_condition_failed`, `lease_harness_unqualified`, `renew_not_current`, `renew_failed`, `credential_source_conflict`.

## Appendix B. Diagrams

PlantUML sources in [`diagrams/`](../diagrams/): `enrol.puml`, `agent-registration.puml`, `lease-at-launch.puml`, `codex-renewal.puml`, `revoke-and-unbind.puml`.
