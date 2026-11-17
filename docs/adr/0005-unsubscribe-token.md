# ADR-0005: Bit-shift hash token for unsubscribe and feedback links

- **Status**: Accepted
- **Date**: 2026-05-30 (first version of Workflow C); recorded 2026-11-17
- **Deciders**: Ridwane777

## Context

Every email has unsubscribe and feedback links that act without a login. The link must prove it was issued by
Persona for this user. Node.js `crypto` is not available in n8n Code nodes by default.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| Plain `uid` in the link | Trivial | Anyone can unsubscribe anyone |
| **32-bit bit-shift hash of `uid + static salt` in a Code node** | Works in a default Code node; same function in B and C | Not cryptographic; small output space; salt in the workflow; no expiry |
| HMAC-SHA256 with n8n's Crypto node, secret in a credential | Standard and secure | One more node in B and C; secret management |
| Random token stored per newsletter in the database | Revocable, single-use possible | Extra column and lookup |

## Decision

Use the bit-shift hash with the static salt `CHANGE_ME_SECRET_SALT`, computed in `Build HTML email` and checked
in `Verify token` and `Verify feedback token`.

## Consequences

- ✅ Links cannot be built by just changing the `uid` without knowing the salt.
- ❌ Demo-grade only: the hash is easy to brute force, the salt ships in the export, tokens never expire.
- ❌ The salt must be changed by hand in three nodes and kept identical.
- Planned replacement: HMAC-SHA256 with a secret outside the workflow (to be recorded as a new ADR).
