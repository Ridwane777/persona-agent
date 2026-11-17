# ADR-0001: Build on n8n (low-code) instead of a coded backend

- **Status**: Accepted
- **Date**: 2026-05-30 (first workflow commit); recorded 2026-11-17
- **Deciders**: Ridwane777

## Context

The project had to deliver, in about ten days, a chatbot for onboarding, a scheduled personalised newsletter
with several news sources and an LLM, and GDPR unsubscribe links. n8n was also a learning goal of the project.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **n8n workflows** | Ready nodes for Supabase, Gmail, HTTP, RSS, AI Agent, memory; visual debugging of each execution; webhooks for free | Logic split across JSON nodes, hard to diff and test; Code nodes have no `crypto` by default; vendor-specific |
| Python (FastAPI + LangChain + APScheduler) | Unit tests, version-friendly code, full control of crypto and auth | Everything (scheduler, email, OAuth, chat UI) written by hand; slower for a first version |
| Node.js (Express + LangChain.js) | Same as Python, one language with the demo page | Same cost as Python |

## Decision

Build everything as n8n workflows (A, A_API, B, C), with Code nodes only where no node fits (normalisation,
schedule check, HTML email, token).

## Consequences

- ✅ All features shipped in the time available, including feedback links and a REST variant of the chatbot.
- ✅ Each execution can be inspected node by node, which made LLM and API debugging fast.
- ❌ No automated tests: the only checks are manual runs. Renaming a node breaks `$('Node name')` references
  silently.
- ❌ A and A_API are two copies of the same logic that must be edited twice.
- ❌ Security primitives are limited in Code nodes (see [ADR-0005](0005-unsubscribe-token.md)).
