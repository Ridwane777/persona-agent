# GDPR Compliance Note - Persona

## 1. Introduction
**Persona** is an AI news aggregation service. To operate, it processes personal data belonging to its users. This note describes the technical and organizational measures in place to comply with the General Data Protection Regulation (GDPR).

## 2. Data collected
The system collects and stores:
- **Username**: for identification.
- **Email**: required to send the newsletter.
- **Password**: stored only as a hash (SHA-256).
- **Interests (topics)**: for personalization.
- **Schedule and timezone**: to plan deliveries.
- **Feedback**: optional comments on received newsletters (via the chatbot or the like / dislike buttons in the email). Feedback is used only to improve that user's next newsletters.
- **Custom prompt (`custom_prompt`)**: optional style instructions set by the user to adjust the newsletter's tone.

## 3. Legal basis
Processing is based on the user's **explicit consent** (GDPR Article 6.1.a). Consent is collected during chatbot onboarding and stored in the database (`gdpr_consent` = true) with a timestamp (`gdpr_consent_date`).

## 4. Retention
Data is kept while the user is active.
- On unsubscribe, preferences and sessions are **deleted immediately**.
- The email address is **anonymized** (e.g. `deleted_123@persona.local`) to keep delivery logs consistent while removing the link to an identifiable person.

## 5. User rights
Users can exercise their rights directly via the chatbot or the email links:
- **Right to erasure**: via the unsubscribe link in every newsletter, or by asking the chatbot to unsubscribe.
- **Right of access and rectification**: users can view and edit their preferences at any time via the chatbot.
- **Right to portability**: on request, stored data can be exported with the Supabase admin tools.

## 6. Data security
- **Hashing**: passwords are never stored in plain text.
- **Isolation**: every database query is filtered by the user ID of the active session, so a user cannot access another user's data.
- **Secure tokens**: unsubscribe links use a unique token generated with a secret salt (`PERSONA_SALT`), preventing malicious unsubscribes by third parties.

## 7. Data transfers
Data is stored on **Supabase** (AWS/GCP infrastructure depending on configuration). News summaries are generated via **OpenRouter** (model: OpenAI GPT-4o-mini). No personal data other than the username is sent to the LLM when summarizing news (only articles and topics are sent).

---
*Last updated: June 4, 2026*
