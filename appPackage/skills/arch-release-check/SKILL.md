---
name: arch-release-check
description: Check that a Kore.ai Agent Platform project is ready to publish or deploy. Use before a release, publish, deployment or customer demo, or when asked "is it ready to ship". Runs ABL lint and package validation, compares the eval set with the last release, and walks a pre-ship checklist for context, secrets, handoffs, tools, voice and speech-to-speech.
---

# Release check

## Steps

1. **Export and lint.** Export the project with `platform_import_export`, save the files to the project folder, then run `debug_lint_abl` and
   `platform_validate_package` on the package. Every finding goes in the report; none are merged.
2. **Eval compare.** Run the eval set against the release target with `platform_eval_runs`
   (`target` set to the `dev`, `staging` or `production` environment, a deployment, or the agent
   version you will ship) and `compare` it with the last release's run.
3. **Budget check.** For the top intents, confirm model calls per turn and latency are within the
   experience contract, using sessions from the eval run.
4. **Context check.** Read the full model requests of a 20-turn test conversation, from "Export
   entire trace" on the session page (see `arch-context-budget`), for growth, truncation markers and
   secrets.
5. **Checklist.** Walk the list below and mark each item pass, fail or not applicable with evidence.

## Pre-ship checklist

- [ ] Lint and package validation are clean.
- [ ] Eval set ran on the release target and was compared with the last release; success held.
- [ ] Model calls per turn and latency are within the experience contract for the top intents.
- [ ] Every defect fixed since the last release has a scenario in the eval set.
- [ ] Each agent declares `BEHAVIOR: role:`, or you have checked its `## Current Context` in a
      trace; no `{{context}}` and no unfilled `{{placeholder}}` in a test trace.
- [ ] No `config.*` or `env.*` in GOAL, PERSONA, LIMITATIONS, SYSTEM_PROMPT or RESPOND; credentials
      are in auth profiles; a trace search for secret values finds nothing.
- [ ] Handoff `CONTEXT:` keys are lowercase and `history` uses a supported form; delegates get
      everything through `INPUT` or `PURPOSE`.
- [ ] Every tool with a large or nested result has `MODEL_CONTEXT`, tested against real, empty and
      error responses; rich responses carry a response `MODEL_CONTEXT`.
- [ ] Sensitive GATHER fields are marked `SENSITIVE: true` with a `SENSITIVE_DISPLAY` mode (and a `PII_TYPE` so the project's rules
      for that type apply).
- [ ] Every tool declares `effect`; irreversible writes confirm and freeze key values; each tool the
      caller depends on has an authored `on_error` message (see `arch-tool-design`).
- [ ] Workflow tools set `mode` and `timeout_ms` explicitly, and the workflow version the release
      needs is activated (see `arch-workflow-design`).
- [ ] Knowledge-base documents are indexed and `max_search_tokens` is set on every knowledge-base
      tool (see `arch-knowledge-search`).
- [ ] `INTENTS` are declared with descriptions.
- [ ] After an import: knowledge bases are populated, and model bindings, auth profiles and config
      variables are set.
- [ ] Final tests ran in a new conversation on the version that will be deployed.
- [ ] Voice: ASR and latency sets re-run if the provider, timeouts, profile or gather fields changed;
      runtime voice settings (fillers, VAD) re-checked after deploy; a fast-tier model is available.
- [ ] Speech-to-speech: the effective orchestration mode verified in traces; every tool exit path
      (success, error, timeout, empty) tested; the welcome is protected from barge-in.

## Output contract

1. **Verdict** in one sentence: ready, ready with risks, or not ready, and the one reason that
   matters most.
2. **Lint table,** grouped by agent: severity | file:line | what is wrong in plain words | what a
   caller would experience | the fix as an ABL snippet.
3. **Eval compare table:** metric | last release | this release | change, plus every scenario whose
   result changed and why.
4. **Checklist** with pass, fail or not applicable and the evidence for each item.
5. **What could not be checked,** and why.
6. **Glossary:** every platform term used, one line each, with an example from this project.

List every finding; do not merge or summarize them.
