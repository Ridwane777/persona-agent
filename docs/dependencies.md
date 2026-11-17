# Dependencies

Every external service, library and secret Persona depends on. Quotas are the free-tier figures noted in
[build-guide.md](build-guide.md) during the project; check the provider's current page before relying on them.

## Services

| Service | Used by | What it does | Free-tier quota (as noted in the build guide) | Failure impact |
| --- | --- | --- | --- | --- |
| n8n (self-hosted, community edition) | All | Runs the workflows, webhooks, hosted chat | Self-hosted: no execution limit | Everything stops |
| Render (web service) | Hosting | Runs n8n in production | Free instance sleeps when idle (see [ADR-0004](adr/0004-external-cron-trigger.md)) | Everything stops |
| cron-job.org | B | Calls `POST /webhook/trigger-newsletter` every minute | Free | No newsletters sent |
| Supabase | A, A_API, B, C | Postgres database | Free project, paused when idle | Chat and newsletters fail |
| OpenRouter (`openai/gpt-4o-mini`) | A, A_API, B | LLM for both agents and the newsletter chain | Paid per token (see [costs.md](costs.md)) | Chat fails; B falls back to raw articles only if the call returns |
| NewsAPI (`/v2/everything`) | B | News search | **100 requests/day**; the free Developer plan is for development, not commercial use | One call per newsletter: caps the service at about 100 newsletters a day |
| The Guardian Open Platform | B | News search | 500 requests/day (the export uses the shared `test` key, which is lower) | Fewer articles |
| BBC News RSS | B | Top stories feed | No key; RSS terms of use apply | Fewer articles |
| Gmail (OAuth2) | B | Sends the newsletter | About 500 emails/day for a personal account | No delivery above the cap |

## Credentials and secrets

| Secret | Where it lives | Notes |
| --- | --- | --- |
| Supabase service role key | n8n credential `Supabase account` | Bypasses Row Level Security; never put it in a client |
| OpenRouter API key | n8n credential `OpenRouter account` | Set a spending limit on the OpenRouter account |
| Gmail OAuth2 client | n8n credential `Gmail account` | Google Cloud OAuth consent screen |
| NewsAPI key | Query parameter in node `NewsAPI` (`YOUR_NEWSAPI_KEY`) | Stored in the workflow, not a credential: moving it to a credential is better |
| Guardian key | Query parameter in node `The Guardian API` (`test`) | Same remark |
| Token salt | Code nodes in B and C (`CHANGE_ME_SECRET_SALT`) | Must be identical in three nodes; see [security.md](security.md) |

No secret is committed: the exports contain only placeholder credential IDs and placeholder values.

## n8n nodes used

`Chat Trigger`, `Webhook`, `Respond to Webhook`, `If`, `Set`, `Code`, `Merge` (SQL mode), `Split In Batches`
(`Loop Users`), `HTTP Request`, `RSS Feed Read`, `Supabase` and `Supabase Tool`, `Gmail`, `AI Agent`,
`Basic LLM Chain`, `Window Buffer Memory`, `OpenRouter Chat Model`. All are built-in n8n nodes; no community
nodes are required.

## Front end

`demo/chat.html` is a single static page with inline CSS and JavaScript. Its only external resource is the
Inter font from Google Fonts.

## Licences

| Component | Licence |
| --- | --- |
| This repository | MIT ([../LICENSE](../LICENSE)) |
| n8n | Sustainable Use License (fair-code): free for internal and personal use; offering n8n itself as a paid service needs a commercial licence |
| Inter font | SIL Open Font License 1.1 |
| News content | Owned by the publishers. Persona stores and emails titles, short summaries and links; the NewsAPI free plan and the Guardian and BBC terms restrict commercial reuse |
| Model output | Subject to OpenRouter's and the model vendor's terms |
