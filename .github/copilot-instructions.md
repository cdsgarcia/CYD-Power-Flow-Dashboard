# Copilot Instructions — CYD-E713B0 ESPHome

## Project Overview

ESPHome configuration for a **CYD (Cheap Yellow Display)** ESP32 device that renders a Home Assistant power flow dashboard on a 240×320 ILI9341 display in landscape mode (320×240 via LVGL rotation 270°). When idle it transitions to a photo screensaver with configurable transitions and play order.

---

## File Roles

| File | Purpose |
|------|---------|
| `cyd-e713b0.yaml` | **Active production config — the only file to edit and flash** |
| `cyd-e713b0-BatteryIconTest.yaml` | Battery icon glyph/color/animation test scaffold (reference only) |
| `cyd-e713b0-SolarIconTest.yaml` | Solar icon glyph/color/animation test scaffold (reference only) |
| `cyd-e713b0-Basic.yaml` | Minimal boot/wifi scaffold (reference only) |
| `cyd-e713b0-test-ble-proxy.yml` | Standalone BLE proxy experiment (reference only) |

Only `cyd-e713b0.yaml` should be flashed to the device.

---

## Build / Flash Commands

```bash
# Validate config without building:
python -m esphome config cyd-e713b0.yaml

# Build only (no flash):
python -m esphome compile cyd-e713b0.yaml

# Full build + flash (USB or OTA):
python -m esphome run cyd-e713b0.yaml

# OTA only (device already online):
python -m esphome upload cyd-e713b0.yaml

# View live logs:
python -m esphome logs cyd-e713b0.yaml
```

> Use `python -m esphome` — bare `esphome` may not be on PATH on Windows without admin rights.

Credentials are in `secrets.yaml` (not committed):
```yaml
wifi_ssid: "primary_ssid"
wifi_password: "primary_password"
wifi_ssid2: "secondary_ssid"      # second AP (different SSID)
wifi_password2: "secondary_password"
```

---

## Hardware

| Function | Pins |
|----------|------|
| TFT SPI (display) | CLK=GPIO14, MOSI=GPIO13, MISO=GPIO12, CS=GPIO15, DC=GPIO2 |
| Backlight (LEDC) | GPIO21 |
| Touch SPI (XPT2046) | CLK=GPIO25, MOSI=GPIO32, MISO=GPIO39, CS=GPIO33, IRQ=GPIO36 |

Touch SPI is wired but `touchscreen:` is not yet configured in the production config.

**Memory:** No PSRAM. With `buffer_size: 10%` (~15 KB render buffer), ~55 KB heap is available for JPEG decode.  
**Confirmed heap minimum** (75-min soak test with all OOM mitigations active): **~45,000 bytes** largest free block during screensaver.  
Recommended photo size = **128×96 RGB565 = 24,576 bytes** → ~20,400 bytes headroom ✅  
160×120 = 38,400 bytes → ~6,600 bytes headroom ⚠️ (thin margin, not recommended — was OOM size before fixes)

---

## LVGL Layout (320×240 landscape)

```
┌─────────────────────────────────┐
│        Time Header (HH:MM:SS AM)│  lbl_time — header_font (size 40)
├─────────────────────────────────┤  horizontal divider at y=34
│  Solar Power   │  Home Load     │  top-left x:0,y:25 / top-right x:160,y:25
│  (top-left)    │  Slots cycling │
├─────────────────────────────────┤
│                                 │
│    Battery Power (full width)   │  val_battery_power — value_font (size 50)
│    cycles with: Charges at /    │  lbl_batt_title changes; val_battery_power shows time
│    Empty at (time estimate)     │
├────────────────┬────────────────┤
│  Battery SOC   │  Home Load     │  bottom-left x:0,y:160 / bottom-right x:160,y:160
│  (bottom-left) │  (bottom-right)│
└────────────────┴────────────────┘
```

Each quadrant `obj` contains: icon label (outer edge), title label (top), value label (center), unit label (bottom).

**Do not change icon/text positions or sizes** — they are optimised for the physical display.

---

## Key Conventions

### Entity IDs — always use substitutions

All HA entity IDs are declared in the `substitutions:` block. Never hardcode entity IDs inside sensor blocks.

| Key | Entity |
|-----|--------|
| `solar_entity` | `sensor.srne_pv_power` |
| `load2_entity` | `sensor.a_c_1f_power_meter_power` |
| `load3_entity` | `sensor.a_c_3f_power_meter_power` |
| `load1_entity` | `sensor.ef_r30241_ac_input_power` |
| `battery_entity` | `sensor.battery_soc_mean` |
| `home_entity` | `sensor.srne_load_l1_apparent_power` *(apparent power — displays as VA/kVA, not W/kW)* |
| `battery_power_entity` | `sensor.total_battery_power` |
| `batt_current_entity` | `sensor.total_battery_current` |
| `srne_charge_limit_entity` | `number.srne_batttery_charge_limit` ⚠️ *'batttery' typo is in the HA entity name — do not fix* |
| `srne_discharge_limit_entity` | `number.srne_batttery_discharge_limit` ⚠️ *same typo* |

### Font glyph trimming — critical for flash size

Adding a character not in a font's `glyphs` list causes a crash or garbled render.

