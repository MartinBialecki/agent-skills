# Peripherals on Arduino core 3.x

Contents: GPIO · ADC · PWM/LEDC · timers · interrupts · I2C · SPI · UART · API change table

All snippets assume Arduino core **3.x** (ESP-IDF 5.x underneath). Wrap every snippet in a `hal/` class implementing a `core/interfaces/` interface — never call these from `core/`.

## GPIO

```cpp
pinMode(pin, INPUT_PULLUP);            // INPUT, OUTPUT, INPUT_PULLDOWN, OPEN_DRAIN
digitalWrite(pin, HIGH);
int v = digitalRead(pin);
```
ESP-IDF equivalent for migration: `gpio_set_direction / gpio_set_level / gpio_get_level` via `driver/gpio.h`.

## ADC

```cpp
analogSetAttenuation(ADC_11db);        // full ~0–3.1 V range
uint32_t mv = analogReadMilliVolts(pin);  // calibrated — always prefer over raw analogRead()
```
- On S3/C3, ADC2 is unusable while Wi-Fi is active; route analog sensors to ADC1 pins for Wi-Fi devices (BLE-only is less restrictive but ADC1 remains the safe default).
- Nonlinearity at rails is real; design dividers to keep the signal in the mid-range, calibrate per unit, store calibration via `IStorage`.

## PWM / LEDC (core 3.x API — old calls are gone)

```cpp
// attach once
ledcAttach(pin, freqHz, resolutionBits);   // channel auto-assigned; returns bool
// write duty by PIN (not channel)
ledcWrite(pin, duty);                      // duty: 0 .. (1<<resolutionBits)-1
ledcChangeFrequency(pin, freqHz, resolutionBits);
ledcDetach(pin);
```
- Servo-friendly: 50 Hz, 10–14 bit. Vibration motor / LED dimming: 1–20 kHz, 8–10 bit (above audible range to avoid whine; LED flicker >1 kHz).
- On S3, ESP32Servo can additionally use MCPWM for jitter-free servo timing — see [motors.md](motors.md).

## Hardware timers (core 3.x)

```cpp
hw_timer_t* t = timerBegin(1000000);       // single arg: frequency in Hz
timerAttachInterrupt(t, &onTimer);         // ISR must be IRAM_ATTR
timerAlarm(t, 1000, true, 0);              // ticks, autoreload, reload count — merged write+enable
```
Use hardware timers only for truly periodic low-latency jobs (sampling clock). For ordinary scheduling use FreeRTOS software timers or task loops with `vTaskDelayUntil` semantics.

## Interrupts

```cpp
attachInterruptArg(digitalPinToInterrupt(pin), isr, thisPtr, FALLING);   // RISING/CHANGE/ONLOW/ONHIGH
void IRAM_ATTR isr(void* arg) { /* copy timestamp, set flag/post queue — nothing else */ }
```
ISR rules: `IRAM_ATTR`, no `Serial`, no heap, no I2C/SPI; hand off to a task via queue or `xTaskNotifyFromISR`.

## I2C

```cpp
Wire.setPins(PINS.i2cSda, PINS.i2cScl);    // call before begin on non-default pins
Wire.begin();
Wire.setClock(400000);                     // 400 kHz fast mode for IMUs/displays
```
- One `Wire` instance per bus, owned by a single hal class; wrap transactions in a mutex if multiple FreeRTOS tasks share the bus.
- Sensor drivers: prefer maintained Adafruit/SparkFun/bosch libs pinned in `lib_deps`. When no Arduino driver exists, port the register map into the hal class — do not leak a datasheet driver into `core/`.

## SPI

```cpp
SPIClass spi(FSPI);                        // S3/C3: FSPI; classic: VSPI/HSPI
spi.begin(sck, miso, mosi, cs);
spi.beginTransaction(SPISettings(8000000, MSBFIRST, SPI_MODE0));
```

## UART

```cpp
HardwareSerial ser(1);
ser.setPins(rxPin, txPin);                 // any order in core 3.x; -1 keeps pin unchanged
ser.begin(115200);
```
Core 3.x changed some default UART pins (e.g. classic ESP32 UART1 RX/TX = GPIO26/27, UART2 = GPIO4/25) — always `setPins()` explicitly from `pins.h` and never depend on defaults.

## Direct ESP-IDF calls worth making from hal/ today

Already linked and stable across the migration — prefer them over Arduino wrappers where they exist:

| Need | Header / call |
|---|---|
| High-res time | `esp_timer.h` → `esp_timer_get_time()` (µs, 64-bit) |
| Short µs delay | `esp_rom_delay_us()` |
| Sleep modes | `esp_sleep.h` — see [power.md](power.md) |
| Power management | `esp_pm.h` → `esp_pm_configure()` |
| NVS storage | `nvs_flash.h` / wrap Arduino `Preferences` (it is NVS) |
| OTA | `esp_ota_ops.h` |
| Chip/restart info | `esp_system.h`, `esp_chip_info()` |

## 2.x → 3.x quick mapping (for fixing old code the AI/user pastes in)

| Removed (2.x) | Use instead (3.x) |
|---|---|
| `ledcSetup(ch,f,res)` + `ledcAttachPin(pin,ch)` | `ledcAttach(pin,f,res)` |
| `ledcWrite(ch,duty)` | `ledcWrite(pin,duty)` |
| `timerBegin(n,div,up)` | `timerBegin(freqHz)` |
| `timerAlarmWrite` + `timerAlarmEnable` | `timerAlarm(t,ticks,reload,count)` |
| `rmtWrite(...)` non-blocking | `rmtWrite` is now blocking; `rmtWriteAsync` for async |
| legacy `driver/i2s.h` Arduino I2S | new `I2SClass` API |
| `analogRead()` + manual scaling | `analogReadMilliVolts()` |
