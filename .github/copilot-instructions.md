# CYD-E713B0 — Repo-Specific Instructions

> Generic conventions, hardware, color thresholds, build commands, and animation details
> are in the CYD skill (`~/.copilot/skills/CYD/SKILL.md`). This file covers only
> E713B0-specific layout, slot logic, screensaver internals, and variable/script references.

---

## File Roles

| File | Purpose |
|------|---------|
| `cyd-e713b0.yaml` | **Production config — only file to edit and flash** |
| `cyd-e713b0-BatteryIconTest.yaml` | Battery icon test scaffold (reference only) |
| `cyd-e713b0-SolarIconTest.yaml` | Solar icon test scaffold (reference only) |
| `cyd-e713b0-Basic.yaml` | Minimal boot/wifi scaffold (reference only) |
| `cyd-e713b0-test-ble-proxy.yml` | BLE proxy experiment (reference only) |

---

## LVGL Layout (320×240 landscape)

```
┌─────────────────────────────────┐
│        Time Header (HH:MM:SS AM)│  lbl_time — header_font (size 40)
├─────────────────────────────────┤  divider at y=34
│  Solar Power   │  Home Load     │  top-left x:0,y:25 / top-right x:160,y:25
│  (top-left)    │  Slots cycling │
├─────────────────────────────────┤
│    Battery Power (full width)   │  val_battery_power — value_font (size 50)
│    cycles with: Charges at /    │  lbl_batt_title cycles; val_battery_power shows time
│    Empty at (time estimate)     │
├────────────────┬────────────────┤
│  Battery SOC   │  Home Load     │  bottom-left x:0,y:160 / bottom-right x:160,y:160
│  (bottom-left) │  (bottom-right)│
└────────────────┴────────────────┘
```

Each quadrant `obj`: icon label (outer edge), title label (top), value label (center), unit label (bottom).
**Do not change positions or sizes** — optimised for the physical display.

---

## Home Load Slot Cycling (top-right quadrant)

Rotates between 3 slots every `Slot Cycle Secs`:

| Slot | Sensor | Default label | Icon | Visible when |
|------|--------|--------------|------|-------------|
| 0 | `${load2_entity}` | "A/C 1st Floor" | snowflake `\ueb3b` | value > `load2_threshold` |
| 1 | `${load3_entity}` | "A/C 3rd Floor" | snowflake `\ueb3b` | value > `load3_threshold` |
| 2 | `${load1_entity}` | "Ecoflow River 3" | elec. services `\ue63c` | **always shown** |

`advance_slot`: tries `(g_slot_idx + 1) % 3`; skips slot 0/1 if value ≤ threshold; slot 2 always eligible.

`update_slot_display` updates: `lbl_slot`, `icon_slot` (snowflake, hidden slot 2), `icon_load1` (shown slot 2 only), `val_slot`, `val_slot_unit`.

HA controls:

| Entity | ID | Range | Default |
|--------|----|-------|---------|
| Slot Cycle Secs | `slot_cycle_secs` | 1–15 s | 5 — shared timer for Home Load + Battery |
| Load 2 Threshold | `load2_threshold` | 0–500 W | 10 |
| Load 3 Threshold | `load3_threshold` | 0–500 W | 10 |

HA-editable labels (`text:`, max 16 chars, label_font charset only):

| ID | Default |
|----|---------|
| `load1_label` | "Ecoflow River 3" |
| `load2_label` | "A/C 1st Floor" |
| `load3_label` | "A/C 3rd Floor" |

---

## Battery Power Slot Cycling (center, full width)

| Slot | Content | Condition |
|------|---------|-----------|
| 0 | Battery power W/kW | Always shown |
| 1 | Target wall-clock time | Only when within `Batt Time Threshold` hours of limit |

Time estimate formula:
```
Charging  (current > +0.5A): hours = (charge_limit  − SOC) / 100 × capacity_Ah / current
Discharging (current < −0.5A): hours = (SOC − discharge_limit) / 100 × capacity_Ah / |current|
```

