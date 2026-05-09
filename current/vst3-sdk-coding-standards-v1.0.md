# Modern Professional Coding Standards for Raw Steinberg VST3 SDK Plug-ins

**Version:** 1.0  
**Generated:** 2026-05-09  
**Audience:** human developers and agentic coding systems  
**Scope:** reusable standards for professional plug-ins built directly on the Steinberg VST3 SDK

---

## 0. Non-negotiable principles

1. **`IAudioProcessor::process()` is a real-time callback.** It must not allocate, block, wait, lock, log, perform I/O, call UI APIs, or throw exceptions.
2. **All host boundary methods must follow VST3 contracts exactly.** Signatures, `PLUGIN_API`, reference counting, `tresult` values, component/controller separation, bus rules, and parameter semantics are not optional.
3. **DSP must be independent from the SDK.** The raw VST3 classes adapt `ProcessData`, parameter queues, state streams, and editor/controller messages to a testable DSP core.
4. **Stable IDs are permanent product contracts.** FUIDs, parameter IDs, unit IDs, and state versions must not be changed casually.
5. **Thread safety and memory safety are first-class design constraints.** Shared mutable state must use atomics, immutable snapshots, or bounded lock-free queues with documented ownership and memory ordering.
6. **The plug-in must validate with Steinberg's validator and real hosts.** Passing unit tests is not enough.
7. **Agent-generated code is held to the same standard as human code.** No hidden allocations, no broad compatibility changes, no untested state migrations.

---

## 1. Current SDK, toolchain, and build baseline

### 1.1 VST3 SDK

Use the latest non-preview Steinberg VST3 SDK pinned by tag or commit. As of this document's research pass, Steinberg's change history listed **VST SDK 3.8.0** dated 2025-10-20, and the public SDK repository described the 3.8.x SDK under the MIT license.

Always verify the current non-preview SDK before starting a new project and record the exact SDK commit/tag in the repository.

### 1.2 Language and compiler

- Project code: **C++20**.
- Windows compiler baseline: **MSVC 19.44+ / Visual Studio 2022**.
- Build architecture: x64 minimum unless the product explicitly supports more.
- Cross-platform builds should also compile with Clang or GCC where supported.

MSVC recommended options for project code:

```cmake
target_compile_features(MyPlugin PRIVATE cxx_std_20)

target_compile_options(MyPlugin PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive- /Zc:__cplusplus /MP>
)
```

Treat warnings in third-party SDK code separately from project code. Do not globally suppress warnings in a way that hides project bugs.

### 1.3 CMake

Steinberg's “creating a CMake plug-in from scratch” tutorial uses `cmake_minimum_required(VERSION 3.15.0)` and states that 3.15.0 is sufficient for that example. Use **CMake 3.15+** as the raw VST3 project minimum unless the selected SDK release documents a higher requirement.

Recommended:

```cmake
cmake_minimum_required(VERSION 3.15...3.30)
project(MyPlugin VERSION 1.0.0 LANGUAGES CXX)
```

The older “CMake 3.14+” baseline is not ideal for new raw VST3 projects because Steinberg's current tutorial baseline is 3.15. Use newer CMake in CI even if the minimum remains 3.15.

### 1.4 Raw VST3 CMake skeleton

```cmake
cmake_minimum_required(VERSION 3.15...3.30)
project(MyPlugin VERSION 1.0.0 DESCRIPTION "MyPlugin Effect" LANGUAGES CXX)

set(vst3sdk_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external/vst3sdk")
add_subdirectory(${vst3sdk_SOURCE_DIR} ${PROJECT_BINARY_DIR}/vst3sdk)
smtg_enable_vst3_sdk()

smtg_add_vst3plugin(MyPlugin
    source/version.h
    source/myplugin_cids.h
    source/myplugin_entry.cpp
    source/myplugin_processor.h
    source/myplugin_processor.cpp
    source/myplugin_controller.h
    source/myplugin_controller.cpp
    source/dsp/DspCore.h
    source/dsp/DspCore.cpp
)

target_link_libraries(MyPlugin PRIVATE sdk)
target_compile_features(MyPlugin PRIVATE cxx_std_20)

smtg_target_configure_version_file(MyPlugin)
```

