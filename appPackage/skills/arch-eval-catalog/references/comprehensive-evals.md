# Comprehensive evals

## What the platform runs

An eval set is a matrix: personas × scenarios × evaluators × variants (repeats, 3 by default), run
against a target (`working_copy`, an environment, a deployment, or one agent version). A pinned
agent version freezes only that agent; everything else comes from the working copy.

| Building block | What to set                                                                                                                                                                                                                                                                                                                                                  |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Persona        | Communication style, domain knowledge, behavior traits, goals, constraints, session variables; an adversarial type (prompt injection, social engineering, off-topic, abusive, edge case); a voice profile for voice                                                                                                                                          |
| Scenario       | Initial message and session state, entry agent, expected outcome, max turns, difficulty, tags; trajectory expectations (`expectedMilestones`, `agentPath`, `requiredToolOrder`, `maxToolCalls`); contracts (`responseOutcomeContract`, `requiredContextFieldContracts`, semantic contracts); `turnLatencyBudgetMs`; `toolFixtures`; modality (text or voice) |
| Evaluator      | `llm_judge` (rubric 1–5 or pass/fail), `code_scorer`, `trajectory` (milestone completion, handoff correctness, path efficiency, tool sequence, groundedness = share of successful tool calls), `human_review`                                                                                                                                                |
| Run            | Target, variants, `runMode: full` or `evaluator_only` (re-judge an earlier run)                                                                                                                                                                                                                                                                              |

A run reports average score, pass rate, scores by evaluator, coverage, errors, cost, standard
deviation and a 95% confidence interval; each conversation records turns, duration, tokens, cost,
the agent path taken and tool calls.

## Coverage, axis by axis

| Axis                  | Comprehensive means                                                                                                                        |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Intents               | Every top intent from the experience contract                                                                                              |
| Scenario families     | Every family in `scenario-families.md`, including failure paths and multi-intent                                                           |
| Personas              | Cooperative, confused, impatient, expert, and every adversarial type                                                                       |
| Difficulty            | Easy, typical and hard cases per intent                                                                                                    |
| Modality and language | Text and voice; each supported language (text scenarios carry language through persona session variables; voice through the voice profile) |
| Configuration         | The working copy and the version you will deploy                                                                                           |
| Repeats               | At least 3 variants per cell; look at the spread                                                                                           |

## Deterministic checks first, judges second

1. Trajectory expectations and contracts decide pass or fail for what must be exact: the path, the
   milestones, the order of tool calls, the values written.
2. Latency budgets assert speed where the contract requires it.
3. Judges score what cannot be checked exactly: helpfulness, tone, safety wording.

## Pitfalls that make scores misleading

- A trajectory evaluator on a scenario with no trajectory expectations cannot score the path. Give
  every scenario at least one expectation.
- A tool-success check passes a conversation that made no tool calls; pair it with
  `requiredToolOrder` or `maxToolCalls`.
- A tool-sequence metric counts calls; order is checked by `requiredToolOrder`. Tool arguments are
  not asserted, so check the values they produce with contracts.
- A safety judge can mark a conversation that ran out of turns as unsafe because the task did not
  finish. Write safety rubrics that rate only what was said, and review max-turn conversations
  separately.
- A judge on the same model as the agent shares its blind spots. Set the judge model explicitly.
- Position-swap judging is on by default for evaluators and doubles judge calls; budget for it.
- A `human_review` evaluator scores the conversation 0 and queues it for review; a judge below its
  human-review confidence threshold keeps its score but is flagged. A run with pending reviews stays provisional and cannot
  be compared or used as a baseline, so keep human review out of gated and baseline sets.
- Pass@k figures are estimated from the overall pass rate, not counted per scenario. For flaky
  scenarios, count per-scenario passes across variants yourself.
- A run's cost policy is not a hard stop; size large runs before starting them.
- A score from a set with no wired scenarios is meaningless; confirm the set is runnable before
  calling a run a baseline.

## In CI

Gate releases on an eval set with a pass threshold and a baseline run to compare against; the gate
fails closed and fails on regression against the baseline. Use the same pinned target and models
in every gated run.

## What you still do outside the platform

- Turn diagnosed production sessions into scenarios by hand.
- Compute model calls per turn and tokens per turn from traces; no evaluator measures them.
- Keep hand labels to check judge agreement outside the platform.
- Stratify smoke sets by hand (one scenario per intent and family) so a small run is
  representative.
