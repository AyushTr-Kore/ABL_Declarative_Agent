# Tool Schema and Tool Discoverability Strategies

**Status:** Proposed implementation guidance
**Scope:** Microsoft 365 Copilot Declarative Agent using the ABL Remote MCP
registry
**Valid strategies:** Flat catalog, capability profiles, and dynamic MCP
discovery

## Problem

The agent currently exposes approximately 42 tools covering nearly 400
operations. The model must evaluate a broad set of tools and action variants
for each request.

This creates several risks:

- incorrect or unnecessary tool selection;
- larger prompt and schema overhead;
- higher latency and downstream execution cost;
- inconsistent descriptions and parameter contracts;
- harder permission, testing, and troubleshooting workflows; and
- more difficult rollout when one tool or operation changes.

The number of model-visible functions and the number of MCP operations are not
the same thing. A function may expose multiple operations through an `action`
discriminator. Reducing only the number of function names does not solve the
problem if a single function still exposes unrelated read, write, and
destructive actions.

## Current package baseline

The current package is a flat model-facing catalog with runtime-provided MCP
schemas:

- `appPackage/ai-plugin.json` declares 42 functions.
- `run_for_functions` binds the same 42 names to the Remote MCP runtime.
- The runtime points to the global `/mcp/arch` resource.
- No checked-in static `mcp-tools-*.json` catalog is currently maintained.
- `appPackage/declarativeAgent.json` contains the agent instructions,
  knowledge, starters, and one MCP action reference.
- `appPackage/manifest.json` registers the declarative agent application; it is
  not a tool catalog.
- `m365agents.yml` provisions the application, OAuth registration, package,
  and publishing stages.

The current state is useful for compatibility and development, but it keeps
the model-facing catalog broad.

## Shared architecture and ownership

All three strategies should use the same canonical tool definitions and
execution path:

```text
Canonical Arch catalog
        |
        +--> selected model-facing projection
        |       functions
        |       static catalog, when used
        |       run_for_functions
        |
        v
Remote MCP resource
  discovery -> authorization -> target checks -> confirmation -> execution
        |
        v
Owning platform service
  Runtime / Studio / Admin / Evaluation / Repair
```

| Concern | Responsible component |
| --- | --- |
| Canonical tool names, actions, and schemas | Arch/MCP catalog owner |
| Model-visible descriptions | Generated `ai-plugin.json` projection |
| Profile exposure and discovery | Remote MCP server |
| Authentication and consent | OAuth/OIDC and `OAuthPluginVault` |
| Tenant, workspace, permission, target, confirmation, and idempotency checks | Remote MCP policy layer |
| Actual business operation | Owning Runtime, Studio, Admin, Evaluation, or Repair service |
| Long-running operation state | Owning service and durable operation store |
| Tool selection | Copilot host/model |
| Package and publishing lifecycle | Microsoft 365 Agents Toolkit |

`run_for_functions` binds a model-facing function to a runtime. It is not an
authorization boundary. Every `tools/call` must be authorized again by Remote
MCP.

## Shared operation response contract

The MCP adapter should return a consistent result envelope for all three
strategies:

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

Synchronous reads return a terminal result. Long-running work returns an
operation reference instead of claiming completion:

```json
{
  "schemaVersion": "1.0",
  "success": true,
  "data": null,
  "operation": {
    "id": "op_123",
    "status": "accepted",
    "statusUrl": "https://example.invalid/operations/op_123",
    "resumeUrl": "https://example.invalid/operations/op_123/resume"
  },
  "error": null
}
```

The owning service owns the operation state. Remote MCP carries the identity,
authorization context, operation ID, and result. The agent reports only the
state returned by the service:

- `accepted` or `queued`: admitted but not started;
- `running`: execution is in progress;
- `completed`: terminal success was confirmed;
- `failed`: terminal failure was confirmed;
- `timed_out`: the service timed out the operation; and
- `unknown`: completion cannot yet be established after timeout or disconnect.

