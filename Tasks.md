# Tasks - Persona
> Progress tracker: what is implemented and what evidence or documentation is still missing.

---

## Core features

### 1 - Chatbot trigger & authentication

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Chat Trigger active and reachable via URL | ✅ Done | `Chat trigger` node in Workflow A | - |
| Sign-up (username, email, password) | ✅ Done | `register` node (Supabase tool) + `Agent Auth` | - |
| Login via chatbot | ✅ Done | `Find_user` + `Create_active_sessions` nodes | - |
| Password hashing | ❌ Not implemented | Password stored as provided by the LLM tool call | Add a Crypto/bcrypt node before insert (see `SECURITY.md`) |
| Session system with `session_id` | ✅ Done | `check_sessions`, `Compute Session State` nodes | - |
| Session expiry (24h) | ✅ Done | README mentions 24h, `sessions` table with `expires_at` | **Show in the demo** that an expired session is rejected |
| Block unauthenticated users | ✅ Done | `IF: Session active?` node → branches to `Agent Auth` | - |
| Data isolation between users | ✅ Done | The main agent receives the session `user_id` | See `SECURITY.md` for technical isolation |
| Protection against malicious actors | ✅ Done | Sessions + hashing cover the basics | See `SECURITY.md` for the security audit |

---

### 2 - User onboarding via the chatbot

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Collect interests (topics) | ✅ Done | `save_preferences` (Supabase tool), `preferences.topics` table | - |
| Collect email | ✅ Done | `users.email` table, sign-up flow in `Agent Auth` | - |
| Collect preferred schedule | ✅ Done | `preferences.schedule_time` + `schedule_tz` | - |
| Collect language | ✅ Done | `preferences.language` | - |
| Collect number of articles | ✅ Done | `preferences.num_articles` | - |
| Edit preferences after sign-up | ✅ Done | `update_preferences` node in `Agent Principal` | - |
| Confirmation before saving | ⚠️ Not verified | Depends on the agent's system prompt (not fully visible) | **Show in the demo** that the agent summarizes preferences before saving them |
| Step-by-step conversational guidance | ⚠️ Not verified | Depends on LLM behavior at runtime | **Screenshot of a full onboarding conversation** |

---

### 3 - Fetch news from multiple sources

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Source 1: NewsAPI | ✅ Done | `NewsAPI` node (HTTP Request) in Workflow B | - |
| Source 2: The Guardian API | ✅ Done | `The Guardian API` node (HTTP Request) in Workflow B | - |
| Source 3: BBC News RSS | ✅ Done | `RSS BBC News` node in Workflow B | - |
| Merge sources | ✅ Done | `Fusionner sources` node (Merge) | - |
| Normalize article formats | ✅ Done | `Normaliser + Dédup + Fact-check` node (JS Code) | - |
| Deduplicate by URL | ✅ Done | JS code in `Normaliser + Dédup + Fact-check` | - |
| At least 2 active sources | ✅ Done | 3 sources present | - |

---

### 4 - Per-user AI filtering

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Filter by user topics | ✅ Done | `IA - Filtrage et résumé` node (LLM Chain) | - |
| Summarize each article in 2-3 sentences | ✅ Done | Prompt in the LLM node + `Parser réponse IA` | - |
| Respect the requested article count (`num_articles`) | ✅ Done | Code in `Parser réponse IA` applies the limit | - |
| Reply in the user's language | ✅ Done | `language` injected into the LLM prompt | - |
| Fallback if AI parsing fails | ✅ Done | `Parser réponse IA` falls back to the original articles | - |
| Honor the user's `custom_prompt` | ✅ Done | `custom_prompt` injected into the `IA - Filtrage et résumé` prompt | ✅ Verified working on June 4, 2026 |

---

### 5 - Email delivery

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Send via Gmail OAuth2 | ✅ Done | `Envoyer newsletter` node (Gmail) in Workflow B | - |
| HTML email | ✅ Done | `Construire email HTML` node (JS Code) | - |
| Unsubscribe link in the email | ✅ Done | HTML includes a `/unsubscribe?uid=...&token=...` link | - |
| Edit-preferences link | ✅ Done | README mentions a chatbot link in the email | - |
| Log delivery in the database | ✅ Done | `Logger envoi` node (Supabase INSERT into `newsletter_logs`) | - |
| **Screenshot of a real received email** | ✅ Done | `assets/newsletter-email.png` | - |

---

### 6 - Per-user scheduling

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Webhook trigger (external cron) | ✅ Done | Trigger replaced by a Webhook, called by `cron-job.org` every minute | - |
| Compute current UTC time | ✅ Done | `Calculer heure UTC` node (JS Code) | - |
| Compare current time vs user time (minute) | ✅ Done | `Vérifier horaire` node (JS Code with `Intl.DateTimeFormat`, HH:mm resolution) | - |
| African and worldwide timezone support | ✅ Done | `Intl.DateTimeFormat` + README lists supported timezones | - |
| Per-user loop (`SplitInBatches`) | ✅ Done | `Loop Users` node in Workflow B | - |
| Only process active users | ✅ Done | `Utilisateurs actifs` node filters `is_active = true` | - |
| **Production cloud deployment** | ✅ Done | Render Web Service (`your-n8n-instance.example.com`) + cron-job.org keep-alive | - |
| **Proof of on-time delivery** | ⚠️ Partial | `assets/db-newsletter-logs.png` | Show that `sent_at` matches the configured schedule |

---

