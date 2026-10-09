---
name: arch-explain-session
description: Explain what happened in one Agent Platform session, turn, handoff, tool call, knowledge-base answer or guardrail decision, or find which sessions failed and why, in plain language. Use whenever asked to analyze, debug, diagnose or explain why an agent did something, missed an input, handed off, looped, went silent on a call or answered wrongly. Produces a turn-by-turn timeline with ownership, state changes and latency, the first wrong event, its layer and the fix.
---

# Explain a session

A vague request ("analyze this trace") tempts a three-sentence answer in platform jargon. Developers
cannot act on that, so they go back to the UI. This skill always returns the same checkable shape.

## Gather the evidence

For any session that is not live in this MCP connection, pass `projectId` and
`source: "historical"` on every `debug_*` read below; the default `live` source reads only local
buffers and returns nothing for past sessions.

1. `platform_connect`, then confirm the project ID (required for history reads) and session ID. Find sessions with
   `debug_session_history` (`action: "list"`; `agentName`, `channel` and `status` take arrays; also
   `outcome`, `range` such as `"7d"`, and `sortBy: "errorCount" | "tokenCount" | "estimatedCost"`).
2. `debug_analyze_session` first: event counts, duration, model calls, current state, detected
   loops, errors, constraint violations and tool failures.
3. `debug_traces` to search instead of scrolling: filter by `types`, `text` or `hasError`, and on
   historical reads also by `turnId`, `requestId`, `traceId` or `failureCode` (`agentName` filters
   live reads only). Voice sessions add event types such as `voice_dead_air`,
   `voice_filler_playback` and `voice_playback_watchdog`. Live reads cut each value to about 200 characters; historical reads leave out the
   LLM prompt, messages and tool list (marked `llmPayloadOmitted`). For the full
   model request, ask the developer to use "Export entire trace" on the session page.
4. `debug_explain_decision` with `turn` or `type` (handoff, completion, gather_extraction) for why
   the agent chose what it chose.
5. `debug_get_span_tree` for execution order and timing; `debug_get_current_state` for gathered
   fields and the active flow step of a live session.
6. `debug_get_flow_graph` (after `debug_load_agent`, or with `projectId`, `sessionId` and `source: "historical"`) (Mermaid) when you need to compare what happened with what the ABL says
   should happen.
7. For a transcript plus exported package, `debug_why_transcript_failed` maps symptoms to ABL file
   and line.

Read [references/layers.md](references/layers.md) to decide which layer an event belongs to, and
[references/transparency-map.md](references/transparency-map.md) for where each kind of evidence
lives.

## Many sessions: triage first

When the question is "which sessions failed" or "is this happening to others":

1. `debug_session_history` with `range`, `status`, `outcome` or `channel` filters, sorted by
   `errorCount`, `tokenCount` or `estimatedCost`.
2. Run `debug_analyze_session` on a sample of 5–10 sessions and group them by their first wrong
   event and layer, not by symptom.
3. Report a table: pattern | layer | sessions affected | example session ID. Then explain one
   example session in full with the output contract below.

## The platform is not a black box

Every runtime decision leaves evidence: the exact model request, decision events, the span tree,
state changes, and the compiled agent the session ran. When a developer says "the model is random"
or "the platform did something weird", treat it as a question the evidence can answer, and answer
it. Never recommend a change you cannot tie to a quoted event.

## Output contract

Answer in exactly this order. Do not skip a part; write "not available" and why instead.

1. **Story.** One plain-language paragraph a new team member could retell: what the caller wanted,
   what happened, where it went wrong.
2. **Timeline table,** one row per turn, or per step inside a multi-step turn:
   - what the caller said, and what ASR heard on voice;
   - the owner: active agent, FLOW step, and who held the floor;
   - model calls and tool calls, with tool arguments;
   - state changes written as `field: before → after (set by …)`;
   - latency per step: end of speech, model, tool, first response.
3. **First wrong event:** the turn and event ID, with the trace text quoted. Everything after it
   is consequence.
4. **Layer:** one of ASR, endpointing, language, routing or handoff, gather extraction, tool input
   or output, knowledge retrieval, guardrail, model reasoning, prompt, configuration, platform. Say why it is that layer and not
   the next one downstream.
5. **Fix at that layer,** as an ABL or configuration change, and **what would not fix it**
   (usually a prompt edit).
6. **Eval scenario** that would have caught it, ready to add to the eval set.
7. **Glossary:** every platform term you used, one line each, with an example from this session.
8. **Confidence and gaps:** what you checked, what you could not check, and which data was missing.

Rules:

- Do not summarize or shorten the timeline.
- Quote evidence instead of paraphrasing it. Prefer an example over a definition.
- If the report is long, write it to `analysis/<session-id>.md` and reply with the path and the
  story paragraph.
- Platform defects need the session ID, the first wrong event and the quoted evidence, not a
  description of symptoms.

A complete worked example is in [references/worked-example.md](references/worked-example.md).

## Root cause, not trial and error

Do not patch each symptom you notice, and do not suggest trying a prompt tweak, a lower temperature
or another model to "see if it helps". Each untargeted change adds noise, hides the cause and often
creates a regression.

1. Write the expected behavior from the ABL and flow graph.
2. Find the first wrong event and quote it.
3. Prove the layer before it was correct.
4. State one hypothesis and check the evidence that would disprove it.
5. Change one thing at that layer, predict the effect, re-run the scenario in a new conversation,
   then the eval set, and compare.

If the developer has already tried several changes, list each one, say why it did or could not work
given the first wrong event, and recommend reverting the ones that do not address the cause. The
signs of trial and error are listed in the transparency map.