Use `SMTG_CREATE_PLUGIN_LINK=0` or equivalent SDK options in developer presets if Windows symbolic links cause local build issues. CI should use a deterministic packaging step.

---

## 2. Repository and architecture

### 2.1 Recommended layout

```text
.
├── CMakeLists.txt
├── CMakePresets.json
├── external/
│   └── vst3sdk/                 # pinned SDK
├── source/
│   ├── myplugin_entry.cpp       # factory definitions
│   ├── myplugin_cids.h          # FUIDs
│   ├── myplugin_processor.*     # AudioEffect/IAudioProcessor
│   ├── myplugin_controller.*    # EditController/IEditController
│   ├── parameters/              # parameter IDs, conversion, metadata
│   ├── state/                   # IBStream state read/write and migration
│   ├── dsp/                     # SDK-independent DSP core
│   └── ui/                      # VSTGUI or other editor layer, optional
├── tests/
│   ├── unit/
│   ├── dsp_regression/
│   ├── state_migration/
│   └── realtime_safety/
└── docs/
```

### 2.2 Component/controller separation

Raw VST3 plug-ins commonly have:

- **Processor/component:** audio processing, audio/event buses, state needed for DSP, parameter queue consumption.
- **Edit controller:** parameter metadata, UI/editor creation, edit gestures, string conversion, unit/program organisation.

Do not share mutable objects directly between processor and controller. Use VST3 parameters, messages, host-mediated state, or the SDK's supported data-exchange mechanisms.

### 2.3 DSP isolation

The DSP core should not include VST3 headers. It should expose plain C++ types and spans/views.

```cpp
struct ProcessSpec {
    double sampleRate = 44100.0;
    int32_t maxBlockSize = 512;
    int32_t numChannels = 2;
    bool doublePrecision = false;
};

class DspCore final {
public:
    void prepare(const ProcessSpec& spec);
    void reset() noexcept;
    void setParameterSnapshot(const ParameterSnapshot& snapshot) noexcept;
    void process(ProcessBlockView<float> block) noexcept;
    void process(ProcessBlockView<double> block) noexcept;
};
```

The VST3 processor translates `Vst::ProcessData` into this interface.

---

## 3. Formatting, naming, and SDK style

### 3.1 Formatting

Use `clang-format` and enforce it in CI. Avoid broad formatting-only diffs unless explicitly requested.

### 3.2 Naming

- Follow VST3 method names and signatures exactly for overrides.
- Use project conventions for internal code.
- Keep FUID constants and parameter IDs in central files.
- Use established DSP abbreviations where clear.

Example:

```cpp
class MyPluginProcessor final : public Steinberg::Vst::AudioEffect {
public:
    Steinberg::tresult PLUGIN_API initialize(Steinberg::FUnknown* context) SMTG_OVERRIDE;
    Steinberg::tresult PLUGIN_API terminate() SMTG_OVERRIDE;
    Steinberg::tresult PLUGIN_API setupProcessing(Steinberg::Vst::ProcessSetup& setup) SMTG_OVERRIDE;
    Steinberg::tresult PLUGIN_API setActive(Steinberg::TBool state) SMTG_OVERRIDE;
    Steinberg::tresult PLUGIN_API process(Steinberg::Vst::ProcessData& data) SMTG_OVERRIDE;
};
```

### 3.3 VST3 boundary types

Use VST3 types at SDK boundaries:

- `Steinberg::tresult`
- `Steinberg::TBool`
- `Steinberg::Vst::ParamID`
- `Steinberg::Vst::ParamValue`
- `Steinberg::Vst::Sample32` / `Sample64`
- `Steinberg::Vst::String128`
- `Steinberg::FUID`

Internally, use standard C++ types when clearer. Do not force VST3 types deep into SDK-independent DSP.

### 3.4 Strings

VST3 uses UTF-16-ish SDK string types. Use SDK helpers correctly.

```cpp
Steinberg::Vst::String128 name {};
Steinberg::UString(name, 128).assign(STR16("Gain"));
```

For numeric conversions, avoid allocation in hot paths. Controller string conversion is non-real-time, but should still be bounded and robust.

