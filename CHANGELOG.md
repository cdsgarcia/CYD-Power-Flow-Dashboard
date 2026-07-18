# Changelog

All notable changes to this project are documented here.

---

## [v1.2.3] — 2026-07-19 — Review Fixes, Solar Substitutions, Load Thresholds

### Fixed
- **BUG** `icon_battery` color in `update_batt_icon_state` state=0 (static/idle) was computed
  from a glyph-index-based `COLORS[]` array that diverged from `val_battery` thresholds at
  SOC 80–84% (icon Blue, value Green) and 50–54% (icon Yellow, value Blue). Now uses
  `batt_color_green/blue/yellow` substitutions — icon and value always match. Removed dead
  `COLORS[]` array. Applied to both `cyd-e713b0.yaml` and `cyd-78d27c.yaml`.
- **BUG** `ha_load1.on_value` guard order in `cyd-78d27c.yaml` reversed — `g_demo_mode` was
  checked before `g_lvgl_ready`. Fixed to check `g_lvgl_ready` first (consistent with all other
  handlers and the safe-call convention).
- **BUG** `load3_label initial_value` said "A/C 2nd Floor" in both files — corrected to "A/C 3rd Floor".

### Changed
- `batt_pwr_blue` threshold lowered from 2000 W to 900 W (new scale: ≥3600 🟢 / ≥900 🔵 / >0 🟡 / =0 🩶 / <0 🟠).
- Removed now-redundant `batt_pwr_yellow` substitution.
- Load slot 0 W changed from Yellow to Green (no idle penalty — any reading shown as normal).
- Per-load thresholds updated to real-world values:
  - Load 1 (Ecoflow River 3): green=140 W / blue=200 W / yellow=400 W
  - Load 2 (A/C 1st Floor): green=600 W / blue=740 W / yellow=900 W
  - Load 3 (A/C 3rd Floor): green=80 W / blue=900 W / yellow=1000 W
- Solar Power color thresholds converted from hardcoded values to substitutions
  (`solar_color_green=3600` / `solar_color_blue=2000` / `solar_color_yellow=900`).
  Applied in `ha_solar.on_value` (LVGL) and `deactivate_screensaver` (E713B0 restore path).
- All stale "A/C 2F" / "A/C 2nd Floor" comments in `cyd-78d27c.yaml` updated to "A/C 3F" / "A/C 3rd Floor".
- Corrected `g_ac_slot_idx` global comment (0=A/C 1F / 1=A/C 3F).
- Corrected AC cycling slot globals comment — removed incorrect `g_load1_val` listing.
- Corrected LVGL container x-offset comment: x=+31 → x=+30 (matches actual YAML value).

---

## [v1.2.2] — 2026-07-18 — Battery Power & Time Estimate Threshold Substitutions

### Changed
- Battery Power (slot 0) thresholds updated and converted to substitutions:
  - New scale: ≥ 3600 W 🟢 / ≥ 2000 W 🔵 / > 0 W 🟡 / **= 0 W 🩶 Grey** / < 0 W 🟠
  - Grey (`0x888888`) at exactly 0 W — matches the cloud icon on PV Power — clearly shows no charging/discharging
  - Substitutions: `batt_pwr_green` / `batt_pwr_blue` / `batt_pwr_yellow`
- Battery Time Estimate (slot 1) colors converted to substitutions:
  - Charging: `batt_chg_green_h` (2h) / `batt_chg_blue_h` (4h) / `batt_chg_yellow_h` (6h)
  - Discharging: `batt_dis_orange_h` (3h) / `batt_dis_yellow_h` (6h) / `batt_dis_blue_h` (9h)
- Applied to both `cyd-e713b0.yaml` and `cyd-78d27c.yaml`.

---

## [v1.2.1] — 2026-07-18 — Home & Load Slot Color Threshold Substitutions

### Changed
- Added named substitutions for Home Apparent Power and Load slot (Load1/Load2/Load3) color
  thresholds — edit once in `substitutions:`, applied everywhere on next flash. Zero runtime overhead.

  ```yaml
  # Home Apparent Power color thresholds (VA)
  home_color_green: "1000"   home_color_blue: "2000"   home_color_yellow: "3000"
  # Load slot color thresholds (W) — each load independent
  load2_color_green: "700"   load2_color_blue: "1000"   load2_color_yellow: "1200"
  load3_color_green: "700"   load3_color_blue: "1000"   load3_color_yellow: "1200"
  load1_color_green: "700"   load1_color_blue: "1000"   load1_color_yellow: "1200"
  ```
