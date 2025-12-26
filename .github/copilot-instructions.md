# ESPTimeCast AI Coding Assistant Instructions

## Project Overview
ESPTimeCast is a WiFi-connected LED matrix clock and weather display for ESP8266/ESP32 platforms using MAX7219 LED matrices. It fetches time via NTP, weather from OpenWeatherMap, and provides a web UI for configuration plus Home Assistant integration for custom scrolling messages.

## Critical Architecture Patterns

### Dual Platform with Shared Logic (Not Duplicated)
- **Separate folders**: `ESPTimeCast_ESP32/` and `ESPTimeCast_ESP8266/` contain their own `.ino` files (~3700 lines each)
- **Nearly identical code**: Both platforms share the same logic, not via includes but through maintained parity
- **Platform differences**: Only in WiFi/HTTP headers (`WiFi.h` vs `ESP8266WiFi.h`), mDNS, timegm() workarounds, and pin definitions
- **When editing**: Changes must be made to BOTH `.ino` files to keep platforms in sync — failing to do so breaks one platform
- **Pin standardization**: All pins are defined at sketch top; ESP32 uses different GPIO numbers than ESP8266 (see pin table in README)

### Global State & Display Loop Architecture
- **No objects for config**: Settings stored as individual global variables (ssid[], brightness, dimmingEnabled, etc.), loaded from LittleFS JSON via `loadConfig()`
- **Display mode cycle**: Modes 0-1 (clock/weather) auto-advance with `advanceDisplayMode()` after `clockDuration`/`weatherDuration` milliseconds
- **Display modes**: 0=Clock, 1=Weather (temp), 2=Weather Description, 3=Countdown, 4-6=Custom Messages (persistent from Web UI)
- **Timer-based switching**: `millis() - lastSwitch > duration` pattern controls when to advance; no task scheduler
- **Volatile state**: `lastSwitch`, `currentScrollCount`, `messageStartTime` track animation progress and must be reset when mode changes

### Three Separate Data Sources (Async, Non-Blocking)
1. **NTP Time**: Fetched on boot + retry logic (30s first retry, 5min subsequent); shows "! NTP" if failed; doesn't block loop
2. **OpenWeatherMap**: Fetched every 5 minutes; stores `mainDesc` (temp) and `detailedDesc` (weather type); API key validated (32 chars)
3. **Web UI Form Data**: Saved to LittleFS as JSON; on POST `/set_custom_message`, message persists until cleared in UI

### MD_Parola Display Library Usage
- `P.displayText()` queues scrolling animation (non-blocking)
- `P.displayAnimate()` advances animation; must call in loop every iteration
- Custom font from `mfactoryfont.h` uses `&` as escape for special characters (e.g., `"s&u&n"` → "SUN" with special U)
- 4 MAX7219 devices chained = 8x32 display; brightness 0-15, -1=display off

## Key Files
- **Main sketches**: [ESPTimeCast_ESP32/ESPTimeCast_ESP32.ino](ESPTimeCast_ESP32/ESPTimeCast_ESP32.ino), [ESPTimeCast_ESP8266/ESPTimeCast_ESP8266.ino](ESPTimeCast_ESP8266/ESPTimeCast_ESP8266.ino) (3710 lines each, must stay in sync)
- **Web UI**: [index_html.h](ESPTimeCast_ESP32/index_html.h) (embedded PROGMEM HTML/CSS/JS, form POSTs to `/set_custom_message`, `/set_brightness`, etc.)
- **Custom font**: [mfactoryfont.h](ESPTimeCast_ESP32/mfactoryfont.h) (8x8 pixel char bitmaps with `&` escape encoding)
- **I18N data**: [days_lookup.h](ESPTimeCast_ESP32/days_lookup.h), [months_lookup.h](ESPTimeCast_ESP32/months_lookup.h), [tz_lookup.h](ESPTimeCast_ESP32/tz_lookup.h) (language maps, IANA timezones)
- **Config storage**: `/config.json` (LittleFS, auto-created from defaults on first boot)

## Development Workflow

### Building & Uploading
1. **Arduino IDE 1.x** (NOT 2.x due to library compatibility issues)
2. **Install board packages**:
   - ESP32: `Espressif Systems` → Select "LOLIN S2 Mini" (or variant)
   - ESP8266: `ESP8266 Community` → Select "Wemos D1 Mini" (or variant)
3. **Install required libraries** (Library Manager):
   - `ArduinoJson` (Benoit Blanchon) — JSON parsing
   - `MD_Parola` + `MD_MAX72xx` (majicDesigns) — LED matrix control
   - `ESPAsyncWebServer` (3.9.1+) + `AsyncTCP`/`ESPAsyncTCP` — Web server
4. **Configure partition scheme**:
   - ESP32: Set "Partition Scheme" → "No OTA (2MB APP/2MB SPIFFS)"
   - ESP8266: Set "Flash Size" → "4MB FS:2MB OTA" (ensures 2MB LittleFS)
5. **Upload sketch**: Click Upload button. **No separate LittleFS upload needed** — web UI is embedded in sketch as `index_html.h`

