---
name: arch-experience-contract
description: Define what a good agent experience means before or early in a build on the Kore.ai Agent Platform. Use when starting a project, when asked for success criteria, latency or cost targets, model-call or token budgets, or when "optimize" has no measurable target. Produces a per-intent experience contract with success definitions, latency targets, model-call budgets and the deterministic steps, marked measured or inferred.
---

# Experience contract

Every later decision (topology, model, history mode, guardrails, voice settings) is checked against
this contract. Without it, nobody can say when the agent is good enough, and optimization has no
target. Write it in week one.

## Gather the evidence

- Read the project's agents and topology: `debug_get_flow_graph` (after `debug_load_agent`, or with `projectId`, `sessionId` and `source: "historical"`) (Mermaid, `includeAppGraph`),
  `platform_package_model` on the exported package (`path`).
- If sessions exist, use `debug_session_history` (`action: "list"`, `range: "7d"`) and
  `platform_analytics` to estimate intent share, turns per conversation, model calls per turn and
  latency. Analyze a few representative sessions with `debug_analyze_session`.
- Ask the developer for what the data cannot show: business success criteria, compliance steps,
  channels and languages.

## Output contract

1. **Story:** two or three sentences on what the agent is for and who uses it.
2. **Contract table,** one row per top intent (the 5–10 that make up most traffic):
   intent | share of sessions | example caller utterance | success definition you can check in a
   transcript or tool call | channel | p50 and p95 first-response latency target | model calls per
   turn budget | input tokens per turn budget | steps that must be deterministic | failure behavior.
3. **Measured or inferred:** mark every number. Measured numbers cite the sessions; inferred numbers
   say what they were inferred from.
4. **Gap list:** the intents furthest from target today, each with one example session ID.
5. **Evidence plan:** which eval set will prove each line (see `arch-eval-catalog`).
6. **Glossary** for any platform term used.
7. **Confidence and gaps:** which numbers rest on few sessions, and what data was not available.

Do not shorten the table. If long, write it to `docs/experience-contract.md` in the project folder.

## Why budgets drive design

Every design choice spends from the same budget. Show the developer the cost before they commit:

| Design choice                                         | What it usually adds per turn                           |
| ----------------------------------------------------- | ------------------------------------------------------- |
| A routing or supervisor agent in front of specialists | One model call on every routed turn                     |
| Handoff to a reasoning specialist                     | The specialist's own call, plus the history it receives |
| `history: full` on handoffs (the default)             | Input tokens that grow every turn                       |
| Model-judged guardrails or classifiers                | One call each, unless replaced by rules                 |
| A generated answer where a template would do          | A model call, output tokens and speech time             |
| A tool call followed by a call to phrase the result   | A second model call per tool turn                       |

## Rules

- Targets are starting points to agree with the customer, not platform guarantees.
- Authentication, money movement and compliance wording are deterministic unless there is a strong
  reason otherwise.
- Voice turns need their own latency line: first audio, not first token.
- Revisit the contract when the intent mix changes; keep it in the project folder next to the ABL.