Display format: `"8:40 AM"` / `"12:30 PM"` (no leading zero on hour).
Title label adds day context: same day / "- Next Day" / "- Next N Days".

`advance_batt_slot` checks eligibility before switching to slot 1 — stays at 0 if ineligible.
`update_batt_display` re-checks on slot 1 and falls back to slot 0 inline if current drops to 0.
Slot 0 independently recomputes the time estimate and publishes HA sensors to prevent oscillation.

HA controls:

| Entity | ID | Range | Default |
|--------|----|-------|---------|
| Battery Capacity Ah | `battery_capacity_ah` | 200–700, step 5 | 300 Ah |
| Batt Time Threshold | `batt_time_threshold_hrs` | 1–48 h | 18 |
| Batt Time Estimate | `batt_time_enabled` | switch | ON |
| Batt Log Enabled | `batt_log_enabled` | switch | OFF |

---

## Doorbell HA Controls

| Entity | ID | Range / Type | Default |
|--------|----|-------------|---------|
| Doorbell Enabled | `doorbell_enabled` | switch | ON |
| Doorbell LED Enabled | `doorbell_led_enabled` | switch | ON |
| Doorbell Duration Secs | `doorbell_duration_secs` | 3–30 s | 10 |

LED-only — no screen effect on E713B0. LED block runs before screensaver gate so it flashes even during screensaver.

---

## Daily Restart HA Controls

| Entity | ID | Range / Type | Default |
|--------|----|-------------|---------|
| Daily Restart Hour | `daily_restart_hour` | 0–23 hr | 3 |
| Daily Restart Enabled | `daily_restart_enabled` | switch | **OFF** |

Fires via `on_time: seconds: 0, minutes: 0` (top of every hour); checks PHT hour via `gmtime_r`.

Published HA sensors (throttled 60s, first call always immediate):

| Entity | ID | Unit | Notes |
|--------|----|------|-------|
| Battery Hours to Full | `batt_hours_to_full` | h | |
| Battery Hours to Empty | `batt_hours_to_empty` | h | |
| Battery Full At | `batt_full_at_min` | min | minutes-since-midnight (0–1440) |
| Battery Empty At | `batt_empty_at_min` | min | minutes-since-midnight |
| Battery Full At Time | `batt_full_at_time` | text | e.g. "10:45 AM" / "Full" / "Not Charging" |
| Battery Empty At Time | `batt_empty_at_time` | text | e.g. "6:30 PM" / "At Limit" / "Not Discharging" |

---

## Battery SOC Icon Glyph Table

| idx | Glyph | Unicode | Color | SOC range |
|-----|-------|---------|-------|-----------|
| 0 | battery_full | `\ue1a4` | Green | ≥ 98% |
| 1 | battery_5_bar | `\uebd4` | Green | ≥ 85% |
| 2 | battery_4_bar | `\uebe2` | Blue | ≥ 70% |
| 3 | battery_3_bar | `\uebdd` | Blue | ≥ 55% |
| 4 | battery_2_bar | `\uebe0` | Yellow | ≥ 40% |
| 5 | battery_1_bar | `\uebd9` | Yellow | ≥ 25% |
| 6 | battery_0_bar | `\uebdc` | Orange | ≥ 10% |
| 7 | battery_alert | `\ue19c` | Orange | < 10% |

State=0 (static): color from `batt_color_*` substitutions — always matches `val_battery`.

---

## Solar Icon Glyph Tables

**State 1 — Sun ping-pong (8 steps, Yellow, `icon_solar`):**

| Step | Glyph | Unicode |
|------|-------|---------|
| 0 | solar_power | `\uec0f` |
| 1 | brightness_7 | `\ue3ac` |
| 2 | brightness_4 | `\ue3a9` |
| 3 | brightness_6 | `\ue3ab` |
| 4 | brightness_5 | `\ue3aa` |
| 5 | brightness_6 | `\ue3ab` |
| 6 | brightness_4 | `\ue3a9` |
| 7 | brightness_7 | `\ue3ac` |