Retries of mutations must use an idempotency key. An `unknown` result must not
cause the agent to submit the mutation again automatically.

## Strategy 1: Flat catalog

### Solution

Expose the complete model-facing catalog. The agent receives all supported
functions and the Remote MCP runtime provides their schemas and operation
enums.

This is the smallest implementation because it preserves the current package
shape and does not require profile selection or a discovery index.

### Implementation steps

1. Keep the canonical Arch registry as the source of truth.
2. Inventory all functions and their action variants.
3. Improve every function description with:
   - purpose and intended use;
   - required workspace, project, agent, or session scope;
   - read, write, or destructive behavior;
   - confirmation requirements; and
   - synchronous or asynchronous result behavior.
4. Keep `functions` and `run_for_functions` exactly aligned.
5. Remove credential, bootstrap, and local-only properties from model-visible
   schemas.
6. Add tool-selection tests for common, ambiguous, read-only, and mutation
   requests.
7. Measure selection accuracy, latency, token usage, policy failures, and
   unnecessary tool calls before choosing a narrower strategy.

### Manifest changes

`appPackage/ai-plugin.json` keeps all function definitions and the same names
in `run_for_functions`:

```json
{
  "functions": ["all model-facing function definitions"],
  "runtimes": [
    {
      "type": "RemoteMCPServer",
      "spec": {
        "url": "https://<environment-host>/mcp/arch"
      },
      "run_for_functions": ["the same function names"],
      "auth": {
        "type": "OAuthPluginVault",
        "reference_id": "${{MCP_DA_AUTH_ID_...}}"
      }
    }
  ]
}
```

Required package changes:

- `ai-plugin.json`: usually no structural change; the current package already
  follows this pattern.
- `declarativeAgent.json`: update routing instructions and starters only when
  needed; it does not contain the tool catalog.
- `manifest.json`: no tool entries are added.
- `m365agents.yml`: replace local or temporary MCP/OAuth URLs with the target
  environment configuration.
- Static catalog: optional. If used, add
  `spec.mcp_tool_description.file` and generate the file from the complete
  canonical catalog.

### Advantages

- Lowest implementation effort.
- Maximum capability visibility.
- No profile-switching or discovery flow.
- Preserves compatibility with uncommon operations.

### Disadvantages

- Highest model-selection confusion.
- Largest schema and prompt overhead.
- More tool descriptions and action contracts to maintain.
- Broadest permission and troubleshooting surface.

### Best fit

Use for development, compatibility, internal users, or as a measured baseline
before reducing the default catalog.

## Strategy 2: Capability profiles

### Solution

Group tools by user workflow and expose only one selected profile to the
default agent. The profile is enforced by Remote MCP and projected into the
Copilot package.

Recommended profile boundaries are:

| Profile | Scope |
| --- | --- |
| `arch-copilot-readonly` | Inspection, project and agent discovery, validation, documentation, and status |
| `arch-copilot-diagnostics` | Sessions, traces, errors, spans, decisions, and diagnosis |
| `arch-copilot-evaluations` | Evaluation personas, scenarios, evaluators, sets, runs, and analysis |
| `arch-copilot-deployment` | Version, deployment, channel, and environment inspection |
| `arch-copilot-builder` | Project building, workflows, repair loops, and controlled changes |
| `arch-copilot-admin` | Authentication profiles, integrations, MCP servers, and privileged administration |

The default production profile should be read-only. Diagnostics and mutation
profiles should be activated through an explicit deployment or package
boundary, not selected merely because the model sees a broad action enum.

### Action-aware filtering

A tool-name allowlist is insufficient when a tool contains mixed actions. For
example, a project or deployment function may support both inspection and
destructive operations.

Use this order of preference:

1. Generate an action-filtered read-only schema from the existing contract.
2. If the schema cannot safely express the filtered action set, create
   separate model-facing read and mutation functions backed by the same owner
   handler.
3. Keep server-side action authorization as the final authority in every case.

Do not rely on instructions alone to prevent a destructive action.

### Implementation steps