---

## 4. VST3 lifecycle standards

### 4.1 `initialize()`

Responsibilities:

- Call base `AudioEffect::initialize(context)` first and propagate failure.
- Add audio/event buses.
- Set controller class ID.
- Initialise host-independent metadata.

Do not allocate sample-rate/block-size-dependent DSP resources here unless they truly do not depend on processing setup.

```cpp
tresult PLUGIN_API MyPluginProcessor::initialize(FUnknown* context)
{
    const auto result = AudioEffect::initialize(context);
    if (result != kResultOk)
        return result;

    addAudioInput(STR16("Stereo In"), Steinberg::Vst::SpeakerArr::kStereo);
    addAudioOutput(STR16("Stereo Out"), Steinberg::Vst::SpeakerArr::kStereo);
    setControllerClass(MyPluginControllerUID);

    return kResultOk;
}
```

### 4.2 `setupProcessing(ProcessSetup&)`

This is where the host provides processing mode, symbolic sample size, max block size, and sample rate.

Responsibilities:

- Validate sample rate and max block size.
- Store precision mode.
- Prepare DSP resources.
- Preallocate temporary buffers.
- Prepare oversampling, filters, delay lines, FFT plans.
- Compute latency if needed.

```cpp
tresult PLUGIN_API MyPluginProcessor::setupProcessing(ProcessSetup& setup)
{
    if (setup.sampleRate <= 0.0 || setup.maxSamplesPerBlock < 0)
        return kInvalidArgument;

    processSetup_ = setup;
    dsp_.prepare({setup.sampleRate,
                  setup.maxSamplesPerBlock,
                  getCurrentChannelCount(),
                  setup.symbolicSampleSize == kSample64});

    return AudioEffect::setupProcessing(setup);
}
```

### 4.3 `setProcessing(TBool)`

Use for processing start/stop transitions. Avoid heavy allocation. Flush or reset transient processing state if required by the algorithm and host state.

### 4.4 `setActive(TBool)`

Use `setActive(true)` to reset runtime state and clear buffers before processing begins. Use `setActive(false)` to release or mark inactive resources if appropriate.

Do not depend on one exact host call sequence beyond the SDK contract. Validator and host behaviour should be tested.

### 4.5 `terminate()`

Release resources and call base terminate. Destructors must still be safe if `terminate()` is not the only path.

---

## 5. `process(ProcessData&)` standards

### 5.1 Hard real-time rules

Inside `process()` and all functions it calls on the processing path:

Forbidden:

- heap allocation/deallocation;
- `std::vector` growth, `std::string` growth, `std::function` creation, `shared_ptr` ownership churn;
- locks, mutexes, condition variables, semaphores;
- `std::atomic::wait`, sleeps, yields, joins;
- file, console, network, registry, COM, Objective-C, CoreFoundation, Win32 calls with unknown latency;
- logging;
- UI/editor access;
- exceptions;
- unbounded loops.

Allowed:

- arithmetic;
- bounded loops over samples/channels/events;
- stack variables with small bounded size;
- preallocated buffers;
- lock-free atomics and queues with bounded operations;
- reading immutable snapshots.

### 5.2 Always consume parameter changes

VST3 hosts may call `process()` with zero samples and no buffers to flush parameter changes. Therefore, handle parameter queues before returning for no audio.

```cpp
tresult PLUGIN_API MyPluginProcessor::process(ProcessData& data)
{
    consumeInputParameterChanges(data.inputParameterChanges, data.numSamples);

    if (data.numSamples <= 0)
        return kResultOk;

    if (data.numOutputs <= 0 || data.outputs == nullptr)
        return kResultOk;

    // process audio...
    return kResultOk;
}
```

Do not assert `data.numSamples > 0`.

### 5.3 Sample size

Handle both supported sample sizes if the plug-in advertises double precision.

```cpp
if (data.symbolicSampleSize == kSample32) {
    process32(data);
} else if (data.symbolicSampleSize == kSample64) {
    process64(data);
} else {
    return kInvalidArgument;
}
```

If only 32-bit processing is supported, report precision support accurately and reject unsupported paths according to SDK expectations.

