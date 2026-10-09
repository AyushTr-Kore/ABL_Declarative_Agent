# ABL knowledge sources

`EmbeddedKnowledge` is disabled starting with app version **1.0.10**. The
agent uses MCP for live platform state and ABL documentation, scoped WebSearch
for public guidance, and CodeInterpreter for supplied evidence.

| Source | Use it for | Current configuration |
| --- | --- | --- |
| Arch MCP | Current workspace, project, agent, session, evaluation, and deployment state | `appPackage/ai-plugin.json` |
| MCP `debug_docs` | ABL language, debug, and import documentation | Included in the pinned MCP catalog |
| WebSearch | Public and changing Kore.ai ABL documentation | Enabled for `https://docs.kore.ai/agent-platform/` |
| CodeInterpreter | User-provided traces, JSON, CSV, packages, and calculations | Enabled |
| EmbeddedKnowledge | Bundled reference-file retrieval | Disabled; files retained in the repo |

## Why we removed EmbeddedKnowledge

The agent's primary purpose is to inspect and diagnose the authenticated ABL
workspace through MCP. Embedded file retrieval gave Copilot another way to
answer project questions using general reference material rather than live
platform evidence.

In the observed version 1.0.9 failure, the request to diagnose OmniSearch had
zero matched, selected, or executed MCP functions. The knowledge execution
referenced the ten bundled ABL `.txt` files. An explicit MCP request on the
same version subsequently matched all 42 functions and executed
`platform_projects(action="list")` with HTTP 200 after confirmation and
OAuth sign-in. This shows that the catalog was provisioned and that routing
and connection completion require separate verification.