| Font | Size | Glyphs | Used for |
|------|------|--------|---------|
| `header_font` | Roboto 40 | `"0123456789: AMP-"` | Clock display + `"--:--:-- --"` fallback |
| `value_font` | Roboto 50 | `"0123456789.: -AMP"` | Numeric values, negative sign, time format `8:40 AM` |
| `label_font` | Roboto 12 | full alphanumeric + `" .%/+-"` | All static text labels, HA-editable Load labels |
| `icon_font` | Material Icons 60 | solar `\uec0f`, snowflake `\ueb3b`, home `\ue88a`, electrical_services `\ue63c`, brightness_4–7 `\ue3a9`–`\ue3ac`, wb_cloudy `\ue42d` | Solar quadrant icon + sun ping-pong glyphs (state 1). **Used by `icon_solar` widget only.** |
| `icon_font_moon` | Material Symbols Outlined 60 | nights_stay `\uf174`, moon_stars `\uf34f`, star_shine `\uf31d`, auto_awesome `\ue65f`, flare `\ue3e4` | Moon cycle glyphs (state 2). **Used by `icon_solar_moon` widget only.** Material Symbols required — these codepoints do not exist in Material Icons. |
| `icon_font_batt` | Material Icons 90 | 8 battery level glyphs | Battery SOC icon |

**Load label text** (editable from HA) must use only `label_font` characters: `A-Z a-z 0-9` and `" .%/+-"`.

### Color coding — consistent across all sensors

```
0x00FF00  Green   — good / high / charging well / almost fully charged
0x0096FF  Blue    — medium
0xFFFF00  Yellow  — moderate / caution / idle (0 W)
0xFFA500  Orange  — low / alert / discharging / long time until charged
```

Icon and value labels for the same sensor **always share identical color thresholds** — update both together.

#### Battery time estimate slot colors (slot 1)

| Charging (hours to full) | Color | Discharging (hours to empty) | Color |
|---|---|---|---|
| < 2 h | 🟢 Green — almost done | < 3 h | 🟠 Orange — almost empty |
| < 4 h | 🔵 Blue | < 6 h | 🟡 Yellow |
| < 6 h | 🟡 Yellow | < 9 h | 🔵 Blue |
| ≥ 6 h | 🟠 Orange — long wait | ≥ 9 h | 🟢 Green — plenty left |

### Battery power sign convention

Battery power can be negative (discharging). All format/unit lambdas test `|| x <= -N` alongside `x >= N` for symmetry.

### `on_boot` priority: -10

Fires after ALL component `setup()` calls (including LVGL). Used to:
1. Restore backlight brightness from NVS
2. Set `g_lvgl_ready = true` — **the LVGL guard flag**
3. `reserve()` pre-allocate `g_photo_queue` (to `ss_photo_count`) and `g_photo_order` (to `ss_cycle_size`) — prevents heap fragmentation from vector growth during the first screensaver activation, protecting the 24 KB JPEG decode budget
4. Call `update_slot_display` and `update_batt_display` to apply NVS-restored values
5. Call `update_batt_icon_state` to set initial battery icon animation state (static at boot)
6. Call `update_solar_icon_state` to set initial solar icon (cloudy / moon depending on time of day)

Do **not** change this priority.

### `g_lvgl_ready` guard

`text:` entities with `restore_value: true` call `on_value` during their `setup()` — before LVGL initialises. The first line of **`update_slot_display`, `update_batt_display`, `update_batt_icon_state`, and `update_solar_icon_state`** is:
```cpp
if (!id(g_lvgl_ready)) return;
```
Any new script that touches LVGL widgets and could be called from `on_value` during setup **must** include this guard.

### Display platform

Uses `mipi_spi` (not the older `ili9xxx`). `color_order: BGR`, `invert_colors: false`, `update_interval: never`, `auto_clear_enabled: false` — LVGL owns the framebuffer entirely.

### Logger level

Production config uses `level: WARN` — do not revert to DEBUG (clock fires every second and floods output).

### Clock time source — UTC+8 offset required

`platform: homeassistant` receives time via the native API. `ha_time.now().hour` reflects the TZ environment variable, which HA may set to UTC regardless of HA's configured timezone. Both `timezone: "PHT-8"` in YAML and manual `+8` offsets are unreliable (`timezone:` is overridden by HA; `+8` causes `+16` after HA timezone sync).

**Fix**: Use `now.timestamp` (raw UTC epoch) with a fixed `+8 × 3600` offset and `gmtime_r` — this bypasses TZ entirely and always produces correct PHT time:

```cpp
time_t pht = now.timestamp + (8 * 3600);
struct tm t; gmtime_r(&pht, &t);
int h = t.tm_hour;  // PHT hour, always correct
```

Applied in **three** places: the clock lambda (`lbl_time`), the battery time estimate in `update_batt_display` (both slot paths), and `update_solar_icon_state` (night detection). `now.minute`/`now.second` replaced with `t.tm_min`/`t.tm_sec` everywhere.

### `entity_category: config`

Applied to all control/diagnostic entities: buttons, `Display Backlight` light, all number/select/switch/text controls.

---

## Home Load Slot Cycling (top-right quadrant)

The top-right quadrant rotates between **3 sensor slots** every `Slot Cycle Secs`:

| Slot index | Sensor sub | Default label | Icon | Visible when |
|-----------|-----------|--------------|------|-------------|
| 0 | `${load2_entity}` | "A/C 1st Floor" (`load2_label`) | snowflake `\ueb3b` | value > `load2_threshold` |
| 1 | `${load3_entity}` | "A/C 3rd Floor" (`load3_label`) | snowflake `\ueb3b` | value > `load3_threshold` |
| 2 | `${load1_entity}` | "Ecoflow River 3" (`load1_label`) | elec. services `\ue63c` | **always shown** |