1. Define profile IDs and tool/action membership in Remote MCP.
2. Define the profile selection mechanism, such as a profile-specific resource
   or an existing server-side resource selector.
3. Generate the selected `functions` list.
4. Generate a matching static MCP catalog when static schemas are required.
5. Generate the matching `run_for_functions` list.
6. Add a drift check that compares the package projection with server-side
   `tools/list`.
7. Add selection tests that verify read-only requests cannot select mutation
   actions.
8. Add authorization tests for every profile and action class.

### Manifest changes

For a profile-specific package, `ai-plugin.json` contains only the selected
projection:

```json
{
  "functions": ["functions in the selected profile"],
  "runtimes": [
    {
      "type": "RemoteMCPServer",
      "spec": {
        "url": "https://<environment-host>/<profile-resource>",
        "mcp_tool_description": {
          "file": "mcp-tools-arch-copilot-readonly.json"
        }
      },
      "run_for_functions": ["the same profile function names"],
      "auth": {
        "type": "OAuthPluginVault",
        "reference_id": "${{MCP_DA_AUTH_ID_...}}"
      }
    }
  ]
}
```

Required package changes:

- `ai-plugin.json`: replace the full function list with one profile projection.
- `mcp-tools-<profile>.json`: add only if static schemas are used; generate
  it from the same source as the function list.
- `declarativeAgent.json`: describe the active profile, its scope, and the
  correct limitation response for requests outside that profile.
- `manifest.json`: keep the package's single declarative-agent registration;
  it does not become a profile catalog.
- `m365agents.yml`: update OAuth `baseUrl` or scopes if the profile uses a
  different resource or audience.
- `manifest.json.validDomains`: update if the profile endpoint uses a new host.

If profile selection requires a different endpoint, the Remote MCP route must
exist before changing the package URL. Do not invent a query parameter or
profile path that the server does not implement.

### Advantages

- Smaller active catalog and lower prompt/schema overhead.
- Better model selection and workflow-specific descriptions.
- Easier permission and test boundaries.
- Clearer ownership for diagnostics, evaluation, deployment, and repair.

### Disadvantages

- Requires profile ownership and maintenance.
- Requires synchronized generated projections.
- A request outside the active profile needs a clear limitation or an explicit
  profile change.
- Mixed-action tools require schema filtering or function splitting.

### Best fit

Use as the default production strategy, beginning with a curated read-only
profile and adding other profiles only when there is a demonstrated workflow
or permission need.

## Strategy 3: Dynamic MCP discovery

### Solution

Expose a small initial surface and discover additional tools from Remote MCP at
runtime. The model receives only the tools needed for the current request or
workflow.

The discovery response must contain useful metadata:

- tool name and purpose;
- supported actions;
- required scope;
- read, write, or destructive behavior;
- expected result duration; and
- required target identifiers.

Discovery is not authorization. A discovered tool must be authorized again
when the model calls it.

### Implementation steps

1. Implement reliable MCP `tools/list` behavior for the selected resource.
2. Add bounded pagination or search for large catalogs.
3. Return concise, high-quality descriptions and schemas.
4. Keep discovery responses tenant- and profile-aware.
5. Cache discovery metadata for a bounded period.
6. Recheck identity, scope, target, confirmation, and permissions in
   `tools/call`.
7. Return an explicit discovery error when the catalog is unavailable.
8. Validate that the target Agents Toolkit and Copilot host support dynamic
   runtime binding before adopting this as the production default.

### Manifest changes

The pure dynamic pattern is:

```json
{
  "functions": [],
  "runtimes": [
    {
      "type": "RemoteMCPServer",
      "spec": {
        "url": "https://<environment-host>/mcp/arch"
      },
      "run_for_functions": ["*"],
      "auth": {
        "type": "OAuthPluginVault",
        "reference_id": "${{MCP_DA_AUTH_ID_...}}"
      }
    }
  ]
}
```

Required package changes:

- `ai-plugin.json`: use an empty function list and wildcard binding if the host
  supports it, or expose only a small fixed discovery/core set.
