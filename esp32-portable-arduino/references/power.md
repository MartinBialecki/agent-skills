# Power: sleep modes, ULP, battery-first design

Contents: mode selection · light sleep + tickless idle · deep sleep rules · ULP · BLE+sleep · budgeting · migration notes

Design rule: decide the power architecture **before** writing application code. Retrofitting sleep into a `delay()`-based sketch means rewriting it — another reason for the non-blocking state-machine rule.

## Mode selection

| Mode | Typical current | State kept | Wake latency | Use when |
|---|---|---|---|---|
| Active (Wi-Fi TX peak) | ~240 mA | all | — | bursts only |
| Modem sleep | ~15–20 mA | all | <1 ms | Wi-Fi must stay associated |
| **Light sleep** | ~0.8–1 mA (CPU gated; ~1–5 mA with BLE maintained) | RAM + peripherals | ~1–3 ms | always-on device that must react fast / keep BLE |
| **Deep sleep** | ~10 µA (C3 ~5 µA) | RTC memory only | full reboot (~200–500 ms) | minutes-to-hours between events |
| ULP active, main cores off | ~1.5–5 µA + sensor cost | RTC memory | — | periodic sensing while asleep |

## Light sleep + tickless idle (the big win for wearables)

Without it, the CPU burns ~40–100 mA baseline between BLE events regardless of radio duty cycle. With it, idle drops to ~1–5 mA.

```cpp
#include "esp_pm.h"
#include "esp_sleep.h"

void PowerManager::enableRuntimeSleep() {
    // sdkconfig: CONFIG_PM_ENABLE=y, CONFIG_FREERTOS_USE_TICKLESS_IDLE=y
    // (settable via build_flags or sdkconfig.defaults in pioarduino projects)
    esp_pm_config_t cfg = {
        .max_freq_mhz = 240,
        .min_freq_mhz = 40,
        .light_sleep_enable = true,
    };
    esp_pm_configure(&cfg);   // RTOS enters light sleep automatically in idle
}
```

Gotchas: UART output gaps during sleep (use `esp_sleep` UART handling or external UART bridge); Wi-Fi+BLE light-sleep coexistence is limited — prefer BLE-only designs for battery; GPIO wake sources must be configured explicitly.

## Deep sleep rules

- Wake = **reboot** from `app_main()`/`setup()`, not continuation. Persist counters/state in `RTC_DATA_ATTR` variables or NVS before sleeping.
- Configure wake sources: `esp_sleep_enable_timer_wakeup(us)`, `esp_sleep_enable_ext0_wakeup(pin, level)`, `ext1` (multi-GPIO), touchpad, ULP.
- Shut radios down before sleep: `nimble_port_stop()/deinit()` or `esp_bt_controller_disable()/deinit()`; connections do not survive any sleep mode.
- USB-serial (native USB on S3/C3) dies in deep sleep — debug via hardware UART pins when developing sleep code.
- Power sensors from a GPIO (ideally RTC-domain GPIO) and switch them off before sleeping.

## ULP coprocessor (wake only on interesting data)

- Chips: classic ESP32/S2/S3 have ULP-FSM (assembly); S2/S3/C6 also have **ULP-RISC-V programmable in C** — prefer RISC-V chips for new designs.
- The ULP accesses RTC GPIOs, ADC, and I2C (S2/S3) and can wake the main CPU on thresholds — e.g. IMU motion interrupt pattern, analog eye-signal threshold — while the main cores stay in deep/light sleep.
- Shared state via RTC slow memory (`RTC_SLOW_ATTR`); one sequential ULP program — implement a mini state machine inside it.
- **Not exposed by the Arduino core**: write ULP via ESP-IDF build (PlatformIO `framework = espidf` component, or plan it for the migration phase). Architect `core/` decisions so the ULP only detects *wake conditions*; classification still runs on the main CPU.

## BLE + sleep (typical wearable budget)

1. Light sleep + tickless idle (above) — largest single factor.
2. BLE: advertising ≥1000 ms; connected interval 300–1000 ms + slave latency 4–10; notifications instead of polling; TX power to minimum reliable; no scanning on the peripheral.
3. Wi-Fi fully off: `esp_wifi_stop(); esp_wifi_deinit();`
4. Duty-cycle sampling: sensor front-end on a timer task; batch-process; queue only events to BLE.
5. Cut peripherals: brownout where safe, unused pull-ups, LEDs, dev-board regulators (dev boards' LDO+USB-UART draw mA — measure a bare module for real numbers).

Rough sanity math: 500 mAh cell, average 2 mA ⇒ ~10 days; average 50 µA ⇒ ~1 year. Compute the average from the duty cycle *before* choosing the battery.

## Migration notes

All calls in this file (`esp_pm`, `esp_sleep`, `nvs`, ULP) are already ESP-IDF — this is the one domain where the "Arduino layer" adds nothing, so write it against IDF headers from day one inside `hal/`. Migration then touches only task wiring, not power logic.