Slot 2 (Load 1) is always eligible — guarantees at least one slot is always visible.

### Cycling logic (`advance_slot` script)
1. Start from `(g_slot_idx + 1) % 3`
2. Try each candidate; skip slot 0/1 if value ≤ threshold
3. Set `g_slot_idx` and call `update_slot_display`

### Display update (`update_slot_display` script)
Uses raw LVGL C API (`lv_label_set_text`, `lv_obj_add_flag/clear_flag`). Updates:
- `lbl_slot` — label text from the matching `load*_label` text entity
- `icon_slot` — snowflake icon; hidden for slot 2
- `icon_load1` — electrical_services icon; shown only for slot 2 (`hidden: true` initially)
- `val_slot` + `val_slot_unit` — value and W/kW unit

Called by: sensor `on_value` (when active slot matches), `advance_slot`, `deactivate_screensaver`, `on_boot`.

### HA-editable slot labels (`text:` entities)

| Entity | ID | Default | Max length |
|--------|----|---------|-----------|
| Load 1 Label | `load1_label` | "Ecoflow River 3" | 16 |
| Load 2 Label | `load2_label` | "A/C 1st Floor" | 16 |
| Load 3 Label | `load3_label` | "A/C 3rd Floor" | 16 |

`on_value` calls `update_slot_display` only if the matching slot is active and not in screensaver.

### Slot cycling HA controls

| Entity | ID | Range | Default | Notes |
|--------|----|-------|---------|-------|
| Slot Cycle Secs | `slot_cycle_secs` | 1–15 s | 5 | Drives **both** Home Load AND Battery slot timers (shared `g_slot_secs`) |
| Load 2 Threshold | `load2_threshold` | 0–500 W | 10 | |
| Load 3 Threshold | `load3_threshold` | 0–500 W | 10 | |

`slot_cycle_secs.on_value` immediately resets `g_slot_secs` — no need to wait for the current countdown to expire.

---

## Battery Power Slot Cycling (center, full width)

Cycles between **2 slots** every `Slot Cycle Secs` (same shared timer as Home Load slots):

| Slot | Content | Title | Condition |
|------|---------|-------|-----------|
| 0 | Battery power in W/kW | "Battery Power" | Always shown |
| 1 | Target wall-clock time | "Charges to N% at" / "Empty to N% at" | Only when within `Batt Time Threshold` hours of limit |

### Time estimate computation
```
Charging  (current > +0.5 A):  hours = (charge_limit  − SOC) / 100 × capacity_Ah / current
Discharging (current < −0.5 A): hours = (SOC − discharge_limit) / 100 × capacity_Ah / |current|
Show slot 1 only when hours < batt_time_threshold_hrs
```
Target time = `now + hours`, displayed as 12h with AM/PM and no leading zero on the hour: `"8:40 AM"`, `"12:30 PM"`.
Day prefix moves to the title label:
- Same day: `"Charges to N% at"` / `"Empty to N% at"`
- +1 day: `"Charges to N% at - Next Day"` / `"Empty to N% at - Next Day"`
- +N days: `"Charges to N% at - Next N Days"` / `"Empty to N% at - Next N Days"`
- N = `srne_charge_limit` (charging) or `srne_discharge_limit` (discharging), cast to int

### Logic notes
- `advance_batt_slot` checks eligibility **before** switching to slot 1 — if ineligible, stays at slot 0.
- `update_batt_display` re-checks eligibility when on slot 1 and falls back to slot 0 inline (e.g. current drops to 0).
- Both `advance_slot` and `advance_batt_slot` fire in the same `g_slot_secs` countdown block — a single shared timer.
- **Slot 0 exit independently recomputes the time estimate** and publishes HA sensors — this prevents oscillation that would occur if slot 0 published zeros while slot 1 published real values as the 5s slot timer alternated.

### Battery HA controls

| Entity | ID | Range | Default |
|--------|----|-------|---------|
| Battery Capacity Ah | `battery_capacity_ah` | 200–700 Ah, step 5 | 300 |
| Batt Time Threshold | `batt_time_threshold_hrs` | 1–48 h | 18 |
| Batt Time Estimate | `batt_time_enabled` | switch | ON |
| Batt Log Enabled | `batt_log_enabled` | switch | **OFF** |

`batt_log_enabled` gates 4 log points: `advance_batt_slot` eligibility, slot-1 fallback, "Battery full/empty at" time string, and invalid-time fallback.

`batt_time_enabled.on_turn_off` immediately publishes correct status to all 6 HA sensors (bypassing the 60s throttle) so HA reflects the disabled state right away.

### Battery Time Estimate — HA sensors (published to HA for graphing)

`update_batt_display` publishes 6 sensors at every exit path, **throttled to once per 60 seconds** using `static bool _first_pub` and `static uint32_t _last_pub_ms` locals. The first call after boot always publishes immediately. LVGL display updates are **not** throttled — they happen on every call.

**Numeric (graphable — `state_class: measurement`):**

| Entity | ID | Unit | Slot 1 value | Slot 0 / fallback |
|--------|----|------|-------------|-------------------|
| Battery Hours to Full | `batt_hours_to_full` | h | hours remaining (charging) | `0.0` |
| Battery Hours to Empty | `batt_hours_to_empty` | h | hours remaining (discharging) | `0.0` |
| Battery Full At | `batt_full_at_min` | min | minutes since midnight of target time | `0.0` |
| Battery Empty At | `batt_empty_at_min` | min | minutes since midnight of target time | `0.0` |

