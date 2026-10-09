---
name: arch-tool-design
description: Design, review and debug tools used by ABL agents on the Kore.ai Agent Platform, including HTTP, MCP, workflow, integration, knowledge-base and table tools; parameters and hidden runtime inputs; effects, confirmation and idempotency; result shaping with MODEL_CONTEXT and on_result; preconditions; errors and retries; auth profiles; timeouts; and testing. Use when adding or changing a tool, when a tool is called wrongly, twice or with missing values, when a result is lost or leaks data, or before shipping write actions.
---

# Tool design

A tool is a contract between the model and your backend. Most tool bugs come from a contract that
leaves something to the model: an ordering, a hidden ID, a confirmation, an error message. Make the
contract structural and keep the model's job small.

## Where each setting lives

- **The agent's `TOOLS:` entry:** the signature, description, parameter rules and behavior keys
  (`effect`, `confirm`, `MODEL_CONTEXT`, `on_result`, `on_error`, `process`).
- **The project tool:** endpoint, method, auth, headers, timeout and retry. Implementation keys
  written in an agent `TOOLS:` entry are dropped with a warning.
- Tool kinds you can create: HTTP, MCP, workflow, integration, knowledge base (SearchAI), table and
  sandbox code. Knowledge-base tools: see `arch-knowledge-search`. Workflow tools: see
  `arch-workflow-design`.

## The contract, key by key

| Concern                     | Key                                                                                                                   | Rule                                                                                                                                                                                                                                                 |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| What it is for              | `description`                                                                                                         | One or two sentences: what it does and when to use it. Without one it defaults to "Execute <name>"                                                                                                                                                   |
| Hidden inputs               | `params:` with `default_source: runtime`, `visibility_model: false`, `context_access.read: [field]`                   | Use for IDs, tokens and anything the model should not fill or see; a missing value fails the call closed                                                                                                                                             |
| Effect                      | `effect: read_only \| state_write \| side_effect`                                                                     | Declare it on every tool. Undeclared HTTP tools are judged by method (GET, HEAD, OPTIONS read-only), knowledge-base tools are read-only, table tools by their operations; workflow, MCP, integration and sandbox tools are treated as side-effecting |
| Facts it publishes          | `provides` with `on_result` `set`                                                                                     | Allow-list what a successful result may write to the session                                                                                                                                                                                         |
| Confirmation                | `confirm: always \| never \| when_side_effects`, `immutable`                                                          | Confirm irreversible writes, freeze amounts and IDs with `immutable`. `confirm: yes` means no confirmation                                                                                                                                           |
| Preconditions               | `process.before` with `require` and `on_fail` (`reask`, `block`, `warn`, `use_original`, `fallback`)                  | Reject missing or invalid inputs before the API call; `reask` is the default on interactive channels                                                                                                                                                 |
| Result shape                | `MODEL_CONTEXT` with `FIELDS` (or `TEMPLATE`)                                                                         | Pass only the fields the model needs; up to 64 paths, 1,000 items, 16,000 characters, otherwise the view fails closed                                                                                                                                |
| Remove sensitive output     | `process.after` with `OMIT` (beta)                                                                                    | Drop echoed tokens and personal data before the model sees them                                                                                                                                                                                      |
| Keep values for later turns | `on_result` `set`                                                                                                     | Results are replaced by a placeholder on the next turn; `store_result` turns off once `set` is used unless you set it back on                                                                                                                        |
| Reply without a model call  | `on_result` `do` with `RESPOND`                                                                                       | `RESPOND` must sit under `do:`, or it is ignored                                                                                                                                                                                                     |
| Errors                      | `on_error` with canonical types (`tool_error` subtypes `rate_limit`, `auth_failure`, `network_error`, `tool_timeout`) | Write what the caller hears; do not retry `auth_failure`                                                                                                                                                                                             |
| Timeouts and retries        | Project tool timeout (milliseconds), `retry`, `idempotency_header`                                                    | Default 30 s, cap 300 s. Write milliseconds; "30s" is rejected by one parser and misread as 30 ms by another. Retry only idempotent reads unless the provider honors an idempotency key                                                              |
| Credentials                 | Auth profiles                                                                                                         | Never secret config in prompts or tool parameters                                                                                                                                                                                                    |

## Idempotency

The runtime claims a receipt before dispatching an external-effect tool, keyed by the tool, its
arguments and the current turn or step, so a duplicate call within that scope is not sent twice. It
protects every tool treated as side-effecting (declared, or inferred for POST, workflow and MCP
tools); a repeat in a later turn is not caught. Make backend writes idempotent as
well; in guarded realtime mode, an unknown outcome must be checked, never retried blindly.

## Errors the caller hears

For retryable errors (rate limit, network, timeout) the built-in behavior retries once after about a
second, then continues with the error result. A
`default:` handler without `RETRY` removes that retry. When nothing matches, the caller hears a
generic "An error occurred. Please try again." Author an `on_error` for each tool the caller
depends on, with a message and a next step.

## Test a tool

1. `platform_tools` `test` runs the real tool with sample inputs; test real, empty and error
   responses.
2. Exercise it through the agent: `debug_load_agent`, `debug_send_message`, then `debug_traces`,
   `debug_get_errors` and `debug_explain_decision`.
3. For MCP tools: `platform_mcp_servers` `test_connection`, `list_tools` and `test_tool`.
4. When the backend is not ready, point the project tool at a mock service you control; the
   platform has no built-in mock.

## Rules

1. Declare `effect` on every tool; mark read-only workflow and MCP tools `effect: read_only`.
2. Confirm irreversible writes explicitly and freeze their key values with `immutable`.
3. Put timeout, retry and auth on the project tool, never in the agent file.
4. Shape every result with more than a few fields with `MODEL_CONTEXT: FIELDS`.
5. Save what later turns need with `on_result` `set`.
6. Hide IDs and credentials with `default_source: runtime` and `visibility_model: false`.
7. Gate writes with `process.before.require` and `on_fail: reask`.
8. Strip echoed secrets and personal data with a `FIELDS` allow-list or `OMIT`.
9. Use auth profiles for credentials.
10. Use canonical error types, author the caller's message, and never retry `auth_failure`.
11. Declare `idempotency_header` when the provider supports it.
12. Make one tool do one job end to end; three sequential lookups cost three model rounds.
13. Keep descriptions short and specific, and keep fewer than about 10 tools per agent.

## Output contract

1. **Story:** what the tool is for and the one change that matters most, in plain words.
2. **Contract table:** tool | effect | confirm | hidden inputs | preconditions | result fields |
   on_result set | on_error | timeout and retry | findings.
3. **Evidence** for each finding (trace event, test result or file line).
4. **Changes** as ABL or project tool snippets, and the scenarios to test them (real, empty, error,
   duplicate call).
5. **Glossary** and **confidence and gaps.**
