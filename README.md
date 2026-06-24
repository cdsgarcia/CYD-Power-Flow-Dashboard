# CYD Power Flow Dashboard

[![Validate](https://github.com/cdsgarcia/CYD-Power-Flow-Dashboard/actions/workflows/validate.yml/badge.svg)](https://github.com/cdsgarcia/CYD-Power-Flow-Dashboard/actions/workflows/validate.yml)
[![GitHub release](https://img.shields.io/github/v/release/cdsgarcia/CYD-Power-Flow-Dashboard)](https://github.com/cdsgarcia/CYD-Power-Flow-Dashboard/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![ESPHome](https://img.shields.io/badge/ESPHome-2024.x+-blue)](https://esphome.io)
[![Platform](https://img.shields.io/badge/Platform-ESP32-red)](https://www.espressif.com)

An ESPHome configuration for the **CYD (Cheap Yellow Display)** ESP32 that renders a real-time Home Assistant power flow dashboard on a 320×240 ILI9341 display. When idle, it transitions to a photo screensaver with configurable transitions and play order.

> 📸 Add a photo of your device to `docs/preview.jpg` to show it here.

---

## Features

- **Real-time power flow** — Solar, Home Load, Battery SOC + Power, all updated live from Home Assistant
- **Battery time estimate** — Calculates and displays target charge/discharge completion time
- **Animated icons** — Battery icon animates charging/discharging direction; solar icon animates production with sun ping-pong (day) or moon cycle (night)
- **Photo screensaver** — Cycles through photos from your HA `/local/screensaver/` folder with fade/slide/instant transitions
- **Configurable from HA** — Timers, thresholds, labels, brightness, screensaver settings all exposed as HA entities
- **PHT clock** — Clock header always shows correct local time (UTC+8) regardless of HA timezone config
- **Heap monitoring** — Exposes SRAM free/largest-block sensors to HA for OOM diagnostics

---

## Hardware

**CYD ESP32-2432S028** (Cheap Yellow Display)

| Component | Details |
|-----------|---------|
| MCU | ESP32-WROOM-32 (no PSRAM) |
| Display | 2.8" ILI9341, 240×320, SPI |
| Touch | XPT2046 resistive (wired, not yet configured) |
| SRAM | 520 KB (no PSRAM) |
| Flash | 4 MB |

### Pin Map

| Function | Pins |
|----------|------|
| TFT SPI | CLK=GPIO14, MOSI=GPIO13, MISO=GPIO12, CS=GPIO15, DC=GPIO2 |
| Backlight | GPIO21 (LEDC PWM) |
| Touch SPI | CLK=GPIO25, MOSI=GPIO32, MISO=GPIO39, CS=GPIO33, IRQ=GPIO36 |

---

## Display Layout (320×240 landscape)

```
┌─────────────────────────────────┐
│          HH:MM:SS AM/PM         │  Clock header
├────────────────┬────────────────┤
│  Solar Power   │  Load Slots    │  Top-right: 3-slot cycling (A/C 1F / A/C 2F / Ecoflow)
├─────────────────────────────────┤
│     Battery Power (full width)  │  Cycles: W/kW ↔ charge/discharge time estimate
├────────────────┬────────────────┤
│  Battery SOC   │  Home Load     │  Bottom-right: home apparent power (VA/kVA)
└────────────────┴────────────────┘
```

---

## Prerequisites

- [ESPHome](https://esphome.io) ≥ 2024.x
- Home Assistant with the following sensors configured:

| Substitution Key | Entity Example | Notes |
|-----------------|---------------|-------|
| `solar_entity` | `sensor.srne_pv_power` | PV power in W |
| `load2_entity` | `sensor.a_c_1f_power_meter_power` | Load slot 2 (W) |
| `load3_entity` | `sensor.a_c_3f_power_meter_power` | Load slot 3 (W) |
| `load1_entity` | `sensor.ef_r30241_ac_input_power` | Load slot 1 (W) |
| `battery_entity` | `sensor.battery_soc_mean` | Battery SOC (%) |
| `home_entity` | `sensor.srne_load_l1_apparent_power` | Home apparent power (VA) |
| `battery_power_entity` | `sensor.total_battery_power` | Battery power W (negative = discharging) |
| `batt_current_entity` | `sensor.total_battery_current` | Battery current A (negative = discharging) |
| `srne_charge_limit_entity` | `number.srne_batttery_charge_limit` | Charge limit % |
| `srne_discharge_limit_entity` | `number.srne_batttery_discharge_limit` | Discharge limit % |

---

## Installation

Choose the method that matches your setup. For full step-by-step instructions including Windows setup, USB drivers, web flasher, and troubleshooting, see the **[Installation wiki page](https://github.com/cdsgarcia/CYD-Power-Flow-Dashboard/wiki/Installation)**.

---

### Method A — ESPHome Dashboard (Web UI)

Recommended for most users. Works via the **HA ESPHome add-on** or the standalone web app at [web.esphome.io](https://web.esphome.io) (Chrome / Edge required).

1. Download `cyd-e713b0.yaml` and `secrets.yaml.example` from the [latest release](https://github.com/cdsgarcia/CYD-Power-Flow-Dashboard/releases)
2. Rename `secrets.yaml.example` → `secrets.yaml` and fill in your WiFi credentials
3. Edit the `substitutions:` block in `cyd-e713b0.yaml` to match your HA entity IDs and `ha_url`
4. Add to ESPHome Dashboard → flash via USB (first time) → OTA for all future updates

---

### Method B — Command Line

Requires Python 3.9–3.12 and ESPHome installed. See the [Installation wiki](https://github.com/cdsgarcia/CYD-Power-Flow-Dashboard/wiki/Installation#method-b--command-line) for full setup including:
- Python + ESPHome install on Windows
- USB driver installation (CH340 / CP2102)
- Compile-only and web flasher options
- Corporate network / firewall workarounds

Quick reference once installed:
```cmd
python -m esphome config cyd-e713b0.yaml    # validate
python -m esphome compile cyd-e713b0.yaml   # build only
python -m esphome run cyd-e713b0.yaml       # build + flash USB
python -m esphome upload cyd-e713b0.yaml    # OTA update
```

---

## Screensaver Photos

Photos are served from Home Assistant's local www folder:

1. Resize your photos before uploading — see the resolution guide below.
2. Upload to: `config/www/screensaver/01.jpg`, `02.jpg`, … `NN.jpg`
3. Set **Photo Count** in the HA device card to match your total photo count.

### Photo resolution guide

| Resolution | Decoded Size | Scale | Headroom | Recommendation |
|-----------|-------------|-------|---------|----------------|
| 160×120 | 38,400 bytes | ×2.0 | ~6,600 bytes | ⚠️ Thin margin — not recommended |
| **128×96** | **24,576 bytes** | **×2.5** | **~20,400 bytes** | ✅ **Recommended** — proven safe |
| 80×60 | 9,600 bytes | ×4.0 | ~35,400 bytes | ✅ Ultra-safe fallback |

> **Why the limit?** The ESP32 has no PSRAM (520 KB SRAM only). The JPEG decoder needs a single contiguous free block ≥ the decoded image size. Confirmed heap minimum after all OOM mitigations = ~45,000 bytes (75-min soak test).

---

## Home Assistant Controls

After flashing, the device exposes the following controls in HA:

| Entity | Type | Description |
|--------|------|-------------|
| Display Backlight | Light | Brightness control (saved to NVS) |
| Powerflow Mins | Number | Minutes before screensaver activates |
| Photo Secs | Number | Seconds per photo in screensaver |
| Photo Count | Number | Total number of photos (0 = disabled) |
| Photos Per Cycle | Number | Photos shown per screensaver activation |
| Animation Duration | Number | Transition animation speed (ms) |
| Transition Effect | Select | None / Fade / Slide* / Random |
| Photo Order | Select | Sequential / Random |
| Slot Cycle Secs | Number | How fast the load/battery slots cycle |
| Battery Capacity Ah | Number | Your battery bank capacity |
| Batt Time Threshold | Number | Hours threshold to show time estimate |
| Batt Time Estimate | Switch | Enable/disable battery time estimate |
| Load 1/2/3 Label | Text | Customisable labels for load slots |

---

## Heap Monitoring

Three diagnostic sensors are published to HA every 5 seconds:

| Sensor | Description |
|--------|-------------|
| `Heap Free` | Total free SRAM bytes |
| `Heap Largest Block` | Largest contiguous free block — must stay ≥ 24,576 for screensaver |
| `Loop Time` | ESP32 main loop time (ms) |

Add a **History Graph** card in HA tracking `sensor.cyd_heap_largest_block` with a reference line at 24576 to monitor screensaver health.

---

## File Structure

| File | Purpose |
|------|---------|
| `cyd-e713b0.yaml` | **Production config — the only file to edit and flash** |
| `secrets.yaml.example` | Credentials template (copy to `secrets.yaml` and fill in) |

---

## Support

If this project has been useful to you, consider buying me a coffee ☕

### <img src="docs/logo-ln.png" width="20"> Lightning
<img src="docs/qr-lightning.png" width="200"><br>
`greatjogging67@walletofsatoshi.com`

### <img src="docs/logo-xrp.png" width="20"> XRP
<img src="docs/qr-xrp.png" width="200"><br>
`rpWJmMcPM4ynNfvhaZFYmPhBq5FYfDJBZu`<br>
Destination Tag: `2135058530`

### <img src="docs/logo-btc.png" width="20"> BTC
<img src="docs/qr-btc.png" width="200"><br>
`bc1q5tqqew0wlpkdz22crltreu5ngc9sdje9hzr4vv`

---

## License

MIT License — see [LICENSE](LICENSE) for details.