**State 2 — Moon cycle (5 steps, Blue, `icon_solar_moon` — Material Symbols Outlined only):**

| Step | Glyph | Unicode |
|------|-------|---------|
| 0 | nights_stay | `\uf174` |
| 1 | moon_stars | `\uf34f` |
| 2 | star_shine | `\uf31d` |
| 3 | auto_awesome | `\ue65f` |
| 4 | flare | `\ue3e4` |

⚠️ Moon codepoints exist only in `gfonts://Material Symbols Outlined` — not in classic Material Icons.

`g_solar_anim_step` resets to 0 only on state change; same-state re-calls do not reset (protects cycle).

---

## Screensaver

### State Machine
1s interval: powerflow counts down `g_pf_secs` → when zero + Photo Count > 0 → `activate_screensaver`.
Screensaver counts down `g_photo_secs` per photo → advances `g_photo_idx` → when all shown → `deactivate_screensaver`.
Slot timers (`g_slot_secs`) pause during screensaver and resume on return.

### `on_download_finished` Flow
1. Guard: if `!g_screensaver`, ignore (deactivated mid-download)
2. `lvgl.image.update` — set new source
3. `lv_anim_del` — cancel running animation first
4. Set initial position/opacity → `lv_obj_clear_flag(HIDDEN)` → start animation:
   - **Fade**: opacity TRANSP→COVER
   - **Slide**: start off-screen x/y → animate to 0
   - **Instant**: reset position/opacity, clear hidden

### Photo Order
`g_photo_queue` — persistent full list `[1…count]`, Fisher-Yates shuffled with `esp_random()`.
Each `activate_screensaver` takes the next `cycle_size` slice into `g_photo_order`.
Queue rebuilt only when `g_photo_queue_pos` reaches end or photo count changes.

### Screensaver HA Controls

| Entity | ID | Range | Default |
|--------|----|-------|---------|
| Powerflow Mins | `ss_powerflow_mins` | 1–15 min | 1 |
| Photo Secs | `ss_photo_secs` | 1–15 s | 5 |
| Photo Count | `ss_photo_count` | 0–99 | 17 (0 = disabled) |
| Photos Per Cycle | `ss_cycle_size` | 1–20 | 3 |
| Animation Duration | `ss_anim_ms` | 500–2500 ms, step 500 | 1000 |
| Transition Effect | `ss_transition` | None/Fade/Slide*/Random | Random |
| Photo Order | `ss_photo_order` | Sequential/Random | Random |

### WARN Logs to Watch
```
Screensaver — photo 3 [page:Slide Down | photo:Fade] fetching http://…/03.jpg
Screensaver → photo 2 [Slide Left] — fetching http://…/02.jpg
→ Power Flow [Slide Right] (next screensaver in 60s)
Screensaver: photo NN fetch failed                        ← check URL / HA
advance_batt_slot: disabled — staying at slot 0 (Power W) ← Batt Time Estimate OFF
```

---

## Sensor → Display Update Mapping

| Sensor ID | Caches to | Updates display when |
|-----------|-----------|---------------------|
| `ha_solar` | `g_solar_val` | `val_solar`/unit: `!g_screensaver`; `update_solar_icon_state`: always |
| `ha_home` | `g_home_val` | `val_home`/icon/unit: `!g_screensaver` |
| `ha_battery` | `g_batt_soc_val` | icon+val: `!g_screensaver`; `update_batt_display`: `!g_screensaver && g_batt_slot==1`; `update_batt_icon_state`: always |
| `ha_load2` | `g_load2_val` | `!g_screensaver && g_slot_idx==0` |
| `ha_load3` | `g_load3_val` | `!g_screensaver && g_slot_idx==1` |
| `ha_load1` | `g_load1_val` | `!g_screensaver && g_slot_idx==2` |
| `ha_battery_power` | `g_batt_power_val` | `!g_screensaver && g_batt_slot==0` |
| `ha_batt_current` | `g_batt_current_val` | `!g_screensaver && g_batt_slot==1` |
| `ha_srne_charge_limit` | `g_srne_charge_limit_val` | `!g_screensaver && g_batt_slot==1` |
| `ha_srne_discharge_limit` | `g_srne_discharge_limit_val` | `!g_screensaver && g_batt_slot==1` |

