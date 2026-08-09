# Migration playbook: Arduino framework → ESP-IDF

Contents: when to migrate · scope · the hybrid phase · API mapping table · step plan · verification

## When to migrate

Triggers: product validated and heading to production; need ULP/secure-boot/flash-encryption/OTA hardening; binary size or RAM pressure; BLE/Wi-Fi tuning beyond Arduino's exposure. Do **not** migrate "because production" while the MVP is still changing weekly — migrate when requirements stabilize.

## Expected scope (if the layered rules were followed)

| Code | Fate |
|---|---|
| `src/core/` (logic, protocols, ML, tests) | unchanged — recompiles as-is |
| mobile app / BLE protocol | unchanged |
| `src/hal/` | rewritten class-by-class against IDF drivers (mechanical, per mapping table) |
| `src/app/`, `main.cpp` | rewired: `setup()/loop()` → `app_main()` + explicit tasks |
| `platformio.ini` → `sdkconfig` + CMake | new build config |

If `core/` contains Arduino calls (rules were broken), expect a `String→std::string`, `millis→esp_timer`, `Preferences→NVS` sweep first — mechanical but pervasive. Avoid it by construction.

## The hybrid phase (Arduino as ESP-IDF component)

Migrate gradually instead of big-bang:

1. Switch PlatformIO env to `framework = espidf, arduino` (pioarduino supports Arduino-as-component; ESP-IDF component manager can also pull `espressif/arduino-esp32: ^3.3`, which pairs with IDF 5.5).
2. Add `sdkconfig.defaults` with at least:
   ```
   CONFIG_AUTOSTART_ARDUINO=y
   CONFIG_FREERTOS_HZ=1000
   CONFIG_PM_ENABLE=y
   CONFIG_FREERTOS_USE_TICKLESS_IDLE=y
   ```
3. Everything still builds and runs. Now port hal classes one by one to native IDF, keeping the rest on Arduino APIs. Gitignore `managed_components/` and `dependencies.lock`.
4. When no file includes `Arduino.h` anymore, flip to `framework = espidf` and delete the Arduino component.

## API mapping table (hal rewrite reference)

| Arduino | ESP-IDF |
|---|---|
| `setup()/loop()` | `extern "C" void app_main()` + `xTaskCreate` loop with `vTaskDelay(pdMS_TO_TICKS(1))` |
| `pinMode/digitalWrite/digitalRead` | `gpio_set_direction / gpio_set_level / gpio_get_level` (`driver/gpio.h`) |
| `analogReadMilliVolts` | `adc_oneshot_read` + calibration (`esp_adc/adc_oneshot.h`) |
| `ledcAttach/ledcWrite` | `ledc_timer_config / ledc_channel_config / ledc_set_duty` (`driver/ledc.h`) |
| `attachInterrupt` | `gpio_isr_handler_add` |
| `millis()` | `esp_timer_get_time()/1000` or `xTaskGetTickCount()*portTICK_PERIOD_MS` |
| `micros()` | `esp_timer_get_time()` |
| `delay(ms)` | `vTaskDelay(pdMS_TO_TICKS(ms))` |
| `delayMicroseconds` | `esp_rom_delay_us` |
| `String` | `std::string` |
| `Serial.printf` | `ESP_LOGI/ESP_LOGD` (`esp_log.h`) — route through the existing `ILog` impl |
| `Preferences` | `nvs_open/nvs_get_*/nvs_set_*` + `nvs_commit` (direct mapping) |
| `Update` | `esp_ota_begin/write/end/set_boot_partition` (`esp_ota_ops.h`) |
| `NimBLE-Arduino` | Apache NimBLE host API (`nimble/host/...`, `ble_hs.h`, GATT tables) |
| `WiFi.h` | `esp_wifi_init/set_config/start` + `esp_event_handler_register(WIFI_EVENT/IP_EVENT)` |
| `HTTPClient` | `esp_http_client` |
| `LittleFS.h` (File objects) | `esp_littlefs` + POSIX `fopen/fread` (VFS) |
| `ESP32Servo` | `driver/ledc.h` or `mcpwm_prelude.h` servo example |
| `FastAccelStepper` | re-wrap on `driver/rmt_tx.h` (library core already hardware-timed) |
| `Wire` (I2C) | `driver/i2c_master.h` (new master bus API) |
| `HardwareSerial` | `driver/uart.h` |
| `ArduinoJson` | keep — works on IDF (set `ARDUINOJSON_ENABLE_STD_STRING=1`, disable Arduino String) |

Already-IDF code written during the Arduino phase (`esp_pm`, `esp_sleep`, `esp_timer`, `esp_ota`, NVS, TFLM/ESP-NN/ESP-DSP) moves untouched.

## Step plan

1. Branch; build current firmware; archive the binary + version tag as the golden reference.
2. Freeze behavior: run the host test suite (`core/`) — green baseline. Add golden I/O recordings (sensor→decision traces) if missing.
3. Enter hybrid mode (above); fix only build errors, no refactors.
4. Port hal classes leaf-first: GPIO/PWM/timing → storage → I2C/SPI sensors → actuators → BLE last (largest surface). One class per commit; hardware test each.
5. Flip to pure `esp-idf`; remove Arduino component; slim `sdkconfig`.
6. Add production features now trivial: secure boot v2, flash encryption, signed OTA with rollback, core-dump partition, brownout config.
7. Full regression: host tests + hardware test matrix + BLE interop with the shipped app version + OTA round trip + power measurements against the Arduino-era baseline.

## Verification checklist

- [ ] Host unit tests pass unchanged (proves `core/` survived)
- [ ] BLE protocol byte-identical (app unmodified, old app version still works)
- [ ] Idle current ≤ Arduino-era number (light sleep) — measure, don't assume
- [ ] OTA: success path + mid-transfer abort + rollback
- [ ] Crash → core dump captured and decodable
- [ ] 72 h soak test on final hardware
