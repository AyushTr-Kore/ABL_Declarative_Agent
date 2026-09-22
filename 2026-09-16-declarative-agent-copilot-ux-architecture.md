# ABL Engineering Assistant: Declarative Agent Copilot UX Architecture

**Date**: 2026-09-16  
**Status**: IMPLEMENTED BASELINE; tenant UX and card verification pending  
**Scope**: Microsoft 365 Copilot Declarative Agent consuming ABL Remote MCP  
**Related feature**: [ABL Remote MCP Server over Streamable HTTP](../features/remote-mcp-server.md)  
**Package**: `AgentsToolkitProjects/Agent Builder`

This document records the recommended architecture and implementation changes
for a better Declarative Agent experience over ABL. It is a UX and integration
document, not a replacement for the Remote MCP protocol or Auth/OIDC contracts.

## Decision summary

Keep the existing pro-code architecture:

```text
Microsoft 365 Copilot
        |
        v
Declarative Agent
  instructions, starters, knowledge, plugin package
        |
        v
Remote MCP HTTPS resource
  Auth/OIDC -> resource/profile -> tools/list -> tools/call
        |
        v
Per-call policy
  scope -> ABL permission -> target -> confirmation -> idempotency
        |
        v
Owner gateway
  Runtime / Studio / Admin
```

The default Copilot should guide users through small, read-only, task-oriented
requests. This package keeps the complete declared Arch catalog available so
users do not lose capability; concise function descriptions and instructions
guide selection, while server-side authorization remains the final boundary.
Curated read-only, diagnostics, builder, and admin profiles remain an optional
future optimization for discoverability and latency.

Use:

- Agents Toolkit for the pro-code Declarative Agent, Remote MCP package,
  source control, Adaptive Card templates, and CI/CD.
- Copilot Studio flows for approval-heavy or long-running workflows.
- Microsoft 365 knowledge sources for stable documentation and runbooks.
- ABL rich content for Teams, Web, and other ABL-owned channel responses.
- A custom-engine agent only for proactive behavior, group collaboration,
  external channels, custom orchestration, or a custom model.

Microsoft describes Declarative Agents as focused extensions using instructions,
knowledge, and actions, while custom-engine agents are intended for more
complex orchestration and proactive scenarios. See the [Microsoft agent
architecture guidance](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/agents-overview).

## Evidence from the current implementation

### Remote MCP

The current Remote MCP source provides a strong security boundary:

- `/mcp/arch` is the global resource and excludes the local-only
  `platform_connect` tool.
- `/mcp/{connectionId}` is connection-scoped and exposes only `chat.execute`.
- `tools/list` returns the selected profile, while `tools/call` rechecks
  authorization.
- Calls are checked against MCP scopes, current ABL permissions, resource
  target, confirmation, and idempotency policy.
- Owner composition, user-bound delegation, deadlines, result bounds, audit,
  and disconnect recovery are explicit boundaries.
- Readiness fails closed when the owner composition is not available.

The source-backed contract is documented in the [Remote MCP README](../../apps/remote-mcp-server/README.md), the [current engineering guide](../../apps/remote-mcp-server/agent-instructions/000-current.md), the [tool profile](../../apps/remote-mcp-server/src/catalog/tool-profile.ts), and the [operation policy](../../apps/remote-mcp-server/src/execution/operation-policy.ts).

### Current Declarative Agent package

The production package currently has these important characteristics:

- `ai-plugin.json` declares the complete function set.
- The package declares 42 model-facing functions and uses dynamic Remote MCP
  discovery; it has no checked-in static `mcp-tools-1.json` catalog.
- `run_for_functions` binds the runtime to those functions.
- Credential and bootstrap properties are excluded from the model-visible
  package.
- The package points to the environment HTTPS `/mcp/arch` resource, while the
  Remote MCP service also supports `/mcp/{connectionId}`.
- Host-managed login selects the active workspace; instructions resolve only
  project scope and retain verified context within the conversation.

The remaining integration risks are:

1. A complete platform catalog makes model selection and user interaction less
   predictable than a curated task catalog.
2. The checked-in package must be regenerated whenever the selected Remote MCP
   profile changes.
3. Adaptive Card rendering must be validated in the target Copilot host; the
   generic MCP catalog remains presentation-neutral.

The static package must be generated from the selected Remote MCP profile, not
from the unfiltered canonical registry.

## Target exposure model

The existing `global-arch` profile should remain available for internal Arch
compatibility. Add a model-facing profile or allowlist for the Copilot.

