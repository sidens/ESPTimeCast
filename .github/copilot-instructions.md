# ESPTimeCast AI Coding Assistant Instructions

## Project Overview
ESPTimeCast is an Arduino-based WiFi-connected LED matrix clock and weather display for ESP8266/ESP32 platforms. It features NTP time synchronization, OpenWeatherMap weather data, a web-based configuration interface, and Home Assistant integration.

## Architecture
- **Dual Platform Support**: Separate folders for ESP32 (`ESPTimeCast_ESP32/`) and ESP8266 (`ESPTimeCast_ESP8266/`) with platform-specific includes and pin configurations
- **Core Components**:
  - Main sketch (`.ino`) handles display logic, WiFi, NTP, weather fetching
  - `MD_Parola` library for LED matrix control (8x32 MAX7219)
  - `AsyncWebServer` for embedded web UI
  - `LittleFS` for persistent configuration storage
- **Data Flow**: NTP → Time display | OpenWeatherMap API → Weather display | Web UI → Config changes | Home Assistant API → Custom messages
- **Display Modes**: Clock (0), Weather (1), Weather Description (2), Countdown (3), Custom Messages (4-6)
- **State Management**: Global variables for config, timers for mode switching, NTP state machine

## Key Files
- `ESPTimeCast_ESP{32,8266}.ino`: Main application logic (3700+ lines)
- `index_html.h`: Embedded web UI (PROGMEM)
- `mfactoryfont.h`: Custom LED matrix font with special character encoding
- `{days,months,tz}_lookup.h`: Internationalization data structures
- `config.json`: Runtime configuration (LittleFS)

## Development Workflow
### Building
1. Use Arduino IDE 1.x (not 2.x)
2. Install board packages: ESP32 (Espressif) or ESP8266 (Community)
3. Install libraries via Library Manager:
   - ArduinoJson (Benoit Blanchon)
   - MD_Parola + MD_MAX72xx (majicDesigns)
   - ESPAsyncWebServer + ESPAsyncTCP/AsyncTCP (ESP32Async)
4. Select board: LOLIN S2 Mini (ESP32) or Wemos D1 Mini (ESP8266)
5. Set partition scheme: "No OTA (2MB APP/2MB SPIFFS)" for ESP32, Flash Size "4MB FS:2MB OTA" for ESP8266
6. Upload sketch (no separate LittleFS upload needed - web files embedded)

### Configuration
- Web UI at `http://esptimecast.local` or device IP
- Settings persist to LittleFS as JSON
- Export/import via `/export` and `/upload` endpoints
- AP mode fallback: SSID "ESPTimeCast", password "12345678"

### Debugging
- Serial output at 115200 baud
- NTP sync status: `[TIME]` logs
- Weather fetch: `[WEATHER]` logs
- Display mode changes: `[DISPLAY]` logs
- Use `printConfigToSerial()` for current settings

## Code Patterns & Conventions
### Pin Assignments (Standardized)
```cpp
// ESP32
#define CLK_PIN 7    // GPIO7
#define CS_PIN 11    // GPIO11  
#define DATA_PIN 12  // GPIO12

// ESP8266  
#define CLK_PIN 14   // D5
#define CS_PIN 13    // D7
#define DATA_PIN 15  // D8
```

### Font Encoding
- Custom font uses `&` as escape for special characters
- Example: `"s&u&n"` renders as "SUN" with special 'U'
- Used in `days_lookup.h` and `months_lookup.h`

### Time Display Logic
- `timeOffsetMinutes`: Affects display only, not internal logic (dimming, countdown)
- 12/24 hour toggle: `twelveHourToggle`
- Colon blink: `colonBlinkEnabled` (replaces ":" with " " when off)

### Display Mode Switching
```cpp
// Timer-based advancement for modes 0-1
if ((displayMode == 0 || displayMode == 1) && millis() - lastSwitch > duration) {
  advanceDisplayMode();
}
```

### Weather Fetching
- Interval: 5 minutes (`fetchInterval = 300000`)
- API: OpenWeatherMap with API key validation (32 chars)
- Units: "metric" (°C) or "imperial" (°F)

### Dimming Logic
- Auto-dimming: Sunrise/sunset from weather API
- Custom dimming: Configurable start/end times
- Brightness: 0-15 (LED levels) or -1 (display off)

### NTP State Machine
```cpp
enum NtpState { NTP_IDLE, NTP_SYNCING, NTP_SUCCESS, NTP_FAILED };
```
- Retry logic: First retry 30s, subsequent every 5min
- Timeout: 30s per attempt, max 30 retries

### Web Server Endpoints
- `/`: Main config UI
- `/set_custom_message`: POST for custom scrolling messages
- `/set_brightness`: POST for brightness control
- `/export`: Download config.json
- `/upload`: Upload config.json
- `/factory_reset`: Erase all config (AP mode only)

### Home Assistant Integration
- REST commands for messages and brightness
- Temporary vs persistent messages
- Expiration by time or scroll count

### Error Handling
- NTP failure: Display "! NTP" with retry animation
- Weather failure: Display "! TEMP" or fallback to clock
- AP mode: Animation loop until WiFi connects

### Internationalization
- Language codes: 2-letter ISO (en, de, fr, etc.)
- Fallback to English if language not found
- Weather descriptions translated via OpenWeatherMap API

### LittleFS Management
- Embedded HTML written to `/index.html` on first boot
- Config saved as `/config.json`
- Uptime tracking in `/uptime.txt`

## Common Tasks
### Adding New Display Mode
1. Increment `displayMode` enum
2. Add case in main loop
3. Update `advanceDisplayMode()` function
4. Handle mode transitions in timer logic

### Adding Web UI Setting
1. Add to `index_html.h` form
2. Add global variable
3. Add to `loadConfig()` and `saveConfig()`
4. Add to `printConfigToSerial()`

### Modifying Pin Layout
- Update `#define` constants
- Test with both ESP32/ESP8266
- Update README wiring table
- Consider hardware changes (VCC from 5V, not 3.3V)

### Testing Changes
- Use serial monitor for logs
- Test NTP sync reliability
- Verify weather API responses
- Check display animations and timings
- Test AP mode fallback

## Platform Differences
- **ESP32**: Uses `WiFi.h`, `HTTPClient.h`, event callbacks
- **ESP8266**: Uses `ESP8266WiFi.h`, `ESP8266HTTPClient.h`, event handlers
- Pin assignments differ but functionally equivalent
- Partition schemes optimized for LittleFS space

## Dependencies
- ArduinoJson: JSON parsing/serialization
- MD_Parola/MD_MAX72xx: LED matrix control
- ESPAsyncWebServer: Web server
- LittleFS: File system
- SNTP: NTP client
- WiFiClientSecure: HTTPS for weather API

## Security Notes
- API keys masked in web UI (`********`)
- WiFi passwords masked
- No authentication on web endpoints
- Config export masks sensitive data</content>
<parameter name="filePath">/Users/stevesiden/dev/ESPTimeCast/ESPTimeCast/.github/copilot-instructions.md