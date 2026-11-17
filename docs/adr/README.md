# Architecture decision records

Format: Nygard / MADR. An accepted ADR is never rewritten; if a decision changes, a new ADR supersedes it and the
old one gets the status "Superseded by ADR-xxxx".

These records were written on 2026-11-17, after the project, from the code and the git history. The decision
dates come from the commit that introduced each choice.

| ADR | Title | Status | Decision date |
| --- | --- | --- | --- |
| [0001](0001-n8n-low-code.md) | Build on n8n (low-code) instead of a coded backend | Accepted | 2026-05-30 |
| [0002](0002-openrouter-gpt-4o-mini.md) | Use gpt-4o-mini through OpenRouter | Accepted | 2026-05-30 |
| [0003](0003-supabase.md) | Use Supabase (Postgres) for storage | Accepted | 2026-05-30 |
| [0004](0004-external-cron-trigger.md) | Trigger the newsletter from an external cron every minute | Accepted | 2026-06-05 |
| [0005](0005-unsubscribe-token.md) | Bit-shift hash token for unsubscribe and feedback links | Accepted | 2026-05-30 |
| [0006](0006-credentials-in-chat.md) | Collect credentials in the chat through agent tools | Accepted | 2026-05-30 |