- `update_slot_display` (E713B0) now selects per-slot threshold variables based on `g_slot_idx`
  instead of a single shared formula — Load1, Load2, Load3 can each have different thresholds.
- `update_ac_slot_display` (78D27C) similarly selects Load2 vs Load3 thresholds per slot.
- Applied to both `cyd-e713b0.yaml` and `cyd-78d27c.yaml`.

---



### Changed
- Battery icon glyph and color thresholds are now declared as named substitutions at the top of
  `cyd-e713b0.yaml`, making them easy to adjust in one place without hunting through the YAML.
  Applied to both `cyd-e713b0.yaml` and `cyd-78d27c.yaml`.

  ```yaml
  # Glyph thresholds (SOC %)
  batt_thresh_alert: "10"   batt_thresh_1bar: "25"   batt_thresh_2bar: "40"
  batt_thresh_3bar:  "55"   batt_thresh_4bar: "70"   batt_thresh_5bar: "85"
  batt_thresh_full:  "98"
  # Color thresholds (SOC %)
  batt_color_green: "80"   batt_color_blue: "50"   batt_color_yellow: "25"
  ```

  Substitutions are compile-time text replacements — zero runtime or heap overhead.
  Changing a value here and reflashing applies it across all locations automatically.

---

## [v1.1.2] — 2026-07-16 — Battery Icon Animation Restart Fix

### Fixed
- Battery icon animation was restarting from the beginning on every sensor update instead of
  completing its cycle. `update_batt_icon_state` unconditionally reset `g_batt_anim_step` on
  every call. Since `ha_batt_current.on_value` fires every few seconds, the 7-step animation
  (~7 seconds per full cycle) was always interrupted and never completed.
  Fix: reset `g_batt_anim_step` only when the animation state changes (`new_state != prev_state`),
  mirroring the existing correct behaviour in `update_solar_icon_state`.

---

## [v1.1.1] — 2026-07-16 — Solar & Home Load Stale Display Fix

### Fixed
- Solar Power and Home Load tiles could show stale values after returning from the screensaver.
  Same root cause as the v1.1.0 Battery SOC fix: `val_solar`, `val_home`, `icon_home` and their
  unit labels are only refreshed in `on_value` handlers which are skipped during screensaver.
  If values changed while the screensaver ran (e.g. PV dropped to 0 W), they stayed stale on return
  until HA sent the next sensor update.

### Changed
- Added `g_home_val` global to cache the Home Load sensor value (mirrors `g_solar_val` pattern).
- `deactivate_screensaver` now restores Solar, Home Load, and Battery SOC labels/colors/units
  from their respective cached globals on every screensaver exit.

---

## [v1.1.0] — 2026-07-16 — Battery SOC Stale Display Fix

### Fixed
- Battery SOC label (`val_battery`) could show a stale value after returning from the screensaver.
  During screensaver, `ha_battery.on_value` LVGL updates are deliberately skipped (screensaver guard).
  `deactivate_screensaver` called `update_batt_display` and `update_batt_icon_state` but did not
  refresh `val_battery` text or color. If SOC changed while the screensaver was running (e.g. 99.7 → 99.3),
  the display remained at the pre-screensaver value until HA sent the next SOC update.
  Fix: restore `val_battery` text and color from the cached `g_batt_soc_val` at the end of
  `deactivate_screensaver`, consistent with how other tiles are already restored on screensaver exit.

---

## [v1.0.1] — 2026-06-28 — Screensaver OOM Fix + Documentation Updates

### Fixed
- OOM crash during screensaver: `ss_img.release()` now called before every `set_url()` — frees old decoded image buffer before new download starts, preventing double-buffer overlap OOM. The heap sensor (5s poll) misses this sub-second window — graph can appear healthy while OOM occurs.

### Changed
- Photos Per Cycle default changed from 4 to 3