### 5.4 Buffer aliasing and null buffers

VST3 buffer pointers may be:

- same for input and output, requiring in-place-safe processing;
- different for input and output;
- shared across buses in some host/instrument cases;
- null for inactive/silent channels.

Rules:

- Check bus and channel counts.
- Check channel buffer pointers before dereference.
- Do not clear output before reading input if input and output alias.
- Clear unused output channels intentionally.
- Honour silence flags where supported.

### 5.5 Channel and bus handling

Do not assume stereo unless the plug-in explicitly only supports stereo and rejects other arrangements.

For each supported layout, define:

- accepted input arrangements;
- accepted output arrangements;
- sidechain behaviour;
- event input/output needs;
- mono-to-stereo or stereo-to-mono policy;
- inactive bus policy.

### 5.6 Process context

`data.processContext` is optional. Use it defensively for tempo, transport, time signature, sample position, and musical timing.

Do not require it for basic audio correctness.

### 5.7 Events

If the plug-in handles MIDI/note events:

- consume `data.inputEvents` in bounded loops;
- validate event types;
- handle sample offsets;
- avoid allocation when scheduling voices;
- preallocate voice pools;
- implement voice stealing deterministically;
- emit output events only if supported and bounded.

---

## 6. Parameters and automation

### 6.1 Normalized values

VST3 parameters use normalized values in `[0.0, 1.0]` at the host boundary. Conversion to plain units belongs in parameter metadata/controller utilities.

```cpp
float cutoffHzFromNormalized(ParamValue normalized) noexcept;
ParamValue normalizedFromCutoffHz(float hz) noexcept;
```

Conversions must be:

- monotonic where expected;
- robust to out-of-range input;
- stable across versions unless migration is provided;
- tested at min, max, default, and representative values.

### 6.2 Parameter IDs

Centralise IDs:

```cpp
namespace Params {
    constexpr Steinberg::Vst::ParamID Gain = 100;
    constexpr Steinberg::Vst::ParamID Cutoff = 101;
}
```

Do not reuse removed IDs. Keep deprecated IDs reserved.

### 6.3 Sample-accurate automation

`IParameterChanges` contains queues of parameter points with sample offsets. Handle them accurately for parameters where timing matters.

Basic pattern:

```cpp
void MyPluginProcessor::consumeInputParameterChanges(IParameterChanges* changes, int32 numSamples) noexcept
{
    if (changes == nullptr)
        return;

    const int32 count = changes->getParameterCount();
    for (int32 i = 0; i < count; ++i) {
        IParamValueQueue* queue = changes->getParameterData(i);
        if (queue == nullptr)
            continue;

        const ParamID id = queue->getParameterId();
        const int32 points = queue->getPointCount();

        for (int32 p = 0; p < points; ++p) {
            int32 sampleOffset = 0;
            ParamValue value = 0.0;
            if (queue->getPoint(p, sampleOffset, value) == kResultOk) {
                parameterScheduler_.pushPoint(id, sampleOffset, value);
            }
        }
    }
}
```

For continuous parameters, either process segments between sample offsets or feed target points into a smoother. For discrete parameters, apply changes at defined sample boundaries with click-free transition if audible.

### 6.4 Controller edit gestures

The controller must use host edit gesture semantics correctly:

- `beginEdit(paramId)` when a user gesture starts;
- `performEdit(paramId, normalizedValue)` during the gesture;
- `endEdit(paramId)` when the gesture ends.

Do not spam host automation from timers or passive UI updates.

### 6.5 Output parameters and meters

Meters should not be ordinary automatable parameters unless host automation is intended. Prefer output parameter changes or other supported mechanisms where appropriate. Meter communication must be bounded and non-blocking.

---

## 7. Dedicated thread-safety and memory-safety standard

### 7.1 Thread model

Assume these contexts:

- Real-time audio processing thread calling `process()`.
- Controller/UI thread(s).
- Host non-real-time threads calling lifecycle, state, and parameter methods.
- Project background workers.

The SDK does not give permission to share mutable state without synchronization. The audio thread simply cannot use synchronization that may block.

