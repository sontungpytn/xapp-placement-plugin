# XAppAdKit JSONC Schema — SDK 0.13.0

Source of truth: `com.xapp.adkit.config.ConfigRepository` parser DTOs at git tag `v0.13.0` of `com.xantus:x-app-ad-kit-sdk`. Schema mirrors the `@SerializedName` JSON keys + parse-time coercion in that file. Cross-project parity invariant: native `template_id` set MUST match `nativeTemplateIdEnum` in `xapp-sdk-web-admin/app/lib/schemas.ts`.

File = 1 JSONC bundling 4 Firebase Remote Config keys + `_project` admin metadata.

## Changes since 0.12.7

| Area | Change |
|---|---|
| `xapp_p_<name>.reuse_chain` | NEW 0.13.0. Optional object `{ "entries": [ { "ad_unit_id": "<id>" }, ... ] }`, default absent / `{entries:[]}`. Ordered allowlist of ad units this placement may BORROW a buffered ad from at the REUSE tier. Borrow walks `entries` in order and takes the first unit whose buffer holds a fresh ad of the SAME format as the placement (format from `ad_chain.entries[0]`). Empty / absent → NO cross-unit borrow (placement uses its own `ad_chain` only). Reuses the `ad_chain` entry shape; `floor_tag` / `load_strategy` are ignored for reuse. Admin schema: `reuseChainSchema = z.object({ entries: z.array(adChainEntrySchema).default([]) })`, optional on the placement. Unknown `ad_unit_id` dropped (same as `ad_chain`). |
| `xapp_config.cross_unit_reuse_enabled` (semantics CHANGED 0.13.0) | Still a bool kill-switch (default true, gated by `late_reuse_enabled`). BUT borrow is now scoped to the placement's `reuse_chain` (list-order, format-matched) — the old "borrow from ANY unit not referenced, matched by AdFormat" behavior is REMOVED (`AdBufferRegistry.borrowOldestForFormat` → `borrowFromChain`). `false`, or an empty `reuse_chain`, → no cross-unit borrow. |
| Cross-unit reuse coverage (CHANGED 0.13.0) | Reuse now applies to ALL formats, not fullscreen-only: fullscreen via `show()`, NATIVE inline via `loadNativeAd()`, BANNER inline via `loadBannerView()` (own-buffer → reuse → live-load). Borrowed inline ad renders under the recipient placement. No new config key beyond `reuse_chain`. |
| Late-reuse for native/banner (NEW 0.13.0) | Late-arriving NATIVE / BANNER ads are now retained in the AdMob loader cache + buffered (previously destroyed), so they are reusable like fullscreen late ads. Still gated by `late_reuse_enabled`. SDK-internal (AdMob native/banner loaders + `PreloadManager.onLateReusable`) — no new config key. |
| `xapp_config.adapter_init_timeout_ms` (NEW, post-0.13.0) | Integer ms, default `0`, coerced `[0, 30000]`. Max wait for `MobileAds.initialize` (all adapters) before the first ad request. `0` = wait fully (prior behavior). `>0` = request ads after the timeout while adapters finish in the background. Admin schema: `z.number().int().min(0).max(30000).default(0)`. SDK `GlobalConfig.adapterInitTimeoutMs`. |

## Changes since 0.12.5

| Area | Change |
|---|---|
| `ui_config.collapse_arrow.targets` | NEW 0.12.6. Array of strings, default `["media"]`. `collapsible_v1` template ONLY. Which sections collapse when the arrow is tapped — subset of `{"media", "cta"}` (case-sensitive). Invalid tokens dropped + WARN; empty after filtering → default `["media"]`; absent → default `["media"]`. NOTE: the arrow lives inside the media section, so `"media"` in targets makes the arrow disappear with it (one-way collapse); `["cta"]` alone keeps the arrow visible. |
| in-app RC defaults (0.12.7) | NOT a JSONC field. SDK 0.12.7 (`feat/rc-defaults-fast-splash`) adds `AppConfig.remoteConfigDefaults` + `ConfigRepository.applyDefaults()` — the host app passes Remote Config defaults at `init()` so first-run splash reads them without blocking on the network fetch. This is an SDK init-API change in app code, NOT a key in the `xapp_*` RC bundle — generator / validator / this schema are unaffected. |

## Changes since 0.12.3

**None — config schema is byte-identical to 0.12.3.** SDK 0.12.4 and 0.12.5 are native-renderer layout fixes only (CTA + AdChoices kept inside the ad box across all media sizes; banner box no longer jumps on first layout). `ConfigRepository` parser DTOs and every `@SerializedName` key are unchanged. All "NEW 0.12.3" markers below remain accurate as field-introduction history.

## Changes since 0.12.0

| Area | Change |
|---|---|
| `xapp_ad_units[*].reload_after_show_delay_ms` | NEW 0.12.3. Long ≥0, default 0. Delay (ms) before the reload-after-show buffer refill. `0` = immediate. Effective delay at show time = `max(this, fullscreen auto-delay derived from the placement's min_interval_sec)`. Only meaningful when `auto_reload_on_show: true`. SDK coerces `< 0` → 0 (no upper cap). **Admin import caps it 0..60000** (`z.number().int().min(0).max(60000)`) — values > 60000 REJECTED on import even though the SDK tolerates them. |
| `xapp_config.firebase_ad_impression_enabled` | NEW 0.12.3. Bool default true. Remote kill-switch — when `false`, `FirebaseBridge` suppresses the `ad_impression` Firebase event; ALL other ad events still emit. Use to avoid double-counting GA4 `totalAdRevenue` when AdMob↔GA4 linking already feeds ad revenue server-side. Admin schema: `z.boolean().default(true)`. (SDK 0.12.x also emits an always-on `ad_revenue` event alongside the gated `ad_impression` — telemetry only, no config key.) |
| `xapp_p_<name>.ui_config_triggered` | NEW 0.12.3. Optional second `ui_config` (SAME shape as `ui_config`, see §6), default null/absent. Used when the app renders the native ad with `triggered = true` (e.g. after a user action on the same screen) — SDK calls `resolvedUiConfig(triggered)`: `true` prefers `ui_config_triggered`, falling back to `ui_config`; `false` always uses `ui_config`. Null → triggered renders fall back to `ui_config`. NATIVE placements only. |
| analytics event names | NEW 0.12.1 (telemetry, NOT a JSONC field). `screen_show` → `x_app_screen_show`, `screen_exit` → `x_app_screen_exit`. `trackScreenShow`/`trackScreenExit` API + bundle keys + screen-preload matching UNCHANGED. Downstream GA4/BQ dashboards on the old event names must migrate. No config-file impact. |