`Battery Full At` / `Battery Empty At` use minutes-since-midnight (0–1440) so they graph as a linear scale. e.g. 10:45 AM = 645, 6:30 PM = 1110.

**Text (display only — dashboard cards):**

| Entity | ID | Charging eligible | Discharging eligible | SOC at limit | Not active |
|--------|----|-------------------|----------------------|--------------|------------|
| Battery Full At Time | `batt_full_at_time` | `"10:45 AM"` | `"Not Charging"` | `"Full"` | `"Not Charging"` / `"Charging"` |
| Battery Empty At Time | `batt_empty_at_time` | `"Not Discharging"` | `"6:30 PM"` | `"At Limit"` | `"Not Discharging"` / `"Discharging"` |

`"Charging"` / `"Discharging"` means actively charging/discharging but estimate is beyond threshold or disabled.

---

## Battery SOC Icon Animation

A separate 250ms `interval` drives glyph cycling on `icon_battery`. State is set by `update_batt_icon_state` (called from `ha_battery.on_value`, `ha_batt_current.on_value`, `deactivate_screensaver`, `on_boot -10`).

### Animation states (priority order)

| State | Condition | Behaviour | Timing |
|-------|-----------|-----------|--------|
| 3 Alert | SOC < 10% (any current) | Flash `battery_alert` glyph on/off | 250ms toggle |
| 2 Discharging | current < −0.5A | Step glyph DOWN: current level → battery_0_bar → repeat | 1000ms/step |
| 1 Charging | current > +0.5A | Step glyph UP: current level → battery_full → repeat | 1000ms/step |
| 0 Static | else | No animation; glyph managed by `ha_battery.on_value` | — |

### Glyph + color table

| idx | Glyph | Unicode | Color | SOC range |
|-----|-------|---------|-------|-----------|
| 0 | battery_full | `\ue1a4` | Green `0x00FF00` | ≥98% |
| 1 | battery_5_bar | `\uebd4` | Green `0x00FF00` | ≥85% |
| 2 | battery_4_bar | `\uebe2` | Blue `0x0096FF` | ≥70% |
| 3 | battery_3_bar | `\uebdd` | Blue `0x0096FF` | ≥55% |
| 4 | battery_2_bar | `\uebe0` | Yellow `0xFFFF00` | ≥40% |
| 5 | battery_1_bar | `\uebd9` | Yellow `0xFFFF00` | ≥25% |
| 6 | battery_0_bar | `\uebdc` | Orange `0xFFA500` | ≥10% |
| 7 | battery_alert | `\ue19c` | Orange `0xFFA500` | <10% |

### Cycling logic
- `g_batt_anim_step` is always **reset to current SOC glyph index** when state changes — animation starts from real battery level.
- **Charging**: `step--`; if `step < 0` → reset to `soc_idx(soc)` (wraps at full, restarts from current SOC).
- **Discharging**: `step++`; if `step > 6` → reset to `soc_idx(soc)` (wraps at empty, restarts).
- **Alert**: only opacity toggled (COVER↔TRANSP) — glyph stays as `battery_alert` (set by `ha_battery.on_value`).
- 250ms interval skips when `!g_lvgl_ready` or `g_screensaver`.
- When state goes to 0 (static), `update_batt_icon_state` restores the correct glyph+color via `lv_label_set_text_static` + `lv_obj_set_style_text_color` immediately.
- All glyph changes use **`lv_label_set_text_static()`** — glyph arrays are `static const char*` in flash, eliminating heap free+malloc churn from repeated cycling.

---

## Solar Power Icon Animation

A 250ms `interval` drives glyph cycling on `icon_solar` for state 1 (producing). State is set by `update_solar_icon_state` (called from `ha_solar.on_value`, `deactivate_screensaver`, `on_boot -10`, and the 1s interval when `g_solar_val <= 0`).

### Animation states

| State | Condition | Glyph | Color | Behaviour |
|-------|-----------|-------|-------|-----------|
| 0 Static | power = 0 + daytime (06:00–18:00) | wb_cloudy `\ue42d` on `icon_solar` | Grey `0x888888` | No animation |
| 1 Glyph-cycle | power > 0 (any time of day) | Sun ping-pong 8 steps on `icon_solar` | Yellow `0xFFFF00` | **2000ms/step** via 250ms interval |
| 2 Glyph-cycle | power = 0 + nighttime (18:00–06:00) | Moon cycle 5 steps on `icon_solar_moon` | Blue `0x0096FF` | **2000ms/step** via 250ms interval |

Night = PHT hour ≥ 18 or < 6. Uses `now.timestamp + (8*3600)` + `gmtime_r` — same method as the clock.

**Dual-widget design**: `icon_solar` (Material Icons) and `icon_solar_moon` (Material Symbols Outlined) occupy the same screen position. Visibility is controlled exclusively via `LV_OBJ_FLAG_HIDDEN` — never opacity — so the hidden widget never participates in rendering and cannot produce ghost artifacts. State 2 hides `icon_solar` and shows `icon_solar_moon`; states 0/1 hide `icon_solar_moon` and show `icon_solar`. This is done unconditionally on every `update_solar_icon_state` call (not only on state change) to prevent ghost icons when the function fires repeatedly in the same state.

### Glyph tables

**State 1 — Sun ping-pong (8 steps, Yellow `0xFFFF00`):**

