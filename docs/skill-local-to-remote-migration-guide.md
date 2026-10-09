# Migrating skills from local clients to remote MCP

## Scope

This guide plans the migration of the packaged Kore.ai Arch skills from local Codex/Claude usage to
the Microsoft 365 Declarative Agent, where tools are called through the configured Remote MCP
server. It records what should change in a later skill-editing pass. **No `SKILL.md` or skill
reference files are changed by this guide.**

The current package has two separate parts:

- `appPackage/manifest.json` declares the package-level skills. A skill provides model guidance; it
  does not install an MCP server or grant a tool.
- `appPackage/ai-plugin.json` declares the model-facing functions and binds them to the Remote MCP
  server. `mcp-tools-copilot-full.json` supplies the tool schemas. Only functions declared and bound
  in this package are callable by Copilot.

Accordingly, a skill must not assume Copilot can start `npx`, edit a developer's local filesystem,
read a local path, open a browser or run a laptop test harness. It should use an exposed MCP function,
ask the user to provide the needed content, or identify the step as manual/unavailable.

## Remote-use rules for the future edit

1. **Connection and workspace:** Do not tell the Copilot user to run `claude mcp add`, `codex mcp
   add`, or `arch-mcp-guidance install`. The host owns the Remote MCP connection and OAuth. Check
   available functions and use `platform_workspaces` (`current`, `list`, or `switch`) to establish
   workspace context. The current package does not expose `platform_connect`. A workspace switch
   may require the Copilot host to reauthorize; the skill cannot force that UI.
2. **Packages and files:** Do not pass a path from the user's computer to a remote tool. Use
   `platform_import_export` to export a project as a relative-path-to-content `files` map. Where the
   schema accepts `files`, pass that map to `platform_package_model`, `platform_validate_package`,
   or `debug_why_transcript_failed`. If the user has a package that is not in the connected project,
   ask them to attach or paste it if the Copilot surface supports that. Otherwise report the missing
   input.
3. **Generated reports:** The current function set has no general local file-writing tool. Return
   reports and draft artifacts in the conversation. Do not claim to have created paths such as
   `analysis/<session-id>.md`, `docs/experience-contract.md`, or `evals/catalog.md` unless a future
   host capability actually writes that file.
4. **Session evidence:** For ended sessions, use the project and session IDs with the historical
   `debug_*` reads. Remote traces can omit full LLM request payloads (`llmPayloadOmitted`); request
   the Studio trace export as an attachment when the exact prompt is needed. Do not infer prompt
   content from a truncated trace.
5. **Manual and physical tests:** Voice audio generation/playback, Studio-only crawl controls, and
   UI-only test playgrounds remain user or CI tasks unless a corresponding Remote MCP function is
   added to this package. The skill can analyze the resulting session or supplied evidence, but must
   not claim it ran those tests.
6. **Writes:** Keep existing approval, confirmation, idempotency, and verification rules for remote
   mutations. A locally available command does not authorize a remote write, and a skill does not
   replace user approval.

## Migration inventory

