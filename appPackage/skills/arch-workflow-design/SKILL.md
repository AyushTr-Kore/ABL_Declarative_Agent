---
name: arch-workflow-design
description: Design, publish, call and debug workflows on the Kore.ai Agent Platform, including workflow-backed agent tools, sync vs async calls, timeouts, input and output mapping, versions (publish and activate), executions, step approvals and human tasks, triggers, retries and failure branches. Use when an agent calls a workflow, a workflow tool times out or returns nothing, the wrong version runs, or a run is stuck waiting.
---

# Workflow design

A workflow is a durable, step-by-step run: each step runs once, retries are per step, and the run
keeps its state while it waits. Agents call workflows through workflow tools. Most problems come
from the call contract (sync vs async, timeout, inputs) or the version that actually runs.

## Calling a workflow from an agent

| Setting         | Rule                                                                                                                                                                                                                                |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mode            | `sync` when the answer is needed this turn; `async` for long runs. Set `mode` (ABL) or `toolMode` (`create_tool`) explicitly: the agent file defaults to sync, while `platform_workflows` `create_tool` defaults to async           |
| Timeout         | Set `timeout_ms` (ABL) or `timeoutMs` (`create_tool`) to what the workflow really needs. Sync calls default to 60 s in the runtime (Studio offers 15 s); keep voice turns short and use async plus a status check for anything slow |
| Inputs          | Declare typed, described inputs (the workflow's input schema or start node inputs) and a matching `param_mapping`. `create_tool` creates a tool with no arguments unless you pass parameters                                        |
| Effect          | Workflow tools are treated as side-effecting unless you declare `effect: read_only`                                                                                                                                                 |
| Unknown outcome | A sync call that cannot read the outcome fails with `TOOL_WORKFLOW_OUTCOME_UNKNOWN`. Never re-fire it; check the run with `get_execution` or the auto-wired `check_workflow_status` first                                           |
| Async results   | An async call returns an execution ID and `running`; the agent checks status with `check_workflow_status`                                                                                                                           |

## Versions

- `publish` freezes a version, but it is not live until `activate_version`. Then deploy the agent.
- The deployment chooses the workflow version, not the tool binding.
- A manual run uses an explicit version, then the active version, then the draft.
- Check what is live with `list_versions`, and which version a run used in its execution record.

## Authoring

- Read start inputs as `{{context.steps.start.output.<name>}}`; the end node's `outputMapping`
  defines what the agent receives; a function node returns the properties it assigns on `context`.
- Wire failure branches and per-step retries; there is no workflow-level retry.
- Every human or data-entry node needs a resolver path (`list_human_tasks`, `resolve_human_task`),
  and approval steps need `approve_step`; otherwise the run waits until it times out.
- Triggers: webhook triggers are managed for you; you create cron and event triggers.
- Pass only the values a workflow needs as inputs, and keep credentials in auth profiles. In
  release 1.7.3, workflow tools receive the project's whole config and environment namespaces,
  secrets included.
- While a backend is not ready, stub it with a function node that returns recorded responses, and
  remove the stub before release.

## Debug a workflow call

1. In the agent session: the workflow tool call event shows inputs, mode, duration and the result
   or error (`debug_traces`, `debug_get_errors`).
2. In the workflow: `list_executions` and `get_execution` show the run, its version, each step's
   status and where it waits or failed. Workflow step traces live in the workflow engine, not in
   the agent's trace.
3. Name the layer: the call contract (mode, timeout, inputs), the version, a step, or the backend.

## Rules

1. Choose sync or async per tool, and set it explicitly.
2. Declare typed inputs and a matching parameter mapping.
3. Publish, activate, then deploy; verify with `list_versions`.
4. Never re-fire after an unknown outcome; check the execution first.
5. Wire failure branches and per-step retries.
6. Give every human step a resolver path.
7. Mark read-only workflow tools `effect: read_only`.
8. Pass only needed inputs; keep credentials in auth profiles.
9. Keep sync workflow calls well inside the voice latency budget.

## Output contract

1. **Story:** what the workflow should do for the caller and where it went wrong, in plain words.
2. **Call table:** tool | workflow and version | mode | timeout | inputs mapped | effect | result
   or error.
3. **Execution timeline** for one run: step | status | duration | input and output summary |
   error.
4. **Fix** at the right layer, and the scenarios to test (success, slow, failure, waiting human
   step, duplicate call).
5. **Glossary** and **confidence and gaps.**
