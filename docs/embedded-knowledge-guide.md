# ABL knowledge sources

This project has three different sources of truth. Keep them separate:

| Source | Use it for | Current configuration |
| --- | --- | --- |
| Arch MCP | Current workspace, project, agent, session, evaluation, and deployment state | `appPackage/ai-plugin.json` |
| WebSearch | Public and changing Kore.ai ABL documentation | Enabled for `https://docs.kore.ai/agent-platform/` |
| EmbeddedKnowledge | Small, stable, approved reference files shipped with this app | Enabled with ten curated Arch ABL/reference documents |

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

## Current embedded references and future additions

The package currently embeds ten focused references. The first five are copied
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

Because v1.8 supports at most 10 embedded files, replace or consolidate a file
before adding another one. Keep organization-specific SOPs separate from
platform contracts so they can be reviewed and updated independently.

The v1.8 capability supports at most 10 embedded files and each file must be
at most 1 MB. Keep files focused, factual, and free of tokens, secrets, or
tenant data. Update the files and the package version when the guidance changes.

## How to add it

1. Add the reviewed plain-text (`.txt`) or supported document file under
   `appPackage/knowledge/`.
2. Add its relative path to the existing capability in
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
10 files and each file at or below 1 MB. The current package uses embedded
files for stable debug/import guidance and WebSearch for changing public
documentation.

## OneDrive and SharePoint documents

If the ABL guide becomes an internal OneDrive or SharePoint document, use the
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
