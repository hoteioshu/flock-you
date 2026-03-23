# Flock-You: Surveillance Device Detector

<img src="flock.png" alt="Flock You" width="300px">

**Standalone BLE surveillance device detector with web dashboard, GPS wardriving, and session persistence.**

Available as part of the OUI-SPY project at [colonelpanic.tech](https://colonelpanic.tech)

---

## Overview

Flock-You detects Flock Safety surveillance cameras, Raven gunshot detectors, and related monitoring hardware using BLE-only heuristics. It runs a WiFi access point with a live web dashboard on your phone, tags detections with GPS from your phone's browser, and exports everything as JSON, CSV, or KML for Google Earth.

No WiFi sniffing — the BLE radio scans continuously while WiFi serves the dashboard, using ESP32 dual-radio coexistence. The ESP32 can simultaneously join your phone's mobile hotspot so your phone retains cellular data while connected.

---

## Detection Methods

All detection is BLE-based:

| Method | Description |
|--------|-------------|
| **MAC prefix** | 20 known Flock Safety OUI prefixes (FS Ext Battery, Flock WiFi modules) |
| **BLE device name** | Case-insensitive substring match: `FS Ext Battery`, `Penguin`, `Flock`, `Pigvision` |
| **Manufacturer ID** | `0x09C8` (XUNTONG) — catches devices with no broadcast name. *From [wgreenberg/flock-you](https://github.com/wgreenberg/flock-you)* |
| **Raven service UUID** | Identifies Raven gunshot detectors by BLE GATT service UUIDs |
| **Raven FW estimation** | Determines firmware version (1.1.x / 1.2.x / 1.3.x) from advertised service patterns |

---

## Features

- **WiFi AP**: `flockyou` / password `flockyou123` — always active at `192.168.4.1`
- **WiFi STA (hotspot mode)**: join your phone's mobile hotspot so your phone keeps cellular data; dashboard also reachable at `flockyou.local`
- **Web dashboard** at `192.168.4.1` (or `flockyou.local` on hotspot) — live detection feed, pattern database, export tools
- **GPS wardriving** — phone GPS via browser Geolocation API auto-starts and tags every detection with coordinates
- **Session persistence** — detections auto-save to flash (SPIFFS) every 15 seconds
- **Prior session tab** — previous session survives reboot and is viewable in the PREV tab
- **Export formats**: JSON, CSV, and KML (Google Earth) — current and prior sessions
- **Serial output** — Flask-compatible JSON over serial for live desktop ingestion
- **200 unique device storage** with FreeRTOS mutex thread safety
- **Crow call boot sounds** — modulated descending frequency sweeps with warble texture
- **Detection alerts** — ascending chirps + descending caw on new device detection
- **Heartbeat** — soft double coo every 10s while a device stays in range

---

## Connecting Your Phone

There are two ways to use the dashboard on your phone. Both can be configured from the **TOOLS → WIFI MODE** section of the dashboard.

### Option A — Direct AP (simple, no internet on phone)

1. Connect your phone to the `flockyou` WiFi network (password `flockyou123`)
2. Open `http://192.168.4.1` in Chrome
3. Your phone will **lose cellular/internet** while connected to the AP

### Option B — Phone Hotspot (recommended, phone keeps cellular)

1. Turn on your phone's **mobile hotspot**
2. Connect to the `flockyou` AP first and open `http://192.168.4.1`
3. Go to **TOOLS → WIFI MODE → CONFIGURE HOTSPOT (STA)**
4. Enter your hotspot SSID and password, tap **CONNECT**
5. After ~20 seconds the ESP32 joins your hotspot. You can now:
   - Disconnect from `flockyou` AP and reconnect to your hotspot
   - Open `http://flockyou.local` in Chrome — dashboard loads over your hotspot with cellular data intact

Credentials are saved to flash and reconnected automatically on every boot.

---

## Enabling GPS (Android Chrome)

The dashboard uses your phone's GPS to geotag detections. GPS starts automatically when you open the dashboard. Because it's served over HTTP, Chrome requires a one-time flag to allow location access:

1. Open a new Chrome tab and go to `chrome://flags`
2. Search for **"Insecure origins treated as secure"**
3. Add `http://192.168.4.1` to the text field (also add `http://flockyou.local` if using hotspot mode)
4. Set the flag to **Enabled**
5. Tap **Relaunch**

After relaunching, the GPS indicator in the stats bar will show `OK` automatically once a fix is acquired.

> **Note:** iOS Safari does not support Geolocation over HTTP. GPS wardriving requires Android with Chrome.

---

## Hardware

**Board:** ESP32-DevKitC-32E (16MB flash)

| Pin | Function |
|-----|----------|
| GPIO 25 | Piezo buzzer |
| GPIO 21 | LED (optional) |

---

## Building & Flashing

Requires [PlatformIO](https://platformio.org/).

```bash
cd flock-you
pio run                       # build
pio run -t upload             # flash
pio device monitor            # serial output (115200 baud)
pio run -t uploadfs           # upload SPIFFS filesystem image
```

To flash to a specific COM port:

```bash
pio run -t upload --upload-port COM14
```

**Dependencies** (managed by PlatformIO):

- `NimBLE-Arduino` — BLE scanning
- `ESP Async WebServer` + `AsyncTCP` — web dashboard
- `ArduinoJson` — JSON serialization
- `SPIFFS` — session persistence to flash

---

## Flask Companion App

The `api/` folder contains a Flask web application for desktop analysis of detection data.

```bash
cd api
pip install -r requirements.txt
python flockyou.py
```

Open `http://localhost:5000` for the desktop dashboard.

**Import support:** JSON, CSV, and KML files exported from the ESP32 can be imported directly into the Flask app. Live serial ingestion is also supported — connect the ESP32 via USB and select the serial port in the Flask UI.

---

## Raven Gunshot Detector Detection

Flock-You identifies SoundThinking/ShotSpotter Raven devices through BLE service UUID fingerprinting:

| Service | UUID | Description |
|---------|------|-------------|
| Device Info | `0000180a-...` | Serial, model, firmware |
| GPS | `00003100-...` | Real-time coordinates |
| Power | `00003200-...` | Battery & solar status |
| Network | `00003300-...` | LTE/WiFi connectivity |
| Upload | `00003400-...` | Data transmission metrics |
| Error | `00003500-...` | Diagnostics & error logs |
| Health (legacy) | `00001809-...` | Firmware 1.1.x |
| Location (legacy) | `00001819-...` | Firmware 1.1.x |

Firmware version is estimated automatically from which service UUIDs are advertised.

---

## Acknowledgments

- **Will Greenberg** ([@wgreenberg](https://github.com/wgreenberg)) — BLE manufacturer company ID detection (`0x09C8` XUNTONG) sourced from his [flock-you](https://github.com/wgreenberg/flock-you) fork
- **[DeFlock](https://deflock.me)** ([FoggedLens/deflock](https://github.com/FoggedLens/deflock)) — crowdsourced ALPR location data and detection methodologies. Datasets included in `datasets/`
- **[GainSec](https://github.com/GainSec)** — Raven BLE service UUID dataset (`raven_configurations.json`) enabling detection of SoundThinking/ShotSpotter acoustic surveillance devices

---

## OUI-SPY Firmware Ecosystem

Flock-You is part of the OUI-SPY firmware family:

| Firmware | Description | Board |
|----------|-------------|-------|
| **[OUI-SPY Unified](https://github.com/colonelpanichacks/oui-spy-unified-blue)** | Multi-mode BLE + WiFi detector | ESP32-S3 / ESP32-C5 |
| **[OUI-SPY Detector](https://github.com/colonelpanichacks/ouispy-detector)** | Targeted BLE scanner with OUI filtering | ESP32-S3 |
| **[OUI-SPY Foxhunter](https://github.com/colonelpanichacks/ouispy-foxhunter)** | RSSI-based proximity tracker | ESP32-S3 |
| **[Flock You](https://github.com/colonelpanichacks/flock-you)** | Flock Safety / Raven surveillance detection (this project) | ESP32-DevKitC |
| **[Sky-Spy](https://github.com/colonelpanichacks/Sky-Spy)** | Drone Remote ID detection | ESP32-S3 / ESP32-C5 |
| **[Remote-ID-Spoofer](https://github.com/colonelpanichacks/Remote-ID-Spoofer)** | WiFi Remote ID spoofer & simulator with swarm mode | ESP32-S3 |
| **[OUI-SPY UniPwn](https://github.com/colonelpanichacks/Oui-Spy-UniPwn)** | Unitree robot exploitation system | ESP32-S3 |

---

## Author

**colonelpanichacks**

**Oui-Spy devices available at [colonelpanic.tech](https://colonelpanic.tech)**

---

## Disclaimer

This tool is intended for security research, privacy auditing, and educational purposes. Detecting the presence of surveillance hardware in public spaces is legal in most jurisdictions. Always comply with local laws regarding wireless scanning and signal interception. The authors are not responsible for misuse.
