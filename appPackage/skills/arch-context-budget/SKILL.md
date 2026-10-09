---
name: arch-context-budget
description: Explain and control what the model sees each turn on the Kore.ai Agent Platform, including system prompt sections, conversation history, session memory, tool results, knowledge-base results, compaction, truncation and secrets in context. Use when the model loses data it was given, repeats itself, when prompts or tokens grow turn after turn, or before shipping a long-conversation or multi-agent project.
---

# Context budget

Every turn ends in one model request: a system prompt, the conversation history and the tool list.
Most "the model forgot", "it's slow" and "the bill is high" complaints come from what went into that
request. Find out from the request itself, not from the reply. If the developer has a single bad
session, use `arch-explain-session` first to find the turn where data was lost.

## Gather the evidence

1. Pick a long session (20+ turns), or run one with `debug_load_agent` and `debug_send_message`.
2. Get the full model requests for turns 1, 5, 10 and 20. The MCP trace tools do not return them:
   Live reads cut each value to about 200 characters; historical reads leave out the
   LLM prompt, messages and tool list (marked `llmPayloadOmitted`). Ask the developer to use **Export entire trace** on the session
   page, or open each LLM step's Request view, and read those payloads.
3. With the MCP tools, locate where cuts happened: `debug_traces` with the session's `sessionId`
   and `text` set to a marker such as `_contextLimited` or `COMPACTED FOR CHAT HISTORY`. For a
   session that is not live, pass `projectId` and `source: "historical"`.
4. In the exported requests, search for the markers `_contextLimited`, `_summarized`, `truncated`,
   `COMPACTED FOR CHAT HISTORY`, `Prior response`, `Prior turn result`, `Result available` and
   "unavailable", and for `{{context}}`, `{{config.`, `{{env.` and unfilled `{{placeholders}}`.

Limits and markers are listed in [references/context-limits.md](references/context-limits.md).

## Two things grow without limit

In release 1.7.3, conversation history and declared session memory have no size cap. History is not
trimmed by tokens or message count, and automatic summarization is off. Memory sections are sent in
full on every call. Long and multi-agent conversations bloat here first.

## Rules that prevent most context bugs

1. **Make visibility explicit.** Declaring `BEHAVIOR: role:` puts the agent in explicit mode:
   `## Current Context` then holds only declared and granted values. One `{{context}}` in an
   allow-list agent admits every session value. On an agent that is already live, values it never
   declared stop reaching the prompt, so declare what it needs and run the eval set before shipping
   the change.
2. **Shape tool results at the source.** Give every tool with more than a few fields a
   `MODEL_CONTEXT`. Without it the platform decides what survives, which is the usual cause of "the
   model forgot the data it just fetched". Voice never summarizes, so this matters twice as much.
3. **Save, don't hope.** On the next turn a tool result becomes a placeholder and a reply that
   followed a tool becomes a short preview. Save what later turns need with `on_result: SET`.
4. **Keep memory small.** Session memory holds IDs and short strings, not objects or lists. Use
   `RESET: per_activation` for scratch values.
5. **Trim handoffs.** `pass` the fields, a one-line `summary`, and `history: summary_only` or a small
   `last_n`. Keys inside `CONTEXT:` must be lowercase.
6. **Set compaction once, at project level** (by API or project import; Studio edits compaction only
   inside the voice profile). The policy is resolved once per conversation and is not refreshed on
   handoff.
7. **Keep the start of the prompt stable.** Identity, rules and tools first; per-turn values later in
   memory. A `{{variable}}` in GOAL changes the start of the prompt and defeats provider caching.
8. **Keep secrets out of prompt text.** Never reference `config.*` or `env.*` in GOAL, PERSONA,
   LIMITATIONS, SYSTEM_PROMPT or RESPOND. Credentials belong in auth profiles.
9. **Give the agent the date** when it reasons about relative dates; the system prompt has none.
10. **Set `max_search_tokens`** on every knowledge-base tool (2,000–4,000 for voice).

## Output contract

1. **Story:** what grows, what gets cut, and what the model lost because of it, in plain words.
2. **Size table:** turn | system prompt by section | history | memory sections | tool schemas |
   total input tokens.
3. **Marker table:** turn | where | quoted marker | what the model lost.
4. **Growth sources,** each with the lever and the ABL change that stops it.
5. **Risks found:** `{{context}}`, `config.*` or `env.*` in prompt text, unfilled placeholders,
   uppercase `CONTEXT:` keys.
6. **Glossary:** each marker and term once, with an example from this session.
7. **Confidence and gaps,** including whether full request payloads were available.

Do not shorten the tables. Test changes in a new conversation: a running conversation keeps the
compiled version it started with.
