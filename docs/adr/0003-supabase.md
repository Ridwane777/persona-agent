# ADR-0003: Use Supabase (Postgres) for storage

- **Status**: Accepted
- **Date**: 2026-05-30 (first workflow commit); recorded 2026-11-17
- **Deciders**: Ridwane777

## Context

The workflows need four tables (users, preferences, sessions, newsletter_logs) reachable from n8n, both from
normal nodes and as agent tools, with a free hosted option.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Supabase** | Hosted Postgres with a free tier; native n8n node usable as an agent tool; SQL editor and table viewer for demos; Supabase Auth available later | Service role key bypasses Row Level Security; free projects pause when idle |
| n8n Data Tables / Google Sheets | Nothing to set up | No constraints, no foreign keys, weak for GDPR deletions |
| Self-hosted Postgres | Full control | Needs hosting and backups; n8n Postgres node is SQL-string based |
| Airtable | Friendly UI | Row limits on free plan; not relational enough |

## Decision

Use Supabase through the n8n Supabase node, authenticated with the service role key.

## Consequences

- ✅ Foreign keys with `ON DELETE CASCADE` and check constraints (`rating`, `feedback_source`).
- ✅ Agent tools are plain Supabase nodes in tool mode, no custom code.
- ✅ Supabase Auth is available to fix the login design ([0006](0006-credentials-in-chat.md)).
- ❌ The service role key gives n8n access to every row; isolation relies on workflow filters only.
- ❌ Filters are limited to what the node offers (for example `save_feedback` updates every log row of the user,
  not only the last one).