| Packaged skill | Local assumption or mismatch | Planned remote treatment |
| --- | --- | --- |
| [arch-agent-guide](../appPackage/skills/arch-agent-guide/SKILL.md) | Includes Claude/Codex MCP setup commands, the Arch skill installer, client `/mcp` checks, `platform_connect`, `AGENTS.md`/`CLAUDE.md` creation, and writing long output into a project folder. | Remove client installation steps from the Copilot path. Check the package's available functions and `platform_workspaces`; ask for missing project context or files in chat; return long output inline. |
| [arch-context-budget](../appPackage/skills/arch-context-budget/SKILL.md) | Requires the full LLM request from the Studio trace export; MCP trace results omit that payload. | Keep the limitation explicit. Use remote trace markers and session metrics first; ask the user to attach the trace export for prompt-level analysis. |
| [arch-eval-catalog](../appPackage/skills/arch-eval-catalog/SKILL.md) | Describes offline package/version-control work and writing `evals/catalog.md`. It also references `platform_arch_auto_loop`, which is not bound to a Copilot function in the current package. | Use the exposed `platform_eval_*` functions for remote eval work and a `files` map for package validation. Return the catalog inline. Do not invoke the auto-loop until it is exposed and verified for this agent. |
| [arch-experience-contract](../appPackage/skills/arch-experience-contract/SKILL.md) | Passes a local `path` to `platform_package_model`, calls `platform_analytics` (not declared in this package), and writes `docs/experience-contract.md`. | Use an exported `files` map. Gather available measurements from `debug_session_history`, `debug_analyze_session`, and `platform_eval_runs`; request other analytics evidence when needed. Return the contract inline and mark missing measures. |
| [arch-explain-session](../appPackage/skills/arch-explain-session/SKILL.md) | Calls `platform_connect`, assumes live reads are local buffers, asks the user to read an exported trace file, and may write `analysis/<session-id>.md`. | Use the host-provided connection and workspace context, then remote historical reads for ended sessions. Request the trace export as an attachment for omitted prompt details; return the explanation inline. |
| [arch-knowledge-search](../appPackage/skills/arch-knowledge-search/SKILL.md) | References `platform_knowledge_bases`, `platform_ingestion_pipelines`, and `platform_analytics`, which are not declared as Copilot functions here. It also directs the user to Studio crawl and Search & Test UI actions. | Either expose and validate the missing functions or mark those actions as manual. Keep Studio crawl/playground work as an explicit handoff until a remote function exists; use available session tools to analyze returned retrieval evidence. |
| [arch-latency-cost](../appPackage/skills/arch-latency-cost/SKILL.md) | Exact prompt/token accounting can depend on full LLM request payloads that remote traces omit. | Use remote session history, analysis, span trees, and eval comparisons for available metrics. Request an attached trace export before making prompt-level claims. |
| [arch-platform](../appPackage/skills/arch-platform/SKILL.md) | Several referenced operations are not declared as Copilot functions: `platform_alerts`, `platform_arch_auto_loop`, `platform_external_agents`, `platform_feedback_templates`, `platform_omnichannel`, and `platform_prompt_library`. | Reconcile each reference: add its function and schema to the Copilot package if approved, or change the skill to report that action unavailable in this host. Use the already exposed project, agent, tool, workflow, workspace, and operation functions where applicable. |
| [arch-release-check](../appPackage/skills/arch-release-check/SKILL.md) | Says to save an export to a project folder and read full prompts from the Studio trace page. | Pass exported package contents as a `files` map to remote lint/validation functions. Request the trace export as an attachment for prompt-level checks. Return the release report inline; do not claim a file was written. |
| [arch-tool-design](../appPackage/skills/arch-tool-design/SKILL.md) | Mostly uses remote platform and debug functions already exposed by this package. Any MCP setup or project-file example must not be treated as a local Copilot command. | Keep the remote tool design/test flow. Route server setup through the exposed `platform_mcp_servers` actions, and retain explicit approval plus readback for mutations. |
| [arch-topology](../appPackage/skills/arch-topology/SKILL.md) | Allows `platform_package_model` to receive a local package `path`. | Prefer `files` from `platform_import_export.export` or a user-supplied package. Use remote flow graphs and eval results for runtime evidence. |
| [arch-voice-design](../appPackage/skills/arch-voice-design/SKILL.md) | Includes a local audio-generation, noise-mixing, and playback harness; also allows package inspection by local `path`. | Use `files` for package inspection. Keep audio execution as a user/CI test against the target channel, then use remote session history and traces to analyze the resulting call. |
| [arch-workflow-design](../appPackage/skills/arch-workflow-design/SKILL.md) | No direct local CLI or filesystem dependency identified; its workflow operations are represented by remote functions in this package. | Keep the remote workflow sequence. Verify the exposed action schema before each operation and return execution IDs/status as evidence. |

## Tool coverage and agent repair

There is no dedicated `arch-agent-repair` skill in the 13-skill package or the current upstream
[`skills/` directory](https://github.com/Koredotcom/agents-mcp-tools/tree/main/skills). Repair guidance
is spread across `arch-platform`, `arch-eval-catalog`, and `arch-release-check`.

Those skills mention `platform_arch_auto_loop`, but it is not in this package's model-facing
`functions`/`run_for_functions` set. The current Copilot package does expose tools sufficient for a
guarded manual repair path: `platform_agents` (including `edit_dsl` with a source-hash guard),
`debug_lint_abl`, `platform_validate_package`, and `platform_eval_*`. A future Copilot-focused `arch-agent-repair`
skill should use that sequence: inspect/export, identify one evidenced defect, request approval for
the exact edit, apply the guarded change, lint/validate, then compare the affected evals. It must not
call the unbound auto-loop function.

Other `platform_*` names used by the skills but not declared in the current Copilot package are
`platform_analytics`, `platform_connect`, `platform_ingestion_pipelines`, and
`platform_knowledge_bases`, along with the operations listed for `arch-platform` above. This means
they are not callable from this Declarative Agent even if an upstream Arch server or another MCP
client supports them. For each one, choose between exposing it in the app package (function
declaration, schema, and Remote MCP binding) or documenting a manual/unavailable step.

## Migration order and acceptance checks

1. **Resolve tool coverage first.** Approve which missing operations should be model-visible in
   Copilot. Keep `functions`, `mcp_tool_description.file`, and `run_for_functions` synchronized.
2. **Rewrite host and file assumptions.** Migrate `arch-agent-guide`, `arch-experience-contract`,
   `arch-explain-session`, `arch-release-check`, `arch-topology`, and `arch-voice-design` to the
   connection, workspace, file-map, and output rules above.
3. **Mark evidence boundaries.** Update context, latency, session, knowledge, and voice guidance so
   every UI or physical test clearly says who performs it and what evidence must be supplied back.
4. **Add a dedicated repair skill only after choosing its tool path.** Prefer the currently exposed
   guarded edit/validate/eval sequence unless the auto-loop function is explicitly added to the
   Copilot package and verified.
5. **Verify in Copilot.** For each migrated skill, check that every named function exists in the
   model-facing catalog; test workspace selection, a historical session, a package `files` map, a
   repair with approval, and the documented manual voice/knowledge handoffs. Confirm actual
   Remote MCP calls in host diagnostics; package/schema validation alone does not prove runtime
   tool selection.

## References

- [Package-level Agent Skills setup](agent-skills.md)
- [Declarative Agent function and Remote MCP binding](../appPackage/ai-plugin.json)
- [Copilot MCP tool schemas](../appPackage/mcp-tools-copilot-full.json)
- [Arch MCP tool catalog](../appPackage/mcp-tools-arch.json)
- [Upstream Kore.ai skills](https://github.com/Koredotcom/agents-mcp-tools/tree/main/skills)
