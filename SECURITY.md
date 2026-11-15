# Security Audit Report - Persona

## 1. Executive summary
Persona was designed with a "security by design" approach. Although the project has no traditional GUI, the chatbot and webhook interactions are secured against common attack vectors.

## 2. Authentication and session management
- **Secret storage (known limitation)**: the password is extracted by the LLM agent (`$fromAI('password')`) and written as-is to the `password_hash` column; no hashing node runs before the insert, and the password transits through the LLM provider. **Fix**: hash it with n8n's Crypto node (or bcrypt in a Code node) before the Supabase `register`/`Find_user` tools, and collect credentials outside the LLM conversation.
- **Session lifecycle**:
  - Each session is tied to a unique `session_id`.
  - Expiry (`expires_at`) is checked on every interaction.
  - Automatic expiry: 24 hours.
- **Brute-force protection**: using LLM tools for auth adds flexibility but needs guardrails. The Auth agent is strictly limited by its system prompt to sign-up/login only.

## 3. Data isolation (multi-tenancy)
- **ID filtering**: all Supabase nodes use strict filters on the user's UUID (`user_id = {{ $json.user_id }}`).
- **Flow separation**: Workflow B (Newsletter) loops over users and fully isolates each newsletter's generation context in separate executions. A user cannot receive another user's news or custom prompt.

## 4. Webhook security (unsubscribe & feedback)
- **Tokenization**: the unsubscribe and feedback links `GET /unsubscribe?uid=...&token=...` and `GET /feedback?uid=...&token=...` are protected by a hashed token.
- **Algorithm**: the token is generated with a **JS bit-shift hash** (the `crypto` module is not available by default in n8n Code nodes) combined with a secret static salt. Acceptable for a demo; replace with HMAC-SHA256 in production.
- **Entropy**: the token combines the user ID with a secret static salt stored in the workflow code.
- **Validation**: Workflow C checks that the user exists AND that the token is valid before any destructive action.
- **Feedback deduplication**: feedback button clicks run an **UPDATE** on the existing `newsletter_logs` row (filtered by `newsletter_id`), so no duplicate rows are created.

## 5. Injection protection
- **Prompt injection**: user input (`custom_prompt`) is treated as raw data and placed in a clearly delimited text block for the LLM. Using templates and JSON for LLM exchanges reduces the risk of hijacking the agent.
- **SQL injection**: Supabase nodes use the PostgREST API, which parameterizes queries automatically, removing direct SQL injection risks.

## 6. Recommendations
1. **Key rotation**: rotate the `PERSONA_SALT` used for unsubscribe tokens regularly.
2. **Rate limiting**: add rate limiting on the n8n webhooks to prevent denial-of-service attacks.
3. **HMAC upgrade** (recommended): replace the current bit-shift hash with a real HMAC-SHA256 via n8n's native "Crypto" node, if available in the target deployment.

---
*Audit performed June 1, 2024 - updated June 4, 2026*
