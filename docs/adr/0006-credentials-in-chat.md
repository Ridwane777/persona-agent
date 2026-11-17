# ADR-0006: Collect credentials in the chat through agent tools

- **Status**: Accepted (known weakness, replacement recommended)
- **Date**: 2026-05-30 (commit "add authentication workflow"); recorded 2026-11-17
- **Deciders**: Ridwane777

## Context

The product is "no GUI": everything happens in the chat. Users need an account so that preferences and the
newsletter are tied to them.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Ask username, email, password in the chat; the auth agent stores `sha256(password)` with `$fromAI`** | Fully conversational; no extra page | Plaintext password goes to the LLM provider and stays in chat memory; unsalted fast hash; model chooses the tool arguments |
| Supabase Auth (magic link or password) on a small page, chat receives the session | Proper password hashing, rate limiting, email verification | Leaves the chat for one step |
| n8n Form Trigger for sign-up, bcrypt in a Code node | Stays in n8n; password never reaches the LLM | Needs bcrypt in the Code node environment |
| Email-only identification with a one-time code | No password at all | Needs an email round trip per login |

## Decision

Keep sign-up and login in the chat. `register` and `Find_user` write and compare
`$fromAI('password', ...).hash('sha256')`.

## Consequences

- ✅ The whole user journey is a single conversation, which was the product idea.
- ✅ Passwords are not stored in plain text.
- ❌ The plaintext password is sent to OpenRouter and the model vendor and kept in the window memory.
- ❌ Unsalted SHA-256 is vulnerable to dictionary attacks if the table leaks.
- ❌ No rate limiting or lockout on login attempts.
- Recommended: supersede with an ADR moving credentials to Supabase Auth or a form with bcrypt/argon2.
