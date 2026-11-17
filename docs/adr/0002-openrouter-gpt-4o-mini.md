# ADR-0002: Use gpt-4o-mini through OpenRouter

- **Status**: Accepted
- **Date**: 2026-05-30 (first workflow commit); recorded 2026-11-17
- **Deciders**: Ridwane777

## Context

Three LLM nodes are needed: two chat agents that must call tools reliably, and one chain that selects and
summarises articles as strict JSON. Budget was close to zero and the model might need to change.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **OpenRouter, `openai/gpt-4o-mini`** | One key for many models, switch by changing a string; low price per token; reliable tool calling | Extra intermediary that sees all prompts; adds a small margin over direct pricing |
| OpenAI API directly | One fewer processor | Locked to one vendor's models |
| Free models on OpenRouter | No cost | Rate limits and weaker tool calling; JSON output less reliable |
| Local model (Ollama) | No data leaves the server | Needs a GPU host; the small Render instance used (Node heap capped at 350 MB) cannot run it |

## Decision

Use the n8n OpenRouter chat model node with `openai/gpt-4o-mini` in `LLM Auth`, `LLM Main` and `LLM Newsletter`.

## Consequences

- ✅ Estimated cost around a tenth of a cent per newsletter (see [../costs.md](../costs.md)).
- ✅ The model can be swapped per node without touching the rest of the workflow.
- ❌ Chat content, including credentials typed in the chat, goes to OpenRouter and the model vendor (see
  [0006](0006-credentials-in-chat.md)).
- ❌ Output is non-deterministic; Workflow B needs a JSON parsing fallback.
