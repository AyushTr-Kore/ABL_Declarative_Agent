# Speed checklist

- [ ] At most two routing hops; deterministic `WHEN` on stored state wherever possible.
- [ ] `role: orchestrator` where the coordinator can answer simple turns (`role: router` always
      spends a call).
- [ ] Status-style answers come straight from the tool with `on_result: do: - RESPOND:`.
- [ ] Fixed steps run as FLOW steps with `REASONING: false`.
- [ ] `EXECUTION.inline_gather: true`, so extraction happens in the same call as the reply. In
      release 1.7.3, on a single-knowledge-base agent with GATHER fields this also turns off the
      automatic first search (later releases make that search opt-in per tool); measure both.
- [ ] `process.before.require` rejects bad inputs before a failing API call and a retry round.
- [ ] `confirm` only on irreversible actions; each confirmation costs a full turn.
- [ ] One tool does one job end to end; independent tools can run in parallel in one round.
- [ ] Every tool with more than a few fields has `MODEL_CONTEXT`; no tool-result summaries on hot
      paths.
- [ ] Fewer than about 10 tools per agent, with one- or two-sentence descriptions; parameters the
      model does not fill are hidden with `default_source: runtime`.
- [ ] Tool timeouts set to what the API needs, not the 30 s default; retries only on idempotent reads.
- [ ] `max_search_tokens` set on every knowledge base; lower Top K and a higher similarity threshold.
- [ ] Session memory holds IDs and short strings only.
- [ ] Routing and summarization use the cheapest model tier that passes evals; extended reasoning
      off for conversational agents.
- [ ] Output guardrails use rule-based or provider checks, or a streaming-safe mode keeps replies
      streaming; `reask` avoided on latency-critical agents.
- [ ] Voice: low-latency profile on, a fast-tier model available, short spoken replies
      (`CONVERSATION: speaking: max_sentences`).

## Measure before and after

- The session's performance view lists every model and tool call in a turn with latency, tokens and
  cost: count the sequential model calls.
- Compare input tokens on turn 1 and turn 20 to see growth.
- Change one thing at a time and re-run the same test conversation in a new session.