### First Boot & AP Mode
- Device auto-enters AP mode if WiFi unavailable
- **SSID**: `ESPTimeCast`, **Password**: `12345678`
- Navigate to `http://192.168.4.1` or `http://setup.esp` in captive portal
- Configure WiFi + OpenWeatherMap API key, then save
- Device reboots and joins WiFi; IP displayed on LED matrix

### Configuration Persistence
- Settings saved to `/config.json` (LittleFS) via `saveConfig()`
- On boot, `loadConfig()` reads JSON and populates globals (char arrays, bools, ints)
- Export/import endpoints: `/export` (downloads config.json), `/upload` (restore from JSON)
- Factory reset: `/factory_reset` (AP mode only, erases all config)

### Debugging
- **Serial output** at 115200 baud logs prefixed by context: `[TIME]`, `[WEATHER]`, `[DISPLAY]`, `[WIFI]`
- `printConfigToSerial()` dumps all current settings
- Use `Serial.printf()` for debugging; avoid blocking operations
- Monitor WiFi events via event callbacks (ESP32: `WiFi.onEvent()`, ESP8266: `WiFiEventHandler`)

## Code Patterns & Conventions

### Pin Assignments (Standardized — October 2024 Update)
**Critical**: Verify your board matches these assignments before first power-on. Display VCC now uses **5V USB rail** (not 3.3V) to prevent regulator damage.

```cpp
// ESP32 (LOLIN S2 Mini)
#define CLK_PIN 7    // GPIO7
#define CS_PIN 11    // GPIO11  
#define DATA_PIN 12  // GPIO12

// ESP8266 (Wemos D1 Mini)  
#define CLK_PIN 14   // D5 (GPIO14)
#define CS_PIN 13    // D7 (GPIO13)
#define DATA_PIN 15  // D8 (GPIO15)

// Both: MAX7219 VCC → 5V USB, GND → GND
```

### Font Encoding
- Custom font uses `&` as **escape character** for special glyphs
- Example: `"s&u&n"` renders as "SUN" where `&u` = special "U" character
- Used in `days_lookup.h` (day abbreviations) and `months_lookup.h` (month abbreviations)
- Non-English chars auto-converted to closest ASCII (e.g., "Ñ" → "N")

### Time Display Logic
- `timeOffsetMinutes`: Shifts **display only**, not internal logic (NTP still correct)
- `twelveHourToggle`: 12/24-hour format
- `colonBlinkEnabled`: Replaces ":" with " " to simulate blink (toggled every ~500ms)

### Display Mode Switching (Auto-Advance)
```cpp
// Modes 0-1 (clock/weather) auto-advance after duration
if ((displayMode == 0 || displayMode == 1) && millis() - lastSwitch > duration) {
  advanceDisplayMode();
}
// Modes 2-6 follow their own logic; countdown (3) is time-based
```
- **Reset state on mode change**: Always reset `currentScrollCount`, `messageStartTime`, `lastSwitch` when entering a new mode
- Failing to reset causes animation glitches or modes to skip

### Weather Fetching & Caching
- **Interval**: Every 5 minutes (`fetchInterval = 300000`)
- **API**: OpenWeatherMap with `units=metric` or `imperial`
- **API Key**: Must be exactly 32 characters; validated in web UI
- **Returned data**: 
  - `mainDesc` = temperature string (e.g., "21°C")
  - `detailedDesc` = weather type (e.g., "scattered clouds"), translated per `language` setting
- **Fallback**: If fetch fails, display "! TEMP"; re-attempt next interval

### Dimming Logic
- **Auto-dimming**: Uses sunrise/sunset from weather API, updates every 5 minutes
- **Custom dimming**: `dimStartHour`/`dimEndHour` control manual dimming window
- **Brightness levels**: 0 (off) → 15 (full); `-1` disables display entirely
- **displayOffByDimming**: Set when in dimming window AND auto-dimming enabled

### NTP State Machine
```cpp
enum NtpState { NTP_IDLE, NTP_SYNCING, NTP_SUCCESS, NTP_FAILED };
```
- **Retry logic**: First retry 30s after failure, subsequent retries every 5min
- **Timeout**: 30s per attempt; max 30 retries per sync attempt
- **Display feedback**: "! NTP" animation on failure; normal time display on success
- **Timezone handling**: SNTP client uses timezone string from `tz_lookup.h` (IANA format)

### Web Server Endpoints
All POST endpoints save to config and trigger `saveConfig()`:
- `/set_custom_message`: `POST` → persistent scrolling message (up to 120 chars)
- `/set_brightness`: `POST` → brightness value (0-15)
- `/set_weather_units`: `POST` → "metric" or "imperial"
- `/export`: `GET` → download `config.json`
- `/upload`: `POST` → restore `config.json`
- `/factory_reset`: `POST` → erase config (AP mode only)

### Home Assistant Integration
- Send temporary messages via POST to `/api/message` (optional parameters):
  - `message=text` (required)
  - `scrolltimes=N` (optional: expire after N scroll cycles)
  - Short messages (≤8 chars) use `weatherDuration`; long messages scroll once per cycle
