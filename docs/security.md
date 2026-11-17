# Security

Scope: the four workflows in [`../workflows/`](../workflows/) as exported. Each point below was checked against
the JSON. Weaknesses are stated as they are, with the fix.

## 1. Credentials and password storage

**What the code does.** The auth agent's `register` and `Find_user` Supabase tools compute
`$fromAI('password', ...).hash('sha256')` and write or compare the result in `users.password_hash`. Passwords are
therefore not stored in plain text.

**Known weaknesses.**

- **Unsalted, fast hash.** SHA-256 without a salt is designed to be fast. If the `users` table leaks, common
  passwords can be recovered with a dictionary or precomputed table, and two users with the same password have
  the same hash.
- **The plaintext password goes through the LLM.** The user types the password in the chat. It is sent to the
  LLM provider (OpenRouter, then the model vendor) as part of the conversation, and it stays in the window
  buffer memory for the rest of the session. `$fromAI` then hands it to the tool.
- **Login is a database filter, not a verification step.** `Find_user` filters on `email` and `password_hash`;
  there is no rate limit or lockout, so brute force is only limited by the auth agent's behaviour.

**Fix.** Collect credentials outside the chat: use Supabase Auth (or an n8n Form / a small login page) and pass
only an authenticated user ID or session to the chatbot. If passwords must be stored by the project, use a slow
salted hash (bcrypt or argon2id) in a server-side step, never in an LLM tool argument. See
[ADR-0006](adr/0006-credentials-in-chat.md).

## 2. Sessions

- A session row (`sessions.session_id`, `user_id`, `expires_at = now + 1 day`) is created by the
  `Create_active_sessions` tool after a successful `Find_user`.
- `check_sessions` keeps only rows with `expires_at > now`, so an expired session falls back to the auth agent.
- The `session_id` comes from the client (`sessionId` in the chat trigger or in the API request body). Anyone
  who knows a live `session_id` is logged in as its user. The demo page generates it in the browser.
- `Create_active_sessions` takes `session_id` from the trigger (not from the model), but `user_id` is still
  chosen by the LLM through `$fromAI`, copied from the `Find_user` result. A manipulated model could pass another
  user's ID. Setting it from the `Find_user` result in a deterministic node would remove that risk.

## 3. Unsubscribe and feedback tokens

**What the code does.** Node `Build HTML email` (Workflow B) computes
`token = abs(bitshiftHash(userId + 'CHANGE_ME_SECRET_SALT')).toString(16)`; nodes `Verify token` and
`Verify feedback token` (Workflow C) recompute it and compare. Only on a match does C anonymise the account or
store the rating.

**Limits (demo-grade).**

- It is a 32-bit non-cryptographic hash, not a MAC: collisions are easy to find and the output space is small.
- The salt is a static string in the workflow code. Anyone with the export, or with the salt left at its default
  value, can compute a valid token for any user ID.
- Tokens never expire and are not single-use.
- Comparison is a plain `===` (not constant time).

**Fix.** HMAC-SHA256 over `userId + newsletterId + expiry` with a secret kept in an n8n credential or environment
variable (n8n's Crypto node supports HMAC), compared in constant time. See
[ADR-0005](adr/0005-unsubscribe-token.md).

## 4. Data isolation

- Workflow A reads user data with filters on the `user_id` of the active session (`check_sessions`).
- Every main-agent tool (`save_preferences`, `update_preferences`, `update_custom_prompt`, `save_feedback`,
  `unsubscribe_user`, `delete_preferences`, `delete_sessions`, `Record GDPR consent`) takes `user_id` from the
  session with `$('Get user info')`, never from the model: a manipulated conversation cannot reach another
  user's rows.
- Workflow B processes one user per loop iteration (`Loop Users`), so one user's custom prompt and feedback are
  not mixed into another user's newsletter.
- n8n uses the Supabase **service role key**, which bypasses Row Level Security. Any workflow bug therefore has
  access to every row.

## 5. Injection

- **Prompt injection** (OWASP LLM01): `custom_prompt` and feedback are user text concatenated into the
  newsletter prompt. A user can only affect their own newsletter. `Build HTML email` escapes every LLM and
  article field (`esc()`) and only keeps `http(s)` links (`safeUrl()`) before inserting them in the HTML
  (OWASP LLM05, improper output handling).
- **Article content** from NewsAPI, The Guardian and the BBC still reaches the prompt unfiltered.
- **SQL injection**: Supabase nodes call PostgREST with structured filters; there is no string-built SQL.

## 6. Other

- No rate limiting on the public webhooks (`api-chat`, `trigger-newsletter`, `unsubscribe`, `feedback`).
- `trigger-newsletter` requires a secret header (n8n Header Auth credential, see [setup.md](setup.md)).
- The A_API response is built with `JSON.stringify`, so any reply text gives valid JSON.

## 7. Recommendations, by priority

1. Move login out of the chat and use bcrypt/argon2 or Supabase Auth (section 1).
2. Set `user_id` of `Create_active_sessions` from `Find_user` in a deterministic node (section 2).
3. Replace the token with HMAC-SHA256 and a secret outside the workflow (section 3).
4. Add rate limiting in front of n8n (section 6).

Related: [legal/gdpr.md](legal/gdpr.md), [dependencies.md](dependencies.md).
