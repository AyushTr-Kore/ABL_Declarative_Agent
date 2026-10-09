---
name: arch-platform
description: Plan, execute, and verify Kore.ai Agent Platform work through the Arch MCP server using code-backed operation and dependency knowledge.
---

# Arch Platform Operations

Use Arch as the Agent Platform control plane. Preserve the user's requested scope and obtain authorization before material external mutations.

## Source of Truth

1. Read `arch://guidance/v1/manifest` and the relevant feature or tool detail.
2. Require `schemaVersion` to equal `"1"` before using catalog dependencies, confidence, or planning prompts. If the manifest is absent or incompatible, stop catalog-driven planning and use only legacy tool discovery and existing project-builder operations; do not infer dependencies.
3. Treat schema-derived operations as executable only when support is `implemented` or `verified`.
4. Treat `authoritative-live` readiness as project-specific; inspect it through existing project-builder resources before planning project mutations.
5. Documentation-only or unsupported entries do not authorize an operation.

## Operating Rules

- Resolve every `requires` dependency before mutation.
- Use opaque auth-profile, integration, and MCP-server references; never place raw secrets in ordinary tool arguments.
- Respect `read`, `write`, `idempotent_write`, `grant_gated_write`, and `destructive_write` safety. Treat plain `write` as non-idempotent unless the live operation proves otherwise.
- Before grant-gated or destructive work, show the exact target and obtain the required confirmation/grant.
- If a mutation has an unknown outcome, verify it; never retry blindly.
- After acting, supply the operation's `verificationRequiredContext`, call its `validatesWith` tool/action, and compare the result with `verificationExpectedEvidence`. If the guidance says the validation read cannot prove the original outcome, report that limitation and rely only on the non-replayed original result.
- Do not claim support for platform features absent from the public Arch operation catalog.

## Whole-project sequence for any MCP client

Read `debug_docs` topic `mcp/project-build-playbook` for the complete client-neutral path;
the connected Studio topic is `guides/arch-mcp-project-build`. In order: connect and confirm
workspace; create/read the project; configure model and secure AuthProfiles; create/test
integrations and MCP servers; author tables, tools and workflows; create reusable prompts,
feedback templates and external-agent entries; create/read/guardedly edit agents and pin
prompt versions; configure optional policy, alerts and channels; publish immutable agent
versions; create/read the typed deployment; run evals and a real channel session. Use the
catalog's `validatesWith` after every mutation. `platform_versions:create` and `promote` are
unsupported legacy actions; use `publish` and deployment promotion. `platform_arch_sop` is an
alternative guided creation path, and `platform_arch_auto_loop` is for later governed repair.
Provider-specific channel provisioning is outside the current direct MCP surface.

For the expanded project path read `debug_docs` topic `mcp/complete-project-surfaces`.
It covers Knowledge Bases, memory declarations, Inbox tasks, Behaviour Profiles,
project Guardrails, and Insights Generator. Use only the feature/tool details needed
for the current stage to keep discovery context bounded. A 429 comes from the
platform request budget; honor Retry-After and never retry a possibly completed
write without an independent read. Synthetic Insights save requires an isolated
project, dry-run token, matching configDigest and explicit confirmation.

## Feedback template and agent sequence

For new project work, use `platform_projects:create`. Create an agent directly with
`platform_agents:create`; project import is reserved for whole-project packages. If the agent
needs customer feedback, create a same-project template with `platform_feedback_templates:create`
and put `FEEDBACK_TEMPLATE: <exact template name>` inside its `RICH_CONTENT: FEEDBACK` block.
The template alone does not display feedback. Read the agent's `sourceHash`, then use
`platform_agents:edit_dsl` with `expectedSourceHash` (`null` for a new draft). A 409 requires
a fresh read and deliberate edit. Existing `save_dsl` remains available for compatibility but
does not use Studio's per-agent edit ACL or source-hash guard. Read template `references` before
rename/delete; that guard scans authored working copies, not all published versions. Build a
version with project context so the compiler resolves the exact template name.

## Prompt Library sequence

Use `platform_prompt_library` for direct project prompt CRUD, draft versions, test/experiment,
promotion, references and lifecycle. A test or experiment requires a configured tenant model and
may incur provider cost. Create an agent with parseable DSL using `platform_agents:create`, then
use `platform_agents:set_prompt_ref` with a same-project `{promptId,versionId}`. Verify with
`platform_agents:get_prompt_ref`, which reads Studio; the older Runtime-backed `get` omits the
library ref. Draft and active versions can be pinned, archived versions cannot. Build and deploy
a new agent version after pinning; old deployments do not change. Read `references` before
archive/delete. If create-with-initialVersion, promotion or agent pin returns an ambiguous error,
read the saved state before retrying. Use project import only for whole-project packages.

## External agents, alerts, and channels

Use `platform_external_agents` for direct registry CRUD and connectivity checks. Create an
AuthProfile first when the remote endpoint needs credentials. Creation can trigger asynchronous
card discovery, so read the entry and run `test_connection` before configuring an agent handoff;
then exercise the actual transfer in a session. A registry entry does not itself create a handoff.

Use `platform_alerts:metrics` to choose a supported metric before creating a rule. Use typed
email or webhook destinations and AuthProfile references, read rules back after writes, and
inspect evaluation history. `test` evaluates the rule; it does not deliver a notification.

Use `platform_omnichannel` to read and patch project policy and configure signed mobile
webhooks. Supply an AuthProfile reference, read the integration back, and use health/test.
Use config-version guards for existing integrations and confirmation for rotation/deletion.
Effective settings defaults, SDK channel keys, agent deployments, provider-specific channels,
and live conversation joins are separate steps. None of these resource edits uses project import.

Use the `plan-platform-operation` prompt for a new multi-feature task and `verify-platform-operation` after a specific action.
