# Architecture: the three-layer template

Contents: layer responsibilities · interface catalog · composition root · pins.h · anti-patterns

## Layers

| Layer | Contains | May include | Must NOT include |
|---|---|---|---|
| `core/` | algorithms, state machines, protocol structs, filters, ML pipeline, business rules | `<cstdint> <cstring> <cmath> <string> <vector> <array> <functional>` | `Arduino.h`, any driver/library header, `millis delay Serial String digitalWrite`, `esp_*` |
| `hal/` | implementations of `core/interfaces/` using Arduino libs, ESP-IDF calls, sensor drivers | everything | business logic decisions |
| `app/` | composition root: constructs HAL objects, injects into core, owns FreeRTOS tasks | Arduino/ESP-IDF setup | algorithms |

Rule of thumb: if a line of code could run unchanged in a PC unit test, it belongs in `core/`. If it names a pin, a driver, or a framework API, it belongs in `hal/`.

## Interface catalog (core/interfaces/)

Define only what the device needs; typical set:

```cpp
// IClock.h — time is an injected dependency; critical for testability and migration
class IClock {
public:
    virtual uint32_t millis() = 0;
    virtual uint32_t micros() = 0;
    virtual void delayMs(uint32_t) = 0;    // only inside hal/app tasks, never in logic
};

// ISensors.h — role-based, not model-based (IImu, not "Mpu6050")
class IImu     { public: virtual bool readAccel(float& x, float& y, float& z) = 0; };
class IAnalogIn{ public: virtual bool readMilliVolts(uint8_t channel, uint32_t& mv) = 0; };
class IPulseOx { public: virtual bool readRaw(int32_t& red, int32_t& ir) = 0; };

// IActuators.h
class IPwmOut  { public: virtual void setDuty01(uint8_t channel, float duty) = 0; };  // LED, vibration, DC motor
class IServoBus{ public: virtual void setAngleDeg(uint8_t joint, float deg) = 0; };

// ILink.h — uplink/downlink to the companion app; protocol bytes defined in core/
class ILink {
public:
    virtual bool send(uint8_t msgType, const uint8_t* payload, size_t len) = 0; // notify
    using RxHandler = void(*)(uint8_t msgType, const uint8_t* payload, size_t len);
    virtual void setRxHandler(RxHandler) = 0;
    virtual bool isConnected() = 0;
};

// IPower.h — design for it from day one even if MVP ignores it
class IPower {
public:
    virtual void lightSleepMs(uint32_t) = 0;
    virtual void deepSleepSec(uint32_t) = 0;   // NOTE: deep sleep = reboot; state via RTC/NVS
};

// IStorage.h — settings/calibration; NVS-backed in hal
class IStorage {
public:
    virtual bool getU32(const char* key, uint32_t& out) = 0;
    virtual bool putU32(const char* key, uint32_t v) = 0;
    virtual bool getBlob(const char* key, uint8_t* buf, size_t& len) = 0;
    virtual bool putBlob(const char* key, const uint8_t* buf, size_t len) = 0;
};

// ILog.h
class ILog { public: virtual void log(int level, const char* tag, const char* fmt, va_list) = 0; };
```

## Core code example (portable forever)

```cpp
// core/CueController.cpp — logic decides; hardware only executes
CueAction CueController::update(const Sample& s, IClock& clock) {
    window_.push(s);
    if (window_.full() && detector_.isRem(window_)) {       // pure math on buffers
        if (clock.millis() - lastCue_ > cfg_.minIntervalMs) {
            lastCue_ = clock.millis();
            return CueAction{ cfg_.pattern, cfg_.intensity };  // value object, not a pin call
        }
    }
    return CueAction::none();
}
```

```cpp
// app/DeviceApp.cpp — translation point between decision and hardware
void DeviceApp::tick() {
    Sample s = sensors_->read();                       // hal
    CueAction a = cueCtrl_.update(s, *clock_);         // core decision
    if (a.active()) cues_->play(a.pattern, a.intensity); // hal executes
}
```

## Composition root + thin main.cpp

```cpp
// main.cpp — the only Arduino-shaped file
#include <Arduino.h>
#include "hal/HalFactory.h"
#include "app/DeviceApp.h"

static DeviceApp* app = nullptr;

void setup() {
    HalModules hal = HalFactory::create(PINS);   // all drivers constructed here
    app = new DeviceApp(hal);
    app->begin();                                 // spawns FreeRTOS tasks internally
}

void loop() {
    app->tick();          // or: vTaskDelete(NULL) if app runs fully in its own tasks
}
```

## boards/pins.h

```cpp
#pragma once
#include <cstdint>
struct PinMap {
    uint8_t i2cSda, i2cScl;
    uint8_t irLedLeft, irPhotoLeft, irLedRight, irPhotoRight;
    uint8_t vibMotor, cueLedR, cueLedG, cueLedB;
    uint8_t battSense;
};
inline constexpr PinMap PINS{ /* sda=*/8, /*scl=*/9, /*...=*/0 };
```

## FreeRTOS in Arduino (maps 1:1 to ESP-IDF)

- `xTaskCreatePinnedToCore(fn, name, stackWords, arg, prio, &handle, coreId)` — use it; do not rely on `loop()` for everything. Core 0 typically runs wireless stacks; pin real-time sampling to core 1.
- Inter-task data: FreeRTOS queues / ring buffers, never naked shared globals. Arduino-framework ISRs: mark callbacks `IRAM_ATTR`.
- `setup()`/`loop()` themselves run in a task (`loopTask`, prio 1, core 1 by default).

## Anti-patterns (reject in review)

- `String` concatenation in periodic code (heap churn) → fixed `char[]` + `snprintf`.
- `Serial.printf` inside `core/` or ISRs.
- Business rules inside BLE callbacks → callback only copies bytes into a queue; `core/` consumes.
- `analogRead()` sprinkled through logic files → behind `IAnalogIn`.
- Global `Wire`/`SPI` instances touched from multiple modules → owned by one hal class, guarded if shared.
- Deep inheritance trees / dynamic allocation churn in `core/` → value types, arena/static buffers (also what TFLite Micro wants).