### Docs
- README streamlined — Installation detail moved to wiki, entity ID corrected
- CHANGELOG consolidated (V1–V8 internal history summarised)
- Removed V1–V8 versioned YAML files from repo (reference YAMLs retained)
- Wiki: screensaver folder permissions guide, photo upload methods (WinSCP, SCP, HA File Editor, Samba), Windows install guide expanded with USB drivers and web flasher option, corporate network / Zscaler troubleshooting

---

## [v1.0.0] — 2026-06-23 — Initial Public Release

### Added
- Real-time power flow dashboard: Solar, Home Load, Battery SOC + Power
- Battery time estimate (slot 1): calculates wall-clock charge/discharge target time
- 6 HA sensors for battery estimate graphing (hours + minutes-since-midnight + text)
- Home Load 3-slot cycling with configurable thresholds and labels
- Battery + solar animated icons (charging/discharging/alert; sun ping-pong; moon cycle)
- Photo screensaver with fade, slide (4 directions), instant, and random transitions
- PHT clock using `now.timestamp + (8*3600)` + `gmtime_r` — bypasses unreliable TZ env
- Persistent Fisher-Yates photo shuffle using `esp_random()` (hardware RNG)
- All controls exposed as HA entities (`entity_category: config`)
- `lv_label_set_text_static()` for all icon glyphs — eliminates heap alloc/free on animation frames
- Screensaver guards on `ha_solar`, `ha_home`, `ha_battery` `on_value` — skips LVGL updates during screensaver
- `ss_img_widget` hidden before `lv_scr_load_anim()` — prevents 3.6 KB RGB565A8 draw-strip OOM
- `on_boot reserve()` — pre-allocates photo vectors before heap fragments
- `debug:` component with 3 HA diagnostic sensors: `Heap Free`, `Heap Largest Block`, `Loop Time`
- WiFi dual-AP fallback
- GitHub Actions CI — validates config on every push

---

## Internal development history (V1–V8, pre-release)

Pre-release versions were developed iteratively and are not tracked individually in this changelog. Key milestones: basic display (V1) → slot cycling + PHT clock (V2) → screensaver (V3) → shuffle + slot merge (V4) → battery time estimate (V5) → battery icon animation (V6) → solar animation (V7) → moon cycle + dual-font (V8) → OOM hardening → public release (v1.0.0).


### Initial public release

### Added
- Real-time power flow dashboard: Solar, Home Load, Battery SOC + Power
- Battery time estimate (slot 1): calculates wall-clock charge/discharge target time
- 6 HA sensors for battery estimate graphing (hours + minutes-since-midnight + text)
- Home Load 3-slot cycling with configurable thresholds and labels
- Battery + solar animated icons (charging/discharging/alert; sun ping-pong; moon cycle)
- Photo screensaver with fade, slide (4 directions), instant, and random transitions
- PHT clock using `now.timestamp + (8*3600)` + `gmtime_r` — bypasses unreliable TZ env
- Persistent Fisher-Yates photo shuffle using `esp_random()` (hardware RNG)
- All controls exposed as HA entities (`entity_category: config`)
- `lv_label_set_text_static()` for all icon glyphs — eliminates heap alloc/free on animation frames
- Screensaver guards on `ha_solar`, `ha_home`, `ha_battery` `on_value` — skips LVGL updates during screensaver
- `ss_img.release()` before every `set_url()` — prevents double-buffer overlap OOM
- `ss_img_widget` hidden before `lv_scr_load_anim()` — prevents 3.6 KB RGB565A8 draw-strip OOM
- `on_boot reserve()` — pre-allocates photo vectors before heap fragments
- `debug:` component with 3 HA diagnostic sensors: `Heap Free`, `Heap Largest Block`, `Loop Time`
- WiFi dual-AP fallback
- GitHub Actions CI — validates config on every push

---

## Internal development history (V1–V8, pre-release)

Pre-release versions were developed iteratively and are not tracked individually in this changelog. Key milestones: basic display (V1) → slot cycling + PHT clock (V2) → screensaver (V3) → shuffle + slot merge (V4) → battery time estimate (V5) → battery icon animation (V6) → solar animation (V7) → moon cycle + dual-font (V8) → OOM hardening → public release (v1.0.0).
