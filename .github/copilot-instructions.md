# Flock-You Project Copilot Instructions

## Project Overview
Flock-You is an ESP32-based BLE (Bluetooth Low Energy) device scanner and tracker. It detects nearby BLE devices, logs detections, serves a web dashboard, and persists session data to SPIFFS flash storage.

## Supported Hardware

Two boards are supported, each with its own PlatformIO environment in `platformio.ini`.

| Environment | Board | Flash | Flash mode | Buzzer GPIO |
|-------------|-------|-------|------------|-------------|
| `esp32devkitc_16mb` | ESP32-DevKitC-32E | 16 MB physical / 4 MB mapped | DIO, 40 MHz | GPIO 25 |
| `xiao_esp32s3` | Seeed XIAO ESP32-S3 | 8 MB | QIO, 80 MHz | GPIO 4 (D3) |

- **Framework:** Arduino (via PlatformIO)
- **Platform:** espressif32 ^6.3.0
- **CPU:** 240 MHz dual-core (LX6 on DevKitC, LX7 on S3)
- **Partitions:**
  - DevKitC: `partitions.csv` — factory 3 MB app + 960 KB SPIFFS
  - XIAO S3: `partitions_s3.csv` — factory 6 MB app + 2 MB SPIFFS

### Board-specific notes

**ESP32-DevKitC-32E:**
- USB-serial via onboard CP2102/CH340 bridge
- `board_build.flash_size = 16MB` causes `_init` crash with `esp32dev` toolchain — **do not add it**
- Flash confirmed DIO mode; QIO causes boot crash on this hardware revision
- **Never use GPIO3** — it is UART0 RX (U0RXD), shared with the USB-UART bridge

**Seeed XIAO ESP32-S3:**
- Native USB CDC — requires `-DARDUINO_USB_MODE=1 -DARDUINO_USB_CDC_ON_BOOT=1` build flags
- `board_build.flash_size = 8MB` is safe and required on S3
- First-time flash: hold BOOT button while connecting USB
- `Serial` output only appears after USB CDC enumeration — boot messages may need a `delay(1500)` to be visible
- Buzzer on GPIO4 (pin D3 on XIAO header)

## Build System
- **Tool:** PlatformIO (NOT Arduino IDE)
- **Config:** `platformio.ini`
- **Default environment:** `esp32devkitc_16mb`
- **Commands:**
  - `pio run` — build default env
  - `pio run -e xiao_esp32s3` — build S3 env
  - `pio run [-e <env>] -t upload` — flash
  - `pio run [-e <env>] -t uploadfs` — upload SPIFFS image
  - `pio device monitor` — serial monitor (115200 baud)

## Buzzer GPIO
`BUZZER_PIN` is set at compile time via `-DFY_BUZZER_PIN=N` per environment. The default fallback in `src/main.cpp` is `25`. Do not hard-code `BUZZER_PIN = 25` — always reference the macro.

```cpp
#ifndef FY_BUZZER_PIN
#  define FY_BUZZER_PIN 25
#endif
#define BUZZER_PIN FY_BUZZER_PIN
```

## Core Libraries
| Library | Version | Purpose |
|---|---|---|
| `h2zero/NimBLE-Arduino` | ^1.4.0 | BLE scanning (use NimBLE API, NOT classic ESP32 BLE) |
| `mathieucarbou/ESP Async WebServer` | ^3.0.6 | Async HTTP server and WebSocket |
| `bblanchon/ArduinoJson` | ^7.0.4 | JSON serialization/deserialization |
| `SPIFFS` | built-in | Persistent flash storage |
| `ESPmDNS` | built-in | mDNS — advertises device as `flockyou.local` on STA network |

## Critical Library Notes

### NimBLE (BLE)
- Always use `NimBLE-Arduino` API, never the classic `BLEDevice` / `BLEScan` API
- Include: `#include <NimBLEDevice.h>`
- Enabled via build flag: `-DCONFIG_BT_NIMBLE_ENABLED=1`
- Callbacks use `NimBLEAdvertisedDeviceCallbacks`

### ESP Async WebServer
- All request handlers are async lambdas: `[](AsyncWebServerRequest *r) { ... }`
- Never block inside a request handler (no `delay()`, no long loops)
- Use `AsyncWebServerResponse` for custom headers
- WebSocket uses `AsyncWebSocket`
- JSON bodies handled via `AsyncCallbackJsonWebHandler` from `AsyncJson.h`
- Recommended build flags for stability:
  - `-D CONFIG_ASYNC_TCP_RUNNING_CORE=1`
  - `-D CONFIG_ASYNC_TCP_STACK_SIZE=16384`

### ArduinoJson v7
- Use `JsonDocument` (NOT deprecated `StaticJsonDocument` or `DynamicJsonDocument`)
- Deserialize: `deserializeJson(doc, input)`
- Serialize: `serializeJson(doc, output)`
- Access: `doc["key"].as<Type>()`

### SPIFFS
- Always check `fySpiffsReady` before any SPIFFS operation
- Always check `SPIFFS.exists(path)` before reading a file
- Mount with `SPIFFS.begin(true)` (true = format on fail)
- Max filename length: 32 chars including leading `/`
- Key files: `FY_SESSION_FILE` (`/session.json`), `FY_PREV_FILE` (`/prev_session.json`), `FY_WIFI_FILE` (`/wifi.json`), `FY_PAT_FILE` (`/patterns.json`)

