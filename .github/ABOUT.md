# About TallyCCU Pro

TallyCCU Pro is an open-source **Camera Control Unit (CCU)** system for [Blackmagic Design](https://www.blackmagicdesign.com/) cameras. It runs on an Arduino Mega 2560 and uses the Blackmagic 3G-SDI Shield to send camera control commands over SDI, while also integrating with [vMix](https://www.vmix.com/) for real-time tally light signals. A built-in web server lets operators control cameras from any browser, and a TCP API allows integration with [Bitfocus Companion](https://bitfocus.io/companion) (e.g. for Stream Deck control).

Current firmware version: **3.7.1**

---

## Repository Structure

```
TallyCCUPro/
├── Arduino/           # All Arduino firmware source files (.ino, .cpp, .h)
├── sdcard/            # Web UI files served from the SD card
│   ├── index.html     # Main camera control interface
│   ├── tally.html     # Tally map configuration page
│   ├── sdcard.html    # SD card file management page
│   └── safemode.html  # Safe Mode diagnostics page
├── tools/             # Desktop configuration utilities
│   ├── TallyCCU_Configurator.py   # Python/Tkinter serial configurator
│   └── TallyCCU_Configurator.exe  # Pre-built Windows executable
├── Images/            # Photos and diagrams used in README
├── .github/           # GitHub configuration and documentation
└── README.md          # Full setup and usage guide
```

---

## Hardware Stack

The firmware targets this exact shield stack:

```
  3  Blackmagic 3G-SDI Shield   ← Top
  2  Arduino Ethernet Shield 2  ← Middle (W5500, includes SD card slot)
  1  Arduino Mega 2560           ← Bottom
```

| Component | Model | Purpose |
|-----------|-------|---------|
| Microcontroller | Arduino Mega 2560 | Main processor (AVR ATmega2560) |
| Network + SD | Arduino Ethernet Shield 2 (W5500) | Ethernet & SD card storage |
| SDI Interface | Blackmagic 3G-SDI Shield for Arduino | Camera control over SDI |
| Power | 9–12 V DC supply | Required for SDI shield |

---

## Key Technologies

| Technology | Where Used |
|-----------|-----------|
| **C++ / Arduino (AVR)** | All firmware source code in `Arduino/` |
| **Ethernet (W5500)** | TCP/IP networking, web server, vMix connection |
| **SdFat library** | SD card file I/O for web assets and presets |
| **BMDSDIControl library** | Blackmagic I²C CCU commands over SDI |
| **AVR Watchdog Timer** | System reliability and boot-loop detection |
| **EEPROM** | Persistent storage for IP config and settings |
| **HTTP / SSE** | Embedded web server + Server-Sent Events for real-time UI sync |
| **Custom TCP protocol** | Port 8098 — bidirectional sync with Bitfocus Companion |
| **Python + Tkinter** | Desktop serial configurator (`tools/`) |
| **HTML / CSS / JavaScript** | Web UI files stored on SD card (`sdcard/`) |

---

## Firmware Modules (`Arduino/`)

| File | Class / Purpose |
|------|-----------------|
| `TallyCCUPro.ino` | Entry point — `setup()` / `loop()`, serial command handler |
| `Configuration.h` | Central constants: ports, EEPROM addresses, timing, pin assignments |
| `SafeMode.cpp/.h` | Boot-loop detection; enters safe mode after 3 watchdog resets |
| `Storage.cpp/.h` | EEPROM abstraction (`StorageManager`) for all persistent settings |
| `Network.cpp/.h` | Ethernet init, IP/gateway/subnet management (`NetworkManager`) |
| `VmixConnector.cpp/.h` | TCP client that connects to vMix on port 8099 and reads tally data |
| `TallyManager.cpp/.h` | Maps vMix inputs to camera IDs; drives tally LED brightness |
| `CCUControl.cpp/.h` | Sends CCU commands to cameras via BMDSDIControl I²C API |
| `bmd_params.cpp/.h` | Parameter table mapping string keys to BMD SDI command codes |
| `webserver.cpp/.h` | Embedded HTTP server (port 80) — serves HTML from SD card, handles API calls, manages SSE connection |
| `CCUBroadcast.cpp/.h` | TCP server (port 8098) — broadcasts CCU changes to Companion clients |
| `SdUtils.cpp/.h` | SD card helpers: begin, file listing, read/write |

---

## How the System Works

### Boot Sequence

1. **SafeMode::begin()** is called first. It reads EEPROM to check whether three or more consecutive watchdog resets occurred, enabling safe mode if so.
2. In **normal mode**, the system initialises EEPROM → SD → Ethernet → Tally → CCU (SDI Shield) → vMix → Web Server → CCU Broadcast, then calls `SafeMode::bootComplete()` to reset the crash counter.
3. In **safe mode**, the SDI Shield is skipped entirely. The web server still runs (serving `safemode.html`), and vMix tally still works, so the operator can diagnose and recover remotely.

### Main Loop (priority order)

| Priority | Task | Interval |
|----------|------|----------|
| 1 | vMix tally data (`VmixConnector::processData`) | Every cycle |
| 2 | Web server requests (`WebServer::processRequests`) | 10 ms |
| 3 | CCU Broadcast / Companion sync (`CCUBroadcast::process`) | 20 ms |
| 4 | Serial command processing | 50 ms |
| 5 | Ethernet maintenance (`Ethernet.maintain`) | 100 ms |

The watchdog timer is reset at the top of each loop iteration to detect hangs.

### Tally System

`VmixConnector` opens a TCP connection to vMix on port 8099 and subscribes to tally events. When vMix reports a program/preview state change, `TallyManager` looks up which camera maps to that vMix input and sets the appropriate tally LEDs (red = program, green = preview) at the configured brightness.

### CCU Control

`CCUControl` wraps the `BMD_SDICameraControl_I2C` library. Camera parameters (iris, focus, zoom, white balance, colour wheels, audio, display settings, etc.) are defined in a lookup table in `bmd_params`. The web interface or Companion sends a parameter key/value pair; `CCUControl::applyParameterByKey` resolves the key through a small LRU cache to avoid repeated O(n) lookups, then sends the appropriate I²C command to the SDI shield.

### Web Interface

The web server serves four HTML files directly from the SD card. JavaScript in those pages communicates with the Arduino via:
- **GET/POST requests** — apply camera parameters, save/load presets, update configuration
- **Server-Sent Events (SSE)** — the server pushes real-time parameter changes to the browser so the UI stays in sync with Companion

### Companion Integration (TCP API — port 8098)

`CCUBroadcast` listens for TCP clients. Once a client sends `SUBSCRIBE`, it receives text messages for every parameter change (`CCU <cameraId> <paramKey> <value>`), preset loads, and preset saves. Companion can also send `PING` to keep the connection alive.

### Preset System

Each camera has up to 5 presets stored as JSON files on the SD card. Presets capture the full camera state (or a selected subset of parameter groups). They can be saved, loaded, renamed, and deleted from the web UI or via the TCP API.

---

## Configuration & Setup Tools

### Serial Configurator (`tools/`)

A Python (Tkinter) GUI application that connects to the Arduino over USB serial at 115 200 baud. It provides point-and-click fields to set local IP, subnet, gateway, and vMix IP, and forwards them as text commands (`ip X.X.X.X`, `vmixip X.X.X.X`, etc.). A pre-built Windows `.exe` is included so no Python installation is needed.

### Serial Commands (via Arduino IDE monitor or configurator)

| Command | Action |
|---------|--------|
| `status` | Print full system status |
| `ip X.X.X.X` | Change device IP |
| `gateway X.X.X.X` | Change gateway |
| `subnet X.X.X.X` | Change subnet mask |
| `vmixip X.X.X.X` | Change vMix IP |
| `tally` | Print current tally map |
| `ram` | Show free RAM |
| `reset` | Restart Arduino |
| `safemode` | Force safe mode on next boot |
| `normalboot` | Exit safe mode and restart |

---

## Safe Mode (v3.7+)

If the Arduino watchdog fires before `SafeMode::bootComplete()` is called (e.g. because the SDI shield hangs waiting for a valid SDI signal), the EEPROM reset counter increments. After **3 consecutive failures**, the device boots into safe mode: the SDI shield is skipped, tally still works, and the web interface shows a diagnostics page. The operator can force a normal boot from the web UI or via `normalboot` over serial once the hardware issue is resolved.

---

## Interfaces at a Glance

| Interface | Port / Method | Use Case |
|-----------|--------------|----------|
| Web UI | HTTP port 80 | Browser-based camera control |
| vMix tally | TCP port 8099 (client) | Receives tally state from vMix |
| Companion / TCP API | TCP port 8098 (server) | Bitfocus Companion / Stream Deck |
| Serial | 115 200 baud | Initial setup, diagnostics |
