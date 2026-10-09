# The platform is not a black box

Every runtime decision leaves evidence, and the authored ABL compiles to a model you can read.
Trial and error ("try a prompt tweak, then a lower temperature, then a different model") happens
when that evidence is not read. Before changing anything, find the answer here. For a session that is not live in this MCP
connection, pass `projectId` and `source: "historical"` to the `debug_*` reads.

| Question                                                  | Where the answer is                                  | How to read it                                                                                                                                                                      |
| --------------------------------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| What exactly did the model receive on this call?          | The LLM request: system prompt, messages, tool list  | "Export entire trace" on the session page, or the LLM step's Request view. MCP trace reads leave the prompt out; use `debug_traces` with `text` to find which events carry a marker |
| Why did it choose this agent, field value or completion?  | Decision events                                      | `debug_explain_decision` with `turn` or `type` (handoff, completion, gather_extraction)                                                                                             |
| In what order did things run, and how long did each take? | The span tree                                        | `debug_get_span_tree`                                                                                                                                                               |
| What state was set, by whom, and when?                    | Session state and trace events                       | `debug_get_current_state` (live), `debug_traces` filtered by agent or text                                                                                                          |
| What did the platform compile from my ABL?                | The compiled agent (IR) and package model            | `platform_package_model`; the session's IR tab shows the source and compiled agent the session ran                                                                                  |
| What should happen next, according to the ABL?            | The flow graph                                       | `debug_get_flow_graph` (Mermaid, `includeAppGraph: true`)                                                                                                                           |
| Is my ABL wrong before it ever runs?                      | Lint and validation                                  | `debug_lint_abl`, `platform_validate_package`                                                                                                                                       |
| Which ABL line caused this transcript?                    | Transcript-to-source diagnosis                       | `debug_why_transcript_failed` with the transcript and exported package                                                                                                              |
| What errors, retries and escalations happened?            | Error events                                         | `debug_get_errors`, `debug_traces` with `hasError: true`                                                                                                                            |
| How does this platform feature actually behave?           | Platform docs and the code-derived operation catalog | `debug_docs` (search by topic), `arch://guidance/v1/*` resources                                                                                                                    |
| Does this happen in other sessions too?                   | Durable session history                              | `debug_session_history` with filters, then `debug_analyze_session` on a sample                                                                                                      |
| Did my change help?                                       | Eval run comparison                                  | `platform_eval_runs` with `compare` and `heatmap`                                                                                                                                   |

## Root-cause discipline

1. **Write the expected behavior** from the ABL and the flow graph before reading the trace.
2. **Find the first wrong event:** the earliest event where actual departs from expected. Quote it.
3. **Name the layer** and prove the layer before it was correct (its input was right).
4. **State one hypothesis** and the evidence that would disprove it. Check that evidence.
5. **Change one thing at that layer.** Predict the effect before running it.
6. **Re-run** the same scenario in a new conversation, then the eval set, and compare.
7. **Record** the hypothesis, the evidence, the change and the result.

## Signs of trial and error

Stop and return to step 2 when any of these is true:

- More than one change was made since the last time the trace was read.
- The change is to the prompt, temperature or model, but nobody can name the first wrong event.
- The same symptom was "fixed" twice.
- The explanation contains "maybe", "seems to" or "the model is random" without quoted evidence.
- A fix works in one test chat but was never run against the eval set.

When the developer has already tried several changes, list each one, say why it did or could not
work given the first wrong event, and recommend reverting the ones that do not address the cause.
