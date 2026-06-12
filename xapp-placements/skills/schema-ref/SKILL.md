---
name: schema-ref
description: Print canonical XAppAdKit (xappsdk) JSONC schema reference for SDK 0.12.5 and named UI presets. Use when user asks about xapp_config / xapp_ad_units / xapp_registry / xapp_p_* shape, native ui_config fields, valid format enums, preload_trigger values, capping defaults, or wants to inspect starter ad-unit preset / native UI presets.
argument-hint: [schema|presets|all]
---

# schema-ref

Output canonical XAppAdKit JSONC schema + preset reference. Use when user asks about schema fields, valid enums, or wants to view presets.

## Steps

1. Read `$CLAUDE_PLUGIN_ROOT/skills/schema-ref/references/schema-0.12.5.md` and/or `$CLAUDE_PLUGIN_ROOT/skills/schema-ref/references/presets.md` based on argument:
   - `schema` → schema file only
   - `presets` → presets file only
   - `all` (default if no arg) → both

2. Print contents verbatim to user. Do not summarize unless they ask.

3. If user asks follow-up clarification on a specific field, answer from the file content. Do not invent fields not in the reference.

## Notes

The schema file mirrors `com.xapp.adkit.config.ConfigRepository` DTOs at git tag v0.12.5. Update this reference when SDK version bumps.