## Changes since 0.11.9

| Area | Change |
|---|---|
| `xapp_ad_units[*].preload_trigger` | Enum gains fourth value `SCREEN`. Now `INIT_CRITICAL \| INIT_DELAYED \| LAZY \| SCREEN`, default `INIT_DELAYED`. `SCREEN` is first-class (no longer a legacy alias). Legacy mapping updated: `"INIT"` → `INIT_DELAYED` (WARN). **`"SCREEN"` no longer maps to LAZY** — it is parsed as the `SCREEN` enum directly. Unknown values → `INIT_DELAYED` + WARN. `SCREEN` semantics: no init preload; buffer fills only when a bound screen appears via `trackScreenShow(screenName)` matching a `preload_on_screens` entry. Mutually exclusive with INIT_*/LAZY. |
| `xapp_ad_units[*].preload_on_screens` | NEW 0.12.0. Array, default `[]`. Only honored when `preload_trigger == "SCREEN"`. Each entry: `screen_name` (string, REQUIRED), `delay_ms` (int 0..60000, default 0), `once` (bool, default true — fill once per session/process; false = every screen_show). Blank/missing `screen_name` → entry dropped + WARN. `delay_ms` outside 0..60000 → coerced. |
| `ui_config.fullscreen` | NEW 0.12.0. Sub-block under `ui_config`, `fullscreen_hero_v1` template ONLY (other templates ignore it). Controls layout order, info background, and action container (countdown + close button). See §6 for full shape. |
| `xapp_ad_units[*].media_aspect_ratio` | Valid set gains `"auto"`. Now: `any \| landscape \| portrait \| square \| auto` (case-insensitive), default null = ANY. `"auto"` = wrap media to the creative's natural size. |

## Changes since 0.11.8

| Area | Change |
|---|---|
| `ui_config.banner` | NEW. Sizing block for the `banner_horizontal_v1` template ONLY (other templates ignore it). `{ height_dp, padding_dp }`. `height_dp` int default 125, coerced ≥1; SDK WARNs when `height_dp < 64` (ad content may clip — AdMob policy risk). `padding_dp` int default 12, coerced ≥0. Defaults equal the legacy hardcoded values, so omitting the block keeps prior render. |
| `ui_config.ad_info.ad_icon.size_dp` | NEW. Int or null (default null). Advertiser-icon width/height in dp, ALL native templates. Coerced ≥1 when set. null = per-template default: `banner_horizontal_v1` 56, `card_media_v1` 50, `card_no_media_v1` 48, `collapsible_v1` 50, `fullscreen_hero_v1` 48. |
| `xapp_ad_units[*].reload_interval_sec` | Plugin default is now `0` for ALL formats (was 30 for native). SDK parser default has always been 0 (`reloadIntervalSec ?: 0`, coerced ≥0); the generator no longer emits 30 on native units. Set a positive value only when timer-refresh is explicitly wanted (banner/native), and only when `meta_mediation_enabled` is false. |

## Changes since 0.11.5

| Area | Change |
|---|---|
| `xapp_config.cross_unit_reuse_enabled` | NEW. Bool default true. When true (AND `late_reuse_enabled` also true), a buffered fullscreen ad in one unit may be borrowed by a placement that does NOT reference that unit, matched by `AdFormat`. Borrow happens only as the last tier before live-load. Revenue stays with the origin unit. Admin schema: `z.boolean().default(true)`. |
| `xapp_p_<name>.reuse_strategy` | NEW. String, default `own_first`. Placement-level (NOT under `ui_config`). Sets the show() resolution tier order. Enum `own_first` / `reuse_before_load` / `reuse_first` (SDK parse is case-insensitive + trimmed; unknown → `own_first` silently). Admin schema is strict `z.enum([...])` → unknown value REJECTED on import. `own_first`=OWN_BUFFER→OWN_PENDING→REUSE→OWN_LOAD (current behavior); `reuse_before_load`=OWN_BUFFER→REUSE→OWN_PENDING→OWN_LOAD; `reuse_first`=REUSE→OWN_BUFFER→OWN_PENDING→OWN_LOAD. |
| `xapp_config.preload.vendor_dedupe` | DEFAULT CHANGED `true` → `false`. Same-`vendor_id` units now load in parallel by default. `vendor_dedupe_spacing_ms` is ignored while false. |

## Changes since 0.8.0

