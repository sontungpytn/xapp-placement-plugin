---
name: add-ad-unit
description: Append a single ad unit to the `xapp_ad_units` pool of an existing XAppAdKit JSONC config. Enforces id regex, vendor_id uniqueness, mediation enum (ADMOB active at runtime; SDK 0.18.0), format enum. Auto-fills `_meta` from git config user.email + current UTC timestamp. Invokes `xapp-validator` after write. Use when user says "add ad unit", "thêm ad unit", "add new vendor unit", "add admob unit to config".
argument-hint: [path-to-jsonc-file]
---

# add-ad-unit

You are appending a single ad unit to the `xapp_ad_units` pool of an existing config file. Be terse.

## Step 0 — Locate file

Same as `add-placement` Step 0.

## Step 1 — Load + parse pool

Read file. Read `$CLAUDE_PLUGIN_ROOT/skills/schema-ref/references/schema-0.18.0.md`.

Extract existing ad_unit `id`s + `vendor_id`s from the `xapp_ad_units` array (text scan OK).

## Step 2 — Collect ad unit spec

Ask user (`AskUserQuestion`):
- `id` — regex `^[a-z][a-z0-9_]*$`. HARD-BLOCK if duplicate of existing.
- `vendor_id` — HARD-BLOCK if duplicate of existing.
- `mediation` — enum `ADMOB | MAX | IRONSOURCE | META` (case-insensitive). Only `ADMOB` is active at runtime; MAX/IRONSOURCE/META are reserved (META requires a `meta_config` object or the unit is skipped). Do not ask user; auto-fill `ADMOB`.
- `format` — enum: `appopen | inter | native | banner | reward | rewinter`.
- `buffer_size` — int >= 1. Default 1 for ALL formats. Only go higher when the user explicitly asks — oversized buffers preload ads that never show, tanking show rate.
- `preload_trigger` — `INIT_CRITICAL | INIT_DELAYED | LAZY | SCREEN`. Default INIT_DELAYED. **NEW 0.12.0**: `SCREEN` triggers preload on specific screen events — when selected, also collect `preload_on_screens` (see below).
- `preload_on_screens` — **NEW 0.12.0**: required (not absent) when `preload_trigger: "SCREEN"` (else the unit never preloads). Array of `{ "screen_name": <string, required non-blank>, "delay_ms": <int 0..60000, default 0>, "once": <bool, default true> }`. Each entry fires preload when the named screen is entered (after `delay_ms` ms; `once: true` = only on first visit per session). Omit entirely when `preload_trigger` ≠ `SCREEN` (SDK ignores it + strips).
- `auto_reload_on_show` — boolean, default true.
- `reload_after_show_delay_ms` — **NEW 0.12.3**: long ≥ 0, default 0 (= immediate refill). Delay before the reload-after-show buffer refill. Only meaningful when `auto_reload_on_show: true`. Effective delay = max(this, fullscreen auto-delay from the placement's `min_interval_sec`). Omit when 0 (default). Admin caps 0..60000 — do NOT exceed 60000 (import rejects).
- `reload_interval_sec` — int ≥ 0. Default 0 for ALL formats (0 = off). Set a positive value only when the user explicitly wants banner/native timer-refresh; SDK coerces it to 0 when `meta_mediation_enabled: true`.
- `max_reload_count` — int ≥ 0. Default 0 (0 = unlimited).
- `floor_tag` — `h|m|l|nofloor`, default `nofloor`.
- `enabled` — boolean, default true.
- `http_timeout_ms` — int 5000–30000 or null (default null; omit when not set). Ask only if user wants an AdMob HTTP request cap. Warn-cross-check: must be `<` the consuming placement's `load_timeout_ms` (else SDK WARN — the coroutine wrapper fires before the HTTP cap).
- `load_timeout_ms` — int 1000–60000 or null (default null; omit when not set; SDK 0.16.0+). Per-unit coroutine load timeout: wins over the consuming placement's `load_timeout_ms` (show-path) and the global preload timeout (`xapp_config.preload.load_timeout_ms`, default 30000). Warn-cross-checks: `http_timeout_ms` must be `<` this value when both set; value above a parallel placement's `load_timeout_ms` gets cut by the chain ceiling.
- `media_aspect_ratio` — native units ONLY. `any | landscape | portrait | square` (case-insensitive) or null (default null; omit when not set). Ask only when format is `native` and the user wants to constrain media aspect.

NOTE: `meta_config` is only read when `mediation: "META"` (not active at runtime). Do NOT generate it for ADMOB units.

## Step 3 — Resolve audit metadata

```bash
git config user.email   # → <email>
date -u +%Y-%m-%dT%H:%M:%S.000Z   # → <now>
```

Use for `_meta.createdBy / updatedBy / createdAt / updatedAt`.

## Step 4 — Construct ad_unit block

Field order: `id, vendor_id, mediation, format, floor_tag, enabled, buffer_size, preload_trigger, preload_on_screens, auto_reload_on_show, reload_after_show_delay_ms, reload_interval_sec, max_reload_count, load_timeout_ms, http_timeout_ms, media_aspect_ratio, _meta`. Omit `preload_on_screens` when `preload_trigger` ≠ `SCREEN`. Omit `reload_after_show_delay_ms` when 0 (default). Omit `http_timeout_ms` / `media_aspect_ratio` when unset (default null). Omit `load_timeout_ms` when unset (default null).

Show user assembled block. Confirm.

## Step 5 — Insert into file

Use `Edit` tool to append a new entry into the `xapp_ad_units` array. Strategy:
- Read the file region containing the array's closing `]`.
- Locate the line of the LAST existing ad_unit's closing `}` (just before the array's `]`).
- Edit replaces `\n    }\n  ],\n` with `\n    },\n    { ...new unit... }\n  ],\n` (adjust indentation to match file).
- If array is empty (`[]`) → replace `[]` with `[\n    { ...new unit... }\n  ]`.

If multiple `]` candidates → fall back to a wider unique anchor including preceding text.

## Step 6 — Validate

Invoke `xapp-validator` via `Task`. Report.

## Step 7 — Summary

```
Added ad_unit <id> (vendor=<vendor_id>, format=<format>) to <path>. Validator: <STATUS>.
```

## Hard rules

- Never accept duplicate `id` or `vendor_id`.
- `mediation` enum is `ADMOB | MAX | IRONSOURCE | META`, but only `ADMOB` is active at runtime (MAX/IRONSOURCE/META reserved; META requires `meta_config`). Always auto-fill `ADMOB`.
- Never generate `meta_config` for ADMOB units — it is only read under `mediation: "META"`.
- Always fill `_meta` block (admin uses for audit).
- Always run validator after write.
