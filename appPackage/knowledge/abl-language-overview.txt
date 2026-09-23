# ABL language overview

> Curated from the Agent Platform ABL reference. The full source is maintained
> in `apps/docs-internal/content/abl-reference/language-overview.mdx`.

Agent Blueprint Language (ABL) is a schema-driven language for defining agent
identity, goals, tools, data collection, flows, memory, guardrails, and
multi-agent behavior. Definitions compile into platform artifacts.

## Minimal structure

An ABL document is a plain-text file made of top-level sections. Every agent
requires `AGENT:` and `GOAL:`; other sections are optional.

```abl
AGENT: Customer_Support

GOAL: |
  Help customers resolve billing questions.

PERSONA: |
  Friendly, patient support representative.

TOOLS:
  lookup_account(account_id: string) -> {name: string, balance: number}
    description: "Retrieve account details"

GATHER:
  account_id:
    prompt: "What is your account number?"
    type: string
    required: true
```

Common file extensions:

| Extension | Purpose |
| --- | --- |
| `.agent.abl` | Agent definition |
| `.tools.abl` | Reusable tool library |
| `.agent.yaml` | YAML representation of an agent |

## Top-level sections

Common sections include:

- `AGENT`, `VERSION`, `DESCRIPTION`, and `LANGUAGE` for metadata;
- `GOAL`, `PERSONA`, `LIMITATIONS`, `IDENTITY`, and `INSTRUCTIONS` for agent
  behavior;
- `TOOLS`, `GATHER`, and `FLOW` for capabilities and execution;
- `MEMORY`, `CONSTRAINTS`, and `GUARDRAILS` for state and safety;
- `DELEGATE`, `HANDOFF`, `ESCALATE`, and `COMPLETE` for collaboration and
  lifecycle;
- `ON_ERROR`, `ON_START`, and `HOOKS` for lifecycle handling; and
- `NLU`, `MULTI_INTENT`, `LOOKUP_TABLES`, `TEMPLATES`, and `ATTACHMENTS` for
  advanced behavior.

## Syntax rules

- Use uppercase section keywords followed by a colon in ABL files.
- Use consistent spaces for indentation; nested properties must be indented.
- Use `#` for comments. Inline comments after an ABL value are not supported.
- Use quoted strings for simple values and `|` blocks for multiline values.
- Use YAML-style `- item` lists with indentation.
- Use `{{variable}}` for runtime interpolation.
- Do not add a global `MODE:` keyword. Agents use reasoning by default; each
  `FLOW` step can set `REASONING: true` or `REASONING: false`.

```abl
FLOW:
  verify_request:
    REASONING: false
    RESPOND: "Please confirm your request."
    ON_INPUT:
      - IF: input == "yes"
        THEN: process_request
      - ELSE:
        RESPOND: "Please answer yes or no."
        THEN: verify_request
```

## YAML format

YAML agent files use `.agent.yaml` and lowercase keys. ABL and YAML compile to
the same intermediate representation when they express the same definition.

```yaml
agent: Customer_Support
goal: |
  Help customers resolve billing questions.
persona: "Friendly support representative"
```

## Authoring guidance

- Make `GOAL` specific and measurable where possible.
- Use `PERSONA` for communication style, not business rules.
- Use `LIMITATIONS` for prompt-level boundaries and `CONSTRAINTS` for
  deterministic runtime checks.
- Keep tools typed and give each one an actionable description.
- Validate the package after structural changes and inspect trace evidence after
  runtime changes.
