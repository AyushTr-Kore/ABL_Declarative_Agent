# ABL behavior profile package contract

> Embedded from the Arch MCP debug fallback documentation. Validate the
> package against the current platform before applying changes.

Agents attach standalone behavior profiles with:

```abl
USE BEHAVIOR_PROFILE: profile_name
```

Behavior profile files should be standalone ABL documents, typically under:

```text
behavior_profiles/<name>.behavior_profile.abl
```

`project.json` can declare behavior profiles by name with a path:

```json
{
  "behavior_profiles": {
    "shared_voice": {
      "name": "shared_voice",
      "path": "behavior_profiles/shared_voice.behavior_profile.abl",
      "priority": 10,
      "when_summary": "Always available",
      "used_by": ["SupportAgent"]
    }
  }
}
```

The snippet is only the `behavior_profiles` section to merge into a complete
exported v2 `project.json`. Do not send a sparse `project.json` containing only
`format_version`, `entry_agent`, or `behavior_profiles`. For a new MCP-created
project, omit `project.json` and let the platform generate the manifest;
include it only when starting from a complete platform export. Current
platforms normalize older sparse manifests for compatibility, but new MCP
packages should not use that legacy shape.

## Compiler and import expectations

- The profile document must exist in the package files.
- The agent `USE BEHAVIOR_PROFILE` name must match a declared or available
  profile.
- Preview diagnostics may report `PROFILE_NOT_FOUND` when a referenced profile
  is missing from the package.
- Behavior profiles compile before agent attachment; invalid profile DSL is a
  package validation issue.

## Repair workflow

1. Use `platform_package_model` to list behavior profiles, profile references,
   and unresolved references.
2. If an agent references a missing profile, add the profile file and
   `project.json` declaration, or remove the `USE BEHAVIOR_PROFILE` line.
3. Use `debug_lint_abl` and `platform_validate_package` to catch syntax,
   dependency, and design issues before import apply.
