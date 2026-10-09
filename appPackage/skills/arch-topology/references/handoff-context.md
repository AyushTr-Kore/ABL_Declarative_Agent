# Handoff context

Most multi-agent token waste and "the specialist asked again" bugs come from the `CONTEXT:` block.

## What crosses the boundary

| Key       | What it does                                                                                                                                                    |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pass`    | Copies named session values to the receiver. Passed values win over values the model extracts, so the receiver never has to re-ask. Values only, never history. |
| `summary` | A template filled from session values (`"Customer {{customer_id}} has a billing question"`). Not written by a model; a missing value renders empty.             |
| `history` | How much conversation the receiver gets. Defaults to `full`.                                                                                                    |

`history` values:

| Value                                    | Receiver starts with                                                |
| ---------------------------------------- | ------------------------------------------------------------------- |
| `full` (default)                         | The whole conversation so far                                       |
| block with `mode: last_n` and `count: N` | The last N messages; a tool call and its result are never split     |
| `summary_only`                           | No history; the summary appears in its prompt                       |
| `none`                                   | No history                                                          |
| `auto`                                   | `summary_only` when a summary is set, otherwise the last 5 messages |

```yaml
HANDOFF:
  - TO: Billing_Agent
    WHEN: intent == "billing"
    CONTEXT:
      pass: [customer_id, account_id]
      summary: 'Customer {{customer_id}} has a billing question about account {{account_id}}'
      history:
        mode: last_n
        count: 6
```

## Silent mistakes

Each of these parses without an error and is silently ignored:

| Mistake                                      | Effect                                         |
| -------------------------------------------- | ---------------------------------------------- |
| Uppercase `HISTORY:` inside `CONTEXT:`       | Ignored; the receiver gets full history        |
| Uppercase `PASS:` inside `CONTEXT:`          | Ignored; nothing is passed                     |
| Inline `history: { mode: last_n, count: 3 }` | Ignored; full history (use the indented block) |
| Misspelled value such as `history: recent`   | Ignored; full history (`pass` still works)     |

Keys inside `CONTEXT:` must be lowercase.

## Other boundary rules

- A delegate starts with no history. Put everything it needs in `INPUT` or `PURPOSE`.
- The receiver does not see the parent's `last_<tool>_result`. List the result, or the fields you
  saved with `on_result: SET`, in `pass`, or the receiver calls the same tool again.
- Authenticate once near the entry and pass the result; specialists should not gather
  authentication again.
- Compaction settings are resolved once per conversation and not refreshed on handoff. Set them at
  project level, or identically on every agent in a handoff chain.
- When control returns, the parent sees the child's customer-facing replies in its own history.
