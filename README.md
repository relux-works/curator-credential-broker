# curator-credential-broker

A credential broker for agent systems. It runs under its own OS user, holds subscription and API credentials in its own store, and leases them to authorised agent launches over a Unix socket whose peer it identifies through the kernel (`getpeereid` / `SO_PEERCRED`).

**Status: design in progress.** Nothing is implemented yet. The specification will live in `spec/`.

## What it is for

- One person, one subscription: a subscription is enrolled once and handed to the launches that person authorised, instead of a separate login per managed home.
- Agents run as separate OS users; a launch asks the broker for a lease for its profile and harness, and receives the credential only in the launched process.
- Authority is a signed grant (`credential.lease` on an account, for a profile, until a time), verifiable down to a root key; delegation can only narrow.
- An account may require a network profile, so its traffic leaves only through a named egress.
- Codex personal plans: a single auth owner refreshes the account; launches receive external access tokens and never hold a refresh token.

## How it fits

- Curator composes launches and asks the broker for a lease at the final executor (curator-spec CIP-0010, CIP-0011).
- curator-host-helper creates the per-agent OS users the broker identifies.
- curator-network-profiles names the egress an account may require.
- curator-trust and waggle define the grant and signature forms the broker verifies.
