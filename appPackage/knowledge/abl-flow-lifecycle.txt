# ABL FLOW retry and lifecycle contract

> Embedded from the Arch MCP debug fallback documentation. Authenticated
> Studio documentation remains authoritative when available.

Use `MAX_ATTEMPTS` with `ON_EXHAUSTED` to bound unsuccessful attempts that may
span customer messages. Runtime consumes an attempt for failed required
`GATHER` input, an authored `ON_INPUT` retry that returns to the protected
step, or a reasoning customer turn that ends without satisfying `EXIT_WHEN`.
Successfully leaving the retry cycle clears the counter. The counter survives
session serialization and is isolated to the active agent thread.

`MAX_TURNS` is different: it limits reasoning, model, and tool cycles inside
one Runtime invocation (one customer turn). It never ends a zone across
customer messages. Pair both limits when one customer message may require
several model/tool cycles but the conversation must stop after a bounded
number of unsuccessful customer messages.

`EXIT_WHEN` is evaluated after the reasoning iteration returns. `always` is the
built-in unconditional sentinel, equivalent to `true`, on every condition
surface. `EXIT_WHEN: always` exits the zone on the same turn and follows
`THEN`; the `THEN` target's `RESPOND` is the authoritative text and the zone
reply is appended. Only the bare identifier is the sentinel; never name a
session variable `always`.

A goal-only zone without `PRESENT` whose `THEN` path cycles straight back into
it re-executes on empty re-entry. An unconditional exit can therefore run to
the FLOW iteration ceiling; add a `PRESENT` or deterministic wait step.

```abl
FLOW:
  resolve_request:
    REASONING: true
    GOAL: "Resolve the customer's request."
    EXIT_WHEN: resolution_ready == true
    MAX_TURNS: 5
    MAX_ATTEMPTS: 3
    ON_EXHAUSTED: escalate_to_human
    THEN: present_resolution
```

For deterministic input retries, an explicit self-transition consumes an
`ON_INPUT` retry and creates a new step occurrence. Entry actions such as
`SET`, `CLEAR`, `LOG`, and `DO` run again for that occurrence. Restoring a
session parked on the step continues the existing occurrence and does not
rerun those actions.

```abl
FLOW:
  confirm_transfer:
    REASONING: false
    MAX_ATTEMPTS: 3
    ON_EXHAUSTED: cancel_transfer
    RESPOND: "Type confirm or cancel."
    ON_INPUT:
      - IF: input == "confirm"
        THEN: execute_transfer
      - IF: input == "cancel"
        THEN: cancel_transfer
      - ELSE:
        RESPOND: "Please type confirm or cancel."
        THEN: confirm_transfer
```

A promptless `ON_INPUT` step is legal. Add `ELSE` when every message must be
consumed. Without `ELSE`, unmatched input remains parked on the same step and
the compiler emits a warning.

`execution.max_flow_iterations` counts accepted step-to-step transitions
cumulatively across customer messages and restored sessions. Runtime refuses
the next nonterminal transition that would exceed the limit. Terminal
completion and an intercepted handoff remain allowed at the exact ceiling;
parent and child agent threads use independent counters.
