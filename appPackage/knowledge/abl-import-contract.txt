# ABL import preview and apply contract

> Embedded from the Arch MCP debug fallback documentation. Always use the
> current MCP response and preview result for a real project operation.

Import endpoints are mounted under Studio project routes:

- `POST /api/projects/:projectId/import/preview`
- `POST /api/projects/:projectId/import/apply`

## Payload fields

- `files`: required file map after local MCP assembly.
- `layers`: optional array of supported layer names. Unsupported names return
  `INVALID_LAYERS`.
- `deleteUnmatched`: optional boolean. `false` maps to merge; `true` maps to
  replace.
- `bindingResolutions`: optional object keyed by resolution ID.
- `previewDigest`: apply acknowledgement digest from preview.
- `acknowledgedIssueIds`: non-blocking preview issue IDs acknowledged by the
  caller.

## Apply acknowledgement rules

- Blocking preview issues must be fixed before apply.
- Non-blocking issues require acknowledgement.
- For a new package, omit `project.json` and let the platform generate it. If
  `project.json` is included, it must be a complete platform-exported v2
  manifest. A sparse object containing only `format_version` or `entry_agent`
  is not a valid v2 project manifest.
- A safe apply payload includes `previewDigest` and all non-blocking issue IDs.
- `platform_import_export` auto-previews and auto-acknowledges non-blocking
  issues when `confirm: true` and no complete manual acknowledgement is
  supplied.
- Partial manual acknowledgement is treated as stale by default and replaced
  by a fresh preview unless `autoAcknowledgeNonBlocking` is `false`.

## Validation evidence

`platform_validate_package` with `projectId` returns `importPreview` details:

- `previewDigest`
- `acknowledgedIssueIdsNeeded`
- `requiresAcknowledgement`
- `acknowledgementReady`
- `canApply`
- `missingAcknowledgementIssueIdCount`
- `suggestedApplyArgs`

Use `suggestedApplyArgs` with `platform_import_export` import when explicit
manual apply control is needed. Never claim an import was applied until the
MCP response confirms it.