| Area | Change |
|---|---|
| `xapp_config.splash_min_duration_ms` | NEW. Long default 1000. Bounds [0, 15000]; out-of-range clamps to default. Min splash dwell to align app-open ad load window vs launch latency. |
| `xapp_config.mute_ad_video` | NEW. Bool default false. `MobileAds.setAppMuted(true)+setAppVolume(0f)` process-wide. One-way within a session (SDK never un-mutes). |
| `xapp_config.use_admob_startpreload` | NEW. Bool default false. When true, AdMob fullscreen units (inter/reward/appopen) delegate to AdMob's own preloaders. Makes `reload_interval_sec`/`max_reload_count`/`auto_reload_on_show`/`preload_trigger`≠INIT_CRITICAL no-ops for delegated units (SDK warns). |
| `xapp_config.late_reuse_enabled` | NEW. Bool default true. Kill-switch — reuse a "late" loaded ad from the unit's buffer instead of live-loading. |
| `xapp_config.preload.circuit_threshold` | NEW. Int coerced ≥1, default 3. Consecutive NO_FILL count that trips the per-unit circuit breaker. |
| `xapp_config.preload.circuit_backoff_sec` | NEW. Int coerced ≥1, default 60. **JSON key is in SECONDS** (SDK stores internally as ms = sec×1000). How long the breaker stays open before retry. |
| `xapp_ad_units[*].http_timeout_ms` | NEW. Int, default null. Range 5000–30000; out-of-range → clamped to null + WARN. AdMob `AdRequest.setHttpTimeoutMillis`. Cross-checked: WARN if `http_timeout_ms ≥` consuming placement's `load_timeout_ms`. |
| `xapp_ad_units[*].load_timeout_ms` | **NEW 0.16.0.** Int, default null. Range 1000–60000; out-of-range → clamped to null + WARN. Coroutine-level load timeout for THIS unit: overrides the consuming placement's `load_timeout_ms` per chain entry (show-path) and the fixed 10s preload timeout. Parallel chains: the chain ceiling stays the placement's `load_timeout_ms`. Cross-checks: WARN if `http_timeout_ms ≥` the unit's effective load timeout; WARN if the override exceeds a parallel placement's ceiling. |
| `xapp_ad_units[*].media_aspect_ratio` | NEW. String, default null. Native only. One of `any` / `landscape` / `portrait` / `square` (case-insensitive). Invalid → null + WARN. |
| `xapp_ad_units[*].mediation` | Enum now `ADMOB` / `MAX` / `IRONSOURCE` / `META`. **Only `ADMOB` is active at runtime**; MAX/IRONSOURCE/META are reserved. `META` requires a `meta_config` object or the unit is skipped. Keep generating `ADMOB`. |
| `ad_chain.load_strategy` | NEW. `waterfall` (default) / `parallel_first` / `parallel_auction`. Replaces the boolean `parallel_load` (still accepted as legacy: `true`→`parallel_auction`, `false`/absent→`waterfall`). Emit `load_strategy`, not `parallel_load`. |
| `xapp_p_*.ad_chain.enable_live_load` | **NEW 0.16.0.** Bool, default false. When false, the show-path OWN_LOAD tier (and the parallel_auction floor-wait sub-auction) is skipped — placement shows only preloaded/reused ads — EXCEPT the lazy carveout (primary entry unit `preload_trigger: lazy` + empty buffer + no pending fill → still live-loads). `show()` returns false with outcome `LIVE_LOAD_DISABLED`. `load()` is NOT gated. |
| `template_id` | 5 templates now: `card_media_v1`, `card_no_media_v1`, `banner_horizontal_v1`, `collapsible_v1` (NEW), `fullscreen_hero_v1` (NEW). Render mode is template-driven — `fullscreen_hero_v1` launches the modal Activity; all others render inline. `NATIVE_FULLSCREEN` format no longer exists (collapsed into `native`). |
| `ui_config.skip_delay_sec` | NEW. Int range 0–15, default 5. Native-fullscreen only — seconds before the close (X) button becomes interactive. |
| `ui_config.margin` | NEW. `[top, right, bottom, left]` ints ≥0, default `[0,0,0,0]`. (Wrong length → default.) |
| `ui_config.border.visible` | NEW field on the border block (default true). Border block is `{visible, color, width_dp}`. |
| `ui_config.collapse_arrow` | NEW block — `collapsible_v1` template only. |
| Per-text `font_size` | NEW on `ad_title`/`ad_body`/`ad_star_rating`/`ad_price` (sp, coerced ≥1). |
| `ad_icon.corner_radius_dp` | NEW (int ≥0, default 0). |
| `cta_button.style.text_font_size` | NEW (sp, default 16, coerced ≥1). |
| `cta_button.style.border.width_sides` | NEW (`[t,r,b,l]` ints ≥0, or null). |
| ad_unit `format` parsing | SDK parser rejects `native_fullscreen` (admin migrates to `native` first). Accepts: `inter`/`interstitial`, `native`, `banner`, `appopen`/`app_open`, `reward`/`rewarded`, `rewinter`/`rewarded_interstitial`. |

## Carried over from 0.8.0 (still in effect)

- `min_fullscreen_interval_sec` (int default 30, coerced ≥0), `launch_cooldown_ms` (long default 4000, coerced ≥0).
- `meta_mediation_enabled` (bool default false; banner/native; SDK coerces `reload_interval_sec=0` + WARN when true).
- Native ad card always bordered (AdMob "Ads disguised as content"). `ad_badge.visible` does not exist — badge always renders. `ad_badge.text` whitelist `{"Ad","Sponsored","Promoted","Quảng cáo"}` → non-whitelisted normalized to "Ad" + WARN.

## Top-level shape

```jsonc
{
  "_project": { ... },          // admin import metadata; SDK ignores
  "xapp_config": { ... },       // RC key
  "xapp_ad_units": [ ... ],     // RC key
  "xapp_registry": [ ... ],     // RC key
  "xapp_p_<name1>": { ... },    // RC key per placement
  "xapp_p_<name2>": { ... }
}
```

Unknown fields anywhere = silently dropped by the SDK Gson parser. Convention prefix `_` (e.g. `_meta`, `_metadata`, `_project`) = admin/team consumption only.

