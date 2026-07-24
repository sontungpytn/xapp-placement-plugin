---
name: create-config
description: Interactive wizard to scaffold a new XAppAdKit (xappsdk) ad placement config JSONC file from scratch. Asks for project metadata (id, name, package_name, app_code, firebase_project_id), bundles starter ad-unit preset (4 AdMob TEST units — appopen/inter/native/reward), then loops through user-defined placements with format-prefix enforcement + native UI preset picker. Writes file to CWD as `<app_code>-ad-placements.jsonc`. Auto-invokes `xapp-validator` agent after write. Use when user says "create xappsdk config", "scaffold ad placements", "new ad placement file", "tạo file ad placement", "tạo config xappsdk".
argument-hint: [output-path]
---

# create-config

You are scaffolding a new XAppAdKit JSONC config file. The user is a Xantus dev or MO team member. Follow steps strictly. Be terse — caveman mode if active.

## Step 0 — Load schema reference

Read `$CLAUDE_PLUGIN_ROOT/skills/schema-ref/references/schema-0.16.7.md` and `$CLAUDE_PLUGIN_ROOT/skills/schema-ref/references/presets.md` into your context. You will need them throughout.

## Step 1 — Collect project metadata

Ask user (use `AskUserQuestion` for each, batched if possible — never assume):
- `id` (lowercase slug, e.g. `controlkit-dev`, `dailydrama`)
- `name` (display name)
- `package_name` (Android applicationId — verify exists by `Bash: grep applicationId <cwd>/app/build.gradle.kts 2>/dev/null` if file present; mismatch → ask user to confirm)
- `app_code` (2-4 letter code — used for default output filename)
- `firebase_project_id` (e.g. `controlkit-app-dev` for dev, `controlkit-prod` for prod)

Default output filename = `<app_code>-ad-placements.jsonc` in CWD. Override with skill argument if provided.

If output file already exists → STOP. Tell user to use `/xapp-placements:add-placement` or `/xapp-placements:add-ad-unit` instead, OR confirm overwrite explicitly.

## Step 2 — Resolve author metadata

Run `Bash: git config user.email`. Capture output as `<email>`. If empty → use `unknown@xantus.network`. Run `Bash: date -u +%Y-%m-%dT%H:%M:%S.000Z`. Capture as `<now>`.

Use `<email>` for `_meta.createdBy` / `_meta.updatedBy` on every ad_unit. Use `<now>` for `_meta.createdAt` / `_meta.updatedAt`.

## Step 3 — Bundle starter ad-unit preset

Use the 4-unit preset from `presets.md` (`admob_appopen`, `admob_inter`, `admob_native`, `admob_reward`). Inject `_meta` on each (Step 2 values).

Ask user: "Use TEST vendor_ids (default) or paste prod AdMob unit IDs now?" If prod → prompt 4 IDs in one batch. Validate each looks like `ca-app-pub-\d+/\d+`.

Ad_unit fields (0.11.5+) — allow but do not block on them; default sensibly (omit or null) unless user supplies:
- `http_timeout_ms`: int 5000–30000 or null (default null). If set, must be `<` consuming placement's `load_timeout_ms`.
- `load_timeout_ms`: int 1000–60000 or null (default null; SDK 0.16.0+). Per-unit coroutine load timeout — overrides placement `load_timeout_ms` per chain entry and the 10s preload default.
- `media_aspect_ratio`: NATIVE units only — `any|landscape|portrait|square` or null (default null).

## Step 4 — Loop: define placements

Repeat until user says "done":

a. Ask placement `name` (must match regex `^(inter|native|ncollap|nfull|nbanner|banner|appopen|reward|rewinter)_[a-z0-9_]+$`).
   - HARD-BLOCK invalid name. Show the format prefix rule. Re-prompt.
   - Detect format from prefix.

b. Pick ad_unit from pool by format match. If only 1 candidate → auto-pick. If multiple → ask user to pick by `id`.

c. Ask: `trigger_event`, `screen_name`, short `description` (1 sentence), `priority` (high|medium|low) for `_metadata` block.

d. Ask capping based on format:
   - `appopen`: usually `session: 1` (cold) or `min_interval_sec: 30` (resume). Ask `cold` vs `resume`.
   - `inter`: `hourly`, `daily`, `session`, `min_interval_sec`. Defaults 4/10/5/90 if user says "default".
   - `reward` / `rewinter`: `session` only (e.g. 20). `modal_loading.enabled=true, max_wait_ms=5000`.
   - `native`: `rotation_interval_sec` (sticky 30-60) OR `capping.session` (modal/install_success). Ask which mode.
   - `banner`: similar to native.

