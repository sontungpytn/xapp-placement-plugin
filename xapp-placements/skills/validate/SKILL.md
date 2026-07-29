---
name: validate
description: Validate an XAppAdKit (xappsdk) ad placement JSONC config against SDK 0.17.1 schema + admin import rules. Invokes the `xapp-validator` agent and shows its report. Use when user says "validate ad config", "check ad placements", "kiểm tra config xappsdk", "validate <file>.jsonc", or asks for a pre-import safety check on a `*-ad-placements.jsonc` file.
argument-hint: [path-to-jsonc-file]
---

# validate

Run the `xapp-validator` agent on a JSONC config file. Be terse.

## Steps

1. Resolve target file:
   - If user passed a path argument → use it. Verify exists via `Bash: test -f <path> && echo OK`.
   - Else `Bash: ls *-ad-placements.jsonc 2>/dev/null`. If 1 match → use it. If 0 → tell user "no `*-ad-placements.jsonc` in CWD; pass a path." If >1 → ask user to pick.

2. Invoke the `xapp-validator` agent via the `Task` tool:
   - `subagent_type: xapp-validator`
   - `description: "Validate <path>"`
   - `prompt: "Validate the xappsdk ad placement config file at <absolute-path>. Apply SDK 0.17.1 schema + admin import rules. Report errors, warnings, and final status."`

3. Print agent's report verbatim to user. Add no commentary unless agent reported errors — in that case append a one-line action hint:
   - HARD errors → "Fix above before admin import."
   - Warnings only → "Safe to import. Address warnings when possible."
   - Clean → "Ready to import."

## Hard rules

- Do not run the validator inline yourself. Always delegate to the agent (separate context, fresh view).
- Do not modify the file.
