# swarma-credential-broker

A credential broker for agent systems. It runs under its own OS user, holds subscription and API credentials in its own store, and leases them to authorised agent launches over a Unix socket whose peer it identifies through the kernel (`getpeereid` / `SO_PEERCRED`).

**Status: design in progress.** Nothing is implemented yet. The specification is [`spec/broker.md`](spec/broker.md); diagrams are in [`diagrams/`](diagrams/).

## What it is for

- One person, one subscription: a subscription is enrolled once and handed to the launches that person authorised, instead of a separate login per managed home.
- Agents run as separate OS users; a launch asks the broker for a lease for its profile and harness, and receives the credential only in the launched process.
- Authority is a signed grant (`credential.lease` on an account, for a profile, until a time), verifiable down to a root key; delegation can only narrow.
- An account may require a network profile, so its traffic leaves only through a named egress.
- Codex personal plans: a single auth owner refreshes the account; launches receive external access tokens and never hold a refresh token.

The broker's v0 trusted-process exception is explicit: the harness can read and copy its leased token. Revocation stops new leases and renewals; a copied token remains usable until expiry or vendor revocation. See [the exposure limits](spec/broker.md#121-what-a-compromised-agent-can-do) and [the required injection qualification](spec/broker.md#122-protected-binding-and-conflicting-sources).

## How it fits

- Curator composes the plan on the caller's side. The first broker consumer is [curator-run under the agent account](spec/broker.md#31-first-consumer-and-execution-boundary): its narrow one-shot mode verifies and executes that plan, obtains the lease and starts the harness (owner decision D-EXECUTOR, 2026-10-11). The board runner and session host ask swarma-dispatcher to launch through curator-run; they do not embed broker clients.
- swarma-user-manager creates the per-agent OS users the broker identifies.
- curator-network-profiles names the egress an account may require.
- curator-trust and waggle define the long-term grant and signature forms. Broker v0 verifies its closed local envelope; a versioned adapter reconciles the forms before 1.0.

The first protected provider launch is Claude with an explicitly enrolled `setup-token`; Codex personal-plan external tokens follow in slice 1. The [provider × source × channel × phase table](spec/broker.md#52-kinds) distinguishes those targets from retained per-home authentication and later or unresolved lanes. Source-policy v1 keeps `per-home` when no entry is configured; v2 requires an explicit entry and migrates existing logins to explicit `per-home`. The first node and broker stores use protected 0600 files on macOS and Linux; Keychain and Secret Service backends are later work.

The repository, standalone binary and service account are named `swarma-credential-broker`; `curator broker` is the Curator front end. The earlier `curator-broker` CLI spelling is historical design text, with no shipped alias or migration. Signed domain tags are `swarma-credential-broker grant/1` and `swarma-credential-broker revocation/1`; `broker/1` remains the peer-authenticated transport. The [grant dialect boundary](spec/broker.md#66-relation-to-the-trust-design) and [key classes](spec/broker.md#43-trust-root-and-key-classes) distinguish this v0 design from the future keeper model. Managed identity checks read the root-owned ledger at `/opt/swarma/lib/user-manager/ledger.json` ([§4.2](spec/broker.md#42-principals)). These are design contracts, not runtime qualification claims.

The existing [registration](diagrams/agent-registration.puml), [launch](diagrams/lease-at-launch.puml), [enrolment](diagrams/enrol.puml), [Codex renewal](diagrams/codex-renewal.puml) and [revocation](diagrams/revoke-and-unbind.puml) sequences describe the intended flows. New component and state diagrams are later work.

## License

Apache License 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
