# ABL agent declaration reference

> Curated from the Agent Platform ABL reference. The full source is maintained
> in `apps/docs-internal/content/abl-reference/agent-declaration.mdx`.

The declaration establishes an agent's identity, metadata, objective,
personality, boundaries, instructions, and execution configuration.

## Required declaration

Every `.agent.abl` document requires a unique `AGENT:` name and a `GOAL:`.

```abl
AGENT: Hotel_Search
VERSION: "1.0.0"
DESCRIPTION: "Helps customers find suitable hotels."
LANGUAGE: "en"

GOAL: |
  Help the customer find and book a hotel matching their preferences,
  budget, and travel dates.
```

Naming guidance:

- Use `PascalCase_With_Underscores`.
- Start with a letter.
- Use only letters, digits, and underscores.
- Keep the name unique within the project.

`VERSION` is optional and uses `major.minor.patch` format. When omitted, the
document version defaults to `1.0.0`. `LANGUAGE` accepts a BCP 47 language code
such as `en`, `es-EC`, or `fr`.

## Identity and behavior

`GOAL` describes what the agent must accomplish. It is the primary objective
used in the runtime prompt and completion decisions.

`PERSONA` describes how the agent communicates: tone, style, expertise, and
transparency. It should not be used as an authorization policy.

`LIMITATIONS` provides prompt-level boundaries and caveats:

```abl
PERSONA: |
  Precise and transparent travel advisor who explains trade-offs.

LIMITATIONS:
  - "Cannot guarantee room availability"
  - "Cannot process payments directly"
  - "Cannot access another user's loyalty account"
```

Use `CONSTRAINTS` for deterministic checks that depend on state or runtime
checkpoints. Prompt limitations alone do not enforce a hard security boundary.

`INSTRUCTIONS` provides procedural guidance that supplements the goal:

```abl
INSTRUCTIONS: |
  Ask for travel dates before searching.
  Explain price, location, and cancellation trade-offs.
  Never claim availability without a current tool result.
```

## IDENTITY shorthand

`IDENTITY` is a compact alternative to separate identity sections:

```abl
IDENTITY:
  role: "Help the customer find a suitable hotel"
  persona: "Patient travel advisor"
  expertise: [hotel chains, boutique properties, price comparison]
  limitations:
    - "Cannot guarantee room availability"
```

For clarity, prefer separate `GOAL`, `PERSONA`, `LIMITATIONS`, and
`INSTRUCTIONS` sections. If both `IDENTITY` and individual sections are used,
the values appearing later in document order take precedence.

## Execution configuration

`EXECUTION` controls runtime behavior such as model selection, token limits,
timeouts, reasoning limits, and per-operation model routing.

```abl
EXECUTION:
  model: claude-sonnet-4-5-20250929
  temperature: 0.2
  max_tokens: 4096
  max_reasoning_iterations: 8
  tool_timeout: 30000
```

Always bound reasoning iterations for production agents. Use a lower-cost model
for simple extraction or response generation when the project policy allows it;
reserve the stronger model for complex reasoning.

## Debug checklist

When behavior is unexpected:

1. Check that `AGENT` and `GOAL` are present and correctly indented.
2. Validate the package and inspect compiler diagnostics.
3. Confirm the goal, instructions, and limitations do not conflict.
4. Inspect tool selection and results in the execution trace.
5. Check `EXECUTION` limits when the agent loops, times out, or returns an
   unexpectedly short response.