The current package does not split these profiles yet. All 42 declared
functions remain available so a user can complete an uncommon task; the agent
instructions, descriptions, and conversation starters provide task-oriented
discovery without removing capability. Introduce a profile only when model
selection, latency, or permission UX shows a measured need.

| Profile | Use | Exposure | Default? |
| --- | --- | --- | --- |
| `arch-copilot-readonly` | Explore and inspect | Workspace/project/agent discovery, inspection, validation, documentation, status | Yes |
| `arch-copilot-diagnostics` | Troubleshoot | Debug sessions, traces, errors, spans, and analysis | No; activate only for diagnosis |
| `arch-copilot-builder` | Repair or author | Project builder, workflow, evaluation, and controlled repair operations | No; confirmation required |
| `arch-copilot-admin` | Administration | Credentials, integrations, MCP servers, deployments, channels, destructive operations | Separate privileged experience |
| `project-chat` | Runtime conversation | `chat.execute` on the bound connection | Only for a connection agent |

The first implementation should add only the read-only profile and keep the
other profiles as explicit follow-up work. Do not create new APIs merely to
create more tool names. Use an existing handler and generated model-facing
projections where possible.

### Action-aware exposure

Several Arch tools use an `action` discriminator containing read, write, and
destructive operations. A tool-name allowlist alone is insufficient when one
tool exposes all of those actions.

For the default profile, use one of these approaches, in order:

1. Generate an action-filtered read-only schema from the existing contract.
2. If the MCP contract cannot safely express that projection, expose separate
   model-facing read and mutation functions backed by the same owner handler.
3. Keep server-side action policy as the final authority in every case.

Do not rely on the model to avoid a destructive enum value just because the
instructions discourage it.

## Required change list

### P0 — Make the contract safe and deterministic

1. Add the `arch-copilot-readonly` Remote MCP profile.
2. Generate the Declarative Agent catalog from that profile.
3. Remove `platform_connect` from every remote Copilot package.
4. Remove token, password, device-code, connection URL, and raw credential
   properties from model-visible schemas.
5. Change the package from local ngrok to an environment-specific HTTPS resource
   before integration testing.
6. Make `functions`, `mcp_tool_description`, and `run_for_functions` exact
   projections of one selected profile.
7. Add a build/CI drift check that fails when the generated catalog and Remote
   MCP profile differ.
8. Keep authorization, tenant isolation, target validation, confirmation, and
   idempotency in Remote MCP. Manifest filtering is not authorization.
9. Make readiness profile-aware when a profile does not require every owner
   service. The currently composed service must not advertise a profile whose
   owner dependency is absent.

### P1 — Improve model selection and result quality

10. Rewrite function descriptions to include purpose, scope, read/write effect,
    confirmation behavior, result state, and next action.
11. Add `outputSchema` to the high-value read-only tools first.
12. Standardize result envelopes with success, data, error, sources/links, and
    operation status where applicable.
13. Add action-aware negative and ambiguity examples to tool-selection tests.
14. Replace mandatory workspace/project listing on every new conversation with
    scope-on-demand behavior.
15. Add task-oriented starters for discovery, inspection, diagnosis,
    validation, comparison, and repair planning.
16. Return exact IDs, status, evidence, and deep links. Do not claim a queued
    or running operation is complete.

### P2 — Add rich presentation and governed actions

17. Add one Adaptive Card for a read-only project, diagnostic, or deployment
    summary.
18. Provide text fallback for every card.
19. Add explicit plan -> confirmation -> execution behavior for mutations.
20. Add durable operation status, retry, cancel, and resume behavior.
21. Use Copilot Studio agent flows for human approval, delay, polling, and
    multi-step business workflow.
22. Keep the default agent read-only; place builder and administration behind a
    separate profile or privileged agent.

### P3 — Improve enterprise grounding and operations

23. Add scoped SharePoint, OneDrive, Teams, Outlook, or Graph knowledge for
    architecture, runbooks, and standards.
24. Keep volatile platform state in MCP rather than uploaded knowledge.
25. Add DLP, sensitivity-label, connector, and prompt-injection controls.
26. Add telemetry for tool selection, latency, permission failures, card
    fallback, confirmation, duplicate operations, and user completion.
27. Validate behavior in Agents Toolkit Playground, tenant Copilot, and fresh
    sessions using repeatable evaluations.

## Runnable functions and runtime binding

The package has three separate layers:

| Layer | Role |
| --- | --- |
| `functions` | Model-facing function names, descriptions, and input contracts |
| `mcp_tool_description.file` | Static MCP tool definitions and schemas |
| `run_for_functions` | Runtime binding for functions that can actually execute |

If a function is present in `functions` but absent from `run_for_functions`, it
is descriptive only. The host may ignore it, reject package validation, or
report that it is unavailable; it must not be considered executable.

