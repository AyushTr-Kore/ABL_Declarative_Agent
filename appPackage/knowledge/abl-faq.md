# ABL engineering FAQ

> Curated from the Agent Platform FAQ and ABL references. The full source is
> maintained in `apps/docs-internal/content/faq/faq.mdx`.

## Authoring

### What is required in an agent?

An agent definition requires `AGENT:` and `GOAL:`. Add `PERSONA:`,
`INSTRUCTIONS:`, `TOOLS:`, `GATHER:`, `FLOW:`, memory, guardrails, and
handoffs only when the use case needs them.

### When should I use FLOW?

- Use no `FLOW` for open-ended conversations where the model should choose the
  next action.
- Use `FLOW` for predictable steps such as intake, onboarding, or a regulated
  transaction.
- Use a mixed flow when most steps are deterministic but selected steps need
  `REASONING: true`.

Adding a flow does not remove the agent's general capabilities. It makes the
declared steps explicit; reasoning can still be enabled on individual steps.

### What is the difference between limitations and constraints?

`LIMITATIONS:` gives the model prompt-level boundaries and caveats.
`CONSTRAINTS:` is for deterministic checks that depend on state or a runtime
checkpoint. Do not use a prompt limitation as the only security control.

### How do I write multiline instructions?

Use a pipe block and indent its content:

```abl
INSTRUCTIONS: |
  Verify identity before account changes.
  Explain uncertainty instead of inventing a result.
```

### Why does my ABL fail to parse?

Check indentation, missing section keywords, unclosed quoted strings, and
invalid step or tool names. Use the Studio diagnostics and validate the
package before testing runtime behavior. Do not add the deprecated global
`MODE:` keyword; use per-step `REASONING:` in `FLOW` instead.

## Tools and runtime behavior

### The agent chooses the wrong tool. What should I check?

Check that the tool is declared and bound, its parameters and return type are
accurate, and its description explains when to use it. Then inspect the trace
for the model's selected tool, actual arguments, result, and error. A clear
description improves selection but does not replace authorization.

### Why did a tool call fail with 401 or 403?

Verify the configured auth profile, token audience and scopes, endpoint, and
the user's authorization to the target resource. Keep secrets in the platform
credential store; never paste them into an ABL definition or request them in
chat.

### How do I collect information from a user?

Use `GATHER` with a clear prompt, type, and `required: true` when the value is
needed. In a `FLOW`, gather at the step where the value is required. Validate
the value again before a side-effecting tool call.

### Why is a flow looping or stopping at the wrong step?

Check the `entry_point`, declared `steps`, exact `THEN` targets, conditional
branches, and `ON_FAIL` paths. Every retry path should have a useful exit or a
bounded attempt count.

## Debugging

### What should I inspect first?

Start with the user input, compiled package diagnostics, execution trace,
selected tool and arguments, tool output, and the final response. Separate
observed errors from hypotheses and reproduce the issue in a fresh session.

### What do trace spans mean?

- Agent spans show top-level execution.
- LLM spans show model prompts and decisions.
- Tool spans show invocation, arguments, output, latency, and errors.
- Step spans show progress through a structured flow.

### Common error categories

`COMPILE_ERROR` usually means the definition or binding is invalid.
`TOOL_NOT_FOUND` means the declared name is not available in the runtime
catalog. `TOOL_TIMEOUT` indicates an endpoint or worker exceeded its timeout.
Session or deployment errors require checking the selected project and
environment rather than changing the prompt first.

## Safety and knowledge

Use guardrails for PII, unsafe content, output checks, and escalation. Use
EmbeddedKnowledge for small, stable, approved references; use WebSearch for
current public documentation and MCP for live tenant or deployment state.
Knowledge files do not override the agent's instructions or permissions.