### 7.2 Direct UI/audio sharing is forbidden

Steinberg's Data Exchange guidance highlights the core issue: modern plug-in UI representation and audio processing should not directly share data because mutexes can block the audio thread. Follow that principle throughout the project.

### 7.3 Atomics

Use atomic scalars for simple cross-thread values:

```cpp
std::atomic<float> latestMeterDb {-120.0f};

// audio thread
latestMeterDb.store(meterDb, std::memory_order_relaxed);

// UI thread
const float db = latestMeterDb.load(std::memory_order_relaxed);
```

Use `memory_order_relaxed` for independent values. Use release/acquire only to publish related data.

### 7.4 Immutable snapshots

For complex configuration changes:

- prepare new configuration off the audio thread;
- validate fully;
- publish pointer/index atomically;
- switch at block boundary;
- retire old objects outside the audio thread.

Do not free a snapshot on the audio thread if its destructor may deallocate.

### 7.5 Lock-free queues

Use SPSC queues for strictly one producer and one consumer. For multiple producers, use separate SPSC queues or a reviewed MPSC structure with bounded behaviour.

Production requirements:

- cache-line padding for indices;
- fixed capacity;
- non-allocating payload operations;
- overflow policy;
- tests for wrap-around;
- TSAN/stress tests where applicable.

No audio-thread blocking on full/empty queues.

### 7.6 ABA, reclamation, and pointer exchange

Avoid raw pointer compare-exchange designs unless reclamation is solved. Prefer:

- index + generation handles;
- immutable snapshot arrays;
- preallocated pools;
- deferred deletion on non-real-time thread;
- epoch-like schemes when necessary.

### 7.7 False sharing

Separate frequently written atomic variables by cache line when written/read by different threads at high frequency. This is especially important for meters, queues, and transport/parameter flags.

### 7.8 Ownership policy

- `std::unique_ptr`: preferred owner outside audio hot paths.
- Raw pointer/reference: non-owning only, lifetime documented.
- `std::shared_ptr`: not allowed in audio processing paths except through an approved immutable snapshot scheme that proves no ref-count churn/deletion occurs on the audio thread.
- Manual `new/delete`: only in SDK-required factories or tightly wrapped ownership utilities.
- Object pools: preallocate; deterministic allocation/free; no unbounded search on audio thread.

### 7.9 Memory safety tooling

Required or strongly recommended:

- MSVC `/analyze` / C++ Core Guidelines checks.
- MSVC AddressSanitizer on Windows test harnesses.
- Clang ASAN/UBSAN builds.
- Clang TSAN builds for thread communication harnesses.
- Fuzz tests for state/preset loading.
- Allocation detection around `process()`.

---

## 8. DSP engineering standards

### 8.1 Denormals

Every processing block must protect against subnormal slowdowns. Use platform-neutral project utilities or CPU flag scopes where appropriate. Ensure any FTZ/DAZ changes are scoped/restored and do not surprise the host.

Also sanitize feedback/filter state after long silence.

### 8.2 Numerical stability

DSP must handle:

- silence;
- full-scale input;
- NaN/Inf input;
- extreme parameter values;
- unusual sample rates;
- feedback boundaries;
- rapid automation;
- offline processing at large block sizes.

Filter designs should be stable under modulation. Consider TPT/state-variable filters for modulated filters. Validate coefficient ranges.

### 8.3 Oversampling

Use oversampling for nonlinear processors when needed to control aliasing.

Rules:

- Prepare oversampling buffers and filters in `setupProcessing()` or `setActive(true)`, not in `process()`.
- Report latency accurately.
- Test aliasing at multiple sample rates.
- Do not change oversampling factor in `process()`.
- Provide quality modes only through safe reconfiguration.

### 8.4 Latency and tail

Implement latency and tail reporting according to VST3 expectations.

- Report latency if lookahead, linear-phase filters, oversampling, or convolution introduces delay.
- Test with impulse alignment.
- Report tail for reverbs, delays, dynamics release, and feedback effects.
- Define reset/bypass behaviour.

### 8.5 Bypass

Bypass should be click-free and latency-aware. If the plug-in uses lookahead/oversampling/convolution, bypass should not shift timing unexpectedly. Crossfade if needed.