## Source Structure
```
src/main.cpp          # All firmware logic (single-file Arduino sketch style)
api/flockyou.py       # Flask companion app for desktop analysis
api/templates/        # Flask HTML templates
datasets/             # Sample/test datasets
partitions.csv        # Partition table — ESP32-DevKitC-32E (4 MB mapped)
partitions_s3.csv     # Partition table — XIAO ESP32-S3 (8 MB)
platformio.ini        # PlatformIO build configuration (two environments)
```

## Firmware Architecture (`src/main.cpp`)
- **Single-file** Arduino-style sketch
- Global prefix convention: `fy` (e.g., `fyServer`, `fySpiffsReady`, `FY_PREV_FILE`)
- Web server instance: `AsyncWebServer fyServer(80)`
- API routes mounted under `/api/`
- BLE scan runs continuously, populates detection map
- SPIFFS stores prior session JSON for history/export

## WiFi Modes
The ESP32 runs in `WIFI_AP_STA` mode at all times:
- **AP mode** (`flockyou` / `flockyou123`) — always active at `192.168.4.1`; phone connects directly, losing cellular
- **STA mode** (optional) — joins a phone hotspot so the phone retains cellular data; dashboard also reachable at `flockyou.local`. Credentials stored in `/wifi.json` (SPIFFS). Non-blocking connect logic runs in `loop()` via `fySTAConnectPending` flag.
- **mDNS** — `flockyou.local` advertised via `ESPmDNS` when STA is connected
- STA state variables: `fySTAConnected`, `fySTAConnecting`, `fySTAConnectPending`, `fySTAIP`, `fySavedSSID`, `fySavedPass`
- Companion mode (BLE GATT client connected) no longer disables WiFi — it only adjusts BLE scan duty cycle

## API Endpoints Pattern
```cpp
fyServer.on("/api/route", HTTP_GET, [](AsyncWebServerRequest *r) {
    // handler - never block here
    r->send(200, "application/json", payload);
});
```

## GPIO Assignments

**ESP32-DevKitC-32E**
| GPIO | Function |
|------|----------|
| 25 | Piezo buzzer (`FY_BUZZER_PIN=25`) |
| 21 | LED (optional) |

**Seeed XIAO ESP32-S3**
| GPIO | XIAO label | Function |
|------|------------|----------|
| 4 | D3 | Piezo buzzer (`FY_BUZZER_PIN=4`) |
| 21 | — | LED (optional, if broken out) |

## Flask Companion App (`api/`)
- Python Flask application
- Run: `cd api && pip install -r requirements.txt && python flockyou.py`
- URL: `http://localhost:5000`
- Accepts JSON, CSV, KML import from ESP32 exports
- Supports live serial ingestion from USB-connected ESP32

## Code Style Conventions
- C++11/14 features are available (ESP-IDF toolchain)
- Use `String` (Arduino) for dynamic strings in firmware
- Prefer `const char*` for string literals in web responses
- Lambda captures in web handlers: capture by value `[=]` for local vars, avoid capturing large objects
- Serial debug: `Serial.println(F("..."))` using `F()` macro for flash strings
- Log level controlled by `-DCORE_DEBUG_LEVEL=0` (set to 5 for verbose)

## Common Pitfalls to Avoid
1. **Never** use `delay()` inside async web server handlers
2. **Never** call SPIFFS functions without checking `fySpiffsReady`
3. **Never** use classic BLE API (`BLEDevice::init`) — always NimBLE
4. **Never** use `StaticJsonDocument` or `DynamicJsonDocument` (ArduinoJson v7 removed them)
5. Avoid allocating large buffers on the stack in tasks/callbacks — use heap (`new`/`malloc`)
6. WebSocket broadcasts must be called from the correct RTOS core — use `CONFIG_ASYNC_TCP_RUNNING_CORE=1`
7. **Never** set `fySTAConnectPending` from inside an async handler and then block waiting for it — the connection completes asynchronously in `loop()`
8. **Never** use `GPIO3` for the buzzer or any output — it is UART0 RX and shared with the USB-UART bridge

## When Adding New API Endpoints
1. Add route in `src/main.cpp` using `fyServer.on(...)`
2. Keep handler non-blocking
3. Return `application/json` for data endpoints
4. Use `Content-Disposition: attachment` header for file downloads
5. Mirror the endpoint in `api/flockyou.py` if desktop app parity is needed

## WiFi API Endpoints
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/wifi/status` | Returns `ap_ip`, `sta_connected`, `sta_connecting`, `sta_ip`, `sta_ssid` |
| `POST` | `/api/wifi/sta` | Body: `application/x-www-form-urlencoded` with `ssid=...&pass=...`. Saves to `/wifi.json` and queues STA connect. |
| `GET` | `/api/wifi/clear` | Removes `/wifi.json`, disconnects STA, reverts to AP-only |

## When Modifying Build Config
- Edit `platformio.ini` only — do not create per-file build overrides
- Settings shared by both boards go in the `[common]` section; board-specific settings go in the environment section
- Adding libraries: append to `lib_deps` in `[common]` with pinned version using `@^X.Y.Z`
- Adding build flags: if both boards need them, add to each env's `build_flags`; if board-specific, add only to that env
- Buzzer GPIO is set via `-DFY_BUZZER_PIN=N` per environment — do not hard-code the value in `src/main.cpp`