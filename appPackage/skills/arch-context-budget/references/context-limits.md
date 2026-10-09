# Context limits and markers

Values are the platform defaults documented for release 1.7.3. Check `debug_docs` for your release;
limits change between releases.

## Where context grows

| Source                   | Default limit                                                       | Lever                                                                                                             |
| ------------------------ | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Conversation history     | None; no token cap, auto-summarization off                          | `prior_turns: compact`, response `MODEL_CONTEXT` on rich replies, `history: last_n` or `summary_only` on handoffs |
| Declared session memory  | None                                                                | Declare only small scalars; `RESET: per_activation`                                                               |
| `## Current Context`     | 8,000 chars per value, 24,000 total (voice profile 6,000 / 16,000)  | `BEHAVIOR: role:`, named references, `prompt_context` budgets                                                     |
| Tool result this turn    | Summarized above 10,000 chars (an extra model call); cut at 102,400 | `MODEL_CONTEXT: FIELDS`, then `structured` strategy with flat `essential_fields`                                  |
| Tool calls in history    | Arguments kept for 6 tool rounds                                    | Pass bulky inputs by ID or with `default_source: runtime`                                                         |
| Knowledge-base results   | 40,000 tokens when more than 10 chunks return                       | `max_search_tokens`, Top K, similarity threshold                                                                  |
| Explicit visibility mode | Declared values appear in memory and in Current Context             | Keep declared memory small                                                                                        |

## Where context gets cut, and the marker

| Cut point                              | Default                     | Marker                                                               | Lever                                                |
| -------------------------------------- | --------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------- |
| One Current Context value              | 8,000 chars                 | `_contextLimited: true` with a preview                               | `value_max_chars`, `projection_strategy`             |
| Whole Current Context                  | 24,000 chars                | Omitted values listed by name                                        | `total_max_chars`                                    |
| Earlier tool result in Current Context | 4,000 chars                 | `_contextLimited`, plus a Prior Tool Result Context note             | `auto_tool_result_value_max_chars`                   |
| Tool result this turn                  | Summarized above 10,000     | `"_summarized": true` or `...[truncated: N chars...]`                | `MODEL_CONTEXT`, `structured_threshold`, `max_chars` |
| Older result in a long turn            | Last 2 rounds stay whole    | `[Result available — see earlier in conversation]`                   | `keep_recent`, `on_result: SET`                      |
| Tool results on later turns            | Always replaced             | `**COMPACTED FOR CHAT HISTORY**`, `[Prior turn result — summarized]` | `on_result: SET`                                     |
| Earlier tool-backed replies            | 200-char preview            | `[Prior response: "…"]`                                              | `assistant_preview_chars`                            |
| A `MODEL_CONTEXT` view                 | 16,000 chars or 1,000 items | an "unavailable" marker; fails closed                                | Select fewer fields                                  |
| Knowledge-base results                 | The token budget            | None; whole chunks dropped                                           | `max_search_tokens`                                  |
| Handoff history                        | Full by default             | Receiver history starts later                                        | `history:`                                           |

## Compaction defaults

| Policy         | Setting                          | Default   | Voice latency profile |
| -------------- | -------------------------------- | --------- | --------------------- |
| tool_results   | strategy                         | summarize | structured            |
|                | max_chars                        | 102,400   | 16,000                |
|                | structured_threshold             | 10,000    | 4,000                 |
|                | keep_recent                      | 2         | 1                     |
| prior_turns    | strategy                         | compact   | compact               |
|                | assistant_preview_chars          | 200       | 120                   |
| prompt_context | value_max_chars                  | 8,000     | 6,000                 |
|                | total_max_chars                  | 24,000    | 16,000                |
|                | auto_tool_result_value_max_chars | 4,000     | 2,000                 |

Policies merge in this order: platform defaults, voice latency profile (voice only), project
settings, the agent's `EXECUTION.compaction`, per-tool hints.

## Tool-result traps

- When N tools run in parallel, each result gets about 100,000 ÷ N characters, never less than 800
  or more than 40,000, and is structured.
- In a chain of three or more sequential tool rounds, the first result drops out before the answer
  unless saved with `on_result: SET` or `keep_recent` is raised.
- Once a tool has `on_result: SET`, `store_result` defaults to false; set it to true if the model
  needs the rest.
- Dotted `essential_fields` such as `plans.name` empty each item; use flat keys or
  `MODEL_CONTEXT: FIELDS: [plans[].name]`.
- In release 1.7.3, on turns where a single-KB agent's automatic first search runs, the model sees
  only the last 5 earlier messages (changed in later releases).

## Secrets

- All project config values, including secret ones, are loaded into conversation state. Never
  reference `config.*` or `env.*` in prompt text, RESPOND or SET.
- Masking is by key name and value pattern. A secret under a neutral name is not masked; give secret
  keys secret-looking names.
- Backends that echo tokens put them in the prompt; allow-list result fields with `MODEL_CONTEXT`.
- Search a few trace requests for the literal values of your secrets before release.