e. Apply timing defaults from `presets.md` table (per format). Override only if user requests.

f. If format == `native` → run **native UI sub-flow**:
   - Ask: "Pick preset (`card_with_media`, `compact_no_media`, `dark_card_solid`, `dark_compact`) OR paste screenshot path."
   - If preset name → load matching block from `presets.md`. Optionally ask user to adjust `background` hex or CTA color.
   - If screenshot path → Use Read tool on image. Use signals from `presets.md` "Screenshot → preset classifier hints" table to pick best preset. Optionally adjust `background` + CTA `colors` to roughly match dominant accent. Tell user which preset matched.
   - Defer fine-grained tuning (badge text, body color, star_rating visibility) — user adjusts in admin UI later.

g. Build `ad_chain` with `load_strategy` (default `"waterfall"`; allow `"parallel_first"` / `"parallel_auction"` if user requests). Do NOT emit the legacy boolean `parallel_load`. Optionally set placement-level `reuse_strategy` (0.11.8: `own_first` default | `reuse_before_load` | `reuse_first`) — omit unless the user wants to favor reusing a ready ad. Optionally set placement-level `reuse_chain` (NEW 0.13.0: optional object `{ "entries": [ { "ad_unit_id": "<id>" } ] }`) — omit unless the user wants to specify an ordered allowlist of units to borrow from at the REUSE tier (same-format only; empty/absent = no cross-unit borrow). Confirm placement block with user (show generated JSONC for this placement). Append to working memory.

h. Ask "Add another placement? (yes / done)".

## Step 5 — Build registry

`xapp_registry` = sorted list of `xapp_p_<name>` keys (alphabetical for stable diffs).

## Step 6 — Assemble + write

Build full JSONC structure in this exact order:
1. Top JSONC header comment (3-4 lines: project name, SDK version 0.16.7, generated date, generator = `xapp-placements plugin`).
2. `_project` block.
3. `xapp_config` block (use preset from presets.md; `debug_mode: true` if firebase_project_id contains `-dev`, else `false`). Include the globals with their defaults: `splash_min_duration_ms: 1000`, `mute_ad_video: false`, `use_admob_startpreload: false`, `late_reuse_enabled: true`, `cross_unit_reuse_enabled: true` (NEW 0.11.8), `firebase_ad_impression_enabled: false` (NEW 0.12.3 — generator emits false; SDK/admin schema default is true. Xantus default: AdMob↔GA4 linking pushes ad revenue server-side, so the client-side `ad_impression` event is OFF to avoid GA4 `totalAdRevenue` double-count. Flip to true only if a project lacks the AdMob↔GA4 link), `adapter_init_timeout_ms: 0` (NEW post-0.13.0 — int ms, default 0, coerced [0,30000]; 0 = wait fully for `MobileAds.initialize` before the first ad request, prior behavior; >0 = request ads after the timeout while adapters finish in the background; SDK `GlobalConfig.adapterInitTimeoutMs`), and under `preload`: `vendor_dedupe: false` (default changed to false in 0.11.8), `circuit_threshold: 3`, `circuit_backoff_sec: 60` (value in SECONDS). Allow user overrides but do not block on them.
4. `xapp_ad_units` array.
5. `xapp_registry` array.
6. Each `xapp_p_<name>` block in registry order.

Write file via `Write` tool to resolved output path.

## Step 7 — Validate

Invoke `xapp-validator` agent via `Task` tool: pass the output file path. Wait for report. If ERRORS → tell user the file was written but blocks admin import; suggest re-running specific add-* skills to fix. If WARN-only → tell user safe to import, list warnings.

## Step 8 — Summary

Print one-line summary:
```
Wrote <path>. <N> placements. <M> ad units. Validator: <STATUS>.
```

## Hard rules

- Never invent placement names/triggers/ad_unit IDs without asking. Defaults OK only when user explicitly says "default".
- Enforce format prefix on every placement name. No exceptions.
- Field `name` inside placement block MUST equal suffix after `xapp_p_`.
- Native placements ALWAYS need `ui_config`. Non-native placements MUST NOT have `ui_config`.
- Preserve `_meta` / `_metadata` / `_project` keys verbatim — SDK drops them, admin uses them.
- JSONC comments OK but keep minimal (file header + section dividers). Heavy comments belong in spec doc, not config.
- 2-space indent. UTF-8. Trailing newline.
