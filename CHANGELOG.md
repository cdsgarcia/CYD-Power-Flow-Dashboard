# Changelog

All notable changes to this project are documented here.

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
