# Costs

**Everything on this page is an estimate, not a measurement.** No token counts or invoices were recorded during
the project. The method and assumptions are given so the numbers can be recomputed, and replaced by measured
values (OpenRouter shows the tokens of each request in its activity page).

## Price used

`openai/gpt-4o-mini` on OpenRouter, list price at the time of writing: **USD 0.15 per million input tokens** and
**USD 0.60 per million output tokens**. Check the current price on the OpenRouter model page.

Token counts below use the rough rule 1 token ≈ 4 characters of English text.

## Newsletter (Workflow B), per user per day

One LLM call per newsletter (`LLM - Filter and summarize`).

| Part | Assumption | Tokens (est.) |
| --- | --- | --- |
| Instructions and preferences | Fixed prompt text, topics, language, custom prompt | 300 |
| Past feedback | A few short entries | 100 |
| Articles in the prompt | Up to 25 (the code keeps the top 25); about 700 characters of JSON each (title, description cut at 400 characters, URL, source, date, score) | 25 x 175 = 4,375 |
| **Input total** | | **about 4,800** |
| Output | 5 articles (default `num_articles`) x about 130 tokens (title, 2-3 sentence summary, URL, source) | **about 650** |

Cost per newsletter: 4,800 x 0.15 / 1,000,000 + 650 x 0.60 / 1,000,000 = 0.00072 + 0.00039 ≈ **USD 0.0011**.

| Users (one newsletter a day) | Estimated LLM cost per month (30 days) |
| --- | --- |
| 10 | USD 0.33 |
| 100 | USD 3.30 |
| 1,000 | USD 33 |

Above about 100 users, the binding limit is not the LLM cost but NewsAPI's free 100 requests a day and Gmail's
sending cap (see [dependencies.md](dependencies.md)).

The every-minute cron (1,440 runs a day) costs no tokens: the LLM is only called for users whose time matches.

## Chatbot (Workflow A / A_API)

Every user message is at least one LLM call; a tool call adds another call to read the result.

| Assumption | Value |
| --- | --- |
| Input per call | System prompt and tool definitions (about 1,000 tokens) + memory (up to 30 messages, about 1,500 tokens) ≈ 2,500 |
| Output per call | About 100 tokens |
| Calls for sign-up, login and onboarding | About 20 (around 12 messages plus tool calls) |

Onboarding one user: 20 x (2,500 x 0.15 + 100 x 0.60) / 1,000,000 ≈ **USD 0.009**, a one-off cost per user.
Later preference changes cost a few calls each.

## Fixed costs

| Item | Cost during the project |
| --- | --- |
| Render, Supabase, cron-job.org, NewsAPI Developer, Guardian, Gmail | Free tiers |
| OpenRouter | Pay as you go, prepaid credit |

A production setup would need at least a paid always-on host and a paid news API plan (the NewsAPI free plan is
not for commercial use). Their prices are not estimated here because no plan was chosen.

## How to measure instead of estimating

1. Run Workflow B for one test user with `FORCE_SEND = true`.
2. Read the input and output tokens of the request on the OpenRouter activity page, or in the execution data of
   the `LLM Newsletter` node.
3. Replace the table values above and mark them "measured" with the date and the number of articles.
