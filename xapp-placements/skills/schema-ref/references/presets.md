# Presets — Ad Units + Native UI

## Starter ad_unit preset (4 standard AdMob units, TEST IDs)

Use as default `xapp_ad_units` when creating new config from scratch. Replace TEST `vendor_id` before prod ship. `_meta` filled at write time.

NOTE 0.11.5: banner/native ad_units whose AdMob mediation chain in AdMob console includes Meta Audience Network MUST set `meta_mediation_enabled: true`. The starter preset omits this field (defaults false). When user wires Meta into the AdMob waterfall, run `/xapp-placements:add-ad-unit` or hand-edit to set `true` on affected banner/native units only — SDK then auto-coerces `reload_interval_sec=0`.

NOTE on mediation: the enum is `ADMOB | MAX | IRONSOURCE | META`, but only `ADMOB` is active at runtime; `MAX`/`IRONSOURCE`/`META` are reserved (META additionally needs a `meta_config` object or the unit is skipped). Keep generating/recommending `ADMOB`.

Optional new ad_unit knobs (0.11.5): `http_timeout_ms` (int 5000–30000 or null; null default; AdMob `setHttpTimeoutMillis`; must be < the consuming placement's `load_timeout_ms`) and `media_aspect_ratio` (NATIVE units only: `any|landscape|portrait|square`, default null). The starter preset omits both (null defaults).

```jsonc
[
  {
    "id": "admob_appopen",
    "vendor_id": "ca-app-pub-3940256099942544/9257395921",
    "mediation": "ADMOB",
    "format": "appopen",
    "floor_tag": "nofloor",
    "enabled": true,
    "buffer_size": 1,
    "preload_trigger": "INIT_CRITICAL",
    "auto_reload_on_show": true,
    "reload_interval_sec": 0,
    "max_reload_count": 0
  },
  {
    "id": "admob_inter",
    "vendor_id": "ca-app-pub-3940256099942544/1033173712",
    "mediation": "ADMOB",
    "format": "inter",
    "floor_tag": "nofloor",
    "enabled": true,
    "buffer_size": 1,
    "preload_trigger": "INIT_CRITICAL",
    "auto_reload_on_show": true,
    "reload_interval_sec": 0,
    "max_reload_count": 0
  },
  {
    "id": "admob_native",
    "vendor_id": "ca-app-pub-3940256099942544/2247696110",
    "mediation": "ADMOB",
    "format": "native",
    "floor_tag": "nofloor",
    "enabled": true,
    "buffer_size": 1,
    "preload_trigger": "INIT_CRITICAL",
    "auto_reload_on_show": true,
    "reload_interval_sec": 0,
    "max_reload_count": 0
  },
  {
    "id": "admob_reward",
    "vendor_id": "ca-app-pub-3940256099942544/5224354917",
    "mediation": "ADMOB",
    "format": "reward",
    "floor_tag": "nofloor",
    "enabled": true,
    "buffer_size": 1,
    "preload_trigger": "LAZY",
    "auto_reload_on_show": true,
    "reload_interval_sec": 0,
    "max_reload_count": 0
  }
]
```

AdMob test app ID: `ca-app-pub-3940256099942544~3347511713` (use in AndroidManifest `<meta-data android:name="admob_app_id">`).

## xapp_config preset

```jsonc
{
  "enabled": true,
  "event_sink_enabled": true,
  "debug_mode": false,
  "min_fullscreen_interval_sec": 30,
  "launch_cooldown_ms": 4000,
  "splash_min_duration_ms": 1000,
  "mute_ad_video": false,
  "use_admob_startpreload": false,
  "late_reuse_enabled": true,
  "cross_unit_reuse_enabled": true,
  "firebase_ad_impression_enabled": false,
  "preload": {
    "max_concurrent_loads": 3,
    "init_delayed_after_ms": 3000,
    "vendor_dedupe": false,
    "vendor_dedupe_spacing_ms": 800,
    "circuit_threshold": 3,
    "circuit_backoff_sec": 60
  }
}
```

`debug_mode` = true for dev firebase project; false for prod.

`min_fullscreen_interval_sec`: cross-placement fullscreen-ad spacing. 30s default per Braly policy. 0 disables.

`launch_cooldown_ms`: cooldown after SDK READY before INTERSTITIAL `show()` allowed. 4000ms default per Play "Better Ads". 0 disables.

`splash_min_duration_ms` (NEW 0.11.5): long, default 1000, bounds [0, 15000] (out-of-range clamps to default). Min splash dwell to align the app-open ad load window vs launch latency.

`mute_ad_video` (NEW 0.11.5): bool, default false. Process-wide AdMob video mute; one-way within a session (SDK never un-mutes).

`use_admob_startpreload` (NEW 0.11.5): bool, default false. When true, AdMob fullscreen units (inter/reward/appopen) delegate preload to AdMob; `reload_interval_sec`/`max_reload_count`/`auto_reload_on_show`/non-INIT_CRITICAL `preload_trigger` become no-ops (SDK warns).

`late_reuse_enabled` (NEW 0.11.5): bool, default true. Kill-switch for reusing a "late" loaded ad from the unit's buffer instead of live-loading.

`cross_unit_reuse_enabled` (NEW 0.11.8): bool, default true. When true (AND `late_reuse_enabled` also true), a buffered fullscreen ad in one unit may be borrowed by a placement that does NOT reference that unit, matched by AdFormat — last tier before live-load. Revenue stays with the origin unit.

`firebase_ad_impression_enabled` (NEW 0.12.3): bool. **SDK/admin schema default true; generator emits `false`.** When false, the SDK suppresses ONLY the `ad_impression` Firebase event (all other ad events still fire) — Xantus default is to let AdMob↔GA4 linking push ad revenue server-side, so the SDK-side event is off to avoid double-counting GA4 `totalAdRevenue`. Set true only if a project needs the client-side `ad_impression` event (e.g. no AdMob↔GA4 link).

`preload.vendor_dedupe` (DEFAULT CHANGED 0.11.8 → false, was true): when true, units sharing a `vendor_id` load sequentially (spaced by `vendor_dedupe_spacing_ms`); when false they load in parallel. Default now false.

`preload.circuit_threshold` (NEW 0.11.5): int coerced ≥1, default 3. Consecutive NO_FILL count that trips the per-unit circuit breaker.

`preload.circuit_backoff_sec` (NEW 0.11.5): int coerced ≥1, default 60. Value is in SECONDS (SDK stores internally as ms = sec×1000) — how long the breaker stays open before retry.

## Placement timing defaults per format

| Format | `load_timeout_ms` | `show_timeout_ms` | `modal_loading.enabled` |
|---|---|---|---|
| appopen | 8000 | 3000 | (forbidden — SDK blocks) |
| inter | 8000 | 3000 | true (max_wait_ms 3000) |
| reward | 10000 | 3000 | true (max_wait_ms 5000) |
| rewinter | 8000 | 3000 | true (max_wait_ms 3000) |
| banner | 5000 | 3000 | (omit) |
| native | 6000 | 3000 | (omit) |

## Native UI presets (5 named — pick by name)

SDK 0.12.3 supports 5 `template_id` values. Pick template by content shape; the named preset below selects template + color scheme + layout knobs together.

- `card_media_v1` — card with hero media. Renders inline. Use for detail/preview screens.
- `card_no_media_v1` — card, no media. Renders inline. Use for compact promo / success screens.
- `banner_horizontal_v1` — narrow horizontal strip (icon + text + CTA inline). Renders inline. Use for feed rows / list slots.
- `collapsible_v1` (NEW) — banner that expands to a full card (uses the `collapse_arrow` block). Renders inline. Use for `ncollap_*` collapsible native placements.
- `fullscreen_hero_v1` (NEW) — fullscreen modal takeover (launches the modal Activity, honors `skip_delay_sec`). All other templates render inline. Use for `nfull_*` interstitial-style native placements.

All native ads in SDK 0.12.3 render with a border (AdMob "Ads disguised as content" policy). Each preset below carries an explicit `border` block matching the card background tone. The border block now also has a `visible` field (default true) alongside `color` + `width_dp`; the presets omit `visible`, so `"visible": true` is the implicit default. Omit the whole block to fall back to SDK default `{visible: true, color: "#E0E0E0", width_dp: 1}`.

`ad_badge.visible` does NOT exist in SDK 0.11.9 — badge always renders. Presets do not include the field. `ad_badge.text` accepts only `{"Ad", "Sponsored", "Promoted", "Quảng cáo"}` — any other value normalizes to "Ad" at parse with WARN.

Optional per-element styling knobs — add to a preset where it improves it, otherwise omit to take defaults:
- `ad_title`/`ad_body`/`ad_star_rating`/`ad_price` accept `font_size` (sp, coerced ≥1). (0.11.5)
- `ad_icon` accepts `corner_radius_dp` (int ≥0, default 0). (0.11.5)
- `ad_icon` accepts `size_dp` (int or null; coerced ≥1 when set; null = per-template default). (NEW 0.11.9)
- `cta_button.style` accepts `text_font_size` (sp, default 16, coerced ≥1). (0.11.5)
- `cta_button.style.border` accepts `width_sides` (`[t,r,b,l]` ints ≥0, or null; outline only — overrides `width` when valid). (0.11.5)
- `banner` block (`{height_dp default 125, padding_dp default 12}`) sizes the `banner_horizontal_v1` template ONLY — other templates ignore it. WARN if `height_dp < 64`. (NEW 0.11.9)

User can also pass `screenshot:<path>` to skill — Claude reads image, picks closest preset, may adjust `background`/CTA colors to match.

### Preset 1: `card_with_media` (light card, full media, orange gradient CTA)

For: detail screens, preview screens. Sticky native at screen bottom.

```jsonc
{
  "template_id": "card_media_v1",
  "background": "#FFFFFF",
  "border_radius": [12, 12, 12, 12],
  "border": { "color": "#E0E0E0", "width_dp": 1 },
  "ad_info": {
    "visible": true,
    "ad_icon": { "visible": true },
    "ad_title": { "visible": true, "text_color": "#000000" },
    "ad_body":  { "visible": true, "text_color": "#B3000000" },
    "ad_badge": { "background_color": null, "text_color": null, "text": "Ad" }
  },
  "ad_metadata": {
    "visible": false,
    "ad_star_rating": { "visible": true },
    "ad_price":       { "visible": true }
  },
  "ad_media": { "visible": true, "aspect_ratio": "16:9" },
  "cta_button": {
    "visible": true,
    "placement": "bottom",
    "style": {
      "type": "solid",
      "background": {
        "solid": null,
        "gradient": { "colors": ["#FF6200", "#FF9500"], "angle": 135 }
      },
      "border_radius": 10,
      "height": 48,
      "text_color": "#FFFFFF"
    }
  }
}
```

### Preset 2: `compact_no_media` (light card, no media, green gradient CTA, install_success)

For: install_success / download_success screens. Compact promo card.

```jsonc
{
  "template_id": "card_no_media_v1",
  "background": "#FFFFFF",
  "border_radius": [16, 16, 16, 16],
  "border": { "color": "#E0E0E0", "width_dp": 1 },
  "ad_info": {
    "visible": true,
    "ad_icon": { "visible": true },
    "ad_title": { "visible": true, "text_color": "#000000" },
    "ad_body":  { "visible": true, "text_color": "#B3000000" },
    "ad_badge": { "background_color": null, "text_color": null, "text": "Ad" }
  },
  "ad_metadata": {
    "visible": false,
    "ad_star_rating": { "visible": true },
    "ad_price":       { "visible": true }
  },
  "ad_media": { "visible": false, "aspect_ratio": "16:9" },
  "cta_button": {
    "visible": true,
    "placement": "bottom",
    "style": {
      "type": "solid",
      "background": {
        "solid": null,
        "gradient": { "colors": ["#4CAF50", "#2E7D32"], "angle": 135 }
      },
      "border_radius": 28,
      "height": 56,
      "text_color": "#FFFFFF"
    }
  }
}
```

### Preset 3: `dark_card_solid` (dark card, no media, solid green CTA top)

For: dark-mode detail screens, theme detail with dark surface.

```jsonc
{
  "template_id": "card_no_media_v1",
  "background": "#1F1F22",
  "border_radius": [16, 16, 16, 16],
  "border": { "color": "#3A3D45", "width_dp": 1 },
  "ad_info": {
    "visible": true,
    "ad_icon": { "visible": true },
    "ad_title": { "visible": true, "text_color": "#FFFFFF" },
    "ad_body":  { "visible": true, "text_color": "#A0A4AB" },
    "ad_badge": { "background_color": "#3A3D45", "text_color": "#FFFFFF", "text": "Ad" }
  },
  "ad_metadata": {
    "visible": false,
    "ad_star_rating": { "visible": true },
    "ad_price":       { "visible": true }
  },
  "ad_media": { "visible": false, "aspect_ratio": "16:9" },
  "cta_button": {
    "visible": true,
    "placement": "top",
    "style": {
      "type": "solid",
      "background": { "solid": "#22C55E", "gradient": null },
      "border_radius": 28,
      "height": 56,
      "text_color": "#FFFFFF"
    }
  }
}
```

### Preset 4: `dark_compact` (dark compact card, no media, green gradient pill CTA)

For: dark modal sheets, bottom sheets with dark backdrop.

```jsonc
{
  "template_id": "card_no_media_v1",
  "background": "#1F1F1F",
  "border_radius": [16, 16, 16, 16],
  "border": { "color": "#3A3D45", "width_dp": 1 },
  "ad_info": {
    "visible": true,
    "ad_icon": { "visible": true },
    "ad_title": { "visible": true, "text_color": "#000000" },
    "ad_body":  { "visible": true, "text_color": "#B3000000" },
    "ad_badge": { "background_color": null, "text_color": null, "text": "Ad" }
  },
  "ad_metadata": {
    "visible": false,
    "ad_star_rating": { "visible": true },
    "ad_price":       { "visible": true }
  },
  "ad_media": { "visible": false, "aspect_ratio": "16:9" },
  "cta_button": {
    "visible": true,
    "placement": "bottom",
    "style": {
      "type": "solid",
      "background": {
        "solid": null,
        "gradient": { "colors": ["#4CAF50", "#2E7D32"], "angle": 135 }
      },
      "border_radius": 24,
      "height": 48,
      "text_color": "#FFFFFF"
    }
  }
}
```

### Preset 5: `horizontal_banner` (banner row, icon + title + body + CTA inline)

For: feed rows, list item slots, narrow horizontal ad strips. Uses `banner_horizontal_v1` template.

```jsonc
{
  "template_id": "banner_horizontal_v1",
  "background": "#FFFFFF",
  "border_radius": [8, 8, 8, 8],
  "border": { "color": "#E0E0E0", "width_dp": 1 },
  "ad_info": {
    "visible": true,
    "ad_icon": { "visible": true },
    "ad_title": { "visible": true, "text_color": "#000000" },
    "ad_body":  { "visible": true, "text_color": "#B3000000" },
    "ad_badge": { "background_color": null, "text_color": null, "text": "Ad" }
  },
  "ad_metadata": {
    "visible": false,
    "ad_star_rating": { "visible": true },
    "ad_price":       { "visible": true }
  },
  "ad_media": { "visible": false, "aspect_ratio": "16:9" },
  "cta_button": {
    "visible": true,
    "placement": "bottom",
    "style": {
      "type": "solid",
      "background": {
        "solid": null,
        "gradient": { "colors": ["#1E88E5", "#1565C0"], "angle": 135 }
      },
      "border_radius": 8,
      "height": 40,
      "text_color": "#FFFFFF"
    }
  }
}
```

## Screenshot → preset classifier hints

When user provides `screenshot:<path>`, Read the image and pick by these signals:

| Signal | Preset | template_id |
|---|---|---|
| Hero image visible at top of ad card | `card_with_media` | `card_media_v1` |
| Light card, no hero image, gradient CTA at bottom | `compact_no_media` | `card_no_media_v1` |
| Dark surface, white text, solid colored CTA at TOP of card | `dark_card_solid` | `card_no_media_v1` |
| Dark surface, no image, gradient pill CTA at bottom | `dark_compact` | `card_no_media_v1` |
| Narrow horizontal strip, icon + text + CTA inline | `horizontal_banner` | `banner_horizontal_v1` |

Adjust `background` hex + CTA `colors` to roughly match dominant accent if visible. Defer detail tuning to admin UI.
