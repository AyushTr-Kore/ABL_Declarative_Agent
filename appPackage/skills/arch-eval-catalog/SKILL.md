---
name: arch-eval-catalog
description: Build and run evaluations for agents on the Kore.ai Agent Platform, including scenario catalogs, personas, evaluators, eval sets, eval runs and run-to-run comparison. Use when asked how to test an agent, to create or review test cases or evals, to measure whether a change helped, or when a project has no comprehensive test coverage. Produces a scenario catalog grouped by family and an eval compare you can trust.
---

# Eval catalog

You cannot optimize what you cannot re-test. An eval set lets a developer cut a model call or shorten
a prompt and know the same day that nothing broke. Coverage means conversation scenarios with
evidence, not a test count.

## Three layers of testing

1. **Offline, before deploying:** keep the exported package in version control and run
   `debug_lint_abl` and `platform_validate_package` on it; `debug_why_transcript_failed` maps a bad
   transcript to ABL file and line. Point tools at mock backends when the real ones are not ready,
   so tests do not wait on another team.
2. **Live scripted conversations:** `debug_load_agent` and `debug_send_message` play multi-turn
   conversations against the working copy; check `debug_get_current_state` after each turn.
3. **Platform evals:** `platform_eval_personas`, `platform_eval_scenarios`,
   `platform_eval_evaluators`, `platform_eval_sets` and `platform_eval_runs`. Target the
   `working_copy`, an environment (`dev`, `staging` or `production`), a deployment or an exact
   agent version. Use `preflight` before a
   big run, `quick` for a smoke check, `compare` between runs and `heatmap` to see which scenarios
   regress.

4. **Repair loops:** `platform_arch_auto_loop` runs governed repair of failing scenarios. Read each
   proposal and the gate results that accepted or blocked it (`list_proposals`) before applying,
   and re-run the eval set afterwards.

## Build the catalog

1. Read the experience contract (`arch-experience-contract`) for the top intents and success
   definitions.
2. Draft with `platform_eval_personas` and `platform_eval_scenarios` using `generate` across every
   generation category (happy path, edge case, error handling, multi-turn, handoff, adversarial).
   Review before accepting; generated catalogs miss hard cases.
3. Cover every family in [references/scenario-families.md](references/scenario-families.md), and
   include the adversarial persona templates (prompt injector, social engineer, off-topic,
   abusive, edge-case explorer).
4. Make every scenario checkable: an `expectedOutcome`, `maxTurns`, and at least one deterministic
   expectation (`expectedMilestones`, `agentPath`, `requiredToolOrder` or `maxToolCalls`). Use
   `responseOutcomeContract` or `requiredContextFieldContracts` for values that must be right, and
   `toolFixtures` to make tool outcomes repeatable.
5. Set `turnLatencyBudgetMs` yourself where latency matters; only an explicit budget is asserted.
6. Turn every diagnosed production failure into a scenario before fixing it.
7. Run each scenario at least three times (`variants`) and read the spread, not just the average.

What "comprehensive" means, the evaluator types, and the pitfalls that make scores misleading are
in [references/comprehensive-evals.md](references/comprehensive-evals.md).

## Calibrate before you trust a score

- Read 5–10 judged conversations by hand before using a score as a gate.
- Re-judge a fixed run with `runMode: evaluator_only` when you change an evaluator, and compare the
  judge with your own labels.
- Set each evaluator's judge model explicitly, and use a different model from the agent's.
- Keep `human_review` evaluators out of gated and baseline sets: pending reviews leave a run
  provisional, and a provisional run cannot be compared.
- Compare like for like: same scenarios, same target, same agent, persona and judge models. A
  compare flags changed sets and variants, but not a changed target or model, so pin those yourself.
- Run `preflight` before a large run; it checks the run's infrastructure (credentials, models,
  runtime reachability), not your scenarios.

## The rule

No prompt, topology, model or voice change is done until the eval set has run against it and been
compared with the previous run. Put the compare in the change note.

## Output contract

Before creating anything, show:

1. **Catalog table:** family | scenario | persona | a 2–4 line example exchange | expected outcome
   you could check in a transcript | evaluator that checks it | efficiency assertion, if any.
2. **Families not covered,** and why.
3. **Decisions needed** from the developer or customer (undefined behaviors).

After a run, show:

1. **Verdict** in one sentence: improvement, regression or no change, and why.
2. **Metric table:** metric | baseline | new | change (success rate, model calls per turn, p50 and
   p95 latency, tokens per conversation).
3. **Changed scenarios:** scenario | before | after | turn where it diverged | why, with quoted
   evidence.
4. **What the run did not test.**
5. **Glossary:** every evaluator, metric and platform term used, one line each, with an example
   from this project.

Do not call a change an improvement unless success held. Do not summarize the scenario list; write
long catalogs to `evals/catalog.md`. Create nothing until the developer approves the catalog.