| Step | Glyph | Unicode | Notes |
|------|-------|---------|-------|
| 0 | solar_power | `\uec0f` | Initial on state entry |
| 1 | brightness_7 | `\ue3ac` | Most rays |
| 2 | brightness_4 | `\ue3a9` | |
| 3 | brightness_6 | `\ue3ab` | |
| 4 | brightness_5 | `\ue3aa` | Peak (reverse starts here) |
| 5 | brightness_6 | `\ue3ab` | |
| 6 | brightness_4 | `\ue3a9` | |
| 7 | brightness_7 | `\ue3ac` | Wraps back to step 0 |

**State 2 — Moon cycle (5 steps, Blue `0x0096FF`) on `icon_solar_moon` widget:**

| Step | Glyph | Unicode | Font | Notes |
|------|-------|---------|------|-------|
| 0 | nights_stay | `\uf174` | Material Symbols Outlined | Initial on state entry — crescent + stars |
| 1 | moon_stars | `\uf34f` | Material Symbols Outlined | Moon with stars |
| 2 | star_shine | `\uf31d` | Material Symbols Outlined | Shining star |
| 3 | auto_awesome | `\ue65f` | Material Symbols Outlined | Sparkles ✨ |
| 4 | flare | `\ue3e4` | Material Symbols Outlined | Starburst — wraps back to step 0 |

⚠️ These codepoints exist **only in Material Symbols Outlined** (`gfonts://Material Symbols Outlined`). Do not look them up in classic Material Icons — the codepoints differ or the glyphs don't exist there.

### Cycling / transition logic
- **State 1**: interval advances `g_solar_anim_step` as `(step+1) % 8` every **2000ms** (gated by `s_tick % 8 == 0`) — sun ping-pong through 8-step array on `icon_solar`.
- **State 2**: interval advances `g_solar_anim_step` as `(step+1) % 5` every **2000ms** (gated by `s_tick % 8 == 0`) — moon cycle through 5-step array on `icon_solar_moon`, blue color.
- No `lv_anim_t` spin — both states fully driven by the 250ms interval shared with battery animation.
- All glyph changes use **`lv_label_set_text_static()`** (not `lv_label_set_text()`) — the glyph arrays are `static const char*` in flash, so no heap copy is needed. This eliminates the free+malloc heap churn that repeated glyph cycling would otherwise cause, reducing fragmentation.
- `g_solar_anim_step` is reset to 0 **only on state change** (`new_state != prev_state`). Repeated same-state calls (e.g., the 1s interval firing every second during state 2) do **not** reset the step — this prevents the moon ping-pong reverse from being interrupted. State 0 (static) is always re-applied safely since there is no step to protect.
- **Screensaver guard**: globals (`g_solar_val`, `g_solar_anim_state`, `g_solar_anim_step`) are always updated; LVGL ops are skipped when `g_screensaver == true` (icon not visible). `deactivate_screensaver` calls `update_solar_icon_state` to restore on return.
- **Time-of-day transitions** (06:00 / 18:00 crossings when solar = 0): the 1s interval calls `update_solar_icon_state` every second when `g_solar_val <= 0`, ensuring the icon switches day↔night at the right moment without waiting for a sensor update.
- **`LV_DRAW_COMPLEX = 1`** is still required — enabled by the screensaver's image scaling (`lv_draw_sw_transform`). Do not disable this.

### Solar HA controls

| Entity | ID | Default |
|--------|----|---------|
| Solar Log Enabled | `solar_log_enabled` | **OFF** |

`solar_log_enabled` gates the `update_solar_icon_state` log: `solar=NNNNw time_valid=1 PHT_hr=HH is_night=0 prev=X → new=Y screensaver=0`. Turn ON to diagnose icon state transitions, turn OFF once confirmed correct.

---

## Screensaver

### State machine
A 1-second `interval` drives a two-state machine (`g_screensaver` global bool):
- **Powerflow** (`g_screensaver = false`): counts down `g_pf_secs`; when zero and `Photo Count > 0`, calls `activate_screensaver`. Also advances both slot timers.
- **Screensaver** (`g_screensaver = true`): counts down `g_photo_secs` per photo; when zero, advances `g_photo_idx`. When all photos shown, calls `deactivate_screensaver`.

Slot timers (`g_slot_secs`) pause during screensaver and resume on return.

### Image memory constraint

`online_image` with `transparency: chroma_key` enables `LV_DRAW_SW_SUPPORT_RGB565A8`, required for SW-renderer image scaling.

**Always hide `ss_img_widget` before calling `lv_scr_load_anim()`** (done in both `activate_screensaver` and `deactivate_screensaver`). If the old decoded image is visible during a slide transition, LVGL allocates a 3.6 KB RGB565A8 draw-strip buffer which OOMs on a fragmented heap → crash.

Photos must be pre-resized to **exactly 128×96** (24,576 bytes decoded) before uploading.  
Upload to HA: `config/www/screensaver/01.jpg`, `02.jpg` … `NN.jpg`.  
`ss_img_widget` uses `scale: 2.5` + `antialias: false` → fills 320×240 exactly.