## 1. `_project` (admin metadata, required for import)

```jsonc
{
  "id": "<slug>",                              // lowercase slug, unique in admin DB
  "name": "<Display Name>",
  "package_name": "<applicationId>",           // must match Android applicationId
  "app_code": "<2-4 letter code>",             // event-prefix tag (e.g. "ck", "dd")
  "firebase_project_id": "<firebase-prod-id>"
}
```

Admin rejects import if `id` not pre-registered or `package_name` mismatches the project record. Dev flavor does NOT import via admin — manual Firebase Console paste.

## 2. `xapp_config` (global SDK config)

```jsonc
{
  "enabled": true,                  // bool, default true. false = SDK no-op
  "event_sink_enabled": true,       // bool, default true. false = no AdEvent emission
  "debug_mode": false,              // bool, default false
  "min_fullscreen_interval_sec": 30,// int default 30, coerced >=0. Cross-placement fullscreen spacing. 0 disables.
  "launch_cooldown_ms": 4000,       // long default 4000, coerced >=0. Cooldown after READY before INTERSTITIAL show(). 0 disables.
  "splash_min_duration_ms": 1000,   // NEW. long default 1000, bounds [0,15000] (else clamp to default).
  "mute_ad_video": false,           // NEW. bool default false. Process-wide AdMob video mute (one-way per session).
  "use_admob_startpreload": false,  // NEW. bool default false. Delegate AdMob fullscreen preload to MobileAds startPreload.
  "late_reuse_enabled": true,       // NEW. bool default true. Late-load buffer reuse kill-switch.
  "cross_unit_reuse_enabled": true, // NEW 0.11.8 (semantics changed 0.13.0). bool default true. Kill-switch for cross-unit
                                    //   reuse: a placement borrows a buffered ad ONLY from units listed in its reuse_chain
                                    //   (list-order, same-format), at the REUSE tier. ALL formats. Empty reuse_chain = no
                                    //   borrow. Gated by late_reuse_enabled=true. Revenue stays with origin unit.
  "firebase_ad_impression_enabled": true, // NEW 0.12.3. bool default true. Kill-switch: false suppresses the ad_impression
                                    //   Firebase event ONLY (all other ad events still emit). Avoid GA4 totalAdRevenue
                                    //   double-count when AdMob<->GA4 linking already feeds revenue server-side.
  "adapter_init_timeout_ms": 0,     // NEW post-0.13.0. int ms default 0, coerced [0,30000]. Max wait for MobileAds.initialize
                                    //   (all adapters) before first ad request. 0 = wait fully (prior behavior); >0 = request
                                    //   ads after timeout while adapters finish in background. SDK GlobalConfig.adapterInitTimeoutMs.
  "preload": {                      // optional sub-object
    "max_concurrent_loads": 3,      // int [1..8], default 3
    "init_delayed_after_ms": 3000,  // long [0..10000], default 3000
    "vendor_dedupe": false,         // bool, default false (CHANGED 0.11.8, was true). true = same-vendor units load sequentially.
    "vendor_dedupe_spacing_ms": 800,// long [0..5000], default 800
    "circuit_threshold": 3,         // NEW. int coerced >=1, default 3. NO_FILL count that trips breaker.
    "circuit_backoff_sec": 60       // NEW. int coerced >=1, default 60. SECONDS (SDK ×1000 → ms). Breaker open duration.
  }
}
```

When `use_admob_startpreload: true`, the SDK emits WARNs for any AdMob fullscreen unit (inter/reward/appopen) that sets `reload_interval_sec > 0`, `max_reload_count > 0`, `auto_reload_on_show: false`, or `preload_trigger` ≠ `INIT_CRITICAL` — those knobs become no-ops under AdMob-managed preload.

## 3. `xapp_ad_units` (ad unit pool, array)

Each entry = one ad unit. Pool-wide unique `id` AND unique `vendor_id`.

