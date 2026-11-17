# GDPR compliance note

Persona is a student portfolio project, not a commercial service. This note describes what the workflows
actually do with personal data and where they fall short of production GDPR compliance.

## 1. Data collected

| Data | Purpose | Where |
| --- | --- | --- |
| Username | Identification in the chat and the email greeting | `users.username` |
| Email | Sending the newsletter | `users.email` |
| Password | Login | `users.password_hash`: unsalted SHA-256 of the password (see section 6) |
| Topics, delivery time, time zone, language, number of articles | Personalisation | `preferences` |
| Custom prompt (`custom_prompt`) | Optional style instructions | `preferences.custom_prompt` |
| Feedback and rating | Improving the next newsletters of the same user | `newsletter_logs.feedback`, `rating` |
| Articles sent | Delivery log | `newsletter_logs.articles` |
| Chat session ID and expiry | Keeping the user logged in | `sessions` |
| Consent flag and date | Proof of consent | `users.gdpr_consent`, `gdpr_consent_date` |

## 2. Legal basis

Consent (GDPR Art. 6(1)(a)). The main agent is instructed to mention GDPR consent before saving preferences, and
the `Record GDPR consent` tool stores `gdpr_consent = true` with a timestamp. Consent capture depends on the
LLM following its prompt; a deterministic consent step (checkbox or explicit yes/no node) would be more robust.

## 3. Retention and erasure

- Data is kept while the account is active. There is no automatic deletion of inactive accounts or old logs.
- Unsubscribe through the email link (Workflow C): `is_active = false`, email replaced by
  `deleted_<user_id>@persona.local`, `preferences` and `sessions` rows deleted.
- Unsubscribe through the chatbot (`unsubscribe_user` tool): `is_active = false` and email anonymised; this path
  does **not** delete `preferences` or `sessions`.
- In both paths, `username`, `password_hash` and `newsletter_logs` (with feedback) are kept. This is
  pseudonymisation, not full erasure; a full Art. 17 erasure would delete or blank those fields too.

## 4. User rights

- **Erasure**: unsubscribe link in every newsletter, or ask the chatbot (limits in section 3).
- **Access and rectification**: preferences can be viewed and changed through the chatbot.
- **Portability**: no self-service export; an operator can export the rows from Supabase.

## 5. Processors and transfers

| Recipient | Data sent | Note |
| --- | --- | --- |
| Supabase | All stored data | Hosting region depends on the project settings |
| OpenRouter and the model vendor (`openai/gpt-4o-mini`) | Everything typed in the chat, **including email and plaintext password during sign-up and login**; in Workflow B: topics, language, custom prompt, past feedback and article metadata (not the username or email) | Provider outside the EU: a data processing agreement and transfer basis are needed in production |
| Google (Gmail) | Recipient email, username, newsletter content | |
| NewsAPI, The Guardian | Topics as search terms | No user identifiers |
| Render, cron-job.org | Hosting and the trigger call | No user data in the trigger |

## 6. Security measures and gaps

- Passwords are stored as unsalted SHA-256, which is fast and open to dictionary attacks if the table leaks, and
  the plaintext password transits through the LLM provider. Fix: collect credentials outside the chat (Supabase
  Auth or a form) and use bcrypt or argon2. Details: [../security.md](../security.md).
- Unsubscribe and feedback links carry a token from a 32-bit bit-shift hash with a static salt
  (`CHANGE_ME_SECRET_SALT` in the export). It stops casual tampering only; HMAC-SHA256 with a secret outside the
  workflow is the fix.
- Queries are filtered by the session's `user_id`, but several chatbot tools receive `user_id` from the LLM
  (see [../security.md](../security.md), section 4).

## 7. Missing for production

Privacy policy and terms published to users, a record of processing activities, DPAs with each processor,
a retention schedule, and a breach procedure.
