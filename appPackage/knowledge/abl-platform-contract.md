# Platform contract for ABL repair tools

> Embedded from the Arch MCP debug fallback documentation. Use MCP results
> for the current workspace; this file describes the package contract.

The MCP package can inspect local folders, zip archives, or import payloads
without requiring callers to manually build a file map.

## Accepted package inputs

- `path`: absolute or relative path to a project folder or `.zip` file.
- `files`: object mapping relative file paths to UTF-8 file content.
- `data.files`: import-style payload file map, normalized the same way as
  `files`.

## Normalization and safety rules

- Backslashes are converted to forward slashes.
- Absolute paths, null bytes, and `..` path traversal are rejected.
- Common archive wrappers, including nested wrappers such as `repo-main/src/`,
  are stripped when they contain `project.json`, `abl.lock`, or a supported
  package content directory.
- Supported content directories include `agents`, `tools`,
  `behavior_profiles`, `config`, `core`, `connections`, `prompts`,
  `guardrails`, `workflows`, `evals`, `search`, `channels`, `vocabulary`,
  `locales`, `deployments`, and `environment`.
- Skipped directories: `.git`, `node_modules`, `dist`, `build`, `.turbo`, and
  `__MACOSX`.
- Skipped files: `.DS_Store`.
- Local MCP assembly is limited to 500 files and 1 MB per file.

## Recommended Arch loop

1. Build with `platform_import_export`, `platform_projects`,
   `platform_agents`, `platform_tools`, and `platform_config`.
2. Optimize with `platform_validate_package`, `platform_package_model`, and
   `debug_lint_abl`.
3. Evaluate with `platform_eval_*` tools for persona, scenario, evaluator,
   set, and run workflows.
4. Debug with `platform_connect`, `debug_traces`, `debug_get_errors`, and
   `debug_why_transcript_failed`.
5. Analyze with `debug_diagnose` and `debug_analyze_session`, then patch and
   repeat until validation and evaluations are clean.
