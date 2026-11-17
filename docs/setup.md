# Setup

This is the single source for installing Persona. Other documents link here instead of repeating the steps.

## 1. Prerequisites

- [n8n](https://n8n.io/) (`npx n8n` or Docker), local or hosted
- A [Supabase](https://supabase.com/) project
- An [OpenRouter](https://openrouter.ai/) account and API key
- A [NewsAPI](https://newsapi.org/) key
- A [Guardian Open Platform](https://open-platform.theguardian.com/) key (the export uses the public `test` key)
- A Gmail account connected to n8n through OAuth2

Quotas, licences and what each service receives are listed in [dependencies.md](dependencies.md).

## 2. Create the Supabase tables

Run this in the Supabase **SQL Editor**:

```sql
-- Users
CREATE TABLE users (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username          TEXT UNIQUE NOT NULL,
  password_hash     TEXT NOT NULL,          -- unsalted SHA-256, see security.md
  email             TEXT NOT NULL,
  created_at        TIMESTAMP DEFAULT now(),
  is_active         BOOLEAN DEFAULT true,   -- false after unsubscribe
  gdpr_consent      BOOLEAN DEFAULT false,
  gdpr_consent_date TIMESTAMP
);

-- Newsletter preferences
CREATE TABLE preferences (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id) ON DELETE CASCADE,
  topics        TEXT[],                     -- e.g. {tech,climate}
  schedule_time TIME NOT NULL DEFAULT '08:00:00',
  schedule_tz   TEXT DEFAULT 'UTC',
  custom_prompt TEXT,
  language      TEXT DEFAULT 'fr',
  num_articles  INT DEFAULT 5
);

-- Delivery logs and feedback
CREATE TABLE newsletter_logs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
  newsletter_id   VARCHAR(255),
  sent_at         TIMESTAMP DEFAULT now(),
  articles        JSONB,
  feedback        TEXT,
  feedback_source VARCHAR(50) CHECK (feedback_source IN ('chat', 'email_link')),
  rating          VARCHAR(50) CHECK (rating IN ('good', 'bad')),
  created_at      TIMESTAMP DEFAULT now(),
  updated_at      TIMESTAMP DEFAULT now()
);

-- Chatbot sessions
CREATE TABLE sessions (
  session_id TEXT PRIMARY KEY,
  user_id    UUID REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT now(),
  expires_at TIMESTAMP
);
```

## 3. Create the n8n credentials

In **n8n > Credentials > New**:

| Credential name in the exports | Type | Where to get it |
| --- | --- | --- |
| `Supabase account` | Supabase API | Supabase > Project Settings > API: project URL and `service_role` key |
| `OpenRouter account` | OpenRouter | [openrouter.ai/keys](https://openrouter.ai/keys) |
| `Gmail account` | Gmail OAuth2 | Google Cloud Console > OAuth 2.0 client |

The exported JSON files contain placeholder credential IDs (`SUPABASE_CREDENTIAL_ID`, `OPENROUTER_CREDENTIAL_ID`,
`GMAIL_CREDENTIAL_ID`). After import, open each Supabase, OpenRouter and Gmail node and select your credential.

## 4. Import the workflows

In n8n, create a workflow, then **... > Import from file**, once per file in [`../workflows/`](../workflows/):

1. `c-unsubscribe-feedback.json` (unsubscribe and feedback webhooks)
2. `a-chatbot.json` (n8n hosted chat) and/or `a-chatbot-api.json` (REST endpoint used by the demo page)
3. `b-newsletter.json` (newsletter generation)

## 5. Replace the placeholders

| Where | Value to replace | With |
| --- | --- | --- |
| B, node `NewsAPI`, query `apiKey` | `YOUR_NEWSAPI_KEY` | Your NewsAPI key |
| B, node `The Guardian API`, query `api-key` | `test` | Your Guardian key (the `test` key is heavily rate limited) |
| B, node `Build HTML email` | `CHANGE_ME_SECRET_SALT` | A long random secret |
| C, nodes `Verify token` and `Verify feedback token` | `CHANGE_ME_SECRET_SALT` | The **same** secret as in B |
| B, node `Build HTML email` | `http://localhost:5678/webhook/...` (unsubscribe, feedback, chat links) | Your public n8n URL |
| `demo/chat.html`, constant `N8N_WEBHOOK_URL` | `https://your-n8n-instance.example.com/webhook/api-chat` | Your Workflow A_API URL |

The token logic and its limits are described in [security.md](security.md).

## 6. Activate

1. **Workflow C**: activate (exposes `GET /webhook/unsubscribe` and `GET /webhook/feedback`).
2. **Workflow A**: in the Chat Trigger, enable public chat, then activate. **Workflow A_API**: activate
   (exposes `POST /webhook/api-chat`).
3. **Workflow B**: activate (exposes `POST /webhook/trigger-newsletter`).
4. Configure an external scheduler (the project used cron-job.org) to `POST` to `/webhook/trigger-newsletter`
   **every minute**. Workflow B compares the current `HH:MM` with each user's `schedule_time`. See
   [ADR-0004](adr/0004-external-cron-trigger.md).

### Hosting variables used in production

The project ran n8n on Render (Web Service, Frankfurt) with:

| Variable | Value |
| --- | --- |
| `N8N_PORT` | `10000` |
| `WEBHOOK_URL` | `https://your-n8n-instance.example.com` |
| `GENERIC_TIMEZONE` | `Africa/Lagos` |
| `NODE_OPTIONS` | `--max-old-space-size=350` |
| `EXECUTIONS_DATA_SAVE_ON_SUCCESS` | `none` |

## 7. End-to-end test

1. Open the Workflow A chat URL (or `demo/chat.html` for A_API) and say "Hello".
2. Sign up with a username, email and password, then log in.
3. Set preferences: topics (e.g. `technology, sport`), time (e.g. `20:00`), language (`fr` or `en`), number of
   articles (e.g. `3`), and accept the GDPR consent.
4. To send without waiting: set your `schedule_time` in Supabase to the current minute, or set
   `FORCE_SEND = true` in node `Check schedule`, then execute Workflow B.
5. In the email, click **I liked it**: `newsletter_logs.rating` is updated.
6. Click **Unsubscribe**: `users.email` becomes `deleted_<id>@persona.local`, `is_active = false`, and the
   user's `preferences` and `sessions` rows are deleted.

Calling the REST chatbot directly:

```bash
curl -X POST https://your-n8n-instance.example.com/webhook/api-chat \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "my-session-id", "message": "Hello!"}'
# {"reply": "..."}
```
