# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ESPTimeCast is a WiFi-connected LED matrix clock and weather display for ESP8266/ESP32 platforms using 4x MAX7219 devices chained into an 8x32 LED matrix. It fetches time via NTP, weather from OpenWeatherMap, and exposes a web UI for configuration plus Home Assistant integration.

## Building & Uploading

**Arduino IDE 1.x only** (NOT 2.x — library compatibility issues).

**Required Libraries** (install via Library Manager):
- `ArduinoJson` (Benoit Blanchon)
- `MD_Parola` + `MD_MAX72xx` (majicDesigns)
- `ESPAsyncWebServer` 3.9.1+ (ESP32Async)
- `AsyncTCP` (ESP32) or `ESPAsyncTCP` (ESP8266)

**Partition scheme (ESP32):** "Minimal SPIFFS (1.9MB APP with OTA / 128KB SPIFFS)" for OTA support (v1.4.1+).
**Flash size (ESP8266):** "4MB FS:2MB OTA"

No separate LittleFS upload is needed — the web UI is embedded in the sketch as `index_html.h` (PROGMEM).

**Debugging:** Serial monitor at 115200 baud. Log prefixes: `[TIME]`, `[WEATHER]`, `[DISPLAY]`, `[WIFI]`, `[CONFIG]`. Call `printConfigToSerial()` to dump all settings.

## Architecture

### Dual-Platform Code Parity (Critical)

Two nearly-identical sketches exist:
- `ESPTimeCast_ESP32/ESPTimeCast_ESP32.ino` (~5100 lines)
- `ESPTimeCast_ESP8266/ESPTimeCast_ESP8266.ino` (~5100 lines)

**Changes must be made to BOTH files.** Platform differences are limited to WiFi/HTTP headers (`WiFi.h` vs `ESP8266WiFi.h`), mDNS registration, `timegm()` workaround, and pin assignments.

Header files (`basic_font.h`, `index_html.h`, `days_lookup.h`, `months_lookup.h`, `tz_lookup.h`, `version.h`) are duplicated in each sketch folder and must also stay in sync.

### Global State (No OOP)

All settings are individual global variables (char arrays, ints, bools) — not a class. They are populated at boot via `loadConfig()` from `/config.json` (LittleFS) and saved via `saveConfig()`. There is no RTOS or task scheduler; the main `loop()` is a non-blocking state machine using `millis()` comparisons.

### Display Mode System

Nine modes cycle via `advanceDisplayMode()`:

| Mode | Name | Auto-Advance |
|------|------|-------------|
| 0 | Clock | Yes, after `clockDuration` (10s default) |
| 1 | Weather (temp) | Yes, after `weatherDuration` (5s default) |
| 2 | Weather description | No (scroll-based) |
| 3 | Countdown | No (time-based) |
| 4 | Nightscout glucose | Yes |
| 5 | Date | Yes |
| 6 | Custom message | Manual |
| 7 | Timer | Manual |
| 8 | Subway | Manual |

On every mode change, reset `lastSwitch`, `currentScrollCount`, and `messageStartTime` — failing to do so causes animation glitches or skipped modes.

### Three Async Data Sources

1. **NTP** — fetched on boot; state machine: `NTP_IDLE → NTP_SYNCING → NTP_SUCCESS/NTP_FAILED`; first retry at 30s, then every 5 min; max 30 retries; displays "! NTP" on failure.
2. **OpenWeatherMap** — fetched every 5 minutes (`fetchInterval = 300000`); stores `mainDesc` (temperature string) and `detailedDesc` (weather type); also updates sunrise/sunset for auto-dimming; API key must be exactly 32 chars.
3. **Web UI** — async `ESPAsyncWebServer` on port 80; HTML served from PROGMEM; config saved to LittleFS on every POST.

### Key Files

- **Main sketches**: `ESPTimeCast_ESP32/ESPTimeCast_ESP32.ino`, `ESPTimeCast_ESP8266/ESPTimeCast_ESP8266.ino` — all application logic in one file per platform
- **`index_html.h`** — embedded PROGMEM HTML/CSS/JS (117KB); form POSTs to `/set_custom_message`, `/set_brightness`, `/export`, `/upload`, `/factory_reset`, `/api/message`
- **`basic_font.h`** — 8x8 pixel character bitmaps; uses `&` as escape for special glyphs (e.g., `"s&u&n"` → SUN icon); fallback when `mfactoryfont.h` is absent
- **`tz_lookup.h`** — IANA timezone → POSIX TZ string mappings for SNTP client
- **`days_lookup.h`**, **`months_lookup.h`** — day/month names in 28 languages
- **`version.h`** — firmware version (`v1.4.1`) and board-type detection macros

### Pin Assignments

```
ESP32 S2 Mini: CLK=7, CS=11, DATA=12
ESP32 default: CLK=18, CS=23, DATA=5
ESP32-S3:      CLK=18, CS=16, DATA=17
ESP32-C3:      CLK=4,  CS=10, DATA=6
ESP8266 D1:    CLK=14, CS=13, DATA=15
MAX7219 VCC → 5V USB (not 3.3V)
```

### Configuration Persistence

- **File**: `/config.json` on LittleFS (auto-created with defaults on first boot)
- **Load**: `loadConfig()` at boot — deserializes JSON into globals
- **Save**: `saveConfig()` — called after any config POST with 1.2s debounce
- **Export/Import**: `/export` (GET → download), `/upload` (POST → restore)
- **Factory reset**: `/factory_reset` endpoint (AP mode only)

### MD_Parola Display Library

- `P.displayAnimate()` must be called every loop iteration to advance animations
- `P.displayText()` queues non-blocking scroll animations
- Brightness: 0–15; `-1` = display completely off (`P.displayShutdown(true)`)
- `HARDWARE_TYPE = MD_MAX72XX::FC16_HW`, `MAX_DEVICES = 4`

## Common Tasks

### Adding a Display Mode
1. Define new mode value (next available: 9)
2. Add case in `loop()` and in `advanceDisplayMode()` (reset all state vars)
3. Update both `.ino` files

### Adding a Web UI Setting
1. Add HTML input to `index_html.h`
2. Add global variable (with default) at sketch top
3. Add to `loadConfig()`, `saveConfig()`, and `printConfigToSerial()`
4. Add/update POST handler
5. Update both `.ino` files

### Home Assistant Integration
POST to `/api/message` with:
- `message=text` (required, max 120 chars)
- `scrolltimes=N` (optional; expire after N scroll cycles)

Temporary messages do not overwrite the persistent Web UI message and expire by time or scroll count.
