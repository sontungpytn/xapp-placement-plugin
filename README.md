# xappsdk — Claude Code marketplace

Standalone [Claude Code](https://docs.claude.com/en/docs/claude-code) marketplace shipping one plugin: **`xapp-placements`** — interactive generators + a proactive validator for Xantus **XAppAdKit (xappsdk)** ad-placement config files.

Targets the Xantus admin import format and matches **SDK `com.xantus:x-app-ad-kit-sdk:0.12.3`** (plugin `v0.8.1`).

Runs on **Claude Code** (CLI) and **Claude Cowork** (desktop app).

## What it does

XAppAdKit reads ad placements from 4 Firebase Remote Config keys, authored as a single JSONC bundle (`xapp_config`, `xapp_ad_units`, `xapp_registry`, `xapp_p_<name>`) plus `_project` admin metadata. Hand-editing that file is error-prone — id regexes, vendor uniqueness, enum sets, native UI shapes, and admin import caps all have to line up with the SDK parser.

`xapp-placements` generates and validates those files so they pass admin import on the first try.

## Install

In Claude Code:

```
/plugin marketplace add sontungpytn/xapp-placement-plugin
/plugin install xapp-placements@xappsdk
```

Restart the session, then verify with `/help` → look for `/xapp-placements:*` skills.

Update later:

```
/plugin marketplace update xappsdk
/plugin update xapp-placements@xappsdk
```

Other methods (local clone, manual cache, Cowork upload, uninstall): see [INSTALL.md](INSTALL.md).

## Skills

| Command | Purpose |
|---|---|
| `/xapp-placements:create-config` | Interactive wizard — scaffold a full JSONC from scratch (project meta + global config + ad units + placements). Starter preset = 4 standard AdMob units (appopen, inter, native, reward) with TEST IDs. |
| `/xapp-placements:add-placement` | Append 1 placement to an existing file. Auto-updates `xapp_registry`. Native template via named preset or screenshot input. |
| `/xapp-placements:add-ad-unit` | Append 1 ad unit to the pool. Enforces id regex + vendor uniqueness. |
| `/xapp-placements:validate` | Run the `xapp-validator` agent against a file. |
| `/xapp-placements:schema-ref` | Print the canonical SDK 0.12.3 schema reference. |

**Agent `xapp-validator`** runs **proactively** after every config write — catches schema violations before they reach admin import.

## Output convention

Default file path: `./<app_code>-ad-placements.jsonc` (CWD). Customizable per skill arg.

## SDK version

Pinned to `com.xantus:x-app-ad-kit-sdk:0.12.3`. When the SDK schema changes, bump the plugin and re-sync the schema reference.

Latest changes synced (`v0.8.x`, SDK 0.12.3):
- `xapp_ad_units[*].reload_after_show_delay_ms` — delay (ms) before the reload-after-show buffer refill (default 0; admin caps 0..60000).
- `xapp_config.firebase_ad_impression_enabled` — kill-switch for the `ad_impression` Firebase event (default true; generator emits `false` to avoid GA4 `totalAdRevenue` double-count when AdMob↔GA4 linking is on).
- `xapp_p_<name>.ui_config_triggered` — optional second native `ui_config` used when rendering with `triggered = true` (NATIVE only).
- Telemetry (0.12.1, no config impact): `screen_show`/`screen_exit` → `x_app_screen_show`/`x_app_screen_exit`; always-on `ad_revenue` event added.

Full per-version history: [xapp-placements/README.md](xapp-placements/README.md#changelog).

## Repo layout

```
.
├── .claude-plugin/marketplace.json   # marketplace manifest (name: xappsdk)
├── INSTALL.md                        # all install methods
├── README.md                         # this file
└── xapp-placements/                  # the plugin
    ├── .claude-plugin/plugin.json
    ├── README.md                     # plugin detail + full changelog
    ├── agents/xapp-validator.md
    └── skills/                       # create-config, add-placement, add-ad-unit, validate, schema-ref
```

## Author

Stephen Dong — `stephen@xantus.network`. Proprietary (Xantus).
