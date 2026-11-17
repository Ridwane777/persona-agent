# Documentation map

Read in this order if you do not want to open the workflows.

| Question | Document |
| --- | --- |
| What is Persona and how is it built? | [../README.md](../README.md) |
| How do I install and test it? | [setup.md](setup.md) |
| What is an agent, a tool, `$fromAI`, memory, an LLM chain? | [concepts.md](concepts.md) |
| Why was each technical choice made, and what did it cost? | [adr/README.md](adr/README.md) |
| Which external services, keys, quotas and licences does it rely on? | [dependencies.md](dependencies.md) |
| How much does it cost to run? (estimate) | [costs.md](costs.md) |
| How secure is it? | [security.md](security.md) |
| What happens to personal data? | [legal/gdpr.md](legal/gdpr.md) |
| What would be done differently? | [retrospective.md](retrospective.md) |
| How was it built step by step? | [build-guide.md](build-guide.md) |

The workflow JSON files in [`../workflows/`](../workflows/) are the source of truth. When a document and a
workflow disagree, the workflow wins and the document should be fixed.
