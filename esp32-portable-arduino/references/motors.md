# Motors & robots: servos, steppers, DC drives, robot arms

Contents: servo · stepper · DC motor · robot-arm architecture · safety · migration notes

All motor code lives in `hal/`; trajectories, kinematics and sequencing live in `core/` as pure math. This split lets you test an arm's motion planner on a PC before touching hardware.

## Servos (robot arms, grippers, pan-tilt)

Use **ESP32Servo** (madhephaestus, core 3.x compatible): Arduino `Servo` semantics, LEDC-based, plus **MCPWM hardware on ESP32-S3** — 20 channels total (8 LEDC + 12 MCPWM) with automatic allocation and fallback. Prefer S3 for multi-joint arms.

```cpp
#include <ESP32Servo.h>
class ServoBus : public IServoBus {
    Servo servos_[6];
public:
    void begin(const uint8_t* pins, uint8_t n) {
        ESP32PWM::allocateTimer(0);
        for (uint8_t i = 0; i < n; i++) {
            servos_[i].setPeriodHertz(50);              // analog servo standard
            servos_[i].attach(pins[i], 500, 2400);      // µs pulse range — calibrate per servo!
        }
    }
    void setAngleDeg(uint8_t j, float deg) override {
        servos_[j].write(constrain(deg, 0.0f, 180.0f)); // rate-limit in core/, not here
    }
};
```

- Power: servos brown out an ESP32 instantly — separate 5–6 V rail with common ground; never power servos from the dev board's 5 V pin.
- Calibrate µs endpoints per servo; store in `IStorage`.
- Smooth motion = trajectory in `core/` (e.g. linear/cosine interpolation at 50 Hz) streaming angles into `IServoBus`; hal stays dumb.

## Steppers (precise axes, CNC-ish motion)

Use **FastAccelStepper** (gin66) — hardware-timed step generation via RMT (and MCPWM on supported chips), so steps stay jitter-free while the CPU handles BLE/other tasks. Avoid `AccelStepper` for ESP32: it bit-bangs steps from the CPU and jitters under load.

```cpp
#include <FastAccelStepper.h>
FastAccelStepperEngine eng; FastAccelStepper* s = nullptr;
eng.init();
s = eng.stepperConnectToPin(STEP_PIN);
s->setDirectionPin(DIR_PIN);  s->setEnablePin(EN_PIN);
s->setAutoEnable(true);
s->setSpeedInHz(2000);        // steps/s — respect driver+mechanics
s->setAcceleration(4000);     // steps/s²
s->moveTo(target);            // non-blocking, hardware-timed
```

- Driver ICs: A4988/DRV8825 for small NEMA14/17; **TMC2209/TMC2208** (UART-configurable, silent StealthChop, StallGuard sensorless homing) for robots used near people.
- Always implement homing (endstop or StallGuard) before trusting absolute positions; after reset the true position is unknown.
- Closed-loop option: magnetic encoders (AS5600 on I2C) per joint — read in hal, fuse in `core/`.

## DC motors (wheels, pumps, fans)

- H-bridge drivers: DRV8833/TB6612 (small), BTS7960 (large). Two LEDC channels per motor (IN1/IN2) or PWM + DIR.
- 10–20 kHz PWM (above audible); ramp duty in `core/` to limit inrush.
- Encoders: quadrature via ESP32 **PCNT** hardware peripheral (ESP-IDF `driver/pulse_cnt.h`, callable from Arduino core 3.x) — not GPIO polling.
- PID lives in `core/` (`setpoint → measurement → duty`), ticked from a fixed-rate task (e.g. 100 Hz); hal provides `IEncoder::ticks()` and `IPwmOut::setDuty01()`.

## Robot-arm reference architecture

```
core/   Kinematics (FK/IK), trajectory generator, joint limits, sequencer/FSM,
        safety monitor (watchdog: no heartbeat from app → hal freeze)
hal/    ServoBus / StepperAxis / EncoderIn / GripperOut / EStopIn
app/    50 Hz motion task (trajectory→servos), 100 Hz PID task, BLE task
```

- Heartbeat pattern: `core/` requires a periodic "I'm alive" from the command source; timeout ⇒ controlled stop. Essential for arms and mobile robots.
- Joint limits enforced in **two** places: planner (`core/`) and actuator clamp (`hal/`) — defense in depth.

## Safety checklist (non-optional for robots)

- E-stop input on a real interrupt pin (`attachInterruptArg`, `IRAM_ATTR`) cutting enables, implemented in hal, asserted in tests.
- Enable pins default to disabled on boot before any motion command.
- Software joint limits + current/thermal monitoring where drivers support it (TMC UART).
- Failsafe on BLE disconnect: hold or controlled stop — pick per device, implement in `core/` FSM, test it.
- USB-vs-battery power path review: dev boards are for bench only.

## Migration notes (Arduino → ESP-IDF)

| Arduino-era piece | ESP-IDF target |
|---|---|
| ESP32Servo (LEDC/MCPWM) | `driver/ledc.h` / `driver/mcpwm_prelude.h` servo example — direct equivalent |
| FastAccelStepper | runs on RMT/MCPWM already; port thin wrapper or re-implement on `driver/rmt_tx.h` |
| PCNT encoder | identical API (`pulse_cnt.h`) — already IDF |
| `constrain/map` macros | replace with `<algorithm>` / own constexpr in core (do this from day one) |