If no runnable functions are configured, the agent can still answer from its
instructions, conversation context, and configured knowledge. It cannot safely
perform live discovery, diagnostics, or mutations. Instructions must tell it to
state that limitation instead of inventing live data.

The dynamic Agents Toolkit pattern is appropriate when the runtime supports
discovery:

```json
{
  "functions": [],
  "run_for_functions": ["*"]
}
```

That pattern should be paired with a URL-only Remote MCP runtime and dynamic
tool discovery. It is useful for development and rapidly changing catalogs.

For production, prefer a curated static package:

```json
{
  "functions": [
    {
      "name": "platform_projects",
      "description": "List verified projects in the selected workspace. Read-only. Ask for clarification when the workspace is ambiguous."
    }
  ],
  "runtimes": [
    {
      "type": "RemoteMCPServer",
      "spec": {
        "url": "https://mcp.example.com/mcp/arch",
        "mcp_tool_description": {
          "file": "mcp-tools-readonly.json"
        }
      },
      "run_for_functions": ["platform_projects"]
    }
  ]
}
```

The exact function list, static file, and runtime binding must be generated
together. `run_for_functions` is a binding mechanism, not a security boundary.

## Tool contract practices

### Names and descriptions

Use names that describe a specific user task. A good description answers:

- what the tool does;
- when it should be called;
- when it must not be called;
- which workspace, project, agent, or environment it uses;
- whether it changes data;
- whether confirmation is required;
- whether the result is terminal or asynchronous.

Avoid generic descriptions such as `Performs project operations`.

### Input schemas

- Use an object root.
- Use explicit scalar types.
- Use enums for bounded actions.
- Keep nesting shallow.
- Make required fields explicit.
- Put semantic rules in descriptions as well as schemas.
- Reject invalid input server-side.
- Do not include credentials or connection bootstrap fields.

ABL schema import is intentionally lossy for some JSON Schema constraints. The
model-facing description cannot replace server-side validation. See [MCP tool
import schema support](../mcp/tool-import-schema-support.md).

### Output schemas

Add output schemas to the read-only tools used by the default agent before
adding cards. The result should distinguish facts from operation state:

```json
{
  "schemaVersion": "1.0",
  "success": true,
  "data": {},
  "sources": [],
  "links": [],
  "operation": null,
  "error": null
}
```

For asynchronous work, return an operation reference with `id`, `status`, and a
status or resume path. The agent must use the terms `started`, `running`,
`completed`, `failed`, `timed out`, and `unknown` accurately.

## Adaptive Card placement

### Microsoft 365 Copilot response cards

For an API-plugin function, define the card under:

```text
functions.<function>.capabilities.response_semantics.static_template
```

The response semantics should also identify the data path and title/subtitle
properties. The tool returns structured data; the plugin template defines how
the host renders it.

Use this path for:

- project summaries;
- diagnostic findings;
- deployment status;
- validation results;
- approval previews.

Do not put card layout in `declarativeAgent.json` or in the MCP tool catalog.
Do not return raw card JSON as a substitute for a stable tool result contract.