**OOM mitigations** (5 in place — confirmed working via 75-min soak test, heap minimum ~45,000 bytes):
1. `on_boot` `reserve()` pre-allocates `g_photo_queue`/`g_photo_order` before heap fragments.
2. All icon glyph changes use **`lv_label_set_text_static()`** — glyph arrays are `static const char*` in flash; no heap copy/free cycle from animation ticking.
3. `ha_solar`, `ha_home`, `ha_battery` `on_value` LVGL updates skipped during screensaver — eliminates ~90 unnecessary heap alloc/free pairs per 60s screensaver.
4. `ss_img_widget` hidden before `lv_scr_load_anim()` — prevents 3.6 KB RGB565A8 draw-strip allocation.
5. `ss_img.release()` called before every `set_url()` — frees old decoded image buffer BEFORE new download starts, preventing double-buffer overlap OOM. (The 5s heap sensor poll misses this sub-second overlap window — graph can look healthy while OOM occurs.)

**Resolution guide** (320×240 display, all measurements confirmed via 75-min soak test):

| Resolution | Decoded Size | Scale | Headroom | Status |
|-----------|-------------|-------|---------|--------|
| 160×120 | 38,400 bytes | ×2.0 | ~6,600 bytes | ⚠️ Thin margin, not recommended |
| **128×96** | **24,576 bytes** | **×2.5** | **~20,400 bytes** | ✅ **Recommended** — proven safe |
| 80×60 | 9,600 bytes | ×4.0 | ~35,400 bytes | ✅ Ultra-safe fallback |

If OOM returns: drop to **80×60** (`scale: 4.0`), or upgrade to CYD S3 N16R8 (8 MB PSRAM).

All photo URLs include a random `?r=XXXXXXXX` query string (e.g. `01.jpg?r=A3F2C100`). This ensures `online_image` always sees a new URL and fires `on_download_finished` — even when the same photo number is selected twice in a row (e.g. `Photo Count = 1`). HA ignores the query parameter and serves the same static file.

### `on_download_finished` flow
1. Guard: if `!g_screensaver`, ignore (screensaver deactivated mid-download)
2. `lvgl.image.update` — set new image source on widget
3. `lv_anim_del` — cancel any running animation **first** (before revealing widget)
4. Per-branch: set correct initial position/opacity → `lv_obj_clear_flag(HIDDEN)` → start animation:
   - **Fade**: reset x/y to 0, clear hidden, animate opacity TRANSP→COVER
   - **Slide**: set off-screen x/y, clear hidden, animate x/y to 0
   - **Instant**: reset x/y to 0, set opa COVER, clear hidden

### Photo order
`g_photo_queue` is the **persistent** full list `[1…count]`, built once and Fisher-Yates shuffled with `esp_random()` (hardware RNG) when Photo Order = Random. Each `activate_screensaver` call takes the **next `cycle_size` slice** from `g_photo_queue_pos` and stores it in `g_photo_order`. The queue is rebuilt (and re-shuffled) only when `g_photo_queue_pos` reaches the end or the photo count changes — guaranteeing all photos appear before any repeats.

### Transition implementation
**Page transitions**: `lv_scr_load_anim()` — direction is reversed on return (Slide Left enter → Slide Right exit).  
**Per-photo fade**: `lv_anim_t` animating `lv_obj_set_style_opa` from `LV_OPA_TRANSP → LV_OPA_COVER` over `ss_anim_ms × 2` ms. `lv_anim_del()` called first to prevent stacked animations.  
**Per-photo slides**: `lv_anim_t` animating `lv_obj_set_x/y` from ±320/±240 → 0 over `ss_anim_ms × 2` ms. Image is positioned off-screen and made fully opaque before the animation; `lv_obj_set_x/y(0,0)` is reset at the start of every `on_download_finished` so interrupted slides don't leave the widget stranded.  
**Per-photo instant**: position and opacity reset directly — logged as `[Instant]`.  
**Random**: fresh `esp_random() % 6` roll (0=Instant, 1=Fade, 2–5=Slides) per transition, independently for enter, exit, and each per-photo transition. Stored in `g_photo_trans` so `on_download_finished` uses the same roll that was made at download time.

### `->obj` rule for raw LVGL C functions
- Regular widgets (`label`, `image`, `obj`): `id(widget)` returns `lv_obj_t*` — pass directly
- Pages (`main_page`, `screensaver_page`): `id(page)` returns `LvPageType*` — use `id(page)->obj`
- ESPHome YAML actions (`lvgl.label.update` etc.): always use `id(widget)` — never `->obj`

### Screensaver HA controls

| Entity | ID | Range | Default |
|--------|----|-------|---------|
| Powerflow Mins | `ss_powerflow_mins` | 1–15 min | 1 |
| Photo Secs | `ss_photo_secs` | 1–15 s | 5 |
| Photo Count | `ss_photo_count` | 0–99 photos | 17 (0 = disabled) |
| Photos Per Cycle | `ss_cycle_size` | 1–20 photos | 3 |
| Animation Duration | `ss_anim_ms` | 500–2500 ms, step 500 | 1000 | Page transitions use this value; per-photo Fade/Slide use 2× |
| Transition Effect | `ss_transition` | None/Fade/Slide*/Random | Random |
| Photo Order | `ss_photo_order` | Sequential/Random | Random |

### WARN logs to watch
```
Screensaver — photo 3 [page:Slide Down | photo:Fade] fetching http://…/03.jpg  ← cycle start
Screensaver → photo 1 [Fade] — fetching http://…/01.jpg       ← per-photo advance (fade)
Screensaver → photo 2 [Slide Left] — fetching http://…/02.jpg ← per-photo advance (slide)
Screensaver → photo 3 [Instant] — fetching http://…/03.jpg    ← per-photo advance (instant)
→ Power Flow [Slide Right] (next screensaver in 60s)           ← returned to powerflow
Screensaver: photo NN fetch failed                             ← check URL / HA
advance_batt_slot: disabled — staying at slot 0 (Power W)     ← Batt Time Estimate switch is OFF
```
When Transition = **Random**, the page-transition log shows the *actual effect chosen*.