### 8.6 SIMD

Use SIMD only behind tested abstraction.

- Keep scalar reference.
- Validate alignment.
- Handle non-vector-length tails.
- Avoid platform intrinsics leaking through the DSP API.
- Compare against reference within tolerance.


### 8.7 Advanced DSP research and implementation checklist

Raw VST3 code should be a host adapter around a high-quality DSP core. The algorithmic standards below apply before SDK details.

#### 8.7.1 Nonlinear and virtual-analog processing

For saturation, clipping, waveshaping, exciters, tape, amp, and nonlinear filter models:

- expect aliasing and test for it;
- use oversampling, antiderivative anti-aliasing, band-limited nonlinear methods, or model-specific anti-aliasing where appropriate;
- validate high drive at low and high sample rates;
- crossfade quality or oversampling changes;
- bound nonlinear feedback paths;
- document latency, quality, and CPU trade-offs.

#### 8.7.2 Modulated filters and time-varying systems

- Prefer stable topologies under modulation.
- Smooth parameters or coefficients carefully.
- Test near zero frequency, Nyquist, high resonance, and fast automation.
- Calculate coefficients in double precision where useful.
- Avoid unsafe coefficient interpolation in structures that can become unstable.

#### 8.7.3 Dynamics processing

- Define detector type, attack/release, knee, ratio, sidechain filtering, stereo linking, and lookahead.
- Report lookahead latency.
- Test impulses, sine bursts, stepped levels, silence recovery, and denormal-prone release tails.
- Make bypass latency-aware and click-free.

#### 8.7.4 Delay, reverb, and feedback networks

- Preallocate buffers.
- Bound feedback coefficients.
- Smooth delay-time changes.
- Test fractional-delay interpolation and modulation artifacts.
- Define reset, bypass, freeze, and tail behaviour.

#### 8.7.5 Convolution

- Parse IR files and build partitions off the audio thread.
- Publish new convolution engines through immutable snapshot handoff.
- Crossfade IR changes.
- Report latency accurately.
- Fuzz malformed IR/state inputs.
- Test huge, zero-length, mono/stereo, and mismatched-rate IRs.

#### 8.7.6 Spectral and time-frequency processing

- Preallocate FFT plans, windows, overlap buffers, and scratch memory.
- Document FFT size, window, hop size, overlap, phase handling, and latency.
- Test overlap-add reconstruction error.
- Handle transport jumps and reset conditions explicitly.
- Do not change FFT size in `process()` unless all resources were already prepared.

#### 8.7.7 Resampling

- Use characterised band-limited/polyphase resampling for quality-critical paths.
- Define passband/stopband/latency/phase trade-offs.
- Test with impulses and sweeps.
- Prepare resampler state outside `process()`.

#### 8.7.8 Instruments and voice management

- Preallocate voice pools and event buffers.
- No per-note allocation.
- Deterministic voice stealing.
- Correct sample-offset handling for note events.
- Bounded modulation matrix processing.
- Correct all-notes-off/all-sound-off behaviour.

#### 8.7.9 Spatial and multichannel processing

- Explicitly document supported bus and speaker arrangements.
- Test channel order and layout conversion.
- Keep mono/stereo/surround/ambisonic code paths isolated enough to test.
- Avoid assuming stereo in generic processing utilities.

#### 8.7.10 ML/model-based DSP

- No model loading, graph compilation, JIT, Python, GPU initialisation, or allocation in `process()`.
- Warm up inference engines before processing.
- Preallocate tensor memory and scratch buffers.
- Bound worst-case inference time.
- Validate model files and state as untrusted input.
- Provide deterministic fallback/bypass if model preparation fails.

#### 8.7.11 Determinism

- Define tolerances instead of assuming cross-platform bit identity.
- Use deterministic seeds.
- Avoid unordered iteration in DSP-affecting code.
- Maintain scalar reference implementations for SIMD/model approximations.


---

## 9. State and preset standards

### 9.1 `getState()` / `setState()`

State methods may allocate and parse because they are not audio callbacks, but they must be thread-safe and robust.

Rules:

- Use versioned binary or structured state.
- Validate every read.
- Check stream errors.
- Clamp unsafe values.
- Do not trust host/preset data.
- Do not serialize raw pointers or ABI-dependent layouts.
- Do not call audio-thread-only functions.
- Avoid changing live DSP objects directly; prepare and publish safely if needed.

### 9.2 Versioning

State format must include:

- magic or type marker;
- version;
- parameter/value map;
- optional feature flags;
- checksum only if useful and documented.

Every version migration requires tests.

### 9.3 Parameter migration

If a parameter changes meaning:

- preserve old ID if possible and transform values;
- reserve removed IDs;
- map old states to new states;
- document automation compatibility implications;
- test old session recall.

---

## 10. UI/controller standards

### 10.1 Controller responsibilities

The controller owns:

- parameter definitions;
- string conversion;
- units/groups;
- UI creation;
- edit gestures;
- controller-side state that is not real-time DSP state.

The controller must not directly mutate processor DSP state.

### 10.2 UI thread

UI/editor code must remain on the UI thread. Do not call editor methods from `process()`. Metering/visualization data must be transferred through atomics or bounded queues.

### 10.3 VSTGUI or other UI toolkit

If using VSTGUI:

- keep resource loading off the audio thread;
- avoid blocking file I/O during editor open where possible;
- handle DPI/scaling;
- implement resizing according to host expectations;
- ensure editor teardown cancels pending callbacks.

---

## 11. Error handling

### 11.1 `tresult`

Use `tresult` at all SDK boundaries. Return values must match the semantics of the operation.

Common values:

- `kResultOk`: success.
- `kResultFalse`: valid request but unsupported/declined.
- `kInvalidArgument`: invalid pointer/value/count.
- `kNotImplemented`: feature not implemented.

### 11.2 Exceptions

Do not allow exceptions to escape any VST3 callback or factory. `process()` and DSP hot paths must be `noexcept` where practical.

### 11.3 Diagnostics

Do not log in `process()`. For audio-thread diagnostics, set atomics/counters and report from non-real-time code.

---

## 12. Testing and validation

### 12.1 Unit tests

DSP and parameter conversion code must be tested without a host.

Required tests:

- parameter normalization round trips;
- min/max/default values;
- automation point scheduling;
- silence/full-scale/NaN/Inf;
- impulse response;
- bypass transitions;
- latency alignment;
- tail/reset behaviour;
- sample-rate and block-size changes;
- mono/stereo/supported bus layouts.

### 12.2 VST3 validator

Run Steinberg's validator in CI for every built VST3 artifact. Treat failures as release blockers unless a documented SDK/validator issue has a tracked waiver.

### 12.3 Host smoke tests

Before release, test in representative hosts. Cover:

- scanning/loading;
- opening/closing editor repeatedly;
- automation write/read;
- preset save/load;
- offline bounce;
- sample-rate changes;
- buffer-size changes;
- bypass;
- sidechain/bus layouts if supported.

### 12.4 Real-time safety tests

Use a standalone harness around `process()` to detect:

- heap allocation;
- lock attempts;
- first-block initialisation costs;
- unbounded queue behaviour;
- random block-size issues;
- repeated setup/active/process cycles.

### 12.5 Sanitizers and fuzzing

- ASAN for memory errors.
- UBSAN for undefined behaviour.
- TSAN for data races in non-real-time test harnesses.
- Fuzzing for state stream parsing and parameter conversion.

---

## 13. Release gates

A release candidate must pass:

- clean configure/build with pinned SDK;
- warnings clean for project code;
- unit tests;
- DSP regression tests;
- state migration tests;
- static analysis;
- sanitizer jobs;
- VST3 validator;
- host smoke tests;
- packaging/signing/notarization where applicable;
- documentation update for version, latency, supported layouts, and known limitations.

---

## 14. Agentic coder rules

Agents modifying raw VST3 projects must obey:

1. Do not change FUIDs, parameter IDs, state versions, or normalization mappings without explicit migration work.
2. Do not add allocation, locks, waits, logging, UI calls, or exceptions to `process()` or DSP functions it calls.
3. Always handle zero-sample process calls and parameter flushing.
4. Always consider input/output buffer aliasing and null channel pointers.
5. Keep SDK-specific code out of the DSP core.
6. Use `tresult` correctly at SDK boundaries.
7. Preserve `PLUGIN_API`, `SMTG_OVERRIDE`, reference counting, and factory patterns.
8. Add tests for DSP, state, and automation changes.
9. Run or document inability to run VST3 validator.
10. Report uncertainty about host threading or lifecycle rather than making permissive assumptions.

---

## 15. Code review checklist

### VST3 compliance

- [ ] Factory and FUIDs are correct and unique.
- [ ] Processor/controller class IDs are stable.
- [ ] SDK method signatures are exact.
- [ ] `tresult` return values are appropriate.
- [ ] Bus arrangements are validated.
- [ ] `setupProcessing()`, `setProcessing()`, `setActive()`, and `process()` follow lifecycle rules.
- [ ] Zero-sample parameter flush calls are handled.
- [ ] Sample32/Sample64 paths are correct.
- [ ] Buffer aliasing/null buffers are safe.
- [ ] Validator passes.

### Real-time safety

- [ ] No allocation/deallocation in `process()`.
- [ ] No locks/waits/sleeps/logging/I/O/UI calls in `process()`.
- [ ] No exception escape.
- [ ] Denormal protection exists.
- [ ] No `shared_ptr` ownership churn.
- [ ] Processing loops are bounded.

### Thread and memory safety

- [ ] Cross-thread communication is atomics/snapshots/bounded queues.
- [ ] Memory orders are justified.
- [ ] Queue producer/consumer assumptions hold.
- [ ] False sharing considered for hot atomics.
- [ ] Object reclamation cannot occur unsafely on audio thread.
- [ ] Sanitizer/static-analysis coverage exists.

### DSP

- [ ] Parameter smoothing/automation is correct.
- [ ] Numerical edge cases tested.
- [ ] Latency/tail/bypass behaviour defined and tested.
- [ ] Oversampling/anti-aliasing tested where relevant.
- [ ] SIMD has scalar fallback.

### State and compatibility

- [ ] State format versioned.
- [ ] Old states/presets migrate.
- [ ] Parameter IDs stable.
- [ ] Removed IDs reserved.
- [ ] Malformed state cannot crash plug-in.

---

## 16. References

- Steinberg VST3 SDK: https://github.com/steinbergmedia/vst3sdk
- VST3 change history: https://steinbergmedia.github.io/vst3_dev_portal/pages/Versions/Index.html
- VST3 API documentation: https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical%2BDocumentation/API%2BDocumentation/Index.html
- VST3 parameters and automation: https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical%2BDocumentation/Parameters%2BAutomation/Index.html
- VST3 Data Exchange: https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical%2BDocumentation/Data%2BExchange/Index.html
- VST3 validator: https://steinbergmedia.github.io/vst3_dev_portal/pages/What%2Bis%2Bthe%2BVST%2B3%2BSDK/Validator.html
- VST3 CMake tutorial: https://steinbergmedia.github.io/vst3_dev_portal/pages/Tutorials/Creating%2Ba%2Bplug-in%2Bfrom%2Bscratch.html
- Microsoft Visual Studio 2022 release notes: https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes/
- Microsoft C++ conformance improvements: https://learn.microsoft.com/cpp/overview/cpp-conformance-improvements
- Microsoft `/fsanitize`: https://learn.microsoft.com/cpp/build/reference/fsanitize
- Microsoft C++ Core Guidelines checker: https://learn.microsoft.com/cpp/code-quality/using-the-cpp-core-guidelines-checkers
- C++ Core Guidelines: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines
- Clang ThreadSanitizer: https://clang.llvm.org/docs/ThreadSanitizer.html
- Clang UndefinedBehaviorSanitizer: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html
- PortAudio callback documentation: https://files.portaudio.com/docs/v19-doxydocs-dev/portaudio_8h.html
- Ross Bencina, “Real-time audio programming 101”: https://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing
- DAFX: Digital Audio Effects, Second Edition: https://www.dafx.de/DAFX_Book_Page_2nd_edition/index.html

