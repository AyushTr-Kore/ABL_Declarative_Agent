---
name: arch-latency-cost
description: Reduce latency, model calls per turn, tokens and cost for agents on the Kore.ai Agent Platform. Use when an agent is slow, makes too many LLM calls, has a high token bill, has long time to first response or first audio, or when asked to optimize. Finds the most expensive turns, shows where each call and token goes, and maps each hotspot to one lever with the eval scenarios to re-run.
---

# Latency and cost

A turn's time is roughly the number of sequential model calls times the time each takes, plus tools
and checks. Count the calls before touching prompts or models. Never optimize without an eval set
(`arch-eval-catalog`): a faster agent that fails more conversations is not an improvement.

## The loop

1. **Measure** model calls, input tokens and latency per turn for the top intents, from real or eval
   sessions. Use `debug_session_history` (`sortBy: "tokenCount"` or `"estimatedCost"`, `range`),
   then `debug_analyze_session` and `debug_get_span_tree` on the worst sessions (pass `projectId`
   and `source: "historical"` for sessions that are not live).
2. **Rank** turns by frequency × cost.
3. **Change one lever** from the table below.
4. **Re-run** the eval set and `platform_eval_runs` `compare` with the baseline: success must hold,
   cost must drop.
5. **Record** before and after numbers in the change note.

## Where a turn's time goes

| Step                  | Runs when                                          | Lever                                                            |
| --------------------- | -------------------------------------------------- | ---------------------------------------------------------------- |
| Input checks          | Every turn with input guardrails                   | Rule-based checks first; model judges only where meaning matters |
| Routing call          | Every turn that starts at a router or orchestrator | Deterministic `WHEN`; a faster model tier for coordination       |
| Main model calls      | 1, plus 1 per tool round                           | Fewer tool rounds; smaller prompts                               |
| Tool calls            | Each round                                         | Fast APIs, parallel calls, `MODEL_CONTEXT`, tight timeouts       |
| Tool-result summaries | A result over 10,000 chars under `summarize`       | `MODEL_CONTEXT` or `structured`                                  |
| Extraction            | GATHER fields outside inline gather                | `EXECUTION.inline_gather: true`                                  |
| Output checks         | Every reply with output guardrails                 | Fewer output judges; a streaming-safe mode                       |
| Reask                 | An output rule with reask fires                    | `block` with a good message on latency-critical agents           |
| Handoff hops          | Each handoff                                       | Flatter topology; let orchestrators answer simple asks           |

## Levers, roughly in order of payoff

1. **Remove routing hops.** Merge agents that always hand to each other; route deterministically on
   stored state.
2. **Make deterministic steps deterministic.** FLOW steps with `REASONING: false` run without a model
   call. Keep a reasoning zone open only while it must own the conversation: a zone whose
   `EXIT_WHEN` waits for state the FLOW already produces stays resident across turns and redoes that
   work. For a side question, use a single-purpose reasoning step (`CONTEXT_UPDATES: false`) that
   returns to the flow through `THEN`, or `EXIT_WHEN: always` to end the zone after one iteration.
3. **Answer straight from the tool.** `on_result: do: - RESPOND:` sends the reply and ends the tool
   loop, saving the call that would phrase it. Add a `MODEL_CONTEXT` next to the `RESPOND` inside `do:` (directly under `on_result:` it is ignored).
4. **Chain results with `on_result: SET`** instead of a separate context-setting round-trip.
5. **One tool, one job end to end.** Three sequential lookups cost three model rounds.
6. **Trim handoff context** (`pass`, `summary`, `history: summary_only` or small `last_n`).
7. **Shrink tool results** with `MODEL_CONTEXT: FIELDS`; filter and paginate at the API.
8. **Fewer tools per agent** (about 10 or fewer), one- or two-sentence descriptions.
9. **Right-size the model per operation** and turn off extended reasoning for conversational turns.
10. **Voice:** keep the low-latency profile on and cover unavoidable waits with fillers.

```yaml
TOOLS:
  get_order_status(order_id: string) -> {status: string, eta: string, carrier: string}
    effect: read_only
    MODEL_CONTEXT:
      FIELDS: [status, eta, carrier]
    on_result:
      do:
        - RESPOND: "Order {{order_id}} is {{result.status}} with {{result.carrier}}, arriving {{result.eta}}."
          MODEL_CONTEXT: "Told the customer order {{order_id}} is {{result.status}}, ETA {{result.eta}}."
```

The full checklist is in [references/speed-checklist.md](references/speed-checklist.md).

## What not to do

- Do not remove a call, prompt line or guardrail that an eval scenario depends on without re-running
  that scenario.
- Do not optimize from one session; optimize on the distribution of the top intents.
- Do not trade a deterministic guarantee (authentication, payment confirmation, compliance wording)
  for latency.

## Field patterns

Illustrative patterns from a production multi-agent assistant with chat and voice:

- One short utterance made six sequential model calls (filler, route, extract, validate, set
  context, reply), and input tokens grew on every call because tool results were re-sent.
- Swapping to a smaller model did not fix the latency and broke extraction. Prompt caching already
  worked, so shrinking the prompt barely helped. Round-trips, not the model tier, set the latency.
- A reasoning zone whose `EXIT_WHEN` waited for full booking state stayed resident and redid work
  the FLOW already owned. A single-purpose step that returns to the flow through `THEN` cut that
  turn from many seconds and several model calls to under a second with none.
- Chaining a validation result with an exact-match `on_result: SET` removed one model call.
- With no `INTENTS` block, the platform inferred undescribed labels and routed safety reports to
  FAQ; declaring intents with descriptions fixed first-turn routing.

## Previews

Preview decision models can answer some routing and safety decisions before the main model. They
are not generally available; see `arch-voice-design` references for status and safe adoption.

## Output contract

1. **Story:** where the time and tokens go, in plain words.
2. **Hot-turn table:** session | turn | caller said | owning agent and step | model calls in order
   with input tokens each | tool calls with result size | latency per step.
3. **Growth evidence:** what grows turn to turn, with the trace marker quoted.
4. **Lever per hotspot:** the change, the expected saving, and the eval scenarios to re-run.
5. **Glossary** and **confidence and gaps.**

Do not shorten the tables. If long, write to `analysis/latency-cost-<date>.md`.