```jsonc
{
  "id": "<id>",                   // REQUIRED. regex ^[a-z][a-z0-9_]*$. unique pool-wide
  "vendor_id": "<network-id>",    // REQUIRED. unique pool-wide (e.g. ca-app-pub-XXX/YYY)
  "mediation": "ADMOB",           // REQUIRED. enum ADMOB|MAX|IRONSOURCE|META (case-insensitive).
                                  //   Only ADMOB active at runtime. META requires meta_config or unit skipped.
  "format": "appopen",            // REQUIRED. one of: inter | native | banner | appopen | reward | rewinter
                                  //   aliases accepted: interstitial, app_open, rewarded, rewarded_interstitial
                                  //   NOTE: "native_fullscreen" REJECTED by SDK — use "native" (template drives fullscreen)
  "floor_tag": "nofloor",         // optional. one of: h | m | l | nofloor (default nofloor)
  "enabled": true,                // optional default true
  "buffer_size": 1,               // int >= 1 (coerced), default 1
  "preload_trigger": "INIT_DELAYED", // enum INIT_CRITICAL | INIT_DELAYED | LAZY | SCREEN. DEFAULT INIT_DELAYED.
                                  //   legacy "INIT"->INIT_DELAYED (WARN); unknown->INIT_DELAYED+WARN.
                                  //   SCREEN (NEW 0.12.0): no init preload; buffer fills on trackScreenShow() matching preload_on_screens. Mutually exclusive with INIT_*/LAZY.
  "preload_trigger_by_segment": {  // NEW (SDK 0.15.0). optional. map<segment, trigger enum>.
                                  //   e.g. "return_user": "INIT_CRITICAL", "new_user": "INIT_DELAYED"
                                  //   Effective trigger at bootstrap = preload_trigger_by_segment[userSegment] ?? preload_trigger.
                                  //   Values use the same enum + legacy/unknown coercion as preload_trigger above. Blank keys dropped.
                                  //   Absent/empty = no override (default). Lets one unit (one vendor_id) preload eager for some
                                  //   segments and lazy for others. No-op for units delegated under use_admob_startpreload.
  },
  "auto_reload_on_show": true,    // bool, default true
  "reload_after_show_delay_ms": 0,// NEW 0.12.3. long >= 0 (coerced), default 0 (=immediate). Delay before reload-after-show
                                  //   buffer refill. Effective = max(this, fullscreen auto-delay from placement min_interval_sec).
                                  //   Only meaningful when auto_reload_on_show=true. Admin caps 0..60000 (rejects > 60000).
  "reload_interval_sec": 0,       // int >= 0 (coerced). 0 = off. banner/native timer-refresh.
                                  //   COERCED to 0 if meta_mediation_enabled=true (Meta forbids auto-refresh)
  "max_reload_count": 0,          // int >= 0 (coerced). 0 = unlimited
  "meta_mediation_enabled": false,// bool default false. true ONLY when AdMob chain includes Meta Audience Network. banner/native only.
  "http_timeout_ms": null,        // NEW. int or null (default null). range 5000-30000; out-of-range -> null + WARN.
                                  //   Must be < consuming placement's load_timeout_ms (cross-check WARN).
  "load_timeout_ms": null,        // NEW 0.16.0. int or null (default null). range 1000-60000; out-of-range -> null + WARN. per-unit coroutine load timeout (preload + show-path entry).
  "media_aspect_ratio": null,     // NEW. string or null (default null). NATIVE only. any|landscape|portrait|square|auto (case-insensitive, NEW 0.12.0: +auto).
                                  //   "auto" = wrap media to creative's natural size.
  "preload_on_screens": [         // NEW 0.12.0. Only honored when preload_trigger == "SCREEN".
    {
      "screen_name": "native_onb_2", // string, REQUIRED. Blank/missing entry dropped + WARN.
      "delay_ms": 200,            // int 0..60000 (coerced), default 0. Wait before fill.
      "once": true                // bool, default true. true = fill once per session (process); false = every screen_show.
    }
  ],
  "_meta": {                      // admin-only audit. SDK ignores
    "createdBy": "<email>", "updatedBy": "<email>",
    "createdAt": "<ISO 8601 UTC>", "updatedAt": "<ISO 8601 UTC>"
  }
}
```

`meta_config` object is only read when `mediation: "META"` (not active at runtime). Do NOT generate it for ADMOB units.

## 4. `xapp_registry` (array of placement key strings)

```jsonc
["xapp_p_appopen_cold", "xapp_p_inter_unlock", "..."]
```

- Every entry MUST start `xapp_p_`. Entries lacking the prefix = dropped + error logged.
- Duplicates deduped (first-seen wins, WARN).
- Order = preload order.

## 5. `xapp_p_<name>` (one key per placement)

```jsonc
{
  "name": "<name>",                 // REQUIRED. MUST equal key suffix after "xapp_p_" (else placement skipped).
                                    // admin regex: ^(inter|native|ncollap|nfull|nbanner|banner|appopen|reward|rewinter)_[a-z0-9_]+$
  "enabled": true,
  "ad_chain": {
    "load_strategy": "waterfall",   // NEW. waterfall (default) | parallel_first | parallel_auction.
                                    // legacy "parallel_load": true -> parallel_auction; false/absent -> waterfall.
    "entries": [
      { "ad_unit_id": "<id>", "floor_tag": "h" }  // ad_unit_id REQUIRED (must exist). floor_tag optional override.
    ]
  },
  "show_config": {
    "capping": { "hourly": 4, "daily": 10, "session": 5 },  // each default Int.MAX (unlimited)
    "min_interval_sec": 90,         // int. 0 = none
    "rotation_interval_sec": 45,    // int. native-only. 0 = no rotation
    "modal_loading": {              // optional. for inter/reward/rewinter (NOT banner/native/appopen)
      "enabled": true,              // default false
      "max_wait_ms": 3000           // default 3000
    }
  },
  "timing_config": {
    "load_timeout_ms": 10000,       // long. default 10000
    "show_timeout_ms": 3000         // long. default 3000
  },
  "segments": [],                   // List<String>. [] or ["*"] = all users
  "reuse_strategy": "own_first",    // NEW 0.11.8. enum own_first | reuse_before_load | reuse_first. default own_first.
                                    //   show() tier order: own_first=OWN_BUFFER>OWN_PENDING>REUSE>OWN_LOAD (current);
                                    //   reuse_before_load=OWN_BUFFER>REUSE>OWN_PENDING>OWN_LOAD; reuse_first=REUSE>OWN_BUFFER>OWN_PENDING>OWN_LOAD.
                                    //   SDK: case-insensitive, unknown->own_first silently. Admin z.enum strict -> unknown REJECTED.
  "reuse_chain": {                  // NEW 0.13.0. OPTIONAL. ordered allowlist of ad units to BORROW from at the REUSE tier.
    "entries": [
      { "ad_unit_id": "<id>" }      // unit must exist in xapp_ad_units; borrow walks entries in order, same-format only.
    ]
  },                                //   empty/absent = no cross-unit borrow. floor_tag/load_strategy ignored for reuse.
  "ui_config": { ... },             // REQUIRED iff chain format == NATIVE. see §6
  "ui_config_triggered": { ... },   // NEW 0.12.3. OPTIONAL second ui_config (SAME shape as ui_config). NATIVE only.
                                    //   Used when host renders with triggered=true; SDK resolvedUiConfig(true) prefers this,
                                    //   falls back to ui_config. null/absent = triggered renders reuse ui_config.
  "_metadata": {                    // admin/team docs. SDK ignores
    "description": "<1 sentence>", "screen_name": "<screen>", "trigger_event": "<EVENT>",
    "priority": "high",             // high | medium | low
    "notes": "<free form>"
  }
}
```