---

## Global Variables Reference

| ID | Type | Purpose |
|----|------|---------|
| `g_lvgl_ready` | bool | LVGL init guard — set in `on_boot -10` |
| `g_screensaver` | bool | false=powerflow, true=screensaver |
| `g_pf_secs` | int | Powerflow countdown (s) |
| `g_photo_trans` | int | 0=Instant 1=Fade 2=SlideL 3=SlideR 4=SlideU 5=SlideD |
| `g_photo_secs` | int | Current photo countdown (s) |
| `g_photo_idx` | int | Position in `g_photo_order` (0-based) |
| `g_photo_order` | `vector<int>` | Current cycle play list |
| `g_photo_queue` | `vector<int>` | Persistent full queue |
| `g_photo_queue_pos` | int | Next index to consume from queue |
| `g_slot_idx` | int | Active slot: 0=Load2, 1=Load3, 2=Load1 |
| `g_slot_secs` | int | Shared slot countdown (Home Load + Battery) |
| `g_load1_val` | float | Cached load1 (W) |
| `g_load2_val` | float | Cached load2 (W) |
| `g_load3_val` | float | Cached load3 (W) |
| `g_batt_slot` | int | 0=Power W, 1=Time estimate |
| `g_batt_power_val` | float | Cached battery power (W) |
| `g_batt_soc_val` | float | Cached battery SOC (%) |
| `g_batt_current_val` | float | Cached battery current (A) |
| `g_srne_charge_limit_val` | float | Cached charge limit (%) |
| `g_srne_discharge_limit_val` | float | Cached discharge limit (%) |
| `g_batt_anim_state` | int | 0=static 1=charging 2=discharging 3=alert |
| `g_batt_anim_step` | int | Glyph index 0–7 |
| `g_solar_val` | float | Cached solar power (W) |
| `g_home_val` | float | Cached apparent power (VA) |
| `g_solar_anim_state` | int | 0=cloudy 1=sun 2=moon |
| `g_solar_anim_step` | int | State1: 0–7 sun; State2: 0–4 moon |
| `g_doorbell_active` | bool | True while LED alert is running |
| `g_doorbell_ticks` | int | Countdown in 250ms ticks |
| `g_doorbell_last_state` | string | HA button state — filters boot deliveries |

---

## Script Reference

| ID | Purpose |
|----|---------|
| `update_slot_display` | Refreshes Home Load slot widgets from cached globals; `g_lvgl_ready` guard first line |
| `advance_slot` | Finds next eligible slot (Load 1 always eligible), sets `g_slot_idx`, calls display update |
| `update_batt_display` | Slot 0: W/kW + independently publishes HA sensors. Slot 1: time estimate or fallback to slot 0. HA publishes throttled 60s. `g_lvgl_ready` guard first line |
| `advance_batt_slot` | Toggles `g_batt_slot` 0↔1; ineligible/disabled → stays at 0 |
| `update_batt_icon_state` | Sets `g_batt_anim_state` from SOC + current; restores glyph+color on → static |
| `update_solar_icon_state` | Sets `g_solar_anim_state` from solar power + PHT hour; toggles widget visibility unconditionally |
| `activate_screensaver` | Builds play order, hides image widget, starts page transition, fetches first photo |
| `deactivate_screensaver` | Returns to main page, restores all tile displays from cached globals |
