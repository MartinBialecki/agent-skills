# Testing: host tests for core/, on-target tests, CI

Contents: why native tests · platformio.ini setup · test layout & example · fakes · on-target · CI sketch

## Why host (native) tests are the payoff of the layered design

Because `src/core/` is pure C++, it compiles for the PC. Algorithms, state machines, protocol parsing, filters and ML pre/post-processing get tested in milliseconds with no hardware, no flashing, no serial cables — and AI assistants can run the suite and iterate autonomously. Most embedded logic bugs (overflows, off-by-one, state-machine edges) are found here; hardware testing then only validates drivers and timing.

## platformio.ini

```ini
[env:native]
platform = native
build_flags =
    -std=c++17
test_build_src = false            ; do NOT build main.cpp/hal for host
; tests add the core sources they need via build_src_filter or symlinks
```

Keep `main.cpp` and everything in `src/hal/` out of the native build (they include Arduino headers). Options: compile `src/core/*.cpp` explicitly from the test via a small `extra_scripts` include, or keep `core/` as a header-mostly/standalone lib in `lib/core/` — `lib/` folders are easy to include from both device and native envs.

## Layout and example

```
test/
├── test_rem_detector/test_main.cpp      # Unity, runs on PC
├── test_protocol/test_main.cpp
├── test_trajectory/test_main.cpp
└── fixtures/night01_ir_channels.csv     # recorded real data
```

```cpp
#include <unity.h>
#include "core/RemDetector.h"
#include "fakes/FakeClock.h"

void test_no_cue_during_nrem() {
    FakeClock clock;
    RemDetector det(defaultCfg());
    for (auto& s : loadCsv("night01_nrem_segment.csv"))
        TEST_ASSERT_FALSE(det.update(s, clock).active());
}

int main(int, char**) {
    UNITY_BEGIN();
    RUN_TEST(test_no_cue_during_nrem);
    return UNITY_END();
}
```

Run: `pio test -e native` (or the pioarduino Test task). Fixtures: record sensor streams on hardware once (dump via BLE/serial to CSV), then replay forever in tests — regression-proof against algorithm changes.

## Fakes for interfaces

One fake per interface, kept in `test/fakes/`: `FakeClock` (steppable time — test timeouts without sleeping), `FakeSensors` (scripted samples), `FakeLink` (capture sent frames, inject received ones), `FakeStorage` (map). Because the interfaces are plain C++, fakes are ~20 lines each and need no mocking framework.

Suggested minimum suite for any device:
- state machine: every transition + illegal-transition handling
- protocol: parse valid/invalid/truncated frames; round-trip encode/decode
- algorithm: replayed real recordings → expected decisions
- boundaries: empty buffers, max length, wrap-around counters (esp. `millis()` overflow at 2^32 — FakeClock makes this testable)

## On-target tests

- Same Unity tests can run on-device (`pio test -e esp32-s3`) for hal drivers: pin toggling, I2C ACK scan, ADC sanity ranges, servo pulse verification with a scope/logic analyzer.
- Keep on-target tests few and hardware-focused; logic belongs in native tests.
- Hardware-in-the-loop smoke: boot → BLE advertises → connect → set/get setting → cue fires → current draw within budget (measure with a power profiler or INA219).

## CI sketch

```yaml
# .github/workflows/firmware.yml (sketch)
- uses: actions/setup-python@v5
- run: pip install platformio        # or pioarduino core
- run: pio test -e native            # fast logic gate
- run: pio run -e esp32-s3           # compile gate for the real target
```

The native test gate gives instant PR feedback; the compile gate catches Arduino-API leakage into `core/` (it fails to build when rules are broken... enforce it harder with a grep check: `! grep -r "Arduino.h" src/core/` as a CI step).
