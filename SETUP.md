# Persona - Setup Guide

## Before importing the workflows

### 1. Supabase tables to create (SQL Editor)

```sql
-- Users table
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username      TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  email         TEXT NOT NULL,
  created_at    TIMESTAMP DEFAULT now(),
  is_active     BOOLEAN DEFAULT true,
  gdpr_consent  BOOLEAN DEFAULT false,
  gdpr_consent_date TIMESTAMP
);

-- Preferences table
CREATE TABLE preferences (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id) ON DELETE CASCADE,
  topics        TEXT[],
  schedule_time TIME NOT NULL DEFAULT '08:00:00',
  schedule_tz   TEXT DEFAULT 'UTC',
  custom_prompt TEXT,
  language      TEXT DEFAULT 'fr',
  num_articles  INT DEFAULT 5
);

-- Newsletter logs table
CREATE TABLE newsletter_logs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id) ON DELETE CASCADE,
  newsletter_id VARCHAR(255),
  sent_at       TIMESTAMP DEFAULT now(),
  articles      JSONB,
  feedback      TEXT,
  feedback_source VARCHAR(50) CHECK (feedback_source IN ('chat', 'email_link')),
  rating        VARCHAR(50) CHECK (rating IN ('good', 'bad')),
  created_at    TIMESTAMP DEFAULT now(),
  updated_at    TIMESTAMP DEFAULT now()
);

-- Sessions table
CREATE TABLE sessions (
  session_id TEXT PRIMARY KEY,
  user_id    UUID REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT now(),
  expires_at TIMESTAMP
);
```

---

## Import into n8n

For each workflow (A, B, C):
1. n8n → **+** (new workflow) → **Import from file**
2. Select the matching JSON file

---

## Credentials to configure (search for "YOUR_" in each JSON)

Replace every placeholder in the relevant nodes:

| Placeholder | Where to find it |
|---|---|
| `YOUR_SUPABASE_CREDENTIAL_ID` | n8n > Credentials > Supabase (Service Role Key) |
| `YOUR_OPENROUTER_CREDENTIAL_ID` | openrouter.ai > Keys |
| `YOUR_GMAIL_CREDENTIAL_ID` | n8n > Credentials > Gmail OAuth2 |
| `YOUR_NEWSAPI_KEY` | newsapi.org (free) |
| `YOUR_GUARDIAN_API_KEY` | open-platform.theguardian.com (free) |
| `YOUR_N8N_BASE_URL` | Your local or hosted n8n URL |
| `YOUR_CHATBOT_URL` | The Workflow A Chat Trigger URL |
| `CHANGE_ME_SECRET_SALT` | Replace with a secret string of your choice (keep it identical in workflows B and C) |

---

## Activation order

1. **Workflow C** (Unsubscribe) → activate first to get the webhook URL
2. **Workflow A** (Chatbot) → activate, copy the Chat Trigger URL
3. **Workflow B** (Newsletter) → activate last

---

## Quick test

1. Open the Workflow A Chat Trigger URL
2. Type "Hello" → the agent should offer sign-up or login
3. Sign up with a username, email and password
4. Log in again in a new conversation
5. Set your preferences (topics, schedule)
6. To test delivery without waiting for the scheduled time, run Workflow B manually
