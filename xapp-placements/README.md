# xapp-placements

Fast generator for XAppAdKit (xappsdk) ad placement config JSONC files. Output matches Xantus admin import format + SDK 0.12.3 schema.

## Install

In Claude Code:

```
/plugin marketplace add sontungpytn/xapp-placement-plugin
/plugin install xapp-placements@xappsdk
```

Restart session, verify `/help` shows `/xapp-placements:*` skills. Other methods (local clone, manual cache, Cowork): see [INSTALL.md](../INSTALL.md).

## What

- `/xapp-placements:create-config` — interactive wizard, scaffold full JSONC from scratch (project meta + global config + ad units + placements). Starter preset = 4 standard AdMob ad units (appopen, inter, native, reward) with TEST IDs.
- `/xapp-placements:add-placement` — append 1 placement to existing JSONC. Auto-update `xapp_registry`. Native templates via named preset OR screenshot input.
- `/xapp-placements:add-ad-unit` — append 1 ad unit to pool. Enforce id regex + vendor uniqueness.
- `/xapp-placements:validate` — invoke `xapp-validator` agent on a file.
- `/xapp-placements:schema-ref` — print canonical SDK 0.12.3 schema reference.

Validator agent (`xapp-validator`) runs **proactively** after every config write — catches schema violations before they hit admin import.

## SDK version

Pinned to `com.xantus:x-app-ad-kit-sdk:0.12.3`. Bump plugin when SDK schema changes.

## Changelog

- **0.8.1** (2026-06-11): Generator now emits `xapp_config.firebase_ad_impression_enabled: false` (was true). Xantus default — AdMob↔GA4 linking pushes ad revenue server-side, so the client-side `ad_impression` event is off to avoid GA4 `totalAdRevenue` double-count. SDK/admin schema default stays true; flip to true only for projects lacking the AdMob↔GA4 link.
- **0.8.0** (2026-06-11): Sync to SDK 0.12.3. New ad-unit `reload_after_show_delay_ms` (long ≥0, default 0; admin caps 0..60000) — delays the reload-after-show buffer refill. New `xapp_config.firebase_ad_impression_enabled` (bool, default true) — kill-switch suppressing ONLY the `ad_impression` Firebase event to avoid GA4 `totalAdRevenue` double-count. New placement `ui_config_triggered` (optional second `ui_config`, same shape) — native render variant for `triggered=true`. NOTE 0.12.1 (telemetry only): Firebase events `screen_show`/`screen_exit` renamed `x_app_screen_show`/`x_app_screen_exit`; SDK also adds an always-on `ad_revenue` event — no config-file impact, but downstream dashboards must migrate.
- **0.7.1** (2026-06-11): Generator default `buffer_size` is now `1` for ALL formats (was appopen=1, inter=2, native=3, reward=2). Oversized buffers preload ads that never show → low show rate. Matches SDK parser default (1). Go higher only on explicit request.
- **0.7.0** (2026-06-08): Sync to SDK 0.12.0. New `preload_trigger: "SCREEN"` (4th value; screen-event-driven preload) + ad-unit `preload_on_screens` array. New `ui_config.fullscreen` block (`layout_order`, `info_bg`, `action_container` with `position`/`duration_sec`/`auto_close`/`close_button{visible,icon}`) — `fullscreen_hero_v1` only. `media_aspect_ratio` gains `auto`.
- **0.6.0** (2026-06-04): Sync to SDK 0.11.9. New native `ui_config.banner` sizing block (`{height_dp default 125, padding_dp default 12}`) — `banner_horizontal_v1` template only; SDK WARNs when `height_dp < 64`. New `ui_config.ad_info.ad_icon.size_dp` (int or null; coerced ≥1 when set; null = per-template default) — advertiser-icon size across all native templates. Generator default `reload_interval_sec` is now `0` for ALL formats (was 30 for native; SDK parser default has always been 0).
- **0.5.0** (2026-06-02): Sync to SDK 0.11.8. New `xapp_config.cross_unit_reuse_enabled` (bool, default true) — cross-unit fullscreen ad borrow, gated by `late_reuse_enabled`. New placement-level `reuse_strategy` (`own_first` default | `reuse_before_load` | `reuse_first`) — show() resolution tier order; admin `z.enum` strict (unknown rejected), SDK falls back to `own_first`. Changed default `preload.vendor_dedupe` true → false (same-vendor units now load in parallel by default).
- **0.4.0** (2026-06-01): Sync to SDK 0.11.5. New xapp_config fields: splash_min_duration_ms, mute_ad_video, use_admob_startpreload, late_reuse_enabled, preload.circuit_threshold + circuit_backoff_sec. New ad_unit fields: http_timeout_ms, media_aspect_ratio. ad_chain.load_strategy (waterfall|parallel_first|parallel_auction) replaces parallel_load. template_id expanded to 5 (added collapsible_v1, fullscreen_hero_v1); fullscreen_hero_v1 renders as modal. New native ui_config knobs: skip_delay_sec, margin, border.visible, collapse_arrow, per-text font_size, ad_icon.corner_radius_dp, cta text_font_size + border.width_sides. Placement-name prefixes add nfull/nbanner; ncollapse renamed ncollap.
- **0.3.0** (2026-05-13): Migrate to SDK 0.8.0. New `xapp_config.min_fullscreen_interval_sec` (default 30) + `launch_cooldown_ms` (default 4000) — Play "Better Ads" policy. New `xapp_ad_units[*].meta_mediation_enabled` flag (banner/native; SDK coerces `reload_interval_sec=0` when true). New `ui_config.border` block (`{color, width_dp}`, AdMob "Ads disguised as content" policy). Removed `ui_config.ad_badge.visible` — badge always renders. `ad_badge.text` whitelist enforced: `{"Ad","Sponsored","Promoted","Quảng cáo"}`.
- **0.2.0** (2026-05-13): Migrate to SDK 0.6.0. Breaking — `Mediation` enum reduced to `ADMOB` only (secondary networks now AdMob waterfall adapters), `AdPlacement.provider` field removed, `AdUnit.meta_config` removed, `NativeTemplateId` expanded to 3 values (`card_media_v1`, `card_no_media_v1`, `banner_horizontal_v1`). New `horizontal_banner` named UI preset.
- **0.1.0** (2026-05-12): Initial release for SDK 0.4.1.

## Output convention

Default file path: `./<app_code>-ad-placements.jsonc` (CWD). Customize per skill arg.

## Author

Stephen — `stephen@xantus.network`
