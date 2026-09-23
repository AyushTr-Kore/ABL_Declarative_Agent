# ABL repair and evaluation loop

> Embedded from the Arch MCP debug fallback documentation. Use this as a
> workflow guide; use MCP evidence for the actual project and session state.

Arch's MCP tools support iterative ABL repair, not only import
troubleshooting.

## Suggested workflow

1. `platform_package_model`: inspect what the compiler sees.
2. `debug_lint_abl`: find design risks such as empty `RESPOND`, empty
   finalization steps, undeclared handoff-condition variables, side-effect tool
   chains, and tool-call plus customer-text reasoning risks.
3. `debug_why_transcript_failed` (or the compatibility alias
   `debug_diagnose_transcript`): correlate transcript symptoms to ABL
   file/line causes, including `finalize -> COMPLETE -> RESPOND: ""`.
4. `platform_validate_package`: run platform validation and import preview when
   `projectId` is available.
5. Use `platform_eval_personas`, `platform_eval_scenarios`,
   `platform_eval_evaluators`, `platform_eval_sets`, and
   `platform_eval_runs` for evaluation workflows.
6. Use `platform_eval_runs` with action `cases` to drill from a failing
   heatmap cell into diagnostic transcript, conversation, trace events, tool
   calls, trajectory, and evaluator scores.
7. Patch the local package and repeat until validation and evaluations agree.

The key debugging question is: **what does the compiler see?**

## Package-model evidence

Use `platform_package_model` for:

- agents;
- tools;
- handoffs and delegates;
- memory variables;
- behavior profile references;
- compiled flow steps;
- constraint observability, including raw constraint bullets, parsed
  constraint AST, inert parser warnings, compiled IR constraints, and runtime
  check phases;
- unresolved references; and
- compiler diagnostics.

## Constraint contract facts

- `rawConstraints` counts authored `CONSTRAINTS` bullets.
- `parsedConstraints` counts only `REQUIRE`, `WARN`, `LIMIT`, and `RESTRICT`
  entries parsed into the AST.
- `inertConstraintWarnings` identifies plain `CONSTRAINTS` bullets ignored by
  the constraints compiler.
- `compiledRuntimeConstraints` counts entries that reached
  `ir.constraints.constraints`.
- `phaseSemantics.labelsOnly` means `always:` and named phases are readability
  labels today, not lifecycle hooks.
- `runtimeChecks` shows semantic runtime surfaces: state-context checks,
  `before_tool_call`, `before_response`, and `after_tool_result` checks.

## Tool contract facts

- `side_effects` plus `confirmation` controls the user approval flow; it is not
  an authorization policy.
- `identity_tier_required` is the current identity gate. Generic tool
  `requires` and `effects` authorization policy is not modeled here.
