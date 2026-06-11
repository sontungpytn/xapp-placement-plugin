---
name: add-placement
description: Append a single ad placement to an existing XAppAdKit JSONC config file. Interactive — asks name (format-prefix enforced), trigger, capping, native UI preset. Auto-updates `xapp_registry` array. Picks ad_unit from existing pool by format match. Invokes `xapp-validator` agent after write. Use when user says "add placement", "thêm placement", "thêm 1 ad placement", "add ad placement to <file>", or references an existing `*-ad-placements.jsonc` and asks to extend it.
argument-hint: [path-to-jsonc-file]
---

# add-placement

You are appending a single placement to an existing XAppAdKit JSONC file. Be terse.

## Step 0 — Locate target file

If user passed a path argument → use it. Else `Bash: ls *-ad-placements.jsonc 2>/dev/null` in CWD; if 1 file → use it; if 0 → tell user to run `/xapp-placements:create-config` first; if >1 → ask user to pick.

## Step 1 — Load file + schema

Read target file. Read `$CLAUDE_PLUGIN_ROOT/skills/schema-ref/references/schema-0.12.3.md` + `presets.md`.

Parse (mentally — no actual JSON parser needed since we Edit text):
- existing `xapp_ad_units` IDs + their `format`
- existing `xapp_registry` entries
- existing `xapp_p_*` keys

## Step 2 — Collect placement spec

a. Ask `name`. ENFORCE regex `^(inter|native|ncollap|nfull|nbanner|banner|appopen|reward|rewinter)_[a-z0-9_]+$`. HARD-BLOCK invalid. Re-prompt with the rule shown.

b. Reject if `xapp_p_<name>` already exists in file.

c. Detect format from prefix.

d. Find candidate ad_units in pool with matching format. If 0 → STOP, tell user to first run `/xapp-placements:add-ad-unit` for that format. If 1 → auto-pick. If >1 → ask user.

e. Ask `_metadata` fields: `description` (1 sentence), `screen_name`, `trigger_event` (uppercase enum-style e.g. `COLD_START`, `BACK_FROM_DETAIL`), `priority` (high|medium|low), `notes` (optional, allow skip).

f. Capping per format (see `create-config` Step 4d). Use defaults if user says "default".

g. Timing per format from `presets.md` table.

h. NATIVE → run native UI sub-flow (preset name OR screenshot path → infer preset). Defer fine details to admin UI.

## Step 3 — Construct placement block

Build the `xapp_p_<name>: { ... }` block. Fields in order: `name`, `enabled` (true), `ad_chain`, `show_config`, `timing_config`, `reuse_strategy` (if non-default), `segments` (`[]`), `ui_config` (if native), `ui_config_triggered` (if native AND user wants a triggered variant), `_metadata`.

- `ad_chain` emits `load_strategy` (`waterfall` (default) | `parallel_first` | `parallel_auction`), NOT the legacy boolean `parallel_load`. Default to `waterfall` unless the user asks for parallel loading.
- `reuse_strategy` (NEW 0.11.8): enum `own_first` (default) | `reuse_before_load` | `reuse_first`. Placement top-level field. Omit unless the user wants to favor reusing a ready ad over loading own (`reuse_before_load` / `reuse_first`). Only effective when `xapp_config.late_reuse_enabled: true`.
- Legacy note: there is no `provider` field on a placement (SDK drops it silently — do not emit it).
- For NATIVE `ui_config`, `template_id` may be any of the 5 values: `card_media_v1`, `card_no_media_v1`, `banner_horizontal_v1`, `collapsible_v1`, `fullscreen_hero_v1`. `fullscreen_hero_v1` renders as a fullscreen modal (others inline) and supports `skip_delay_sec` (int [0..15], default 5 — close-button delay).
- `ui_config_triggered` (NEW 0.12.3): OPTIONAL second `ui_config` with the SAME shape, NATIVE only. Emit ONLY if the user explicitly wants a distinct look for the triggered render pass (host calls render with `triggered=true`). When present, the SDK prefers it for triggered renders and falls back to `ui_config` otherwise. Omit by default — most placements need only `ui_config`.

Show user the assembled JSONC. Ask confirm.

## Step 4 — Insert into file

Use `Edit` tool. Two edits:

a. **Append to `xapp_registry` array** — Edit replaces the closing `]` of the registry block with `,\n    "xapp_p_<name>"\n  ]`. Inspect existing indentation carefully. If you can't find unique anchor (e.g. multiple `]` lines), use the registry's last entry line as anchor.

b. **Insert placement block** — Find a stable anchor for insertion point. Strategy: insert BEFORE the file's closing `}` (last `}` on its own line). Edit replaces `\n}\n` with `\n,\n  "xapp_p_<name>": { ... assembled block ... }\n}\n`. (Take care: previous entry's closing `}` may or may not have a trailing comma. Inspect — if previous placement block ends `}` without comma before final `}`, add the comma.)

Preferred strategy: read last ~40 lines of file with `Read offset=`, identify exact closing pattern, then `Edit` with full unique block.

## Step 5 — Validate

Invoke `xapp-validator` agent via `Task`. Report result.

## Step 6 — Summary

```
Added placement xapp_p_<name> to <path>. Validator: <STATUS>.
```

## Hard rules

- Never insert duplicate `xapp_p_<name>`.
- Never leave `xapp_registry` out of sync with placement blocks.
- Never skip validator after write.
- Preserve existing file's indentation style (2-space typical).
