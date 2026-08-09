# Toolchain: VS Code + pioarduino, chips, platformio.ini

Contents: why pioarduino · install · platformio.ini templates · chip selection · core 2.x vs 3.x · common build problems

## Why pioarduino (not PlatformIO IDE)

Espressif stopped publishing arduino-esp32 to PlatformIO after core **2.0.17**. Core 3.x (ESP-IDF 5.x based, current: 3.3.x on IDF 5.5) is available for VS Code only via the community fork **pioarduino**. The classic `platform = espressif32` still works but builds against the old 2.x core.

Consequences:

- New chips (C6, H2, P4, C5) and all current fixes require core 3.x → pioarduino.
- Core 2.x is fine for classic ESP32/S3 if a legacy library demands it — pin it explicitly (`platform = espressif32@6.12.0`).
- Never mix: one project = one platform line.

## Install

1. VS Code → Extensions → install **pioarduino IDE** ("Trust Publisher & Install"; accept the Core CLI install prompt). Restart VS Code.
2. The original PlatformIO IDE extension may stay installed but do not run both on the same ESP32 project.
3. Arduino IDE is acceptable for one-off sensor experiments only; it is not the project IDE.

## platformio.ini templates

```ini
; --- ESP32-S3 dev board, Arduino core 3.x ---
[env:esp32-s3]
platform = https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip
board = esp32-s3-devkitc-1
framework = arduino
monitor_speed = 115200
build_flags =
    -D CORE_DEBUG_LEVEL=3
    ; -D ARDUINO_USB_CDC_ON_BOOT=1   ; enable when using native USB Serial on S3
; lib_deps =
;     h2zero/NimBLE-Arduino@^2
;     mathertel/OneButton@^2

; --- Classic ESP32 ---
[env:esp32]
platform = https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip
board = esp32doit-devkit-v1
framework = arduino
monitor_speed = 115200

; --- ESP32-C3 (BLE-first, low power) ---
[env:esp32-c3]
platform = https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip
board = esp32-c3-devkitm-1
framework = arduino
monitor_speed = 115200

; --- Host-side unit tests for src/core/ (no hardware) ---
[env:native]
platform = native
build_flags = -std=c++17
test_build_src = false        ; tests compile core/ sources explicitly, not main.cpp
```

Pin the platform URL to a specific release tag (e.g. `.../download/55.03.32/platform-espressif32.zip`) once the project stabilizes — `stable` moves.

## Chip selection guide

| Need | Pick | Why |
|---|---|---|
| On-device ML / DSP (TFLM, vision, audio) | **ESP32-S3** | Xtensa LX7 + vector/SIMD instructions (ESP-NN), octal PSRAM options, native USB OTG, both ULP-FSM and ULP-RISC-V, 8 LEDC + 12 MCPWM channels |
| Battery BLE sensor, smallest BOM | **ESP32-C3** | RISC-V, BLE 5.0, deep sleep ~5 µA, cheap; only 6 LEDC channels, no classic BT |
| New BLE design, Thread/Zigbee roadmap | **ESP32-C6 / H2** | C6: Wi-Fi 6 + BLE 5.3 + 802.15.4; H2: BLE + 802.15.4 only (no Wi-Fi) |
| Legacy compatibility, dual-core Xtensa, classic BT | **ESP32** | Mature but highest power draw (~240 mA active Wi-Fi), deep sleep ~10 µA |
| Camera / display-heavy | **ESP32-S3** (octal PSRAM) or **P4** | P4: high-performance, no Wi-Fi/BT on-die |

For battery BLE products also compare: C3/H2 radios are materially more efficient than classic ESP32.

## Core 2.x → 3.x: what breaks (generate 3.x code only)

- **LEDC**: `ledcSetup()`/`ledcAttachPin()`/`ledcWrite(pin,...)` removed → `ledcAttach(pin, freq, resBits)`, `ledcWrite(pin, duty)` (pin-addressed; channel auto-allocated). Details: [peripherals.md](peripherals.md).
- **Timers**: `timerBegin(freq)` single-arg; `timerAlarm()` merges write+enable; many getters removed.
- **RMT**: API reworked (pin-based args, `rmtWriteAsync`, `rmtInit` returns bool).
- **UART**: default pins changed on some SoCs; `setPins()` callable any order.
- **ADC**: use `analogReadMilliVolts()` (calibrated) instead of raw `analogRead()` math.
- **I2S**: legacy driver removed → new `I2SClass` (ESP_I2S) API; old audio libs may need core 2.x or updates.
- Full list: Espressif "Migration from 2.x to 3.0" guide — check it whenever a compiler error names a removed function.

## Common build/flash problems

- **`framework-arduinoespressif32@3.x` fails with `platform = espressif32`** → expected; switch the platform line to the pioarduino URL.
- **S3 shows no serial output** → add `-D ARDUINO_USB_CDC_ON_BOOT=1` and use the native USB port, or monitor via the UART port.
- **Upload hangs / port busy** → close Arduino IDE and any other monitor; hold BOOT while resetting for manual download mode.
- **Brownout resets on BLE/Wi-Fi bursts** → supply/decoupling issue, not code; check before debugging firmware.
- **PSRAM not detected on S3** → set `board_build.arduino.memory_type = qio_opi` (or `qio_qspi` per module) in platformio.ini.
- **`managed_components/` and `dependencies.lock`** → gitignore them when using IDF-component mode.
- Keep every dependency in `lib_deps` with a pinned version — reproducible builds are part of the migration story.