Microsoft stores uploaded embedded files in SharePoint Embedded containers.
Our deployed agent reported `OneDriveAndSharePoint` with ten file-specific
scopes even though the source configured `EmbeddedKnowledge`. See
[Microsoft's embedded-file storage documentation](https://learn.microsoft.com/en-us/microsoft-365/admin/manage/agent-details#manage-agents-with-embedded-file-content-as-a-knowledge-source).
Removing the capability removes this configured bundled-file retrieval path.

This change does not establish that EmbeddedKnowledge caused the unrelated
Word/PPT citations in an earlier response, and it does not force Copilot to
select an MCP tool. We lose automatic grounding from these bundled files.
Use `debug_docs` with `query` or `topic` for documentation, and MCP project or
diagnostic tools for actual workspace state. If MCP cannot run, report the
connection or evidence gap instead of treating documentation as project data.

The 42 pinned functions, runtime bindings, tool schemas, OAuth registration,
and app IDs are unchanged by this removal. The `.txt` references remain under
`appPackage/knowledge/` for maintainers but are not referenced or packaged.

## Verify after provisioning

1. Rebuild and validate the package. Inspect the ZIP to confirm the agent has
   only `CodeInterpreter` and the scoped `WebSearch`, with no bundled knowledge
   files and all 42 pinned MCP tools still present.
2. Provision using the existing app and OAuth IDs. Start a fresh conversation
   and enable developer mode with `-developer on`.
3. Confirm the deployed version is 1.0.10 and that the previous ten-scope
   `OneDriveAndSharePoint` capability is absent. If it remains, inspect the
   deployed configuration and agent ID before attributing the result to MCP.
4. Test both a project-list/diagnosis request and an ABL documentation request.
   Complete host confirmation or sign-in when required. Verify actual server
   `tools/call` requests for `platform_projects` and `debug_docs`, including
   successful responses; a generated answer or `tools/list` alone is insufficient.

The developer card distinguishes matching, selection, and execution for each
prompt; see [Microsoft's developer-mode guidance](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/debugging-agents-vscode#function-matching-and-selection).

Do not put security rules, consent policy, or agent operating instructions in
EmbeddedKnowledge. Keep those in `appPackage/instruction.txt`. Knowledge files
provide facts and reference material; they must not override the agent's system
instructions or the user's request.

## Public Kore.ai documentation

WebSearch is the best fit for the public documentation because it can follow
updated pages without copying the site into the app package. The configured
site scope is:

- [ABL overview](https://docs.kore.ai/agent-platform/abl)
- [ABL language overview](https://docs.kore.ai/agent-platform/abl/reference/language-overview)
- [Agent declaration](https://docs.kore.ai/agent-platform/abl/reference/agent-declaration)
- [Tools](https://docs.kore.ai/agent-platform/abl/reference/tools)
- [Flows](https://docs.kore.ai/agent-platform/abl/reference/flow)
- [Memory and constraints](https://docs.kore.ai/agent-platform/abl/reference/memory-and-constraints)
- [Guardrails](https://docs.kore.ai/agent-platform/abl/reference/guardrails)
- [Debug routing traces](https://docs.kore.ai/agent-platform/abl/how-to/debug-routing-traces)

The agent should link the narrowest page that supports a documentation answer.
WebSearch answers do not prove the state of a tenant or deployment; verify
those claims through MCP.

## Retained references

The repo retains ten unreferenced files. The first five are copied
from the Arch MCP debug fallback documentation:

- `appPackage/knowledge/abl-flow-lifecycle.txt`
- `appPackage/knowledge/abl-platform-contract.txt`
- `appPackage/knowledge/abl-import-contract.txt`
- `appPackage/knowledge/abl-behavior-profiles.txt`
- `appPackage/knowledge/abl-repair-loop.txt`

The next five are curated from the local ABL authoring, FAQ, and examples
documentation:

- `appPackage/knowledge/abl-language-overview.txt`
- `appPackage/knowledge/abl-agent-declaration.txt`
- `appPackage/knowledge/abl-tools-reference.txt`
- `appPackage/knowledge/abl-faq.txt`
- `appPackage/knowledge/abl-examples-by-pattern.txt`

## If embedded knowledge is reintroduced later

Only reintroduce this capability when bundled grounding is required and its
routing has been tested separately from live MCP state. The v1.8 manifest
allows at most ten files, each at most 1 MB. Keep files factual and free of
tokens, secrets, or tenant data.

1. Add the reviewed plain-text (`.txt`) or supported document file under
   `appPackage/knowledge/`.
2. Add an `EmbeddedKnowledge` capability with its relative path in
   `appPackage/declarativeAgent.json`, for example:

```json
{
  "name": "EmbeddedKnowledge",
  "files": [
    { "file": "knowledge/abl-authoring-basics.txt" },
    { "file": "knowledge/abl-engineering-sop.txt" }
  ]
}
```

3. Keep `CodeInterpreter` and `WebSearch` as separate capabilities. They serve
   different jobs: CodeInterpreter analyzes supplied evidence, WebSearch finds
   current public guidance, and EmbeddedKnowledge supplies the team's curated
   baseline.
4. Validate and package the app with Microsoft 365 Agents Toolkit, then test
   questions that should be answered from the embedded files and questions that
   require current MCP state.

Do not add an empty `EmbeddedKnowledge` entry. Keep the file list at or below
10 files and each file at or below 1 MB. Update the instructions and package
version when intentionally enabling it again.

## OneDrive and SharePoint documents

This capability is also disabled in the current package. If tenant-hosted
documents are explicitly required in the future, use the
Microsoft `OneDriveAndSharePoint` capability with the tenant-accessible URL or
SharePoint identifiers. A public Kore.ai URL is not a OneDrive source; keep it
under WebSearch. Confirm that every intended user can access the referenced
document before publishing the agent. The URL form looks like this:

```json
{
  "name": "OneDriveAndSharePoint",
  "items_by_url": [
    { "url": "https://contoso.sharepoint.com/sites/abl/Shared%20Documents/ABL-guide.docx" }
  ]
}
```

Add that object beside `WebSearch` and `CodeInterpreter` in the
`capabilities` array only after the document is available to the target users.