---

## Sensor → Display Update Mapping

Sensors either update LVGL **directly** (no caching) or **cache + conditionally update**. Understanding this prevents confusion about why a sensor change doesn't update the display.

| Sensor ID | Caches to | Updates display when |
|-----------|-----------|---------------------|
| `ha_solar` | `g_solar_val` | `val_solar`/`val_solar_unit` only when `!g_screensaver`; calls `update_solar_icon_state` always (has internal screensaver guard) |
| `ha_home` | `g_home_val` | `val_home`/`icon_home`/`val_home_unit` only when `!g_screensaver` |
| `ha_battery` | `g_batt_soc_val` | `icon_battery`+`val_battery` only when `!g_screensaver`; calls `update_batt_display` only if `!g_screensaver && g_batt_slot == 1`; always calls `update_batt_icon_state` |
| `ha_load2` | `g_load2_val` | `!g_screensaver && g_slot_idx == 0` |
| `ha_load3` | `g_load3_val` | `!g_screensaver && g_slot_idx == 1` |
| `ha_load1` | `g_load1_val` | `!g_screensaver && g_slot_idx == 2` |
| `ha_battery_power` | `g_batt_power_val` | `!g_screensaver && g_batt_slot == 0` |
| `ha_batt_current` | `g_batt_current_val` | `!g_screensaver && g_batt_slot == 1` |
| `ha_srne_charge_limit` | `g_srne_charge_limit_val` | `!g_screensaver && g_batt_slot == 1` |
| `ha_srne_discharge_limit` | `g_srne_discharge_limit_val` | `!g_screensaver && g_batt_slot == 1` |

Cached values (`g_load*_val`, `g_batt_*_val`, `g_solar_val`, `g_home_val`) are always up-to-date even if the display isn't updated — `update_slot_display` and `update_batt_display` always read from globals, not re-read from sensors. `val_solar`, `val_home`, `val_battery`/`icon_battery` are restored immediately on screensaver exit via `deactivate_screensaver` (reads cached globals), so the display is always current when the power flow screen appears.

**Why `ha_solar`/`ha_home`/`ha_battery` are guarded**: `lvgl.label.update` calls `lv_label_set_text()` internally, which does a heap free+malloc on every call. During screensaver the widgets aren't visible (on `main_page`) so these allocs are pure waste that fragment the heap, eventually causing JPEG decode OOM.

---

## Color Threshold Quick Reference

Icon and value labels **always share the same thresholds** — update both together.
All color thresholds are declared as named `substitutions:` — edit there, never inline.

**Solar Power** (W) — `solar_color_green/blue/yellow` substitutions
| Range | Color |
|-------|-------|
| > 3600 | 🟢 Green |
| > 2000 | 🔵 Blue |
| > 900  | 🟡 Yellow |
| ≤ 900  | 🟠 Orange (includes 0 W / no sun) |

**Home Load** (VA) — `home_color_green/blue/yellow` substitutions
| Range | Color |
|-------|-------|
| < 1000 | 🟢 Green (includes 0 VA) |
| < 2000 | 🔵 Blue |
| < 3000 | 🟡 Yellow |
| ≥ 3000 | 🟠 Orange |

**Home Load Slots** (W — top-right cycling) — per-load `load1/2/3_color_green/blue/yellow` substitutions

| Load | Green below | Blue below | Yellow below | Orange |
|------|-------------|------------|--------------|--------|
| Load 1 (Ecoflow River 3) | 140 W | 200 W | 400 W | ≥ 400 W |
| Load 2 (A/C 1st Floor)   | 600 W | 740 W | 900 W | ≥ 900 W |
| Load 3 (A/C 3rd Floor)   | 80 W  | 900 W | 1000 W | ≥ 1000 W |

All values (including 0 W) follow the table — no special idle color.

**Battery SOC** (%) — `batt_color_green/blue/yellow` substitutions
| Range | Color |
|-------|-------|
| ≥ 80 | 🟢 Green |
| ≥ 50 | 🔵 Blue |
| ≥ 25 | 🟡 Yellow |
| < 25 | 🟠 Orange |

**Battery Power** (W, slot 0) — `batt_pwr_green` / `batt_pwr_blue` substitutions
| Range | Color |
|-------|-------|
| ≥ 3600 | 🟢 Green (fast charge) |
| ≥ 900  | 🔵 Blue (moderate charge) |
| > 0    | 🟡 Yellow (slow / trickle) |
| = 0    | 🩶 Grey `0x888888` (idle — matches cloud icon) |
| < 0    | 🟠 Orange (any discharge) |

**Battery Time Estimate** (slot 1) — see *Battery time estimate slot colors* above.

---

## Global Variables Reference

