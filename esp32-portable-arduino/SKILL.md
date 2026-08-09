---
name: esp32-portable-arduino
description: "Design and write ESP32 firmware in C++ with the Arduino framework (PlatformIO/pioarduino in VS Code), architected in three layers (core / hal / app) so the firmware can later migrate to ESP-IDF for production with minimal rewrite. Use whenever the user asks to: create, structure, review, refactor or extend ESP32 firmware (any variant: ESP32, S2, S3, C3, C6, H2); build connected devices such as wearables, sensor nodes, sleep/lucid-dreaming devices, robotic arms, robots, motor controllers, BLE peripherals paired with Flutter/mobile apps; run on-device ML (TFLite Micro / ESP-DL); optimize ESP32 battery life (sleep modes, ULP); set up PlatformIO or pioarduino projects; or migrate an Arduino-framework project to ESP-IDF (including hybrid 'Arduino as ESP-IDF component' mode). Trigger keywords: ESP32, PlatformIO, pioarduino, Arduino framework, ESP-IDF migration, NimBLE, BLE firmware, robotic arm ESP32, stepper/servo control, deep sleep, ULP, TFLite Micro, portable firmware, hardware abstraction layer."
---

# ESP32 Portable Arduino

Write ESP32 firmware in C++ / Arduino framework that is fast to develop with AI assistance **and** cheap to migrate to ESP-IDF when the product matures. The entire skill rests on one principle:

> **Business logic is pure C++ and never touches Arduino APIs. All framework/hardware calls live behind a thin HAL. Migration = rewrite the HAL, keep the core.**

## When the user has NOT chosen a chip yet

Recommend by use case (details: [references/toolchain.md](references/toolchain.md)):

- **ESP32-S3** — default choice for new designs: AI vector instructions (TFLite Micro), native USB, both ULP types, most PWM channels.
- **ESP32-C3 / C6** — battery BLE-first devices; lowest power, RISC-V, fewer GPIO/peripherals.
- **Classic ESP32** — only when required by existing hardware; mature but highest power draw.

## Toolchain (current state — do not rely on stale memory)

- Use **VS Code + pioarduino IDE extension** for Arduino core 3.x. The original PlatformIO IDE extension is frozen at arduino-esp32 core 2.0.17 — Espressif does not publish core 3.x to PlatformIO. pioarduino is the community fork; same UX.
- `platformio.ini` must point at the pioarduino platform release (exact line in [references/toolchain.md](references/toolchain.md)).
- Arduino core **3.3.x runs on ESP-IDF 5.x** — many Arduino APIs changed vs 2.x (LEDC, RMT, timers, UART pins). Never generate 2.x-style `ledcSetup()`/`ledcAttachPin()` code; the old functions are gone. See the API-change table in [references/peripherals.md](references/peripherals.md).
- ESP-IDF APIs (`esp_pm.h`, `esp_sleep.h`, `esp_timer.h`, `esp_ota_ops.h`, NVS) are callable directly from Arduino code — use this deliberately for power management and OTA instead of Arduino wrappers.

## Mandatory project structure

Generate every new project in this shape (full template + file examples: [references/architecture.md](references/architecture.md)):

```
project/
├── platformio.ini          # pioarduino platform, envs (device + native test)
├── src/
│   ├── core/               # PURE C++: zero #include <Arduino.h>, zero driver calls
│   │   ├── interfaces/     # ISensors, IActuators, ILink, IPower, IClock, ILog, IStorage
│   │   └── ...             # state machines, filters, protocols, algorithms, ML pipeline
│   ├── hal/                # ONLY place Arduino/ESP-IDF/driver code appears
│   ├── app/                # composition root: builds HAL, injects into core, top-level tasks
│   └── main.cpp            # thin: setup()/loop() delegate to app/
├── test/                   # PlatformIO native-env unit tests run core/ on the PC
└── boards/pins.h           # single header with all pin assignments
```

## Non-negotiable coding rules

Enforce these in all generated code; treat violations as defects:

1. **`src/core/` contains no Arduino includes or calls** — no `Arduino.h`, `millis()`, `delay()`, `Serial`, `String`, `digitalWrite`, no library headers. Time, logging, storage, sensors, links all arrive through injected interfaces.
2. **All hardware access goes through `hal/`** implementations of `core/interfaces/`. One class per peripheral role, not per sensor model (e.g. `IImu`, not `Mpu6050`).
3. **No `delay()` in logic.** Use non-blocking state machines driven by `IClock::millis()`. A device must serve BLE, sampling and actuators concurrently; `delay()` also has no direct ESP-IDF equivalent.
4. **Never use Arduino `String` in `core/`** — `std::string` or fixed buffers. `String` does not exist in ESP-IDF and fragments heap.
5. **All pins in `boards/pins.h`** — no raw GPIO numbers scattered in code.
6. **Logging through `ILog`** — not `Serial.println` sprinkled in logic.
7. **BLE/wire protocol structs are plain C++ (POD, fixed-size) defined in `core/`** — the mobile app contract must survive the ESP-IDF migration unchanged.
8. **Compile `core/` on the host** — every project gets a `[env:native]` PlatformIO target + unit tests so logic is testable on PC without hardware. See [references/testing.md](references/testing.md).
9. Prefer `constexpr`, fixed-size buffers, and FreeRTOS primitives already available in Arduino (`xTaskCreate`, queues) — these map 1:1 to ESP-IDF, unlike Arduino convenience wrappers.
10. When generating code with AI, state the constraint explicitly in the prompt: *"pure C++ against these interfaces, no Arduino API calls"* for `core/`; *"implement interface X using library Y"* for `hal/`.

## Domain guides — load as needed

Read the matching reference before writing code in that domain; each contains current library choices, working code patterns, and migration notes:

| Domain | Reference | Covers |
|---|---|---|
| Architecture | [references/architecture.md](references/architecture.md) | Full layer template, interface catalog, pins.h, composition root, anti-patterns |
| Toolchain | [references/toolchain.md](references/toolchain.md) | pioarduino install, platformio.ini for ESP32/S3/C3, chip selection, build issues |
| Peripherals | [references/peripherals.md](references/peripherals.md) | GPIO, ADC, I2C/SPI, UART, LEDC/PWM, timers, interrupts — **core 3.x API changes** |
| BLE | [references/ble.md](references/ble.md) | NimBLE-Arduino server, characteristics/notifications, connection params & power, BLE OTA, mobile-app contract |
| Motors/robots | [references/motors.md](references/motors.md) | Servo (ESP32Servo/MCPWM), steppers (FastAccelStepper), DC+driver ICs, robot-arm patterns, safety |
| Power | [references/power.md](references/power.md) | Sleep modes, ULP coprocessor, RTC memory, BLE+light-sleep, battery budgeting |
| On-device ML | [references/ml.md](references/ml.md) | TFLite Micro + ESP-NN on S3, INT8 quantization, alignment pitfalls, ESP-DL, Edge Impulse |
| Migration | [references/migration.md](references/migration.md) | Arduino→ESP-IDF playbook: hybrid mode, API mapping table, step plan, verification |
| Testing | [references/testing.md](references/testing.md) | native-env host tests for core/, Unity, on-target tests, CI sketch |

## Standard workflow for a new device

1. Identify device class (wearable/sensor node, actuator/robot, gateway) and chip — see [references/toolchain.md](references/toolchain.md).
2. Scaffold the mandatory structure; write `platformio.ini` with device env + `[env:native]`.
3. Define `core/interfaces/` from the device's responsibilities (sense / decide / act / communicate / store / sleep) — before any driver code.
4. Implement `core/` logic + host unit tests; validate algorithms on PC with recorded data.
5. Implement `hal/` per peripheral using the domain guide; bring up one peripheral at a time.
6. Wire up in `app/` + thin `main.cpp`; integration-test on hardware.
7. Apply the power guide early if battery-powered — retrofitting sleep is painful.
8. When heading to production, follow [references/migration.md](references/migration.md) — expected rewrite scope: `hal/` + `main.cpp` only; `core/`, tests and the mobile app contract carry over unchanged.
