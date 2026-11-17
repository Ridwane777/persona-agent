# Concepts

This page explains the ideas behind Persona for someone who has not used n8n or LLM agents before. Node names
refer to the files in [`../workflows/`](../workflows/).

## 1. LLM agent with tools

A plain LLM call takes text and returns text. An **agent** wraps the model in a loop: the model receives the
conversation plus a list of tools (name, description, parameters); it either answers, or asks for a tool call;
the framework runs the tool and gives the result back to the model, which continues until it produces a final
answer. This "reason, act, observe" loop was popularised by ReAct (Yao et al., 2023), and the idea that a model
can learn when and how to call external APIs was studied in Toolformer (Schick et al., 2023).

In Persona, the n8n **AI Agent** node is the loop. Its tools are Supabase nodes in "tool" mode:

- Auth agent (`Auth Agent (logged out)`): `register`, `Find_user`, `Create_active_sessions`.
- Main agent (`Main Agent (logged in)`): `save_preferences`, `update_preferences`, `update_custom_prompt`,
  `save_feedback`, `unsubscribe_user`, `Record GDPR consent`.

The tool description (for example "Saves the user's newsletter preferences. Call it after collecting all
settings AND obtaining explicit GDPR consent.") is the only thing the model knows about the tool, so it doubles
as an instruction.

## 2. `$fromAI` tool arguments

In a tool node, a field written as `{{ $fromAI('email', 'The user\'s email address', 'string') }}` is not filled
by the workflow: n8n exposes it as a parameter of the tool, and the model supplies the value when it calls the
tool. This is how the agent turns "I'm bob@example.com" into a database insert.

The consequence: anything filled by `$fromAI` is model output. It can be wrong (a hallucinated ID), or steered
by a malicious message (prompt injection). Values that the workflow already knows, such as the logged-in
`user_id`, are safer as normal expressions. `Record GDPR consent` does that with `$('Get user info')`; the other
main-agent tools still take `user_id` from `$fromAI` (see [security.md](security.md)).

## 3. Window buffer memory

An LLM is stateless: each call only sees what is sent. The **Window Buffer Memory** node stores the
conversation per session key and sends the last *k* exchanges back with each new message. Persona uses
`contextWindowLength` 20 for the auth agent (`Auth Memory`) and 30 for the main agent (`Main Memory`), keyed by
the chat `sessionId`.

Trade-offs: a larger window keeps more context but costs more tokens on every turn (see [costs.md](costs.md)),
and anything said in the chat, including a password typed during login, stays in memory for that session.

## 4. Two-agent split: auth vs main

Workflow A does not give one agent all the tools. A deterministic branch decides first:

1. `check_sessions` looks for a non-expired row in `sessions` for this `sessionId`.
2. `Session active?` routes to the auth agent (no session) or to the main agent (session found).

The auth agent can only sign up and log in; the main agent can only act on an account. A logged-out user cannot
talk the model into calling `unsubscribe_user`, because that tool is not connected to the agent they are talking
to. This is the "least privilege" answer to OWASP LLM06 (excessive agency): the control is in the graph, not in
the prompt.

## 5. LLM chain vs agent

Workflow B uses `LLM - Filter and summarize`, a **Basic LLM Chain**: one prompt in, one completion out, no tools
and no loop. The steps of a newsletter are fixed (fetch, filter, summarise, send), so letting a model decide the
order would add cost and failure modes without benefit. The chain is asked for strict JSON, and
`Parse LLM response` extracts it with a regular expression; if parsing fails, the original articles are used so
the user still gets an email.

Rule of thumb used here: agent when the next step depends on what the user says; chain when the steps are known.

## 6. Cross-source fact-check heuristic

`Normalize + Dedup + Fact-check` gives each article a `factCheckScore`: the number of articles from a
**different source** whose title contains at least one of the first three words longer than four letters of
this article's title. Articles are sorted by this score, the top 25 go to the LLM, and the prompt asks it to
prefer higher scores.

What it measures: how widely a story is covered. What it does not measure: truth. Known limits:

- Several outlets can repeat the same wire story, including a wrong one.
- Keyword matching is crude: "Trump" or "Ukraine" will match unrelated stories; a paraphrased headline will not.
- NewsAPI returns many publishers, so "different source" is not the same as "independent source".
- The Guardian and BBC feeds are English; French topics match fewer articles.

A real verification step would compare claims (for example with embeddings or a dedicated fact-check API) and
show the user the sources, not a single number.

## 7. No RAG

Retrieval-augmented generation (RAG) stores documents as embeddings in a vector database and retrieves the most
similar ones before prompting. Persona does not do this. Articles are fetched live for each newsletter, filtered
by keyword, and put directly in the prompt. With at most 25 short articles per call, everything fits in the
context window, so a vector store would add infrastructure without improving the result. RAG would become
useful to search an archive of past articles or past feedback across many users.

## 8. Security concepts relevant to LLM apps

The OWASP Top 10 for LLM Applications lists the risks that apply here: prompt injection (LLM01), sensitive
information disclosure (LLM02, a password in the chat), improper output handling (LLM05, LLM text inserted into
HTML), and excessive agency (LLM06, tools with model-chosen IDs). Their status in Persona is in
[security.md](security.md).

## References

- Yao, S. et al. (2023). *ReAct: Synergizing Reasoning and Acting in Language Models*. ICLR 2023.
  <https://arxiv.org/abs/2210.03629>
- Schick, T. et al. (2023). *Toolformer: Language Models Can Teach Themselves to Use Tools*. NeurIPS 2023.
  <https://arxiv.org/abs/2302.04761>
- OWASP. *Top 10 for Large Language Model Applications* (2025 edition). <https://genai.owasp.org/llm-top-10/>
- n8n docs. *AI Agent node*.
  <https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/>
- n8n docs. *Let AI specify tool parameters (`$fromAI`)*. <https://docs.n8n.io/advanced-ai/examples/using-the-fromai-function/>
- n8n docs. *Simple Memory (window buffer memory) node*.
  <https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorybufferwindow/>
- n8n docs. *Basic LLM Chain node*.
  <https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm/>
