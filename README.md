# XeWe OS Laptop Cooling Pad — temperature-driven fans and ARGB on ESP32

Personal project (XeWe Labs) · 2026-03-21 → 2026-05-26 · Solo: Max Dokukin · Status: Completed (archived)

![Web UI: fan curve editor with live temperature marker](static/media/resources/web_ui_curve.webp)

## Overview

Firmware for a DIY laptop cooling pad, built on an early copy of [XeWe OS](https://github.com/xewe-labs/xewe-os)
(ESP32 modular firmware with a text command line). A contactless MLX90614 infrared sensor reads a temperature every
500 ms; once per second a user-defined temperature → fan-speed curve sets two 4-wire PWM fans, and two 12-LED WS2812B
strips fade from a "coldest" to a "hottest" colour across the same curve. The curve, colours, fans, strips and sensor
pins are stored in ESP32 NVS and can be changed from the serial CLI or from a small web page served by the device,
which draws the curve and a live operating point. The web page was prototyped in Flask in
[xewe-os-frontend-builder](https://github.com/xewe-labs/xewe-os-frontend-builder) and embedded as PROGMEM headers.
It was the first device built on XeWe OS; the XeWe OS modules it inherits (serial port, NVS, system, command parser,
WiFi, web interface) are documented in that project.

## Highlights

- Four device modules added to XeWe OS: `Fan`, `MLX90614`, `ARGB`, `TempController` (`src/Modules/`), 21 device CLI commands
- Tachometer RPM from falling-edge interrupts, ≥ 1 s windows, 3-sample median and EMA (α = 0.3), rounded to 10 RPM — `Fan.cpp`
- Piecewise-linear fan curve (default 22 °C → 0 %, 27 °C → 100 %, 32 °C → 100 %) persisted as one NVS string — `TempController.cpp`
- Frame-rate-independent colour glide (rate 8 /s, Δt capped at 0.1 s) replaced a fixed 1 s linear fade on 2026-05-26 — `ARGB.cpp`
- Web UI (canvas curve editor, colour pickers) polling `/api/state` every 1 s; same files as the Flask prototype, exported to headers
- Hotspot mode (added 2026-05-26): the device can run its own access point with a random 8-digit password — `Wifi.cpp`

## How it works

```
MLX90614 (I2C 0x5A, every 500 ms) → TempController (every 1,000 ms)
    ├─ curve lookup (°C → %) → ×255/100 → Fan.set_all → 25 kHz 8-bit PWM on GPIO 3 and 10
    └─ colour lerp cold→hot over the curve's range → ARGB.set_all_rgb → glide → WS2812B on GPIO 6 and 7
Fan tach (GPIO 0, 1) → ISR pulse count → RPM (median + EMA) → JSON / status
Browser ⇄ WebServer :80  (/api/state, /ui/config GET/POST, /cmd)     Serial 115,200 baud ⇄ CommandParser
```

- **Fan** (`src/Modules/Hardware/Fan/`) — adds fans with or without a tachometer (`$fan add`, `add_w_tach`, `set`, `set_all`, `remove`, `print_json`); PWM 25 kHz, 8-bit; first boot registers PWM 3 / tach 0 and PWM 10 / tach 1. RPM = pulses × 30,000 / Δt ms (two pulses per revolution), readings above 10,000 RPM discarded.
- **MLX90614** (`src/Modules/Sensors/Temperature/MLX90614/`) — reads object (RAM 0x07) and ambient (RAM 0x06) temperature over I2C (default SDA 4, SCL 5, address 0x5A), `raw × 0.02 − 273.15`; returns 255.0 °C as an error value when offline so the curve drives the fans to their top point. Commands: `read`, `scan`, `set_addr`, `set_pins`, `print_json`.
- **TempController** (`src/Modules/Software/TempController/`) — holds the sorted curve points (0–100 %), interpolates linearly between them, scales to 0–255 PWM and drives all fans; below the first point the LED strips switch off, above the last they show the hot colour. Commands: `add_point`, `remove_point`, `set_cold`, `set_hot`, `print_json`.
- **ARGB** (`src/Modules/Hardware/ARGB/`) — Adafruit NeoPixel strips (12 LEDs, GRB, 800 kHz), default pins 6 and 7; each update moves the current colour toward the target by `diff × 8 × Δt` until it is within 0.5. Temperature-driven colour changes skip NVS writes. Commands: `add`, `remove`, `set_state`, `set_rgb`, `print_json`.
- **WebInterface** (`src/Modules/Software/WebInterface/`) — serves `templates/index_html.h`, `static/script_js.h`, `static/styles_css.h` from flash; JSON routes `/api/state`, `/fan/data`, `/argb/data`, `/mlx90614/data`, `/ui/config` (GET, POST replaces the curve and colours); `/cmd?c=<command>` runs any CLI command.
- **Inherited XeWe OS core** — `SystemController` (module list, begin order, cooperative loop), `SerialPort`, `Nvs`, `System` (device name fixed to "XeWe Laptop Chiller" on first boot), `CommandParser`, `Wifi` (station or hotspot).

CLI syntax is XeWe OS's: `$<module> <command> <args>`, e.g.

```text
$tempcontroller add_point 40.5 50
$tempcontroller set_cold #00FFFF
$fan status
$mlx90614 status
$argb set_rgb 6 255 0 0
$help
```

## Results

| Reading | Value | Note |
|---|---|---|
| Fan speed feedback | 840 and 820 RPM | both fans at PWM 50/255, serial `$fan status`, 2026-03-22 screenshot |
| Sensor | 28.37 °C object, 28.77 °C ambient | `$mlx90614 status`, same screenshot |
| Live curve | 34.1 °C → 100 % fan | web UI with a 26 °C → 0 %, 34 °C → 100 % curve, 2026-03-22 screenshot |

These are bench readings from the screenshots in `static/media/resources/`; no cooling (laptop temperature) measurements
are recorded in the repository.

## Getting started

Hardware as wired in the code: an ESP32 board (the screenshots show an ESP32-C3 build; `build/release_matrix.csv` lists
c3, c6 and s3), two 4-wire PWM fans (PWM on GPIO 3 and 10, tach on GPIO 0 and 1), an MLX90614 on I2C (SDA 4, SCL 5), two
WS2812B strips of 12 LEDs on GPIO 6 and 7. Pins can be changed at runtime with the CLI.

```bash
git clone https://github.com/xewe-labs/xewe-os-laptop-cooling-pad
cd xewe-os-laptop-cooling-pad/build/scripts

# set up the build environment (arduino-cli, ESP32 core, libraries from build/libraries/required_libraries.txt)
./setup_build_enviroment_mac.sh      # or ./setup_build_enviroment_linux.sh

# build and flash: ./build.sh -c <c3|c6|s3> [-p <port>]
./build.sh -c c3 -p /dev/cu.usbmodem11143201
```

Then open a serial monitor at 115,200 baud. On first boot the device names itself "XeWe Laptop Chiller", registers the
two fans and asks whether to join an existing WiFi network or start a hotspot; the web page is then at the printed IP
address. The code depends on Adafruit NeoPixel and ArduinoJson besides the ESP32 Arduino core. Per-module debug output
is switched on in `src/Debug.h`.

## Documents

- [Web UI screenshot](static/media/resources/web_ui_curve.webp) and [serial status screenshot](static/media/resources/serial_status.webp)
- `doc/ADDING_A_MODULE.md`, `doc/MODULES.md`, `doc/PROJECT_STRUCTURE.md` — XeWe OS documents inherited from the template; they describe the Pins and Buttons modules, which this build replaced with the device modules above
- `static/firmware/releases/1.0.0/` — XeWe OS 1.0.0 images inherited from the template, not cooling-pad firmware
- Web UI prototype and export script: [xewe-os-frontend-builder](https://github.com/xewe-labs/xewe-os-frontend-builder) (`laptop-cooling-pad/`, `export/export.py`)
- Base firmware: [XeWe OS](https://github.com/xewe-labs/xewe-os) · project page [maxdokukin.com/projects/xewe-os](https://maxdokukin.com/projects/xewe-os)

## License

PolyForm Noncommercial 1.0.0 + No AI Use Addendum v1.0 — see [LICENSE.md](LICENSE.md) and [LICENSE-NO-AI.md](LICENSE-NO-AI.md).