Cross-rules enforced by the SDK parser:
- All `ad_chain.entries[].ad_unit_id` must exist in `xapp_ad_units`. Unknown ref = entry dropped.
- All entries in one chain must share the same `format`. Mixed formats = placement skipped.
- Placement skipped if 0 resolvable entries.
- `name` MUST equal key suffix — mismatch = placement skipped.

## 6. `ui_config` (NATIVE format only)

`ui_config_triggered` (NEW 0.12.3, optional) uses the EXACT same shape below — it is just a second render variant the host selects via `triggered=true`.


```jsonc
{
  "template_id": "card_media_v1",   // enum: card_media_v1 | card_no_media_v1 | banner_horizontal_v1 | collapsible_v1 | fullscreen_hero_v1
                                    //   default card_media_v1. fullscreen_hero_v1 -> modal Activity; others render inline.
  "skip_delay_sec": 5,              // NEW. int [0..15], default 5. fullscreen_hero_v1 only — close button delay.
  "background": "#FFFFFF",          // hex #RRGGBB or #RRGGBBAA. default #FFFFFF
  "border_radius": [12,12,12,12],   // 4 ints >=0 (TL, TR, BR, BL). wrong length -> default
  "margin": [0,0,0,0],              // NEW. 4 ints >=0 (top, right, bottom, left). wrong length -> default
  "border": {                       // always applied to native card (AdMob policy)
    "visible": true,                // NEW field. default true
    "color": "#E0E0E0",             // hex, default #E0E0E0
    "width_dp": 1                   // int, default 1, coerced >=1
  },
  "collapse_arrow": {               // NEW. collapsible_v1 template only
    "border_width": 0,              // int >=0, default 0 (0 = no border)
    "border_color": "#000000",      // hex, drawn only when border_width > 0
    "border_radius": 20,            // int >=0, default 20
    "icon_color": "#000000",        // hex, default #000000
    "icon_size": 16,                // int >=1, default 16
    "container_size": 20,           // int >=1, default 20
    "container_bg_color": "#00000000", // hex, default transparent
    "container_shadow": false,      // bool, default false
    "targets": ["media"]            // NEW 0.12.6. string[] subset of {"media","cta"}, default ["media"].
                                    //   sections to collapse on arrow tap. invalid tokens dropped+WARN; empty->default.
  },
  "banner": {                       // NEW 0.12.0. banner_horizontal_v1 template ONLY (others ignore)
    "height_dp": 125,               // int default 125, coerced >=1. WARN when < 64 (clip risk).
    "padding_dp": 12                // int default 12, coerced >=0. row inner padding (all sides).
  },
  "fullscreen": {                   // NEW 0.12.0. fullscreen_hero_v1 template ONLY; other templates ignore.
    "layout_order": ["badge", "media", "info"], // permutation of exactly {badge, media, info}; else default + WARN. Default ["badge","media","info"].
    "info_bg": "#00000000",         // hex ARGB, default "#00000000". Invalid hex -> default.
    "action_container": {           // top-anchored slot (countdown + close)
      "position": "right",          // "left" | "right", default "right". Invalid -> default + WARN.
      "duration_sec": 5,            // int 0..15 (coerced), default 5. Countdown before close becomes active.
      "auto_close": false,          // bool, default false.
      "close_button": {
        "visible": true,            // bool, default true.
        "icon": "x"                 // "x" | "arrow_forward", default "x". Invalid -> default + WARN.
      }
    }
  },
  "ad_info": {
    "visible": true,
    "ad_icon": { "visible": true, "corner_radius_dp": 0, "size_dp": null }, // corner_radius_dp (int >=0, default 0); size_dp NEW 0.12.0 (int or null, coerced >=1 when set; null = per-template default 56/50/48/50/48)
    "ad_title": { "visible": true, "text_color": "#000000",   "font_size": 16 }, // font_size NEW (sp >=1)
    "ad_body":  { "visible": true, "text_color": "#B3000000", "font_size": 14 },
    "ad_badge": {                   // NO `visible` field — badge always renders
      "background_color": null,     // null OR hex; null = template default
      "text_color": null,           // null OR hex; null = white
      "text": "Ad"                  // whitelist: "Ad"|"Sponsored"|"Promoted"|"Quảng cáo" (else -> "Ad" + WARN)
    }
  },
  "ad_metadata": {
    "visible": false,
    "ad_star_rating": { "visible": true, "text_color": "#B3000000", "font_size": 12 },
    "ad_price":       { "visible": true, "text_color": "#B3000000", "font_size": 12 }
  },
  "ad_media": {
    "visible": true,
    "aspect_ratio": "16:9"          // regex w:h both > 0. default 16:9
  },
  "cta_button": {
    "visible": true,
    "placement": "bottom",          // "top" | "bottom" (default bottom)
    "style": {
      "type": "solid",              // "solid" | "outline"
      "background": {
        "solid": null,              // hex OR null
        "gradient": { "colors": ["#1E88E5","#1565C0"], "angle": 135 } // >=2 hex; default blue gradient. gradient wins over solid.
      },
      "border_radius": 8,           // int >=0, default 8
      "height": 48,                 // dp int >=1, default 48
      "text_color": "#FFFFFF",
      "text_font_size": 16,         // NEW. sp int >=1, default 16
      "border": {                   // outline only
        "color": "#FF6200",         // default #FF6200
        "width": 1,                 // int >=0, default 1
        "width_sides": null         // NEW. [t,r,b,l] ints >=0 OR null (overrides width when valid)
      }
    }
  }
}
```

