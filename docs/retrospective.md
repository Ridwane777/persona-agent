# Retrospective

A look back at the whole project, written on 2026-11-17 from the code, the git history and the known issues.

## Timeline

| Date (git) | What happened |
| --- | --- |
| 2026-05-30 | Workflows A (auth), B (newsletter) and C (unsubscribe) first committed; real Gmail sending |
| 2026-05-31 | Production tuning (Gmail, memory limits), GDPR consent flow, README |
| 2026-06-01 | GDPR and security notes, feedback loop from past newsletters |
| 2026-06-02 to 06-04 | In-email feedback links, feedback fixes |
| 2026-06-05 | External cron every minute, chatbot exposed as REST (A_API) |
| 2026-06-07 | Demo landing page, screenshots |
| 2026-06-08 | Last workflow fix (unsubscribe flow) |
| 2026-10-26, 11-15 | Documentation translated to English and cleaned |

The core was built in about ten days by one person (all commits are from Ridwane777). Planned dates were not
written down, so this table cannot compare planned with actual; that is itself a lesson (see Organisation).

## What went well, to keep

- **Deterministic routing around the agents.** The `If` on the session before choosing the auth or main agent
  limits what a logged-out user can trigger, without relying on the prompt.
- **Chain, not agent, for the newsletter.** Fixed steps plus a JSON-only prompt and a parsing fallback made B
  predictable.
- **Pre-filtering before the LLM.** Topic filtering, deduplication and the 25-article cap keep each call small
  (about a tenth of a cent, see [costs.md](costs.md)).
- **Feedback loop.** Ratings from the email links reach the next prompt for the same user.
- **Honest documentation of limits**, kept next to the code ([security.md](security.md)).

## Technical: what to do better

| Problem (verified in the code) | Named alternative | When it becomes worth it | Its cost |
| --- | --- | --- | --- |
| Password typed in the chat, sent to the LLM, stored as unsalted SHA-256 | Supabase Auth (magic link) or an n8n Form with bcrypt/argon2; chat receives only the session | Before any real user | One page or form, about a day |
| Static-salt bit-shift token, no expiry | HMAC-SHA256 with n8n's Crypto node and a secret in a credential, with an expiry date in the signed data | Before sending to real addresses | Two nodes, about two hours |
| `save_feedback` (chat) updates every log row of the user; the Supabase node cannot target only the latest | Filter on the latest `newsletter_id` | Once users have several newsletters | Under an hour |
| A and A_API duplicate 25 nodes | One sub-workflow (Execute Workflow) called by both triggers | As soon as the logic changes again | Half a day |
| One NewsAPI call per user per day hits the 100/day free cap | Fetch once per topic per hour and cache in Supabase; or a paid news API | Above about 50 to 100 users | A cache table and a second workflow; or a paid plan |
| Every-minute polling with no catch-up | Queue the next send time per user and process due rows (`WHERE next_send_at <= now()`), so a missed minute is caught up later | When missed emails are reported | A column and a query change |
| No tests | Export the Code-node functions to a small JS module with unit tests, and a script that checks node references in the JSON | From the start of the next project | A day to set up |

## Organisation

- **Solo work, no review.** Every change went straight to `main`. A second reader, or at least a checklist
  before each merge (node references, secrets, prompt changes), would have caught the auth prompt expression bug
  and the time zone fallback.
- **No written plan.** Deadlines and scope were tracked in a grading checklist, not in dated milestones. Next
  time: a short plan with dates per feature, updated when something is cut.
- **Decisions recorded late.** The ADRs in [adr/](adr/README.md) were reconstructed months later. Writing each
  one on the day of the choice would have kept the rejected options and the reasons.

## Tooling and hardware limits

- **Free hosting** (Render free instance, Node heap capped at 350 MB) forced the external cron and ruled out a
  local model.
- **n8n Code nodes without `crypto`** pushed the token design to a weak hash.
- **Workflow JSON is hard to review**: prompts and code are escaped strings inside JSON, so diffs are unreadable.
  Keeping prompts and Code-node sources as separate files, injected at build time, would help.
- **No staging environment**: tests ran against the same Supabase project and Gmail account as the demo.

## Deadlines and scope

- Shipped: chatbot (two variants), newsletter with three sources, feedback by chat and by email link, GDPR
  consent and unsubscribe, demo page.
- Not done: secure login, real fact checking, tests, a privacy policy and terms, deployment documentation as
  code. The security fixes above were identified but postponed in favour of features.

## Next actions

1. Store `schedule_tz` from the chat, so users can ask for local time.
2. Move login to Supabase Auth or a form with bcrypt/argon2 (new ADR superseding [ADR-0006](adr/0006-credentials-in-chat.md)).
3. Replace the token with HMAC-SHA256 (new ADR superseding [ADR-0005](adr/0005-unsubscribe-token.md)).
4. Merge A and A_API through a sub-workflow.
5. Measure real token usage and replace the estimates in [costs.md](costs.md).

## Fixed on 2026-11-17

Found while writing this retrospective, fixed in the workflows: `user_id` bound to the session in every
main-agent tool; auth prompt turned into an n8n expression so the real session ID reaches the model, and
`session_id` taken from the trigger; time zone fallback to `Africa/Lagos` removed; HTML escaping and `http(s)`-only
links in the email; JSON-safe API replies; unsubscribing through the chat also deletes preferences and sessions,
like the email link; secret header required on `trigger-newsletter`.