Microsoft documents static Adaptive Card templates for API-plugin functions in
[Return rich responses with Adaptive Cards](https://learn.microsoft.com/en-us/training/modules/copilot-declarative-agent-action-api-plugin-adaptive-cards-vsc/2-return-rich-responses-adaptive-cards).

Remote MCP support for this exact response-semantics path must be validated in
the target Copilot host. If the host does not apply the template to a Remote
MCP result, use an API-plugin façade for the card-capable read operation or
render the card through the ABL channel adapter.

### ABL channel responses

For Teams, Web, Slack, and other ABL-owned channels, create cards in the
response owner:

- scripted agent: the exact `FLOW` step that owns `RESPOND`;
- reasoning agent: an authored `ON_START`, `COMPLETE`, or lifecycle hook;
- hybrid agent: a deterministic presentation or confirmation step after the
  reasoning step.

Use the existing `RESPOND` rich-content path with `adaptiveCard` and a plain
text response. See [channel rich-content guidance](../../packages/arch-ai/src/knowledge/cards/platform/channels-messaging.ts)
and [rich-content placement guidance](../../packages/arch-ai/src/knowledge/cards/generated/rich-content.ts).

The generic Remote MCP transport should remain presentation-neutral. It returns
structured data; each host or channel adapter chooses the supported rendering.

### Teams entry points

Teams manifest command lists, static tabs, compose extensions, and sample
prompts are discovery mechanisms. They are not a dynamic tool catalog. Use
runtime suggested actions for contextual follow-ups such as:

- Inspect details
- Show recent failures
- Validate configuration
- Compare environments

For streamed Teams responses, send cumulative text and final attachments in the
final activity. Do not depend on an attachment-only terminal message.

### Card design rules

- Show three to five important fields.
- Provide one primary action and at most a few secondary actions.
- Include source or deep-link evidence.
- Include readable text fallback.
- Keep action payloads opaque and reauthorize them server-side.
- Never include secrets or hidden permissions in card data.
- Support accessibility, localization, and the client’s supported card schema.
- Test empty, long, error, permission-denied, card-only, and text-only cases.
- Use cards for decisions and summaries, not for every conversational answer.

## Conversation strategy

### Scope resolution

1. Trust the active workspace selected by host-managed login; never enumerate
   or switch workspaces from Copilot.
2. Reuse verified project scope inside the conversation.
3. Resolve exact project and agent IDs before calling a scoped tool.
4. Ask one clarification question when a project or agent name is ambiguous.
5. Never silently switch project, agent, or environment.
6. Show the selected project target before a write or destructive operation.

### Read-only journey

```text
User request
  -> Resolve only the required scope
  -> Call one read-only tool
  -> Summarize verified result
  -> Add card, source, and deep link when useful
```

### Diagnostic journey

```text
Identify project, agent, session, and time range
  -> Call the smallest diagnostic set
  -> Summarize finding
  -> Show evidence and likely next step
```

### Mutation journey

```text
Inspect -> validate -> plan/diff -> confirm -> execute -> check status
```

The agent must not silently turn “fix this” into a production mutation.

### Long-running journey

Use a Copilot Studio flow or durable ABL operation when the task contains
approval, delay, polling, callback, deployment, or human review. The original
Copilot turn must return an operation reference rather than attempting to hold
an unbounded conversational turn open.

## Knowledge and Microsoft feature strategy

Use Microsoft 365 knowledge sources for stable content:

- architecture and platform concepts;
- runbooks and troubleshooting procedures;
- development standards;
- deployment policy;
- ownership and support information.

Use Remote MCP for live or consequential content:

- current projects and agents;
- current deployment state;
- traces and diagnostics;
- validation;
- mutations and approvals;
- operation status.

Scope SharePoint, OneDrive, Teams, Outlook, and Graph sources narrowly. Use an
“only specified sources” style of grounding when precision matters. Remove stale
documents and record owners and version dates. Microsoft documents these
knowledge and publishing options in [Extend Microsoft 365 Copilot with
agents](https://learn.microsoft.com/en-us/microsoft-copilot-studio/microsoft-365-copilot-extend-with-agents).

| Need | Use | Reason |
| --- | --- | --- |
| Exact MCP contract and pro-code control | Agents Toolkit | Source control, direct APIs, custom MCP, cards, CI/CD |
| Approval and business workflow | Copilot Studio | Connectors, agent flows, human-in-the-loop, workflow testing |
| Simple Q&A over a small knowledge set | Agent Builder | Low/no-code personal or group productivity agent |
| Site or library-specific documentation assistant | SharePoint agent | Narrow content boundary |
| Current operational state | Remote MCP | Live authorization and platform truth |
| Stable procedures and standards | Microsoft 365 knowledge | Grounding without tool calls |
| Proactive or custom orchestration | Custom-engine agent | Full hosting and orchestration control |

Copilot Studio can attach an agent flow as a tool or run it independently. Use
that capability around Arch operations rather than duplicating the Arch catalog.
See [What is Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/fundamentals-what-is-copilot-studio)
and [Microsoft’s Declarative Agent tool comparison](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/declarative-agent-tool-comparison).

## Security, reliability, and governance practices

### Security

- Keep raw credentials in OAuthPluginVault or the approved Auth boundary.
- Never place credentials in tool schemas, instructions, card data, or logs.
- Reauthorize every tool call and card action.
- Use verified identity, tenant, workspace, project, and permissions.
- Conceal cross-tenant and unauthorized resources appropriately.
- Require explicit confirmation for writes and destructive actions.
- Use idempotency keys for mutations.
- Treat retrieved documents, tool descriptions, logs, and tool output as
  untrusted content.
- Do not let instructions inside external content override agent policy.

### Reliability

- Bound request, result, concurrency, and execution time.
- Distinguish admission from completion.
- Recover timeout and disconnect outcomes through durable operation status.
- Make retries safe or reject them explicitly.
- Never perform a second external mutation when a prior result is unknown.
- Preserve trace and operation IDs across Copilot, MCP, owner, and channel
  boundaries.

### Documentation and contract ownership

- Canonical Arch tool definitions remain the source of truth for tool contracts.
- Remote MCP owns exposure profiles and the inbound resource boundary.
- Auth/OIDC owns authentication, consent, token issuance, refresh, and
  revocation.
- Runtime, Studio, and Admin own their operations.
- Declarative Agent package files are generated projections of the selected
  profile, not a second catalog.
- Any schema, tool, or response change must update its authoritative source and
  generated projections in the same change.

## Evaluation and acceptance criteria

### Selection and conversation tests

- “What can you do?” produces a concise answer without a tool call.
- A project list uses the correct read-only tool.
- An ambiguous project name causes clarification.
- A diagnostic request selects diagnostic tools only after target resolution.
- A write request performs inspect and plan before confirmation.
- A user cannot cause a silent workspace or project switch.
- A request outside the profile receives a clear limitation.

### Security tests

- No secret-bearing property appears in generated `functions` or MCP schemas.
- `platform_connect` is absent from remote Copilot packages.
- Missing or invalid authentication returns the expected challenge.
- Unauthorized and cross-tenant targets do not leak resource existence.
- Confirmation is consumed atomically before mutation admission.
- A repeated idempotency key does not duplicate a mutation.

### Reliability tests

- Sync reads return bounded terminal results.
- Async work returns an operation reference.
- Timeout or disconnect returns a recoverable status, not false success.
- Retry and resume preserve operation identity.
- Owner gateway failure keeps readiness or execution fail-closed.

### Card tests

- Valid card renders in the target Copilot or channel.
- Plain text remains useful when card rendering is unavailable.
- Card actions validate identity, target, permission, expiry, and operation
  state again.
- Empty, long, error, permission-denied, and operation-running results render
  safely.
- Card fields do not expose secrets or untrusted markup.

### Operational metrics

Track at minimum:

- correct, incorrect, and unnecessary tool-selection rate;
- tool success and policy-error rates;
- authentication, permission, and confirmation rates;
- time to first useful result and terminal completion;
- card render versus text-fallback rate;
- async recovery and duplicate-operation rate;
- user completion, retry, abandonment, and feedback;
- token and downstream cost.

## Delivery sequence

### Phase 0: Contract alignment

Create the read-only profile, remove sensitive and local-only tools, switch to
the real HTTPS resource, generate the static package, and add drift checks.

### Phase 1: Read-only Copilot

Improve descriptions and output schemas, update scope instructions, add
task-oriented starters, and validate tool selection and authorization.

### Phase 2: Cards and actions

Add one read-only Adaptive Card, then add plan/confirmation/execute behavior for
one safe mutation. Validate both Copilot response semantics and ABL channel
rendering independently.

### Phase 3: Long-running and enterprise workflows

Add operation status/resume and use Copilot Studio flows for approval-heavy
workflows. Add scoped Microsoft 365 knowledge, DLP, and prompt-injection
controls.

### Phase 4: Reassess the agent model

Only move to a custom-engine agent if requirements include proactive behavior,
multi-user/group collaboration, external channels, custom models, or
orchestration that Declarative Agents cannot express.

## Open verification items

The following are not proven by repository source inspection alone:

- Remote MCP owner gateway behavior in a deployed environment;
- real Shared Auth/OIDC tenant and consent behavior;
- Microsoft 365 Copilot tool selection against the Remote MCP profile;
- whether the target Copilot host applies `response_semantics` directly to
  Remote MCP results;
- Adaptive Card rendering and action routing in the target tenant;
- production latency, throttling, and async recovery behavior.

These require environment-backed integration or tenant tests. Source-level
package and unit tests should be completed first, but they must not be reported
as proof of live Copilot behavior.

## Implementation source map

| Concern | Current source |
| --- | --- |
| Remote MCP resource and protocol | `apps/remote-mcp-server/` |
| Global and connection tool profiles | `apps/remote-mcp-server/src/catalog/tool-profile.ts` |
| Per-action safety policy | `apps/remote-mcp-server/src/catalog/server-capability-map.ts` |
| Call authorization and target checks | `apps/remote-mcp-server/src/execution/operation-policy.ts` |
| Delegation, deadlines, result bounds, recovery | `apps/remote-mcp-server/src/execution/tool-dispatcher.ts` |
| Canonical Arch tool catalog | `packages/mcp-debug/src/remote-catalog.ts` |
| Tool schemas and output contracts | `packages/mcp-debug/src/tools/` |
| Schema import limitations | `docs/mcp/tool-import-schema-support.md` |
| ABL rich content and card placement | `packages/arch-ai/src/knowledge/cards/` |
| Cross-feature capability matrix | `docs/feature-matrix.md` |
