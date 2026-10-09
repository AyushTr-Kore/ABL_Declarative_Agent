# Declarative Agent skills

## Approach: package-level Agent Skills

This app includes Agent Skills in the Teams app package. The root `agentSkills` property in
`appPackage/manifest.json` points to folders under `appPackage/skills/`; each folder contains a
`SKILL.md` and any files it references. Keep this declaration in the Teams app manifest, not in
`declarativeAgent.json`.

The manifest schema allows up to 20 skill folders. This package currently includes 13 Kore.ai Arch
skills copied from the local Codex installation under `~/.codex/skills/arch-*`. The same skill
folders can be sourced from the [agents-mcp-tools skills directory](https://github.com/Koredotcom/agents-mcp-tools/tree/main/skills).

## Add or update a skill

1. Copy the selected skill folder into `appPackage/skills/<skill-name>/`.
2. Keep its `SKILL.md` and every relative file it links to, such as `references/` documents. The
   `name` in the YAML frontmatter must match the folder name; provide a concise `description` so the
   agent can select the skill for relevant requests.
3. Add `{ "folder": "skills/<skill-name>" }` to `agentSkills` in `appPackage/manifest.json`.
4. Keep the MCP function catalog in `appPackage/ai-plugin.json` in sync with the tool names the
   skills describe. Skills supply guidance; they do not create an MCP connection or expose tools.
5. Rebuild and validate the package using the `teamsApp/zipAppPackage` and
   `teamsApp/validateAppPackage` actions in `m365agents.yml`.

For Codex skill folders, package `SKILL.md` and the referenced resources. Client-specific files such
as `agents/openai.yaml` and `.arch-mcp-guidance.json` are not needed by this M365 package. The command
`npx -y -p @koreai/arch-mcp-tools@1.8.0 arch-mcp-guidance install --client claude --scope project`
installs for Claude Code at project scope; it does not add files to this app package.

## Verify the built ZIP

After `teamsApp/zipAppPackage` and `teamsApp/validateAppPackage` complete, inspect the generated ZIP
under `appPackage/build/` and confirm it contains `manifest.json`, each declared
`skills/<skill-name>/SKILL.md`, and every referenced resource. Package validation checks the
artifact and manifest; it does not prove that a particular Copilot host activates or selects every
skill. Verify skill behavior in the target tenant and host separately.

Do not run the full `provision` or `publish` lifecycle just to validate a package: those actions can
create or update tenant resources.

## References

- [Microsoft Teams app manifest schema v1.30](https://developer.microsoft.com/en-us/json-schemas/teams/v1.30/MicrosoftTeams.schema.json)
- [Agent Skills specification](https://agentskills.io/specification)
- [Microsoft 365 Copilot declarative agents overview](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/overview-declarative-agent)
- [Microsoft 365 Copilot Cowork plugin development](https://learn.microsoft.com/en-us/microsoft-365/copilot/cowork/cowork-plugin-development)
- [Kore.ai agents-mcp-tools skills](https://github.com/Koredotcom/agents-mcp-tools/tree/main/skills)