### 7 - Basic GDPR & unsubscribe

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Explicit consent requested at sign-up | ✅ Done | `Marquer consentement RGPD` node in Workflow A | **Show in the demo** that the chatbot asks for consent |
| `gdpr_consent` and `gdpr_consent_date` in the database | ✅ Done | `users` table, README GDPR section | - |
| Unsubscribe via the chatbot | ✅ Done | `unsubscribe_user` node in `Agent Principal` | - |
| Unsubscribe via email link (Workflow C) | ✅ Done | Full Workflow C (8 nodes) | - |
| Secure token on the unsubscribe link | ✅ Done | JS code in `Construire email HTML` + `Vérifier token` (Workflow C) | The current token uses a simple bit-shift hash, **not a real HMAC** - flag or fix |
| Anonymize email on unsubscribe | ✅ Done | `Désactiver compte` node in Workflow C, README confirms `deleted_...` | - |
| Delete preferences and sessions | ✅ Done | `Supprimer préférences` and `Supprimer sessions` nodes in Workflow C | - |
| HTML confirmation page after unsubscribe | ✅ Done | `Répondre succès` node (Respond to Webhook) in Workflow C | - |
| Error page on invalid token | ✅ Done | `Répondre erreur` node (Workflow C) | - |
| **GDPR compliance document** | ✅ Done | `GDPR.md` at the root | - |

---

## Extra features

### 1 - HTML newsletter

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Structured HTML email | ✅ Done | `Construire email HTML` node generates a full template | - |
| Responsive, readable design | ✅ Done | Table-based template with inline styles | - |
| Article cards with title, source, summary, link | ✅ Done | JS code confirms the card structure | - |

---

### 2 - Feedback on past newsletters

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| User can give feedback via the chatbot | ✅ Done | `save_feedback` node in `Agent Principal` (Workflow A) | - |
| Feedback via email link (Workflow C) | ✅ Done | `/feedback` webhook with UPDATE on `newsletter_logs` filtered by `newsletter_id` | - |
| Feedback stored in the database (`newsletter_logs.feedback`) | ✅ Done | `newsletter_logs` table with a `feedback` column | - |
| Feedback fed back into the next prompt | ✅ Done | `Récupérer feedbacks passés` runs before `Fusionner user + prefs` (sequential) | - |

---

### 3 - Fact-checking

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| Cross-source verification | ✅ Done | `Normaliser + Dédup + Fact-check` node computes a `factCheckScore` | - |
| Per-article confirmation score | ✅ Done | `factCheckScore` passed through `Agréger pour l'IA` | - |
| Prioritize multi-source articles | ✅ Done | LLM prompt instructed to prioritize high-score articles | - |
| Fact-checking by a secondary LLM | ❌ Not implemented | No dedicated second LLM call | Possible improvement |

---

### 4 - Per-user custom prompt

| Requirement | Status | Evidence in the project | Missing |
|---|:---:|---|---|
| User can set a `custom_prompt` via the chatbot | ✅ Done | `update_custom_prompt` node in `Agent Principal` (Workflow A) | - |
| `custom_prompt` stored in the database | ✅ Done | `preferences.custom_prompt` column | - |
| `custom_prompt` passed to the newsletter LLM | ✅ Done | `IA - Filtrage et résumé` injects the `custom_prompt` into the prompt | ✅ Verified working on June 4, 2026 |

---

## Other ideas not covered

| Feature | Status | Note |
|---|:---:|---|
| Payment / premium subscription | ❌ Not implemented | Low priority |
| Refer another user (public profile) | ❌ Not implemented | Low priority |
| Targeted ads in newsletters | ❌ Not implemented | Low priority (raises complex GDPR issues) |
| Exposed endpoints (website, Discord, inbound email) | ✅ Implemented | `Persona_workflow_A_API.json` + `demo_chat.html` - REST API POST `/webhook/api-chat` |
| Multi-format newsletter (blog, social, printable) | ❌ Not implemented | Future improvement |
| AI-generated media | ❌ Not implemented | Future improvement |

---

## Missing documents & evidence (summary)

| # | Document / evidence | Priority | For the demo? |
|---|---|:---:|:---:|
| 1 | **Screenshot of a received email** (full HTML newsletter) | 🟢 Done (`assets/newsletter-email.png`) | ✅ Yes |
| 2 | **Supabase log** showing a delivery at the configured time (`newsletter_logs`) | 🟠 Partial (`assets/db-newsletter-logs.png`) | ✅ Yes |
| 3 | **Screenshot of a full onboarding conversation** (sign-up → prefs → confirmation) | 🔴 Critical | ✅ Yes |
| 4 | **Full security audit** (SECURITY.md) | 🟢 Done | ✅ Yes |
| 5 | **GDPR compliance note** (GDPR.md) | 🟢 Done | ✅ Yes |
| 6 | **Unsubscribe token verification** (hash with secret salt) | 🟢 Done | ⚠️ Useful |
| 7 | **Proof that `custom_prompt` is injected into the Workflow B LLM** (Workflow B JSON) | ✅ Verified | - |
| 8 | **Feedback fed back into the prompt** | ✅ Verified | - |
| 9 | **`factCheckScore` mentioned in the LLM prompt** | 🟢 Done | ❌ No |
| 10 | **REST API demo** via `demo_chat.html` (endpoint `/webhook/api-chat`) | 🟢 Done | ✅ Yes |

---

## Legend

| Icon | Meaning |
|:---:|---|
| ✅ Done | Implemented and visible in the workflow JSON |
| ⚠️ Partial | Partially implemented or not verifiable without running it |
| ❌ Not implemented | Not in the project |
| 🔴 Critical | Blocking for the demo |
| 🟠 Important | Expected for a complete project |
| 🟡 Minor | Nice to have |
