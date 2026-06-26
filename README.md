# xappsdk — Claude Code marketplace

Standalone [Claude Code](https://docs.claude.com/en/docs/claude-code) marketplace shipping one plugin: **`xapp-placements`** — interactive generators + a proactive validator for Xantus **XAppAdKit (xappsdk)** ad-placement config files.

Targets the Xantus admin import format and matches **SDK `com.xantus:x-app-ad-kit-sdk:0.13.0`** (plugin `v0.10.0`).

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

Uninstall:

```
/plugin uninstall xapp-placements@xappsdk
/plugin marketplace remove xappsdk
```

Other methods (local clone, manual cache, Cowork upload): see [INSTALL.md](INSTALL.md).

## Skills

| Command | Purpose |
|---|---|
| `/xapp-placements:create-config` | Interactive wizard — scaffold a full JSONC from scratch (project meta + global config + ad units + placements). Starter preset = 4 standard AdMob units (appopen, inter, native, reward) with TEST IDs. |
| `/xapp-placements:add-placement` | Append 1 placement to an existing file. Auto-updates `xapp_registry`. Native template via named preset or screenshot input. |
| `/xapp-placements:add-ad-unit` | Append 1 ad unit to the pool. Enforces id regex + vendor uniqueness. |
| `/xapp-placements:validate` | Run the `xapp-validator` agent against a file. |
| `/xapp-placements:schema-ref` | Print the canonical SDK 0.13.0 schema reference. |

**Agent `xapp-validator`** runs **proactively** after every config write — catches schema violations before they reach admin import.

## Output convention

Default file path: `./<app_code>-ad-placements.jsonc` (CWD). Customizable per skill arg.

## SDK version

Pinned to `com.xantus:x-app-ad-kit-sdk:0.13.0`. When the SDK schema changes, bump the plugin and re-sync the schema reference.

Latest changes synced (`v0.10.0`, SDK 0.13.0):
- `xapp_p_<name>.reuse_chain` (NEW 0.13.0) — optional ordered allowlist of ad units a placement may borrow a buffered ad from at the REUSE tier (same-format, list-order). `{ "entries": [ { "ad_unit_id": "<id>" }, ... ] }`. Empty/absent = no cross-unit borrow.
- `xapp_config.cross_unit_reuse_enabled` semantics CHANGED 0.13.0 — borrow now scoped to `reuse_chain` (list-order, format-matched); old "borrow from ANY unref'd unit" behavior removed. Still a bool kill-switch (default true, gated by `late_reuse_enabled`).
- Cross-unit reuse now covers ALL formats (fullscreen, NATIVE inline, BANNER inline) — was fullscreen-only.
- Late-reuse for NATIVE/BANNER — late-arriving ads retained + buffered (SDK-internal, no new config key).
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
