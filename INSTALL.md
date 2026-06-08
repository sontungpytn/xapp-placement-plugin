# xapp-placements — Install

Bundle = standalone Claude Code (CC) marketplace containing 1 plugin: `xapp-placements@xappsdk`.

Works on **Claude Code** (CLI) and **Claude Cowork** (desktop app).

## Method 1 — Claude Code marketplace (recommended)

```bash
# 1. Unzip somewhere stable (NOT in ~/.claude — keep it as a source dir you can update later)
mkdir -p ~/xantus-plugins
unzip xapp-placements-v0.7.0.zip -d ~/xantus-plugins
# Result: ~/xantus-plugins/xappsdk/  (contains .claude-plugin/ + xapp-placements/)
```

In Claude Code:

```
/plugin marketplace add ~/xantus-plugins/xappsdk
/plugin install xapp-placements@xappsdk
```

Restart Claude Code session. Verify: `/help` → look for `/xapp-placements:*` skills.

## Method 2 — Manual cache install (no marketplace)

```bash
unzip xapp-placements-v0.7.0.zip -d /tmp/xappsdk
mkdir -p ~/.claude/plugins/cache/xappsdk/xapp-placements/0.7.0
cp -R /tmp/xappsdk/xappsdk/xapp-placements/* ~/.claude/plugins/cache/xappsdk/xapp-placements/0.7.0/
mkdir -p ~/.claude/plugins/marketplaces/xappsdk/.claude-plugin
cp /tmp/xappsdk/xappsdk/.claude-plugin/marketplace.json ~/.claude/plugins/marketplaces/xappsdk/.claude-plugin/
```

Then in Claude Code: `/plugin install xapp-placements@xappsdk`. Restart.

## Method 3 — Claude Cowork (desktop app upload)

Drag-drop `xappsdk/` dir into Cowork's plugin upload UI. Cowork registers it at `marketplaces/local-desktop-app-uploads/`. Skills + agent load.

## Uninstall

```
/plugin uninstall xapp-placements@xappsdk
/plugin marketplace remove xappsdk
```

## What you get

- `/xapp-placements:create-config` — interactive wizard, scaffold full ad-placement JSONC
- `/xapp-placements:add-placement` — append placement to existing file
- `/xapp-placements:add-ad-unit` — append ad unit to pool
- `/xapp-placements:validate` — validate file against SDK 0.12.0 schema
- `/xapp-placements:schema-ref` — print canonical schema reference
- Agent `xapp-validator` — proactively runs after every config write

SDK pinned: `com.xantus:x-app-ad-kit-sdk:0.12.0`.
