# BLE: NimBLE peripheral for a companion mobile app

Contents: library choice · minimal server · protocol design · notifications · connection params & power · OTA · mobile contract · migration notes

## Library choice

- **NimBLE-Arduino** (h2zero, v2.x for core 3.x) — default. Far lower RAM/flash than Bluedroid; that headroom pays for ML models and buffers.
- Avoid the legacy `BLEDevice` (Bluedroid) examples the AI may suggest; they waste ~100 KB RAM and are obsolete for new designs.
- The ESP32 is the **peripheral/server**; the phone is the **central/client**. Phone does the scanning (saves device power).

## Minimal server pattern (hal/ implementation of ILink)

```cpp
#include <NimBLEDevice.h>

class NimBleLink : public ILink {
    NimBLEServer* server_ = nullptr;
    NimBLECharacteristic* txCh_ = nullptr;   // device → phone (notify)
    RxHandler rx_ = nullptr;
public:
    void begin(const char* name) {
        NimBLEDevice::init(name);
        NimBLEDevice::setPower(ESP_PWR_LVL_P3);       // trim to what the use case needs
        NimBLEDevice::setSecurityAuth(false, false, true); // adjust for real pairing later
        server_ = NimBLEDevice::createServer();
        auto* svc = server_->createService(SERVICE_UUID);
        txCh_ = svc->createCharacteristic(TX_UUID, NIMBLE_PROPERTY::NOTIFY);
        auto* rxCh = svc->createCharacteristic(RX_UUID, NIMBLE_PROPERTY::WRITE | NIMBLE_PROPERTY::WRITE_NR);
        rxCh->setCallbacks(new RxCb(this));
        svc->start();
        auto* adv = NimBLEDevice::getAdvertising();
        adv->addServiceUUID(SERVICE_UUID);
        adv->start();
    }
    bool send(uint8_t type, const uint8_t* p, size_t n) override {
        if (!server_->getConnectedCount()) return false;
        uint8_t frame[3 + 32];                        // header + payload (fixed protocol)
        frame[0] = 0xA5; frame[1] = type; frame[2] = (uint8_t)n;
        memcpy(frame + 3, p, n);
        txCh_->setValue(frame, 3 + n);
        txCh_->notify();
        return true;
    }
    bool isConnected() override { return server_->getConnectedCount() > 0; }
    void setRxHandler(RxHandler h) override { rx_ = h; }
    void onRx(const uint8_t* p, size_t n) { if (rx_ && n >= 3) rx_(p[1], p + 3, p[2]); }
};
```

Rules: callbacks (`RxCb`) only copy bytes into a FreeRTOS queue — parsing and decisions happen in `core/` on a task, never inside the BLE stack context.

## Protocol design (core/, frozen contract with the app)

- Binary, fixed header: `[sync][msgType][len][payload...]`; little-endian POD structs for payloads; optional CRC8 for settings writes.
- Keep every message ≤ negotiated MTU − 3; request MTU 247 on connect; chunk anything larger at the protocol layer.
- Define structs in `core/protocol.h` — **pure C++, no Arduino types** — so both firmware generations and the Flutter app agree byte-for-byte.

## Notifications vs reads / writes

- Stream sensor/telemetry via **notifications** (server-initiated) — polling reads force the central to wake the peripheral constantly.
- Phone → device via **write-with-response** for settings (need ack), **write-no-response** for high-rate commands.

## Connection parameters & power (tune after it works)

- Connection interval: short (15–50 ms) only while streaming fast; negotiate 300–1000 ms for idle telemetry.
- **Slave latency**: let the device skip N connection events when it has nothing to send (e.g. interval 50 ms + latency 10 ⇒ effective wake every 550 ms, still 50 ms response when data exists). The central (iOS/Android) ultimately decides — always offer min/max ranges.
- Advertising interval: ≥1000 ms when merely discoverable.
- Disable Wi-Fi entirely on BLE-only devices (`esp_wifi_stop()` + `esp_wifi_deinit()`).
- Biggest single win for a battery device: automatic **light sleep + tickless idle** — see [power.md](power.md).
- Lower-power radios: ESP32-C3/H2/C6 beat classic ESP32 for BLE-first products.

## OTA over BLE (plan from v1)

- Reserve an OTA characteristic (write = chunks, notify = progress/ack); bootloader side uses `esp_ota_ops.h` (callable from Arduino today).
- Chunk with index + CRC; verify whole image before `esp_ota_set_boot_partition()`; keep factory partition for recovery.
- Test OTA failure paths (mid-transfer disconnect) before shipping — consumer devices live or die by this.

## Mobile (Flutter) contract checklist

- Service/characteristic UUIDs, MTU negotiation, notification enable (CCCD), bonding behavior — document all in `core/protocol.h` comments; the app mirrors them.
- Flutter side: `flutter_blue_plus` (maintained standard; original flutter_blue is deprecated). Subscribe to notify characteristic, parse the same binary header.
- Bonding/just-works vs passkey: decide early; changing security later breaks installed apps.

## Migration notes (Arduino → ESP-IDF)

- NimBLE-Arduino wraps the same Apache NimBLE host ESP-IDF uses (`nimble/` in IDF). Migration rewrites this hal class against `host/ble_hs.h` GATT tables and gap event handlers; protocol structs, queues and `core/` consumers are untouched.
- Bluedroid-only Arduino calls (if any slipped in) have no 1:1 future — keep everything NimBLE.
