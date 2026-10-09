---
name: arch-topology
description: Choose or review the agent topology of a Kore.ai Agent Platform project, including single agent vs scripted FLOW vs supervisor with specialists, handoff, delegate, fan-out, escalation, handoff context and multi-intent requests. Use when asked "which topology", "should I add an agent or specialist", "why does it route wrongly", "how many model calls does routing cost", or to explain how a multi-agent project works.
---

# Topology

Start with the smallest topology that meets the experience contract. Add an agent only when evals
show the single agent failing for a reason another agent fixes. Every extra agent adds a routing
decision, a context boundary and usually a model call.

## Gather the evidence

- `debug_get_flow_graph` (after `debug_load_agent`, or with `projectId`, `sessionId` and `source: "historical"`) with `format: "mermaid"` and `includeAppGraph: true`.
- `platform_package_model` on the exported package (`path` or `files`) for compiler-visible agents, tools, handoffs and diagnostics.
- `debug_explain_decision` with `type: "handoff"` on real sessions to see routing decisions (pass
  `projectId` and `source: "historical"` for sessions that are not live).
- `platform_eval_runs` with `compare` to measure a topology change instead of arguing it.

## Orienting on an existing project

When the developer asks "explain this project to me", produce before any recommendation:

1. The full app flow graph as Mermaid.
2. An agent table: agent | what it owns, in plain words | how it is reached (deterministic `WHEN` or
   model routing) | tools | handoffs out with `pass`, `summary` and `history` | model calls on a
   typical turn.
3. One top-intent conversation, turn by turn, from a real or scripted session.
4. The three design choices most likely to cost latency or tokens, with evidence.

## Decision guide

Ask in order and stop at the first yes:

1. **Is it a fixed procedure** (authentication, payments, compliance wording)? Use a scripted FLOW
   with GATHER steps, and DIGRESSIONS for side questions.
2. **Can one agent pass the evals** (one persona, one domain, a tool list it handles)? Use a single
   reasoning agent with tools. This is the default: cheapest and easiest to debug.
3. **Does the user talk to distinct domains** with different tools, policies or owners? Use a
   supervisor with specialists through HANDOFF, passing fields and small history.
4. Otherwise, **fix the agent before adding agents**: tighter instructions, fewer tools, DELEGATE
   for silent look-ups.

Delegate, fan-out and escalate are added inside whichever topology you land on.

| Pattern                  | ABL construct                         | Use when                            | Cost to watch                                                |
| ------------------------ | ------------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| Single reasoning agent   | One agent with `TOOLS`                | One persona and domain              | Prompt and tool list growing until tool choice gets confused |
| Scripted flow            | `FLOW` with `GATHER`                  | Fixed procedures, high volume       | Rigid if users wander                                        |
| Supervisor + specialists | `HANDOFF`                             | Distinct domains the user talks to  | One routing call per routed turn; context at the boundary    |
| Returnable handoff       | `HANDOFF` with return and `ON_RETURN` | Do something, then come back        | Parent waits; design the return explicitly                   |
| Delegate                 | `DELEGATE` with `INPUT`, `RETURNS`    | A sub-task with no user interaction | A full sub-agent turn; use a tool if one API call would do   |
| Fan-out                  | Supervisor over independent sub-tasks | Independent look-ups combined       | Parallel calls still cost tokens                             |
| Escalate                 | `ESCALATE`                            | Human takeover                      | Triggers, so it is not the default failure path              |

Handoff context rules (lowercase keys, history defaults) are in
[references/handoff-context.md](references/handoff-context.md). Read them before reviewing any
handoff; several mistakes parse without error and are silently ignored.

## Anti-patterns

- A supervisor in front of a single specialist.
- One specialist per intent instead of per domain, so every turn routes.
- More than two routing levels; each hop is at least one full model call.
- `role: router` where `role: orchestrator` could answer greetings and trivial asks itself.
- Specialists handing back to the supervisor for simple follow-ups.
- Model routing where a deterministic `WHEN` on stored state would do.
- Splitting a long prompt into agents when the fix is tighter instructions or fewer tools.
- Managing a multi-intent queue in prompt text.

## Field patterns

These are illustrative patterns from production projects; the numbers are directional, not
benchmarks. Measure your own project with an eval compare.

- On a large voice router, routing straight to the leaf agent was more accurate than a
  supervisor → domain → leaf chain: the domain hop added a less accurate decision the leaf could
  not recover from.
- On the same router, loading the caller's profile at session start improved routing more than
  any rewording of intent descriptions. Add context before adding prompt text.
- Money-out and safety routes made deterministic rules improved sharply; the same guidance as a
  model hint changed nothing. Safety intents need deterministic overlays.
- Leaf agents with two-way discriminators against their look-alike neighbors routed far better than
  leaves without them; leaves with no route were silently absorbed by a neighbor.
- Before conditioning routing or escalation on `intent.confidence` in a
  reasoning supervisor, check with `debug_explain_decision` that they are populated for your agent.
- A supervisor → supervisor → specialist voice chain produced long silences on a single turn.

## Multi-intent

When a customer asks for two things at once, the risks are an agent switch while a question is
unanswered, a "yes" accepting a request that was not offered, and sensitive values crossing agents.

- Do not hand-roll a queue of requests in prompt text.
- Mark sensitive GATHER fields `SENSITIVE: true` with a `SENSITIVE_DISPLAY` mode (and a `PII_TYPE` so the project's rules for that type apply).
- Decide with the customer what should happen for dependent requests ("check my balance, then
  transfer if there's enough"), contradictory requests, cancelling everything, and changing one's
  mind after accepting, then test each.
- Choose the project's multi-intent `completion_signal` deliberately (a project setting; a value
  written in ABL is ignored): the default `delivered_response` runs a small check after each queued
  reply; `task_signal` lets the agent signal completion itself.

## Output contract

1. **Recommendation** in two plain sentences.
2. **Comparison table:** option | model-call sequence per turn for the top intents (for example
   route → extract → reply) | expected first-response latency | context passed at each boundary |
   main failure risk.
3. **Walkthrough** of one top-intent conversation in the recommended option, turn by turn: caller
   said | owning agent and step | what each model call decides | state set.
4. **Handoff table** for existing projects: from → to | trigger (deterministic or model) | `pass`
   fields | `summary` | `history` | calls on a routed turn | merge candidate?
5. **What would change the recommendation,** and the eval scenarios that would prove it.
6. **Glossary** with an example from this project for each term.
7. **Confidence and gaps:** what you measured, what you inferred, and what was not available.

Do not shorten the tables.
