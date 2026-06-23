# Changelog

All notable changes to this project are documented here.

---

## [Unreleased]

### Added
- `lv_label_set_text_static()` for all static glyph strings — eliminates heap alloc/free on every icon animation frame
- Screensaver guards on `ha_solar`, `ha_home`, `ha_battery` `on_value` handlers — skips LVGL label updates during screensaver (widgets not visible), reducing heap fragmentation
- `debug:` component with 3 HA diagnostic sensors: `Heap Free`, `Heap Largest Block`, `Loop Time` — exposed every 5s for real-time SRAM monitoring in HA

---

## [v1.0.0] — 2025-06-23

### Initial public release

---

## [V8] — Internal

### Added
- Moon cycle icon animation (5-step) for nighttime solar state using **Material Symbols Outlined** font (dual-widget design: `icon_solar` + `icon_solar_moon`)
- Dual-font solar icon: state 1 (producing) on `icon_solar` (Material Icons), state 2 (night) on `icon_solar_moon` (Material Symbols Outlined)
- `LV_OBJ_FLAG_HIDDEN` visibility control on every `update_solar_icon_state` call — prevents ghost icon artifacts

### Fixed
- Battery alert threshold corrected: `battery_alert` glyph at `< 10%`, `battery_0_bar` at `≥ 10%`
- `BatteryIconTest.yaml`: COLORS[] idx5 was Orange — corrected to Yellow

---

## [V7] — Internal

### Added
- Solar icon ping-pong animation (8-step sun glyph sequence) when solar power > 0
- `update_solar_icon_state` script: state 0 (day/idle → cloudy), state 1 (producing → sun animation), state 2 (night/idle → moon placeholder)
- 250ms animation interval shared by battery + solar icon cycling
- `solar_log_enabled` switch to gate diagnostic logs

---

## [V6] — Internal

### Added
- Battery icon animation: charging (glyph steps up), discharging (glyph steps down), alert flash at < 10% SOC
- `update_batt_icon_state` script with 4 animation states (static / charging / discharging / alert)
- `g_batt_anim_step` reset to current SOC glyph index on state change

---

## [V5] — Internal

### Added
- Battery time estimate (slot 1): calculates wall-clock target time using current, capacity, SOC, and charge/discharge limits
- 6 HA sensors published for graphing: `Heap hours to full/empty`, `Battery full/empty at` (minutes since midnight + text)
- `advance_batt_slot` script with eligibility checks; `batt_time_enabled` switch
- `batt_log_enabled` switch gating 4 log points
- HA sensor publish throttled to 60s with `static` locals to avoid HA flooding

---

## [V4] — Internal

### Added
- `entity_category: config` on all control/diagnostic entities
- Persistent Fisher-Yates photo shuffle using `esp_random()` (hardware RNG)
- `g_photo_queue` persistent full queue; slice-per-activation pattern prevents repeats before all photos shown
- Battery + load slot timer merged into single shared `g_slot_secs` countdown

### Fixed
- Renamed all AC-specific globals/scripts to generic Load slot naming

---

## [V3] — Internal

### Added
- Photo screensaver with fade, slide (4 directions), instant, and random transitions
- `lv_scr_load_anim()` page transitions; per-photo `lv_anim_t` animations
- `ss_img_widget` hidden before `lv_scr_load_anim()` to prevent RGB565A8 draw-strip OOM
- `?r=XXXXXXXX` random query string on every photo URL — forces `on_download_finished` even on repeat photos
- `Photos Per Cycle`, `Photo Order`, `Transition Effect`, `Animation Duration` HA controls

---

## [V2] — Internal

### Added
- Home Load 3-slot cycling (Load 1 / 2 / 3) with configurable thresholds and labels
- Battery power slot cycling (W/kW slot)
- PHT clock using `now.timestamp + (8*3600)` + `gmtime_r` — bypasses unreliable TZ env
- Color thresholds for solar, home, battery SOC, battery power, load slots
- WiFi dual-AP fallback

---

## [V1] — Internal

### Initial prototype
- Basic ILI9341 display via `mipi_spi` platform
- LVGL 320×240 landscape layout (270° rotation)
- Solar power, home load, battery SOC quadrant display
- Home Assistant native API sensor binding
