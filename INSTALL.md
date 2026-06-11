# xapp-placements — Install

Repo = standalone Claude Code (CC) marketplace containing 1 plugin: `xapp-placements@xappsdk`.

Works on **Claude Code** (CLI) and **Claude Cowork** (desktop app).

## Method 1 — Install from GitHub (recommended)

In Claude Code:

```
/plugin marketplace add sontungpytn/xapp-placement-plugin
/plugin install xapp-placements@xappsdk
```

Restart Claude Code session. Verify: `/help` → look for `/xapp-placements:*` skills.

Update later:

```
/plugin marketplace update xappsdk
/plugin update xapp-placements@xappsdk
```

> Private repo? Use the SSH URL instead (requires GitHub SSH key configured):
> `/plugin marketplace add git@github.com:sontungpytn/xapp-placement-plugin.git`

## Method 2 — Local clone / zip

```bash
# Clone (or unzip a release) somewhere stable — NOT in ~/.claude
git clone git@github.com:sontungpytn/xapp-placement-plugin.git ~/xantus-plugins/xappsdk
```

In Claude Code:

```
/plugin marketplace add ~/xantus-plugins/xappsdk
/plugin install xapp-placements@xappsdk
```

Restart Claude Code session.

## Method 3 — Manual cache install (no marketplace)

```bash
git clone git@github.com:sontungpytn/xapp-placement-plugin.git /tmp/xappsdk
mkdir -p ~/.claude/plugins/cache/xappsdk/xapp-placements/0.8.1
cp -R /tmp/xappsdk/xapp-placements/* ~/.claude/plugins/cache/xappsdk/xapp-placements/0.8.1/
mkdir -p ~/.claude/plugins/marketplaces/xappsdk/.claude-plugin
cp /tmp/xappsdk/.claude-plugin/marketplace.json ~/.claude/plugins/marketplaces/xappsdk/.claude-plugin/
```

Then in Claude Code: `/plugin install xapp-placements@xappsdk`. Restart.

## Method 4 — Claude Cowork (desktop app upload)

Drag-drop the cloned repo dir into Cowork's plugin upload UI. Cowork registers it at `marketplaces/local-desktop-app-uploads/`. Skills + agent load.

## Uninstall

```
/plugin uninstall xapp-placements@xappsdk
/plugin marketplace remove xappsdk
```

## What you get

- `/xapp-placements:create-config` — interactive wizard, scaffold full ad-placement JSONC
- `/xapp-placements:add-placement` — append placement to existing file
- `/xapp-placements:add-ad-unit` — append ad unit to pool
- `/xapp-placements:validate` — validate file against SDK 0.12.3 schema
- `/xapp-placements:schema-ref` — print canonical schema reference
- Agent `xapp-validator` — proactively runs after every config write

SDK pinned: `com.xantus:x-app-ad-kit-sdk:0.12.3`.
