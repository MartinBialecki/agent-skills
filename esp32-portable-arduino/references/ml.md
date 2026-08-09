# On-device ML: TFLite Micro / ESP-DL on ESP32

Contents: chip & framework choice · pipeline · TFLM integration · quantization · pitfalls · DSP front-end · where inference lives in the architecture

## Choices

- **Chip: ESP32-S3** for anything ML — Xtensa LX7 vector (SIMD) instructions; INT8 ops hardware-accelerated via **ESP-NN** kernels; octal PSRAM variants for bigger models.
- **Framework:**
  - **TFLite Micro (TFLM) + ESP-NN** — default: best ecosystem, most examples, portable pipeline. Runs in Arduino-framework projects as a library/component.
  - **ESP-DL** — Espressif's own; deeper S3 optimization for vision/speech, smaller operator set; consider when TFLM profile shows you need the last bits of speed.
  - **Edge Impulse** — fastest path for non-ML-specialists (data capture → train → exports a ready Arduino library with TFLM inside); closed SaaS aspects, good for prototypes and theses.
- Models that fit: INT8-quantized ~50–300 KB. Typical S3 inference times at 240 MHz with vector acceleration: IMU gesture net ~2 ms, KWS ~12 ms, MNIST ~8 ms, 96×96 person detection ~145 ms (PSRAM).

## Where inference lives in the architecture

- Model array, preprocessing (filters/feature extraction) and post-processing are **`core/`** — pure C++, unit-testable on the host with recorded data. This is the project's crown jewel; keep it framework-free.
- TFLM runtime is a `hal/` (or app-level) dependency: `core/` defines `IInference { bool invoke(const Features&, Scores&) }`; the hal implementation wraps `tflite::MicroInterpreter`. Swapping TFLM→ESP-DL or moving inference off-device later then costs one file.
- Alternative valid topology: stream features over BLE and run inference in the phone app (tflite_flutter) — model updates ship via app store, no OTA needed. Decide per product constraints.

## Pipeline (train → deploy)

1. Train small model (Keras/TFLite); target ≤300 KB INT8.
2. **Quantize fully to INT8** with a representative dataset (PTQ); QAT if accuracy drops. FP32 fallback ops are ~6× slower on S3 — check the op list.
3. Convert: `xxd -i model.tflite > model_data.cc` (or `ld -r -b binary`); place in `hal/ml/`.
4. Integrate TFLM (see below) with ESP-NN enabled; verify acceleration is actually linked.
5. Feed **quantized** inputs: map sensor values via `input->params.scale` and `input->params.zero_point` — never raw 0–255/float into an INT8 tensor.

## TFLM integration sketch (hal side)

```cpp
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"

alignas(16) static uint8_t tensor_arena[kArenaSize];   // 16-BYTE ALIGNMENT IS MANDATORY on S3

bool TflmInference::begin() {
    model_ = tflite::GetModel(g_model_data);
    resolver_.AddFullyConnected(); resolver_.AddConv2D(); /* only ops the model uses */
    interpreter_ = new tflite::MicroInterpreter(model_, resolver_, tensor_arena, kArenaSize);
    return interpreter_->AllocateTensors() == kTfLiteOk;
}
```

Keep the op resolver minimal — every registered op costs flash.

## The three classic deployment bugs

1. **Unaligned tensor arena** → StoreProhibited crashes / slowdowns. `alignas(16)` or place in PSRAM with proper alignment.
2. **ESP-NN not actually linked** (reference kernels silently used) → small conv taking >50 ms means acceleration is off; check build/link config and measure with `esp_timer_get_time()` around `Invoke()`.
3. **Ignoring quantization params** on input/output → garbage scores; always scale/zero_point-map both directions.

## DSP front-end

- Use **ESP-DSP** (`esp-dsp` component, callable in Arduino builds) for FFT/filters/windowing; MFCC for audio ≈2–3 ms/frame on S3.
- Fixed-point or float32 preprocessing is fine; keep it in `core/` behind interfaces so unit tests can replay recorded CSV data through the exact same code path as firmware.
- Sampling discipline: hardware-timer or dedicated task at a fixed rate into a ring buffer; inference task consumes windows — never sample inside the inference call.

## Migration notes

TFLM, ESP-NN, ESP-DSP and ESP-DL are ESP-IDF components already; the hal wrapper moves nearly unchanged. The `core/` preprocessing/model logic and its host tests do not change at all — this is the payoff scenario for the layered architecture.