Hex regex: `^#[0-9A-Fa-f]{6}([0-9A-Fa-f]{2})?$`. Aspect ratio regex: `^\d+:\d+$` both > 0. Any color/aspect failing its regex falls back to the field default. Gradient with < 2 valid hex colors is discarded (falls back to solid/default).

## 7. Validation summary (validator agent enforces all)

HARD ERRORS (admin will reject):
1. `_project.id` empty / not lowercase-slug.
2. `_project.package_name` mismatches the target app's Android applicationId.
3. ad_unit `id` fails regex `^[a-z][a-z0-9_]*$`.
4. ad_unit `id` duplicate within pool.
5. ad_unit `vendor_id` blank or duplicate within pool.
6. ad_unit `format` not in valid enum (after alias resolution); `native_fullscreen` is invalid — migrate to `native`.
7. ad_unit `mediation` not in `{ADMOB, MAX, IRONSOURCE, META}`. Practically generate `ADMOB`; flag MAX/IRONSOURCE/META as not-runtime-active.
8. ad_unit `preload_trigger` unknown value (legacy `"INIT"` → `INIT_DELAYED` + WARN; `"SCREEN"` is now first-class, not a legacy alias; any other unknown → `INIT_DELAYED` + WARN).
9. registry entry missing `xapp_p_` prefix.
10. registry references a placement key with no corresponding `xapp_p_<name>` object.
11. `xapp_p_<name>` object exists but key not in registry.
12. placement `name` ≠ key suffix.
13. placement name fails regex `^(inter|native|ncollap|nfull|nbanner|banner|appopen|reward|rewinter)_[a-z0-9_]+$`.
14. ad_chain entry references nonexistent `ad_unit_id`.
15. ad_chain entries have mixed formats.
16. `ad_chain.load_strategy` not in `{waterfall, parallel_first, parallel_auction}` (legacy `parallel_load` bool tolerated — migrate to `load_strategy`).
17. NATIVE-format placement missing `ui_config`.
18. non-NATIVE placement carries `ui_config` (WARN — SDK ignores).
19. hex color invalid format.
20. `border_radius` / `margin` array length ≠ 4 or contains negative.
21. `aspect_ratio` not `w:h` both > 0.
22. `cta_button.placement` not `top|bottom`.
23. `cta_button.style.type` not `solid|outline`.
24. `modal_loading` present on `appopen`/banner/native placement (SDK blocks — WARN).
25. `rotation_interval_sec` > 0 on non-native placement (SDK ignores — WARN).
26. `ui_config.template_id` not in the 5-value enum `{card_media_v1, card_no_media_v1, banner_horizontal_v1, collapsible_v1, fullscreen_hero_v1}` — admin rejects; SDK render router falls back. ABSENT = OK (defaults `card_media_v1`).
27. `min_fullscreen_interval_sec` < 0 / `launch_cooldown_ms` < 0 (SDK coerces; admin rejects negative).
28. `meta_mediation_enabled: true` AND `format` not in `{banner, native}`.
29. `border.width_dp` < 1 (SDK coerces to 1 + WARN; admin should reject negative).
30. `border.color` not matching hex regex.
31. `splash_min_duration_ms` outside [0, 15000] (SDK clamps to 1000; admin should reject out-of-range).
32. `http_timeout_ms` present and outside 5000–30000 (SDK clamps to null + WARN; admin should reject out-of-range).
33. `media_aspect_ratio` not in `{any, landscape, portrait, square, auto}` (SDK → null + WARN); also flag when set on a non-native unit. NEW 0.12.0: `"auto"` is now valid.
34. `skip_delay_sec` outside [0, 15] (SDK coerces; admin should reject).
35. `preload.circuit_backoff_sec` / `circuit_threshold` < 1 (SDK coerces to ≥1).
36. `xapp_p_<name>.reuse_strategy` present AND not in `{own_first, reuse_before_load, reuse_first}` — admin `z.enum` rejects import (SDK silently falls back to `own_first`). ABSENT = OK (defaults `own_first`).
37. `xapp_config.cross_unit_reuse_enabled` present AND not boolean.
38. NEW 0.12.0: `ui_config.banner.height_dp` / `banner.padding_dp` present AND not an integer, or negative (`height_dp` coerced ≥1, `padding_dp` coerced ≥0; admin should reject non-int / negative). ABSENT = OK (defaults 125 / 12).
39. NEW 0.12.0: `ui_config.ad_info.ad_icon.size_dp` present AND not an integer or < 1 (SDK coerces ≥1 when set; null = per-template default). ABSENT / null = OK.
40. NEW 0.12.3: `xapp_ad_units[*].reload_after_show_delay_ms` present AND not an int / negative / **> 60000** (SDK coerces ≥0 with no upper cap; admin `z.number().int().min(0).max(60000)` REJECTS > 60000). ABSENT = OK (default 0).
41. NEW 0.12.3: `xapp_config.firebase_ad_impression_enabled` present AND not boolean (admin `z.boolean()`). ABSENT = OK (default true).
42. NEW 0.12.3: `xapp_p_<name>.ui_config_triggered` present on a NON-native placement (only consumed for NATIVE renders) — admin/SDK ignore it for non-native; flag. Its inner shape is validated by the SAME `ui_config` rules (rules 19–26, 29–30, 38–39) — apply all of them to `ui_config_triggered` too.
43. NEW 0.12.6: `ui_config.collapse_arrow.targets` present AND not an array of strings, OR contains a token outside `{"media", "cta"}` (SDK drops invalid tokens + WARN, empty → default `["media"]`; admin should reject non-subset). ABSENT = OK (default `["media"]`).
44. NEW 0.13.0: `xapp_p_<name>.reuse_chain` present AND not an object `{ entries: [...] }`, OR an `entries` item missing/blank `ad_unit_id` (admin `z.object({ entries: z.array(adChainEntrySchema).default([]) })`). Unknown `ad_unit_id` is dropped by the SDK (like `ad_chain`) — flag as WARN, not error. ABSENT / `{entries:[]}` = OK.
45. NEW 0.13.0: `xapp_config.cross_unit_reuse_enabled` already covered by rule 37 (boolean) — no change.
46. NEW post-0.13.0: `xapp_config.adapter_init_timeout_ms` present AND not an integer, or outside [0, 30000] (admin `z.number().int().min(0).max(30000).default(0)`; SDK coerces). ABSENT = OK (default 0). Generator emits `0` (wait fully = prior behavior) — `0` is expected, not an error.
47. NEW (SDK 0.15.0): `xapp_ad_units[*].preload_trigger_by_segment` present AND (a) any key blank/empty (SDK drops blank keys with WARN; admin `z.record(z.string().min(1), preloadTriggerEnum)` rejects), or (b) any value not a valid trigger after alias resolution — same enum + legacy/unknown coercion as `preload_trigger` (rule 8): legacy `"INIT"` → `INIT_DELAYED` + WARN; unknown → `INIT_DELAYED` + WARN in the SDK; admin `z.enum` rejects the raw unknown on import. ABSENT / `{}` = OK (no override; effective trigger = `preload_trigger`).