- Temp messages **do not overwrite** persistent Web UI messages
- Expires by time or scroll count; then displays next mode (clock → weather → …)

## Error Handling
- **NTP failure**: Display "! NTP" with retry animation; log to serial at `[TIME]` prefix
- **Weather failure**: Display "! TEMP" or fallback to clock; log at `[WEATHER]` prefix
- **WiFi disconnect**: Auto-attempts reconnect every 5 seconds; AP mode fallback after timeout
- **LittleFS errors**: Gracefully fall back to hardcoded defaults; log to serial
- **API validation**: OpenWeatherMap key must be exactly 32 chars; rejected if shorter/longer
- **JSON parsing errors**: `ArduinoJson` silently fails on malformed data; use `printConfigToSerial()` to verify

## Internationalization
- Language codes: 2-letter ISO (en, de, fr, es, ja, etc.)
- Fallback to English if language not found in lookup tables
- Weather descriptions translated via OpenWeatherMap API response
- Day/month names use special `&` encoding for non-ASCII chars
- Non-English characters auto-converted to ASCII closest match (e.g., "Ñ" → "N", "é" → "e")

## LittleFS Management
- **Embedded HTML**: Written to `/index.html` on first boot (though web UI served from PROGMEM)
- **Config storage**: `/config.json` with full settings (WiFi, API key, display modes, dimming, etc.)
- **Uptime tracking**: `/uptime.txt` tracks system health
- **Auto-initialization**: If `/config.json` missing, loads hardcoded defaults and saves
- **Space management**: ESP32 = 2MB LittleFS, ESP8266 = 2MB LittleFS; no cleanup needed unless adding large files

## Common Tasks
### Adding New Display Mode
1. Define new `displayMode` value (currently 0-6 used; next = 7)
2. Add case in main loop `loop()` function
3. Initialize mode state in `advanceDisplayMode()` (reset timers, `lastSwitch`, scroll counters)
4. Add logic to `P.displayAnimate()` or `P.displayText()` calls
5. Test mode transitions and state resets with serial monitor
6. **Update both `.ino` files** to keep platforms in sync

### Adding Web UI Setting
1. Add form input to `/*** Web UI HTML ***/` section in `index_html.h` 
2. Add global variable for the setting (top of sketch, with type and default)
3. Add to `loadConfig()` (JSON deserialization with default fallback)
4. Add to `saveConfig()` (JSON serialization)
5. Add to `printConfigToSerial()` (debugging output)
6. Add POST handler or modify existing handler to process form data
7. Test with AP mode; verify persistence across reboot

### Modifying Pin Layout
- Update `#define` constants at sketch top
- Update README wiring table (critical for user reference)
- Test with both ESP32 and ESP8266 boards
- **Consider power supply**: MAX7219 draws significant current from 5V; 3.3V regulator may not handle it
- Add comments explaining hardware justification for changes

### Testing Changes
- **Serial monitor** (115200 baud) for real-time logs
- Test NTP sync on boot and retry behavior (force WiFi timeout to verify)
- Verify weather API responses with network traffic inspection
- Check display animations and mode transitions with stopwatch
- Test AP mode fallback (disconnect from WiFi, power cycle)
- Verify config persistence: change setting, power off/on, check if retained
- **Platform verification**: Build & upload to both ESP8266 and ESP32 to catch platform-specific bugs

## Platform Differences
- **ESP32**: Uses `WiFi.h`, `HTTPClient.h`, event callbacks (`WiFi.onEvent()`)
- **ESP8266**: Uses `ESP8266WiFi.h`, `ESP8266HTTPClient.h`, event handlers (`WiFiEventHandler`)
- **Pin numbers differ significantly** but functionally equivalent; verify pin constants at sketch top
- **Partition schemes optimized**: ESP32 uses "No OTA (2MB APP/2MB SPIFFS)", ESP8266 uses "4MB FS:2MB OTA"
- **timegm() missing**: Both platforms lack standard timegm(); custom emulation in sketch for countdown timestamp handling
- **mDNS**: Both support, but registered differently (see sketch initialization)

## Dependencies
- **ArduinoJson**: JSON parsing/serialization (required for config persistence)
- **MD_Parola/MD_MAX72xx**: LED matrix control (animation, scrolling, special effects)
- **ESPAsyncWebServer**: Web server for config UI (non-blocking, efficient memory)
- **LittleFS**: File system for `/config.json` and `/index.html`
- **SNTP**: NTP client for time synchronization
- **WiFiClientSecure**: HTTPS for secure OpenWeatherMap API calls

## Security Notes
- API keys masked in web UI (`********`) but stored in plaintext in LittleFS `/config.json`
- WiFi passwords stored plaintext in `/config.json`
- No authentication on web endpoints — assumes local network only
- Config export masks API keys in response headers but not in downloaded file
- Consider device physical access security for local network use cases</content>
<parameter name="filePath">/Users/stevesiden/dev/ESPTimeCast/ESPTimeCast/.github/copilot-instructions.md