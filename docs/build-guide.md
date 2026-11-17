# Complete Guide - Persona on n8n
> Personalized AI newsletter aggregator | First-time n8n usage

---

> **Status of this guide.** It was written as the build plan before the workflows existed, so some snippets
> differ from the final exports (for example, Workflow B is started by a webhook that an external cron calls,
> not by a Schedule Trigger, and the auth agent hashes passwords inside the Supabase tools with
> `$fromAI('password', ...).hash('sha256')` rather than in a separate Code node). The JSON files in
> [`../workflows/`](../workflows/) are the source of truth. Installation steps live only in [setup.md](setup.md).

## Table of contents

1. [n8n in 5 minutes](#1-n8n-in-5-minutes)
2. [Overall architecture](#2-overall-architecture)
3. [Supabase - database schema](#3-supabase---database-schema)
4. [Core 1 - Chatbot trigger & authentication](#4-core-1---chatbot-trigger--authentication)
5. [Core 2 - User onboarding through the chatbot](#5-core-2---user-onboarding-through-the-chatbot)
6. [Core 3 - Fetching news from multiple sources](#6-core-3---fetching-news-from-multiple-sources)
7. [Core 4 - Per-user AI filtering](#7-core-4---per-user-ai-filtering)
8. [Core 5 - Sending the newsletter by email](#8-core-5---sending-the-newsletter-by-email)
9. [Core 6 - Per-user scheduling](#9-core-6---per-user-scheduling)
10. [Core 7 - Basic GDPR & unsubscribe](#10-core-7---basic-gdpr--unsubscribe)
11. [Extra 1 - HTML newsletter](#11-extra-1---html-newsletter)
12. [Extra 2 - Feedback on past newsletters](#12-extra-2---feedback-on-past-newsletters)
13. [Extra 3 - Fact-checking](#13-extra-3---fact-checking)
14. [Extra 4 - Per-user custom prompt](#14-extra-4---per-user-custom-prompt)
15. [Recommended build order](#15-recommended-build-order)
16. [Common n8n beginner mistakes](#16-common-n8n-beginner-mistakes)
17. [Pre-demo checklist](#17-pre-demo-checklist)

---

## 1. n8n in 5 minutes

### Key vocabulary

| Term | Definition |
|---|---|
| **Workflow** | A chain of nodes - your "program" |
| **Node** | One step in the workflow (e.g. send an email, call an API) |
| **Trigger** | The node that starts the workflow (chatbot, cron, webhook...) |
| **Expression** | A dynamic value, syntax `{{ $json.champ }}` |
| **Credentials** | API keys stored securely in n8n |
| **Execution** | One run of a workflow |
| **Item** | A JSON object passed between nodes |

### How data flows

Each node receives an array of **items** (JSON objects) and returns a new one. To access the previous node's output:

```
{{ $json.nom_du_champ }}           → field value
{{ $json["field with space"] }}   → field with a space/special character
{{ $('Node name').item.json.x }} → value from a specific node by name
```

### Most-used nodes

- **Chat Trigger** - chatbot entry point
- **AI Agent** - LLM with conversation memory
- **HTTP Request** - external API call (news, etc.)
- **Supabase** - read/write to the DB
- **Code (JavaScript)** - custom logic
- **IF / Switch** - conditional branching
- **Send Email / Gmail** - sending emails
- **Schedule Trigger** - runs at fixed times
- **Set** - create/modify item fields

---

## 2. Overall architecture

The project consists of **3 separate workflows**:

```
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW A - Chatbot (always active)                        │
│                                                              │
│  Chat Trigger → AI Agent → Supabase (user CRUD)             │
│                          → Authentication                    │
│                          → Onboarding / preference updates  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW B - Newsletter delivery (runs every hour           │
│              or every X minutes)                             │
│                                                              │
│  Schedule Trigger → Supabase (who is due now?)              │
│                   → HTTP Request (news sources)             │
│                   → AI Agent (filtering + summary)          │
│                   → Send Email                              │
│                   → Supabase (delivery log)                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW C - Unsubscribe (webhook or email link)            │
│                                                              │
│  Webhook Trigger → Supabase (delete/deactivate)             │
│                  → Confirmation email                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Supabase - database schema

Four tables: `users` (account, `password_hash`, `is_active`, GDPR consent), `preferences` (topics,
`schedule_time`, `schedule_tz`, `language`, `num_articles`, `custom_prompt`), `newsletter_logs` (one row per
newsletter sent, plus `feedback` and `rating`) and `sessions` (chat `session_id` to `user_id`, with `expires_at`).

The SQL to create them and the Supabase credential steps are in [setup.md](setup.md) (single source, so the
schema is not duplicated here). Use the **service role key** in n8n: it bypasses Row Level Security, so keep it
secret.

---

## 4. Core 1 - Chatbot trigger & authentication

### Goal

Let a user identify through the chatbot, and block any unauthenticated access to personal data.

### Nodes

```
[Chat Trigger] → [AI Agent] → [Code: extract intent] → [IF: logged in?]
                                                              ↓ No
                                                         [Ask login/register]
                                                              ↓ Yes
                                                         [Rest of workflow]
```

### Step by step

**1. Add the Chat Trigger**
- Create a new workflow
- Add the **"Chat Trigger"** node
- It generates a chatbot URL you can open in a browser
- Optionally enable **"Allow file uploads"**

**2. Store the session identity**

The Chat Trigger provides a unique `sessionId` per conversation. It is the key to look up a user's state.

```javascript
// In a Code node, read the sessionId:
const sessionId = $('Chat Trigger').item.json.sessionId;
```

**3. Authentication logic**

On every incoming message, before anything else:

```
[Chat Trigger]
      ↓
[Supabase: SELECT * FROM sessions WHERE session_id = '{{sessionId}}']
      ↓
[IF: session found and not expired?]
    ↙ Yes                    ↘ No
[User logged in]          [AI Agent: ask for login]
```

**4. Registration node**

When the user wants to create an account:

```javascript
// Code node - password hashing (simple)
const crypto = require('crypto');
const password = $json.password;
const hash = crypto.createHash('sha256').update(password).digest('hex');
return [{ json: { password_hash: hash } }];
```

Then a **Supabase** node in `INSERT` mode on the `users` table.

**5. Login node**

```javascript
// Code node - verification
const inputHash = crypto.createHash('sha256').update($json.password).digest('hex');
const storedHash = $('Supabase: get user').item.json.password_hash;
const isValid = inputHash === storedHash;
return [{ json: { isValid } }];
```

If valid → INSERT into `sessions` with `expires_at = now() + 24h`.

### Security tests

- [ ] Try to access another user's data by manually changing the email in the chat
- [ ] Check that an expired session_id no longer grants access
- [ ] Check that an unknown user gets a clear error message

---

## 5. Core 2 - User onboarding through the chatbot

### Goal

A new user provides their interests, email and preferred time in natural conversation.

### Nodes

```
[AI Agent with memory] → [Code: extract data] → [Supabase INSERT preferences]
```

### AI Agent configuration

**Recommended starter model:** OpenAI GPT-4o-mini (cheap and fast), or any comparable small model available on OpenRouter.

**Agent system prompt:**

```
You are the setup assistant for Persona, a personalized newsletter service.

Your job is to collect the following from the user conversationally:
1. Their interests (minimum 1, maximum 10 topics)
2. Their email address
3. The time they want to receive the newsletter (HH:MM format)
4. Their timezone (if unknown, suggest UTC+1 for France)

Rules:
- Do not ask for everything at once; guide the user step by step
- Confirm each item before moving to the next
- Once everything is collected, summarize and ask for confirmation
- When the user confirms, reply ONLY with this JSON (nothing else):
  {"status": "complete", "email": "...", "topics": [...], "time": "HH:MM", "tz": "..."}
- Let the user change any item
- Always remind the user they can unsubscribe at any time

The user is already authenticated. Session: {{$json.sessionId}}
```

**Enable memory:** in the AI Agent, set **Memory** → "Window Buffer Memory" with a 20-message window so the agent keeps conversation context.

### Extracting the collected data

After the AI Agent, add an **IF** node that checks whether the reply contains `"status": "complete"`:

```javascript
// Code node - parse the AI reply
const response = $json.output;
try {
  // The AI may wrap the JSON in text; extract it
  const jsonMatch = response.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    const data = JSON.parse(jsonMatch[0]);
    if (data.status === 'complete') {
      return [{ json: { isComplete: true, ...data } }];
    }
  }
} catch(e) {}
return [{ json: { isComplete: false, message: response } }];
```

### Saving to Supabase

When `isComplete: true`:
1. Supabase node `INSERT` into `preferences`
2. Supabase node `UPDATE` on `users` → set `gdpr_consent: true` and `gdpr_consent_date: now()`
3. Reply to the user: "Great! Your newsletter will be sent every day at [time]. See you soon!"

---

## 6. Core 3 - Fetching news from multiple sources

### Goal

Fetch news articles from at least 2-3 different sources.

### Recommended sources (free APIs)

| Source | API | Free limit | n8n node |
|---|---|---|---|
| **NewsAPI** | newsapi.org | 100 req/day | HTTP Request |
| **The Guardian** | open-platform.theguardian.com | 500 req/day | HTTP Request |
| **RSS Feeds** | Any site with an RSS feed | Unlimited | RSS Feed Read |
| **GNews** | gnews.io | 100 req/day | HTTP Request |

### Fetch workflow (in Workflow B)

```
[Schedule Trigger]
        ↓
[Supabase: list of all unique topics of all active users]
        ↓
[Split In Batches] ← process topic by topic
        ↓
[HTTP Request: NewsAPI] + [RSS Feed Read: Guardian]
        ↓
[Merge] ← combine results from both sources
        ↓
[Code: dedup + cleanup]
        ↓
[Continue to AI filtering]
```

### HTTP Request node for NewsAPI

- **Method:** GET
- **URL:** `https://newsapi.org/v2/everything`
- **Query Parameters:**
  - `q` : `{{ $json.topic }}`
  - `language` : `fr`
  - `sortBy` : `publishedAt`
  - `pageSize` : `10`
  - `apiKey` : (store your key in credentials, or inline to get started)

### RSS Feed node

n8n has a native **RSS Feed Read** node. Just set the feed URL:
- Le Monde : `https://www.lemonde.fr/rss/une.xml`
- BBC : `https://feeds.bbci.co.uk/news/rss.xml`
- Reuters : `https://feeds.reuters.com/reuters/topNews`

### Data cleanup (Code node)

```javascript
// Normalize articles from all sources
const articles = $input.all().map(item => {
  const a = item.json;
  return {
    json: {
      title: a.title || a.headline || 'Untitled',
      description: a.description || a.summary || a.content?.substring(0, 300) || '',
      url: a.url || a.webUrl || a.link,
      source: a.source?.name || a.sectionName || 'Unknown source',
      publishedAt: a.publishedAt || a.webPublicationDate || new Date().toISOString(),
      topic: $('Split In Batches').item.json.topic
    }
  };
});

// Dedup by URL
const seen = new Set();
return articles.filter(a => {
  if (seen.has(a.json.url)) return false;
  seen.add(a.json.url);
  return true;
});
```

---

## 7. Core 4 - Per-user AI filtering

### Goal

For each user, the AI picks the relevant articles among all fetched ones and writes a short summary.

### Architecture

```
[Fetched articles] + [User preferences from Supabase]
                ↓
         [AI Agent: filtering]
                ↓
         [Code: parse AI JSON]
                ↓
         [Filtered articles with summaries]
```

### Filtering prompt

In an **AI Agent** or **Basic LLM Chain** node:

```
You are a newsletter editor. Here are a subscriber's preferences:
- Interests: {{ $json.topics }}
- Preferred language: {{ $json.language }}
- Number of articles: {{ $json.num_articles }}

Here is the list of articles available today:
{{ JSON.stringify($('Merge articles').all().map(i => i.json)) }}

Select exactly {{ $json.num_articles }} articles that best match this subscriber's interests.
For each selected article, write a 2-3 sentence summary in {{ $json.language }}.

Reply ONLY with this JSON, no text before or after:
{
  "articles": [
    {
      "title": "...",
      "summary": "...",
      "url": "...",
      "source": "..."
    }
  ]
}
```

> ⚠️ Watch the context size! Many articles can exceed the model's context window. Limit input to 20-30 articles.

### Token optimization (Extra)

To save tokens, pre-filter by topic before calling the AI:

```javascript
// Code node - simple keyword pre-filter before the AI
const userTopics = $json.topics; // e.g. ['tech', 'AI', 'climate']
const articles = $('Merge').all().map(i => i.json);

const filtered = articles.filter(a => {
  const text = (a.title + ' ' + a.description).toLowerCase();
  return userTopics.some(topic => text.includes(topic.toLowerCase()));
});

// Keep at most 20 articles for the AI
return filtered.slice(0, 20).map(a => ({ json: a }));
```

---

## 8. Core 5 - Sending the newsletter by email

### Goal

Send a well-formatted email to the user with the selected articles.

### Email options in n8n

| Option | Pros | Cons |
|---|---|---|
| **Gmail** (native node) | Simple, free | 500 emails/day max |
| **Send Email (SMTP)** | Works with any provider | Requires SMTP config |
| **Brevo (formerly Sendinblue)** | 300 free emails/day, good deliverability | Account needed |
| **Resend** | Simple, 100 free emails/day | API Key |

### Gmail node configuration

1. In n8n → Credentials → New → "Gmail OAuth2"
2. Follow the Google OAuth guide
3. In the Gmail node:
   - **To:** `{{ $json.email }}`
   - **Subject:** `Your newsletter for {{ new Date().toLocaleDateString('fr-FR') }}`
   - **Email Type:** HTML (for Extra 1) or Text
   - **Message:** see below

### Simple plain-text email template (to start)

```javascript
// Code node - build the email body
const articles = $json.articles; // AI output
const userName = $json.username;
const date = new Date().toLocaleDateString('en-GB', { weekday: 'long', day: 'numeric', month: 'long' });

let body = `Hello ${userName},\n\n`;
body += `Here is your news selection for ${date}:\n\n`;
body += `${'─'.repeat(50)}\n\n`;

articles.forEach((article, i) => {
  body += `${i + 1}. ${article.title}\n`;
  body += `   Source: ${article.source}\n`;
  body += `   ${article.summary}\n`;
  body += `   → Read: ${article.url}\n\n`;
});

body += `${'─'.repeat(50)}\n`;
body += `To change your preferences or unsubscribe, reply to this message or talk to our chatbot.\n`;
body += `\nPersona - Your AI-personalized newsletter`;

return [{ json: { emailBody: body, emailSubject: `📰 Your newsletter for ${date}` } }];
```

---

## 9. Core 6 - Per-user scheduling

### Problem

n8n does not natively support a different schedule per user. Solution: a **Schedule Trigger running every hour** that checks which users are due at that moment.

### Check logic

```
[Schedule Trigger: every hour]
        ↓
[Code: compute current UTC hour]
        ↓
[Supabase: SELECT users + preferences WHERE schedule_time = current_hour AND is_active = true]
        ↓
[IF: results found?]
        ↓ Yes
[SplitInBatches: one user at a time]
        ↓
[Fetch news + Filter + Send]
```

### Code node - hour check

```javascript
// Compute the current hour, rounded to the hour
const now = new Date();
const currentHour = now.getUTCHours().toString().padStart(2, '0') + ':00:00';

return [{ json: { currentHour } }];
```

### Supabase query

In the Supabase node, use a custom query ("Execute Query" mode):

```sql
SELECT u.id, u.email, u.username, p.topics, p.language, p.num_articles, p.custom_prompt
FROM users u
JOIN preferences p ON p.user_id = u.id
WHERE p.schedule_time = '{{ $json.currentHour }}'
  AND u.is_active = true
  AND u.gdpr_consent = true;
```

> Timezones are tricky. To start, ask users for their time in UTC. For an advanced version, use the `luxon` library in a Code node to convert.

---

## 10. Core 7 - Basic GDPR & unsubscribe

### Minimum to implement

**Consent:** during chatbot sign-up, the agent must explicitly state:
- That the email will be stored and used to send newsletters
- That the user can unsubscribe at any time
- Consent must be recorded in the DB (`gdpr_consent = true`)

**Unsubscribe:** two methods:

#### Method 1 - Through the chatbot

In the Workflow A AI Agent, handle the "unsubscribe" intent:

```javascript
// If the AI detects an unsubscribe intent
// Supabase UPDATE users SET is_active = false WHERE id = userId
// Supabase UPDATE users SET email = 'DELETED_' + id WHERE id = userId (anonymization)
```

#### Method 2 - Link in every email (Workflow C)

Add at the bottom of every newsletter:

```
To unsubscribe: https://your-n8n.example.com/webhook/unsubscribe?uid={{ userId }}&token={{ token }}
```

For email feedback, also add to the footer:

```
I liked it: https://your-n8n.example.com/webhook/feedback?uid={{ userId }}&token={{ token }}&newsletter_id={{ newsletterId }}&rating=good
I didn't like it: https://your-n8n.example.com/webhook/feedback?uid={{ userId }}&token={{ token }}&newsletter_id={{ newsletterId }}&rating=bad
```

Then create **Workflow C** with a **Webhook Trigger**:

```
[Webhook: GET /unsubscribe]
        ↓
[Code: verify uid + token]
        ↓
[Supabase: UPDATE is_active = false]
        ↓
[Respond to Webhook: "You have been unsubscribed."]
```

> **Link security:** the `token` must be a unique per-user hash (`sha256(userId + secret_salt)`) so an attacker cannot unsubscribe someone else.

### Right to erasure (data deletion)

```javascript
// Option 1: hard delete
// DELETE FROM users WHERE id = userId

// Option 2: anonymization (safer, keeps logs)
// UPDATE users SET email = 'deleted_' + id, username = 'deleted_' + id, is_active = false
// DELETE FROM preferences WHERE user_id = userId
```

---

## 11. Extra 1 - HTML newsletter

### Why

HTML emails are far more readable and professional. n8n + Gmail fully support HTML.

### Simple HTML template

```javascript
// Code node - generate the HTML
const articles = $json.articles;
const userName = $json.username;
const date = new Date().toLocaleDateString('en-GB', { weekday: 'long', day: 'numeric', month: 'long' });

const articleCards = articles.map(a => `
  <div style="border-left: 3px solid #6366f1; padding-left: 16px; margin-bottom: 24px;">
    <h2 style="margin:0 0 4px; font-size:16px; color:#1e1e2e;">
      <a href="${a.url}" style="color:#6366f1; text-decoration:none;">${a.title}</a>
    </h2>
    <p style="margin:0 0 8px; font-size:12px; color:#888;">${a.source}</p>
    <p style="margin:0; font-size:14px; color:#333; line-height:1.6;">${a.summary}</p>
  </div>
`).join('');

const html = `
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"></head>
<body style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; background: #f9f9f9;">
  <div style="background:#6366f1; color:white; padding:24px; border-radius:8px 8px 0 0; text-align:center;">
    <h1 style="margin:0; font-size:24px;">📰 Persona</h1>
    <p style="margin:8px 0 0; opacity:0.8;">Your newsletter for ${date}</p>
  </div>

  <div style="background:white; padding:24px; border-radius:0 0 8px 8px;">
    <p style="color:#555;">Hello <strong>${userName}</strong>, here is today's selection:</p>
    ${articleCards}
    <hr style="border:none; border-top:1px solid #eee; margin:24px 0;">
    <p style="font-size:12px; color:#aaa; text-align:center;">
      To unsubscribe: <a href="UNSUBSCRIBE_LINK" style="color:#6366f1;">click here</a>
    </p>
  </div>
</body>
</html>
`;

return [{ json: { html } }];
```

In the Gmail node, select **"HTML"** as the message type and use `{{ $json.html }}`.

---

## 12. Extra 2 - Feedback on past newsletters

### Concept

The user can write in the chatbot: "Yesterday's newsletter was good, but I don't like pure tech articles, prefer applied AI". The AI remembers these preferences for future newsletters.

### Implementation

**In Workflow A (chatbot):**

```javascript
// The AI Agent detects feedback and stores it
// Add to the system prompt:

"If the user gives feedback on their newsletter or preferences,
save it with the following JSON:
{\"action\": \"save_feedback\", \"feedback\": \"[feedback text]\"}
This feedback will be used to personalize future newsletters."
```

When the agent returns `action: save_feedback`:
- Supabase node: `UPDATE newsletter_logs SET feedback = '...' WHERE user_id = x ORDER BY sent_at DESC LIMIT 1`
- Or create a `user_notes` table with accumulated preferences

**In Workflow B (newsletter delivery):**

Fetch the user's latest feedback and inject it into the filtering prompt:

```javascript
// Add to the AI prompt:
`Feedback from this user's previous newsletters:
${previousFeedbacks.join('\n')}
Take this feedback into account in your selection.`
```

---

## 13. Extra 3 - Fact-checking

### Simple concept

Before including an article, check that at least one other source mentions it.

### Lightweight implementation

```javascript
// Code node - simple cross-check
// For each article, check whether another source covers the same topic

const articles = $input.all().map(i => i.json);
const checkedArticles = [];

for (const article of articles) {
  // Look for articles with similar keywords
  const keywords = article.title.split(' ')
    .filter(w => w.length > 4)
    .slice(0, 3);

  const confirmedBy = articles.filter(a =>
    a.url !== article.url &&
    a.source !== article.source &&
    keywords.some(kw => a.title.toLowerCase().includes(kw.toLowerCase()))
  );

  checkedArticles.push({
    ...article,
    factCheckScore: confirmedBy.length, // 0 = unconfirmed, 1+ = confirmed
    confirmedBySources: confirmedBy.map(a => a.source)
  });
}

// Prioritize confirmed articles
return checkedArticles
  .sort((a, b) => b.factCheckScore - a.factCheckScore)
  .map(a => ({ json: a }));
```

### Advanced AI version

Use a second LLM call asking the AI to assess an item's consistency with other fetched sources.

---

## 14. Extra 4 - Per-user custom prompt

### Concept

Each user can customize the tone, format and length of their newsletter through the chatbot.

### In the chatbot (Workflow A)

Add to the AI Agent:

```
If the user wants to customize their newsletter (tone, format, length, style),
capture the request and reply with:
{"action": "update_prompt", "custom_prompt": "..."}

Examples of custom prompts:
- "Write as if addressing an AI expert, with technical vocabulary"
- "Be brief, 3 articles max, telegraphic style"
- "Use a light, humorous tone"
```

### In Workflow B

Use the `custom_prompt` from the `preferences` table in the LLM call:

```javascript
const systemPrompt = user.custom_prompt
  ? `${basePrompt}\n\nCustom instructions for this user: ${user.custom_prompt}`
  : basePrompt;
```

---

## 15. Recommended build order

The optimal order to build the project without getting blocked:

```
WEEK 1 - Working base
━━━━━━━━━━━━━━━━━━━━━

☐ Step 1: Create the Supabase tables (section 3)
☐ Step 2: Partial Workflow B - fetch and display news (section 6)
          → Just to check your APIs work
☐ Step 3: Workflow A - Chat Trigger + simple replying AI Agent (sections 4-5)
          → No authentication yet
☐ Step 4: Add onboarding to Workflow A (section 5)
☐ Step 5: Complete Workflow B - AI filtering + email sending (sections 7-8)

WEEK 2 - Hardening
━━━━━━━━━━━━━━━━━━

☐ Step 6: Add authentication (section 4)
☐ Step 7: Per-user scheduling (section 9)
☐ Step 8: GDPR + unsubscribe, Workflow C (section 10)

WEEK 3 - Extras
━━━━━━━━━━━━━━━

☐ Step 9: HTML newsletter (section 11) ← high priority for the demo
☐ Step 10: User feedback (section 12)
☐ Step 11: Fact-checking (section 13)
☐ Step 12: Custom prompt (section 14)
```

---

## 16. Common n8n beginner mistakes

### Malformed expressions

```
❌ {{ json.email }}        → missing $
✅ {{ $json.email }}

❌ {{ $json['email'] }}    → works but less readable
✅ {{ $json.email }}

❌ {{ $node['Mon Node'].json.email }}   → old syntax
✅ {{ $('Mon Node').item.json.email }}  → new syntax
```

### Node receives nothing

If a node seems to receive no data, the previous node usually returned an empty array. Add an **IF** or **Code** node to debug:

```javascript
// Debug Code node
console.log('Received data:', JSON.stringify($input.all()));
return $input.all(); // pass-through
```

### AI Agent vs Basic LLM Chain

- **AI Agent**: has memory, can use tools (e.g. query Supabase), multi-turn. Use for the chatbot.
- **Basic LLM Chain**: single call, no memory. Use for filtering/summarizing in Workflow B.

### Credentials in the exported JSON

Before sharing your JSON file:
1. n8n > Settings > Export
2. Open the JSON
3. Find `"credentials"` fields and replace values with placeholders
4. Check HTTP Request nodes and remove auth headers

### "Workflow could not be started" error

Check that:
- The workflow is **activated** (toggle top right)
- The trigger node is configured
- Credentials are valid (small green indicator on the node)

---

## 17. Pre-demo checklist

### Core features

- [ ] The chatbot is reachable at the Chat Trigger URL
- [ ] A new user can register
- [ ] An existing user can log in
- [ ] A logged-in user can provide preferences (topics, email, time)
- [ ] The workflow fetches news from at least 2 sources
- [ ] The AI filters news based on user preferences
- [ ] An email with the selected news is sent to the user
- [ ] The newsletter is sent at the user's chosen time
- [ ] A user can unsubscribe
- [ ] Personal data is protected (user A cannot see user B's data)

### Security

- [ ] Passwords are hashed (not stored in plain text)
- [ ] Sessions expire
- [ ] The unsubscribe link contains a signed token
- [ ] The exported JSON contains no sensitive credentials

### GDPR

- [ ] Consent is requested and recorded in the DB
- [ ] Unsubscribing deletes or anonymizes data
- [ ] The newsletter email contains an unsubscribe link

### For the presentation

- [ ] Prepare a demo account (do not use your real email)
- [ ] Have cached news or make sure the APIs work on the day
- [ ] Be able to explain why n8n vs coding from scratch (speed, no-code, native integrations, open-source)
- [ ] Show the workflow visually in n8n (a strong selling point)

---

*Guide for the Persona project - n8n v1.x | Supabase*