- `mcp_tool_description.file`: omit it for pure runtime discovery.
- `declarativeAgent.json`: instruct the agent to discover capabilities before
  selecting an unfamiliar tool and to report discovery failures explicitly.
- `manifest.json`: no tool changes.
- `m365agents.yml`: update only the environment MCP/OAuth configuration when
  required.

If wildcard binding is unsupported, keep a small explicit discovery function
and do not assume that an unbound server tool is executable.

### Advantages

- Small initial model context.
- Scales as the operation count grows.
- New tools can become discoverable without rebuilding the entire static
  package.
- Less manual package maintenance for rapidly changing catalogs.

### Disadvantages

- Adds discovery latency.
- Depends heavily on accurate tool metadata.
- Discovery failures can prevent otherwise valid requests.
- Runtime behavior is harder to reproduce than a fixed static package.
- Host support must be proven in the target Copilot environment.

### Best fit

Use for large or rapidly changing catalogs, long-tail operations, and
development environments. Keep the core discovery contract small and stable.

## Comparison

| Concern | Flat catalog | Capability profiles | Dynamic discovery |
| --- | --- | --- | --- |
| Implementation effort | Low | Medium | Medium/high |
| Model context size | Largest | Smaller | Smallest initially |
| Tool-selection accuracy | Lowest | Highest for known workflows | Depends on metadata |
| Catalog maintenance | Centralized but broad | Profile maintenance required | Discovery index maintenance |
| Runtime latency | Lowest discovery latency | Low | Higher first-use latency |
| Permission isolation | Broadest surface | Stronger profile boundary | Runtime-enforced |
| Reproducibility | Highest | High per profile | Lower unless discovery is versioned |
| Recommended use | Compatibility and baseline | Production default | Long tail and fast-changing tools |

## Recommended rollout

### Phase 1: Establish the baseline

1. Keep the current flat package working.
2. Replace temporary MCP/OAuth URLs with an environment HTTPS resource.
3. Confirm exact equality between `functions` and `run_for_functions`.
4. Remove credentials and bootstrap fields from all model-visible schemas.
5. Record tool-selection, latency, token, permission, and failure metrics.

### Phase 2: Add capability profiles

1. Create `arch-copilot-readonly` first.
2. Generate its function list and static catalog from the profile.
3. Add action-aware filtering.
4. Add diagnostics, evaluation, deployment, and builder profiles only when
   their workflow or permission boundaries are justified.
5. Add package/server drift checks.

### Phase 3: Add dynamic discovery where it helps

1. Keep a small stable core catalog.
2. Add bounded discovery for long-tail operations.
3. Test discovery latency, metadata quality, and failure recovery.
4. Promote dynamic discovery only after target-host behavior is verified.

## Validation checklist

### Package checks

- Parse every JSON package file.
- Validate the app package with Agents Toolkit.
- Confirm all referenced files exist.
- Confirm `functions`, static catalog names, and `run_for_functions` match.
- Confirm no credential or bootstrap fields are model-visible.

### MCP checks

- `tools/list` returns only the selected profile when profiles are enabled.
- `tools/call` rechecks authorization and target scope.
- Unsupported actions are rejected server-side.
- Tenant and workspace isolation is preserved.
- Timeouts and disconnects return recoverable operation state.
- Repeated mutation requests with the same idempotency key do not duplicate
  work.

### Copilot checks

- Common requests select the intended tool.
- Ambiguous project or agent names cause clarification.
- Read-only requests cannot select destructive action variants.
- Requests outside a profile receive a clear limitation.
- Dynamic discovery failure does not produce an invented tool or result.
- Long-running work is reported as an operation, not false completion.

Local JSON and package validation do not prove live Copilot tenant behavior.
Each selected strategy still requires validation in the target tenant and
environment.

## Decision

Use capability profiles as the production direction, retain the flat catalog as
the compatibility baseline, and introduce dynamic discovery for the long tail
after its host behavior and metadata quality are verified.
