---
name: m365-agent-builder-lifecycle
description: Use when provisioning, publishing, validating, or debugging this Microsoft 365 Declarative Agent package; catches EmbeddedKnowledge file-type errors, preserves session changes, and verifies lifecycle outcomes.
---

# M365 Agent Builder Lifecycle

Use this skill for the Microsoft 365 Agents Toolkit package in this repository.
Keep the Copilot package lifecycle separate from the ABL runtime/project
lifecycle.

## Package contract

- `appPackage/manifest.json` registers `appPackage/declarativeAgent.json`.
- `declarativeAgent.json` loads `instruction.txt`, `ai-plugin.json`, and
  optional local knowledge files.
- `m365agents.yml` runs create/auth, package, package validation, manifest
  update, and Copilot publish. Publish repeats package and validation steps.
- `EmbeddedKnowledge.files` accepts only `.doc`, `.docx`, `.ppt`, `.pptx`,
  `.xls`, `.xlsx`, `.txt`, and `.pdf`; it allows at most 10 files and 1 MB per
  file. `.md` is rejected even when the JSON is otherwise valid.

## Fix the current provisioning failure

The package now uses ten `.txt` files under `appPackage/knowledge/`; the
original failure came from referencing `.md` files. For future local bundled
knowledge, rename package files to `.txt` and update every reference. The
content can remain plain text with Markdown-style headings; the extension is
the platform allowlist check. Do not point the manifest at a file that was not
renamed or copied.

Use another capability only when its source model is intended:

- `OneDriveAndSharePoint` for tenant-accessible hosted documents;
- `WebSearch` for changing public documentation; or
- no `EmbeddedKnowledge` capability when bundled knowledge is not required.

Do not move reference material into `instruction.txt` just to bypass the
allowlist; that file is the agent's operating instructions, not its knowledge
store. Convert to `.docx` or `.pdf` only when document formatting matters.

Before provisioning, verify that every embedded path exists, has a supported
suffix, is no larger than 1 MB, and keeps the list at or below 10 files. Run
the toolkit package validator after this check; a successful zip step alone
does not prove that Copilot accepted the declarative-agent document.

## Session and commit safety

At the start of a task:

1. Run `git status --short` and inspect existing diffs for
   `appPackage/`, `m365agents.yml`, and `docs/`.
2. Treat pre-existing changes as protected. Do not reset, checkout, or
   overwrite them, and do not claim they were made by this session.
3. Keep source files separate from generated `appPackage/build/` output and
   ignored environment files.

When a manifest changes during the session:

1. Rebuild the package from source before validating or provisioning it.
2. Inspect the generated `declarativeAgent.json` inside the zip, not only the
   source file.
3. Before any commit, review the exact diff and stage only the requested files;
   never use `git add -A` for this workflow.
4. Commit only when the user explicitly requests a commit. Otherwise leave the
   change uncommitted and report its status.
5. After a requested commit, verify `git show --stat --oneline HEAD` and
   `git status --short`.

Provisioning can create or update tenant resources before a later publish
stage fails. Do not blindly retry a deterministic manifest error or create a
second app; fix and repackage the source, then reuse or inspect the existing
resource.

## Lifecycle checks

Use this order and report each result separately:

1. **Source:** JSON parses; referenced files exist; embedded-file limits and
   extensions pass; no secrets are added to the package.
2. **Package:** run the repository's ATK zip and validation stages, then inspect
   the zip contents and expanded declarative-agent document.
3. **Provision/publish:** distinguish app creation, OAuth registration,
   manifest update, and Copilot publish. A pass in one stage is not proof of
   the others.
4. **Runtime:** start the agent, test one EmbeddedKnowledge question and one
   MCP-backed request, and record whether the failure is package, auth,
   routing, tool execution, or response delivery.
5. **Evaluation:** after provisioning is confirmed, run the checked-in eval
   prompts or the configured evaluation command. Do not use evaluation output
   to prove provisioning succeeded.

For actual ABL project import, validation, runtime diagnosis, or eval repair,
use the repository-independent `artemis-agent-build-repair` skill when it is
available. This skill owns the outer M365 package boundary; that skill owns
the ABL project/runtime boundary.

## Completion evidence

Do not report success from a generated zip, PM2 status, or a successful local
JSON parse alone. A complete handoff names the changed files, source/package
validation result, provisioning or publishing stage reached, runtime smoke
result, evaluation result if run, and remaining uncommitted changes.