WARNINGS:
- `buffer_size` > 5 (excessive memory).
- `load_timeout_ms` < 3000 (too tight) or > 15000 (too lax).
- session capping > 50 (likely abuse).
- gradient `colors.length` < 2 (gradient discarded by parser).
- ad_unit `vendor_id` starts `ca-app-pub-3940256099942544/` (AdMob test ID — flag pre-ship).
- `meta_mediation_enabled: true` AND `reload_interval_sec > 0` (SDK auto-coerces to 0 + WARN — file should reflect the coerced value).
- ad_badge `text` not in whitelist (SDK normalizes to "Ad" + WARN).
- `use_admob_startpreload: true` AND a delegated AdMob fullscreen unit sets `reload_interval_sec > 0` / `max_reload_count > 0` / `auto_reload_on_show: false` / `preload_trigger` ≠ INIT_CRITICAL (those knobs become no-ops — SDK WARN).
- NEW (SDK 0.15.0): `use_admob_startpreload: true` AND a delegatable fullscreen unit (`mediation: ADMOB`, `format` inter/reward/appopen) carries a non-empty `preload_trigger_by_segment` — the per-segment override is a no-op under AdMob-managed preload, same as `preload_trigger` ≠ `INIT_CRITICAL` (SDK/admin WARN).
- `http_timeout_ms` ≥ consuming placement's `load_timeout_ms` (coroutine wrapper fires before HTTP cap — SDK WARN).
- legacy `parallel_load` boolean present on `ad_chain` (migrate to `load_strategy`).
- legacy `provider` field on placement or `meta_config` on an ADMOB unit (SDK drops silently — strip).
- `collapse_arrow` present on a non-`collapsible_v1` template (SDK ignores).
- NEW 0.12.6: `collapse_arrow.targets` contains tokens outside `{"media", "cta"}` — SDK drops the invalid entries + WARN (empty result → default `["media"]`).
- `skip_delay_sec` set on a non-`fullscreen_hero_v1` template (SDK ignores for inline renderers).
- NEW 0.11.8: placement `reuse_strategy` ≠ `own_first` (or `xapp_config.cross_unit_reuse_enabled: true`) while `xapp_config.late_reuse_enabled: false` — reuse tiers are disabled, so the strategy / cross-unit borrow is a no-op (gated by `late_reuse_enabled`).
- NEW 0.13.0: `reuse_chain` entry references a unit whose `format` differs from the placement's format (from `ad_chain.entries[0]`) — borrow skips it at runtime (same-format only); likely a config mistake.
- NEW 0.13.0: `reuse_chain` non-empty while `xapp_config.cross_unit_reuse_enabled: false` OR `late_reuse_enabled: false` — cross-unit borrow is disabled, so `reuse_chain` is a no-op.
- NEW 0.13.0: `reuse_chain` references an `ad_unit_id` not in `xapp_ad_units` — SDK drops it (entry never borrowable); flag to fix or remove.
- NEW 0.11.8: `xapp_config.preload.vendor_dedupe` default is now `false` (was `true`) — same-`vendor_id` units load in parallel unless explicitly set `true`.
- NEW 0.12.0: `ui_config.banner.height_dp` < 64 — SDK WARNs (ad content may clip; AdMob policy risk). Only effective on the `banner_horizontal_v1` template.
- NEW 0.12.0: `ui_config.banner` block present on a non-`banner_horizontal_v1` template — SDK ignores (other templates do not consume it) — strip.
- NEW 0.12.0: `preload_trigger == "SCREEN"` AND `preload_on_screens` empty (or absent) — SDK WARNs (unit will never preload).
- NEW 0.12.0: `preload_on_screens` present on a unit whose `preload_trigger` ≠ `"SCREEN"` — SDK ignores + WARN.
- NEW 0.12.0: `preload_on_screens[].screen_name` blank — entry dropped + WARN; `delay_ms` outside 0..60000 coerced.
- NEW 0.12.0: `fullscreen.layout_order` not a permutation of `{badge, media, info}` — SDK uses default `["badge","media","info"]` + WARN.
- NEW 0.12.0: `fullscreen.action_container.position` not in `{left, right}` — SDK uses default `"right"` + WARN.
- NEW 0.12.0: `fullscreen.close_button.icon` not in `{x, arrow_forward}` — SDK uses default `"x"` + WARN.
- NEW 0.12.0: `fullscreen` block present on a non-`fullscreen_hero_v1` template — SDK ignores (strip).
