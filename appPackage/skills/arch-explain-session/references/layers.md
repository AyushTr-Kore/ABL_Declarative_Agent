# Which layer is it?

Pick the earliest layer where the evidence is already wrong. If the input to a layer was wrong, the
fault is upstream.

| Layer                | Typical symptom                                                  | Evidence to check                                                         | Fix lives in                                                                                              |
| -------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| ASR                  | Wrong words, missing digits, wrong entity value                  | ASR final text vs what the caller said; confidence                        | ASR provider, vocabulary, per-field listening settings                                                    |
| Endpointing          | Caller cut off mid-answer; long silence after "yes"              | Pause length vs end-of-speech timeout for that turn                       | `VOICE: end_of_speech_turn_type` or `end_of_speech_timeout_ms` on the turn                                |
| Language             | Reply in the wrong language; language drifts back                | Detected language, interaction language, locale events                    | Language switch configuration and FLOW, not the prompt                                                    |
| Routing or handoff   | Wrong agent answers; ping-pong between agents                    | Handoff decision events, `WHEN` evaluation, routing call                  | `WHEN` conditions, intent descriptions, topology                                                          |
| Gather extraction    | Field empty, wrong or from the wrong turn                        | Extraction event input and output                                         | GATHER field definition, validation, `process.before.require`                                             |
| Tool input or output | Tool called with wrong or partial arguments; result lost         | Tool call arguments, raw result, `MODEL_CONTEXT` view, truncation markers | Tool contract, `MODEL_CONTEXT`, `on_result: SET`                                                          |
| Knowledge retrieval  | Answer missing facts the documents contain; wrong document cited | Search tool call: query, chunks returned, token budget                    | Search instructions, `max_search_tokens`, Top K, similarity threshold, chunking (`arch-knowledge-search`) |
| Guardrail            | Valid message blocked; reply replaced or held back               | Guardrail check events: rule, score, action                               | The rule's condition, kind and action; the policy's fail mode                                             |
| Model reasoning      | Correct inputs, wrong decision                                   | The model request for that call (Export entire trace)                     | Instructions, tool descriptions, model tier                                                               |
| Prompt               | Placeholder unfilled, contradicting instructions                 | System prompt sections in the request                                     | GOAL, PERSONA, LIMITATIONS text                                                                           |
| Configuration        | Behavior differs from the ABL; defaults changed                  | Channel and project runtime settings, release notes                       | Explicit configuration values                                                                             |
| Platform             | Behavior contradicts documented contract with correct inputs     | Event ordering, missing or duplicated events                              | Report with session ID, first wrong event and quoted evidence                                             |

Common traps:

- A model that "forgot" data usually never received it: check truncation and compaction markers
  before blaming the model.
- A declarative `WHEN` evaluated before the model set the variable in the same turn looks like a
  routing bug; the cause is ordering.
- A voice result that was never spoken may be a provider that does not speak tool results without
  a trigger, not a model failure.