| ID | Type | Purpose |
|----|------|---------|
| `g_lvgl_ready` | bool | `false` until `on_boot -10` fires; guards all LVGL calls from early `on_value` |
| `g_screensaver` | bool | false = powerflow, true = screensaver |
| `g_pf_secs` | int | Powerflow countdown (s); reset by `deactivate_screensaver`; set by `ss_powerflow_mins.on_value` |
| `g_photo_trans` | int | Per-photo reveal: 0=Instant 1=Fade 2=SlideLeft 3=SlideRight 4=SlideUp 5=SlideDown; set before each download |
| `g_photo_secs` | int | Current photo countdown (s) |
| `g_photo_idx` | int | Position in `g_photo_order` (0-based) |
| `g_photo_order` | `std::vector<int>` | Current activation's play list — slice of `g_photo_queue` |
| `g_photo_queue` | `std::vector<int>` | Persistent full queue (shuffled/sequential); rebuilt only when exhausted or count changes |
| `g_photo_queue_pos` | int | Next index to consume from `g_photo_queue` |
| `g_slot_idx` | int | Active Home Load slot: 0=Load 2, 1=Load 3, 2=Load 1 |
| `g_slot_secs` | int | Shared countdown for both Home Load AND Battery slot advances |
| `g_load1_val` | float | Cached `${load1_entity}` |
| `g_load2_val` | float | Cached `${load2_entity}` |
| `g_load3_val` | float | Cached `${load3_entity}` |
| `g_batt_slot` | int | Battery slot: 0=Power W, 1=Time estimate |
| `g_batt_power_val` | float | Cached `${battery_power_entity}` |
| `g_batt_soc_val` | float | Cached `${battery_entity}` (%) |
| `g_batt_current_val` | float | Cached `${batt_current_entity}` (A, + charging, − discharging) |
| `g_srne_charge_limit_val` | float | Cached `${srne_charge_limit_entity}` (%) |
| `g_srne_discharge_limit_val` | float | Cached `${srne_discharge_limit_entity}` (%) |
| `g_batt_anim_state` | int | Battery icon animation state: 0=static 1=charging(step up) 2=discharging(step down) 3=alert(flash) |
| `g_batt_anim_step` | int | Current glyph index in cycling animation (0=battery_full … 6=battery_0_bar); reset to SOC level on state change |
| `g_solar_val` | float | Cached solar power (W) from `${solar_entity}`; used by `update_solar_icon_state` |
| `g_home_val` | float | Cached `${home_entity}` (VA); restored by `deactivate_screensaver` |
| `g_solar_anim_state` | int | Solar icon animation state: 0=static-cloudy 1=glyph-cycle(producing) 2=glyph-cycle(night) |
| `g_solar_anim_step` | int | Current glyph index: state 1 → 0–7 (sun ping-pong on `icon_solar`); state 2 → 0–4 (moon cycle on `icon_solar_moon`) |

---

## Script Reference

| ID | Purpose |
|----|---------|
| `update_slot_display` | Updates all 5 Home Load slot widgets from cached globals; first line guards `g_lvgl_ready` |
| `advance_slot` | Finds next eligible slot (Load 1 always eligible), updates `g_slot_idx`, calls `update_slot_display` |
| `update_batt_display` | First line guards `g_lvgl_ready`. Slot 0: shows W/kW, independently recomputes time estimate and publishes HA sensors (slot 0 uses `_`-prefixed locals to avoid naming conflicts with slot 1 path). Slot 1: checks eligibility, shows time estimate or falls through to slot 0. HA sensor publishes throttled to 60s via `static _first_pub`/`_last_pub_ms`. Called by ha_battery_power (slot 0), ha_batt_current / ha_battery / ha_srne_* (slot 1), advance_batt_slot, batt_time_enabled.on_turn_off, deactivate_screensaver, and on_boot |
| `advance_batt_slot` | Toggles `g_batt_slot` 0↔1; ineligible/disabled → stays at 0. Logs (when batt_log_enabled): "disabled" / "slot1→0" / full eligibility details |
| `update_batt_icon_state` | Sets `g_batt_anim_state` and resets `g_batt_anim_step` based on current `g_batt_soc_val` / `g_batt_current_val`. Priority: SOC<${batt_thresh_alert}%→alert, current<−0.5A→discharge, current>+0.5A→charge, else→static. When going to static (state 0), immediately restores correct glyph via `lv_label_set_text_static` and color via `batt_color_*` substitutions (≥80 Green, ≥50 Blue, ≥25 Yellow, else Orange) — matches `val_battery` thresholds exactly. Called from `ha_battery.on_value`, `ha_batt_current.on_value`, `deactivate_screensaver`, `on_boot -10` |
| `update_solar_icon_state` | Sets `g_solar_anim_state` based on `g_solar_val` and PHT time. **State 0** (day + no power): shows `wb_cloudy` grey on `icon_solar`, static. **State 1** (producing >0W): shows `solar_power` glyph on `icon_solar` initially, 250ms interval ping-pongs through 8-step sun sequence (Yellow). **State 2** (night + no power): shows `nights_stay` on `icon_solar_moon` initially, 250ms interval cycles through 5-step moon sequence (Blue). Visibility toggled via `LV_OBJ_FLAG_HIDDEN` on **every call** (not only on state change) to prevent ghost icon artifacts. Called from `ha_solar.on_value`, `deactivate_screensaver`, `on_boot -10`, and 1s interval (when `g_solar_val <= 0`) |
| `activate_screensaver` | Builds play order, hides image widget, starts page transition, fetches first photo |
| `deactivate_screensaver` | Hides image widget, transitions back to main page, calls both display updates + `update_batt_icon_state` + `update_solar_icon_state`. Also restores `val_solar`/`val_solar_unit`, `val_home`/`val_home_unit`/`icon_home`, and `val_battery`/`val_battery` color from cached globals so all tiles are current the moment the power flow screen appears |
