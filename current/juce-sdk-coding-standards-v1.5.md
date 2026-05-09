# Modern Professional Coding Standards for JUCE-Based VST3 Plug-ins

**Version:** 1.5  
**Generated:** 9 May 2026  
**Audience:** human developers and agentic coding systems  
**Scope:** one enforceable coding standard for professional JUCE-based audio plug-ins, with VST3 as a required target format and DSP engineering rules included where they affect code correctness, real-time safety, or maintainability.

---

## 0. Normative Vocabulary

This document uses the following terms deliberately:

- **MUST** and **MUST NOT** define mandatory requirements.
- **SHOULD** and **SHOULD NOT** define strongly recommended practice. A project may depart from them only with a documented technical reason.
- **MAY** defines an allowed option.
- **PROJECT POLICY** marks a choice that each product team must make once and then apply consistently.
- **EXAMPLE** marks illustrative code or configuration. Examples must still be adapted to the pinned JUCE version, supported hosts, target platforms, and product requirements.

Code keywords, identifiers, literals, file paths, format names, compiler options, tool names, API names, and quoted external names are not localised.

---

## 1. Purpose and Non-Negotiable Principles

This standard is intended to be short enough to use during implementation and review. It is not a general DSP textbook, release handbook, or JUCE tutorial. It includes DSP engineering requirements only where the algorithm choice or implementation pattern can create host crashes, audio glitches, data loss, real-time failures, or unmaintainable plug-in code.

Non-negotiable principles:

1. **The audio callback MUST be deterministic.** `AudioProcessor::processBlock()` and every function it calls MUST NOT allocate, block, wait, log, perform file/network I/O, call GUI APIs, call host-notifying lifecycle APIs, or throw exceptions.
2. **DSP MUST be testable outside JUCE.** Put algorithms in SDK-independent classes. JUCE adapts host buffers, parameters, state, and UI to the DSP model.
3. **Parameter IDs and state formats are product contracts.** Do not rename, reorder, remove, or reuse parameter IDs without an explicit migration plan and tests.
4. **Thread safety is designed at boundaries.** Shared mutable state between audio, UI, host, and worker threads MUST use atomics, immutable snapshots, or bounded queues with documented ownership.
5. **Real-time uncertainty is a bug until proven otherwise.** If an operation has unbounded duration, may allocate, may call the OS, or may wait for another thread, it is forbidden on the audio thread.
6. **DSP changes require evidence.** SIMD, approximations, lookup tables, oversampling, branch removal, and model-based inference MUST be justified by profiling or measurable quality requirements and protected by tests.
7. **Generated or agent-written code has no lower bar.** It MUST pass the same build, test, analysis, and validation gates as human-written code for the same change scope.

---

## 2. Baseline Toolchain and Dependencies

### 2.1 Language and Compiler

Project code MUST use **C++20** or newer where the project explicitly raises the baseline.

Projects MUST define and record supported compiler/toolchain baselines for every release platform. At minimum:

- Windows: MSVC / Visual Studio version, Windows SDK version, and architecture set.
- macOS: Xcode / Apple Clang version, macOS deployment target, signing/notarisation requirements, and architecture set.
- Linux where supported: compiler, standard library, glibc or runtime baseline, distribution/container image, and architecture set.

Compiler baselines are time-sensitive. When this standard is revised or a product starts a new cycle, confirm supported compiler versions against official vendor/toolchain sources and record the checked date and rationale in dependency or build metadata.

MSVC builds SHOULD enable modern conformance switches:

```cmake
target_compile_features(MyPlugin PRIVATE cxx_std_20)

target_compile_options(MyPlugin PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive- /Zc:__cplusplus /MP>
)
```

Recommended but project-dependent:

- `/WX` in CI after third-party warnings are isolated.
- `/analyze` or C++ Core Guidelines checks in a dedicated CI job.
- Clang or `clang-cl` builds for additional diagnostics and sanitiser coverage.

Code MUST NOT rely on compiler extensions unless isolated behind documented compatibility wrappers.

### 2.2 JUCE and Third-Party Dependencies

Projects SHOULD use the latest stable, non-preview JUCE SDK that has been verified for the product's supported hosts and platforms. The JUCE dependency MUST be pinned by release tag, immutable commit, or another reproducible dependency mechanism.

Production products MUST NOT track JUCE `master`. Record the JUCE tag/commit, selected VST3 SDK source where relevant, third-party dependency versions, source hashes, and checked date.

Before starting a new project or upgrading an existing one, verify the current stable non-preview JUCE release from the official JUCE release source and record the selected version and reason for selection.

### 2.3 CMake and Build System

For JUCE projects using the current JUCE CMake API and the preset schema used in this document, require **CMake 3.25 or newer** unless the product records a different validated baseline.

```cmake
cmake_minimum_required(VERSION 3.25...3.30)
project(MyPlugin VERSION 1.0.0 LANGUAGES CXX)
```

The second version in `cmake_minimum_required(VERSION 3.25...3.30)` is the CMake policy-version maximum, not a cap on which CMake executable may run the project. Product templates SHOULD set that maximum to the newest CMake version validated in CI.

Use CMake presets for repeatable configuration:

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "windows-msvc-debug",
      "generator": "Visual Studio 17 2022",
      "architecture": "x64",
      "binaryDir": "${sourceDir}/build/windows-msvc-debug"
    }
  ]
}
```

Use `juce_add_plugin()` for plug-ins. Examples use the literal target name `MyPlugin`; replace it with the actual target. CMake snippets MUST NOT rely on undeclared placeholder variables such as `${PROJECT_TARGET}`.

```cmake
add_subdirectory(external/JUCE)

juce_add_plugin(MyPlugin
    COMPANY_NAME "Your Company"
    PLUGIN_MANUFACTURER_CODE Ycmp
    PLUGIN_CODE Yplg
    PRODUCT_NAME "MyPlugin"
    FORMATS VST3 Standalone
    IS_SYNTH FALSE
    NEEDS_MIDI_INPUT FALSE
    NEEDS_MIDI_OUTPUT FALSE
    IS_MIDI_EFFECT FALSE
    COPY_PLUGIN_AFTER_BUILD FALSE
)

target_sources(MyPlugin PRIVATE
    src/plugin/PluginProcessor.cpp
    src/plugin/PluginEditor.cpp
    src/dsp/DspCore.cpp
    src/state/StateModel.cpp
)

target_compile_features(MyPlugin PRIVATE cxx_std_20)

target_link_libraries(MyPlugin PRIVATE
    juce::juce_audio_utils
    juce::juce_dsp
)
```

`COPY_PLUGIN_AFTER_BUILD` MAY be enabled for local developer presets only. CI SHOULD package artefacts explicitly.

Modern JUCE projects SHOULD include specific module headers in each translation unit rather than depending on generated `JuceHeader.h`. Use `juce_generate_juce_header(MyPlugin)` only for legacy migration and track its removal as technical debt.

### 2.4 Warning and Configuration Definitions

Project code MUST build cleanly at high warning levels. Suppressions MUST be narrow, justified, and never used to hide undefined behaviour, lifetime, conversion, or truncation issues.

Recommended JUCE definitions:

```cmake
target_compile_definitions(MyPlugin PUBLIC
    JUCE_MODAL_LOOPS_PERMITTED=0
    JUCE_STRICT_REFCOUNTEDPOINTER=1
)
```

Use `PUBLIC` or a shared interface configuration target when a JUCE definition must be applied consistently to plug-in wrappers, helper libraries, and tests. `PRIVATE` is acceptable only when the project verifies that no other target can compile JUCE-dependent code with different configuration.

Validate all JUCE-specific definitions against the pinned JUCE version.

---

## 3. Repository Structure and Layering

Recommended layout:

```text
.
|-- CMakeLists.txt
|-- CMakePresets.json
|-- cmake/
|-- external/
|   `-- JUCE/
|-- src/
|   |-- plugin/       # JUCE AudioProcessor/AudioProcessorEditor glue
|   |-- dsp/          # SDK-independent DSP core
|   |-- parameters/   # parameter IDs, ranges, smoothing metadata
|   |-- state/        # preset/state serialisation and migration
|   |-- ui/           # components, look-and-feel, assets
|   `-- platform/     # narrow platform wrappers only
|-- tests/
|   |-- unit/
|   |-- dsp_regression/
|   |-- state_migration/
|   `-- realtime_safety/
|-- tools/
`-- docs/
```

Layering rules:

- `src/dsp` MUST compile without JUCE unless a specific exception is documented.
- `src/parameters` SHOULD contain standard C++ parameter metadata; JUCE parameter object creation belongs in plug-in glue.
- `src/plugin` owns APVTS or another approved state model, bus layout negotiation, lifecycle callbacks, host adaptation, and state entry points.
- `src/ui` owns components, attachments, look-and-feel, and assets.
- `src/platform` contains narrow wrappers only. Platform APIs MUST NOT leak into DSP hot paths.

`AudioProcessor` MUST orchestrate DSP; it MUST NOT contain large algorithms inline.

A DSP module SHOULD expose a small pure-C++ interface:

```cpp
struct PrepareSpec {
    double sampleRate = 44100.0;
    int    maxBlockSize = 512;
    int    numChannels = 2;
};

struct AudioBlockView {
    float* const* channels = nullptr;
    int           numChannels = 0;
    int           numSamples = 0;
};

struct ParameterSnapshot {
    float gain = 1.0f;
};

class DspCore final {
public:
    void prepare(const PrepareSpec& spec);
    void reset() noexcept;
    void setParameters(const ParameterSnapshot& snapshot) noexcept;
    void process(AudioBlockView block) noexcept;
};
```

The view is non-owning. The JUCE adapter owns the buffer lifetime and must pass valid pointers only for the duration of `process()`.

---

## 4. C++ Style, Formatting, and API Boundaries

### 4.1 Formatting

Use `clang-format` in CI. A compact starting configuration:

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
ColumnLimit: 110
PointerAlignment: Left
AllowShortFunctionsOnASingleLine: Empty
NamespaceIndentation: All
SortIncludes: CaseSensitive
```

Agents and developers MUST format only files they edit unless explicitly asked to reformat the project.

### 4.2 Naming

Use consistent, descriptive names. Recommended defaults:

- Types: `PascalCase` (`DspCore`, `ParameterSnapshot`).
- Functions and local variables: `camelCase` (`prepareToPlay`, `targetGain`).
- Constants: `kPascalCase` or `PascalCase`; choose one project-wide.
- Member variables: `member_` or `mMember`; choose one project-wide.
- Namespaces: short, stable, lowercase or PascalCase, but consistent.

Allow established DSP/audio abbreviations: `fft`, `ifft`, `rms`, `iir`, `fir`, `lfo`, `dc`, `midi`, `ui`, `simd`, `dB`, `LUFS`. Preserve domain-correct casing for unit-like terms (`dB`, `LUFS`, `Hz`, `kHz`).

### 4.3 Headers and Includes

- Use `#pragma once` in headers.
- Do not put `#pragma once` in `.cpp` files.
- Include the matching header first in `.cpp` files.
- Prefer forward declarations in headers when they reduce dependency weight.
- Keep JUCE includes out of DSP headers unless unavoidable.

### 4.4 Comments

Comments SHOULD explain invariants, timing constraints, ownership, and algorithm choices. Avoid stale file banners with author/date fields.

Good comments:

```cpp
// Audio-thread only. Reads atomics and preallocated buffers only.
void processBlock(juce::AudioBuffer<float>&, juce::MidiBuffer&);

// TPT one-pole smoother: stable under fast cutoff changes.
```

Bad comments:

```cpp
sample++; // increment sample
```

### 4.5 Ownership and Lifetime

- Use `std::unique_ptr` for unique ownership outside hot loops.
- Use raw pointers or references as non-owning views only when lifetime is obvious and documented.
- Avoid `std::shared_ptr` and `std::weak_ptr` on the audio path. Reference count updates are atomic operations and final release may delete on the wrong thread.
- Do not capture `this` in async callbacks unless cancellation/lifetime is guaranteed.
- Editors must tolerate processor destruction ordering and host editor recreation.
- Use `juce::Component::SafePointer` for UI callbacks targeting components, not for audio-thread communication.

---

## 5. JUCE Lifecycle and Host Integration

### 5.1 Constructor

Allowed:

- Define static parameter layout.
- Construct APVTS or another approved state model.
- Initialise small host-independent value types.
- Register immutable metadata.

Avoid:

- Heavy allocation.
- Host-dependent work.
- File I/O.
- Assumptions about sample rate, block size, channel count, or active bus layout.
- Decisions based on `wrapperType`.

### 5.2 `prepareToPlay(double sampleRate, int samplesPerBlock)`

This is the primary place to allocate sample-rate/block-size-dependent resources.

Required:

- Validate `sampleRate > 0` and `samplesPerBlock >= 0`.
- Prepare all DSP modules.
- Preallocate temporary buffers for the maximum expected block size.
- Prepare oversamplers, delay lines, filters, smoothers, FFT plans, SIMD buffers, and model inference state.
- Reset denormal-prone state.
- Compute/report latency when it changes, from this or another non-real-time configuration path.

Do not assume the host will keep the same block size forever. `samplesPerBlock` is a hint or maximum in many contexts. `processBlock()` MUST tolerate smaller, larger, and zero-length buffers. If a block exceeds the prepared maximum, use a pre-agreed safe path: split processing into chunks, bypass safely and flag a diagnostic, or resize only from a non-real-time path where the host lifecycle permits it.

### 5.3 `processBlock()`

Every floating-point `processBlock()` implementation MUST begin with denormal protection:

```cpp
void MyProcessor::processBlock(juce::AudioBuffer<float>& buffer,
                               juce::MidiBuffer& midi)
{
    juce::ScopedNoDenormals noDenormals;

    // No allocation, locks, waits, logging, GUI calls, host notifications, or exceptions.
}
```

Rules:

- Do not allocate or free memory.
- Do not lock, wait, sleep, yield, join, or call `std::atomic::wait`.
- Do not call `MessageManagerLock`, modal UI, file/network APIs, logging APIs, or OS APIs with unknown latency.
- Do not call `copyState()`, `replaceState()`, `setStateInformation()`, `getStateInformation()`, `setLatencySamples()`, `updateHostDisplay()`, or other host-notifying/non-real-time JUCE APIs.
- Do not mutate APVTS `ValueTree` state.
- Do not construct or destroy objects whose constructors/destructors allocate, lock, log, notify, or deallocate.
- Do not use `shared_ptr` ownership churn.
- Do not access UI components.
- Do not assume channel count or block size is constant.
- Handle bypass, silence, zero samples, more output channels than input channels, and MIDI/control events.

Preferred single-bus pattern:

```cpp
void MyProcessor::processBlock(juce::AudioBuffer<float>& buffer,
                               juce::MidiBuffer& midi)
{
    juce::ScopedNoDenormals noDenormals;

    const int numSamples = buffer.getNumSamples();

    handleMidiInput(midi);       // Events may be present in zero-sample blocks.
    applyMidiOutputPolicy(midi); // Clear/pass-through/output policy; non-allocating.

    if (numSamples == 0)
        return;

    const int totalInputChannels  = getTotalNumInputChannels();
    const int totalOutputChannels = getTotalNumOutputChannels();

    for (int ch = totalInputChannels; ch < totalOutputChannels; ++ch)
        buffer.clear(ch, 0, numSamples);

    updateAudioThreadParameters(numSamples);

    AudioBlockView view {
        buffer.getArrayOfWritePointers(),
        std::min(totalInputChannels, totalOutputChannels),
        numSamples
    };

    dspCore_.process(view);
}
```

Products with sidechains, multiple buses, mono-to-stereo processing, surround layouts, or instruments MUST define and test their own bus/channel mapping policy. Prefer one explicit view per bus instead of hiding bus semantics in channel counts.

### 5.4 `releaseResources()` and Destruction

`releaseResources()` may release large buffers and reset prepared flags, but destructors must still be safe. Hosts can instantiate, prepare, scan, bypass, release, and destroy in different orders.

Plug-in constructors, destructors, and scan-time paths MUST be cheap, side-effect-free at process scope, and exception-safe.

### 5.5 State Methods

`getStateInformation()` and `setStateInformation()` are not audio callbacks. They may lock or allocate if necessary, but MUST be robust against host threading and malformed data.

Rules:

- Use versioned state.
- Validate all input data.
- Never trust preset files.
- Never serialise raw pointers or platform-dependent object layouts.
- Preserve backwards compatibility.
- Do not call audio-thread-only methods.
- Treat APVTS `copyState()` and `replaceState()` as non-real-time operations. Verify their locking behaviour against the pinned JUCE version and guard calls so they cannot contend with the audio callback in product-specific flows.

### 5.6 Bus Layouts

Bus layout negotiation is part of the plug-in contract.

Rules:

- Declare initial/default buses in the constructor via `BusesProperties`.
- Add sidechain buses only if the product supports them.
- Implement `isBusesLayoutSupported()` to accept exactly the layouts the DSP and tests cover.
- Reject unequal main input/output channel counts for effects unless the product explicitly supports mono-to-stereo or upmix/downmix.
- Allow disabled buses where the format permits.
- Re-evaluate prepared resources when the layout changes; the host will call `prepareToPlay()` again.
- Test mono, stereo, mono-to-stereo, sidechain, surround, and ambisonic configurations advertised by the product.

### 5.7 `AudioPlayHead` and Transport State

Reading the host playhead is allowed in `processBlock()`, but it is fallible and host-dependent.

Rules:

- Call `getPlayHead()` at most once per `processBlock()` call; the returned pointer may be `nullptr`.
- Use `getPosition()`. Do not use deprecated `CurrentPositionInfo` in new code.
- Treat optional position fields as optional. Do not collapse missing time fields to zero for seek/loop detection.
- `PositionInfo::getIsPlaying()`, `getIsRecording()`, and `getIsLooping()` return booleans, not optional values.
- Detect transport jumps only when current and previous comparable positions are both valid.
- Do not retain pointers or references to playhead objects beyond the current block.
- Do not block, allocate, or call back into the host while reading the playhead.

EXAMPLE:

```cpp
// lastSampleTime_ is a project-owned std::optional<int64_t>.
void MyProcessor::readTransport() noexcept
{
    auto* playHead = getPlayHead();
    if (playHead == nullptr)
        return;

    auto pos = playHead->getPosition();
    if (! pos.hasValue())
        return;

    const bool isPlaying = pos->getIsPlaying();

    if (auto sampleTime = pos->getTimeInSamples()) {
        if (isPlaying && lastSampleTime_.has_value() && *sampleTime < *lastSampleTime_)
            resetTimeDependentState();

        lastSampleTime_ = *sampleTime;
    } else {
        lastSampleTime_.reset();
    }

    if (auto bpm = pos->getBpm()) {
        // relaxed: independent scalar transport display value.
        currentBpm_.store(static_cast<float>(*bpm), std::memory_order_relaxed);
    }
}
```

### 5.8 Wrapper and Format-Specific Behaviour

`AudioProcessor::wrapperType` MAY be used only to gate format-specific behaviour that cannot be expressed through JUCE abstractions, such as AAX-specific bypass routing or AU-specific MIDI quirks.

Rules:

- Do not branch on `wrapperType` to select algorithms or change user-facing behaviour.
- Read it from non-audio lifecycle or message-thread code after wrapper initialisation.
- Do not make constructor-time decisions from `wrapperType`.
- Do not branch on it inside `processBlock()` for performance-relevant paths.

### 5.9 Double-Precision Processing

Double-precision processing is PROJECT POLICY.

Default policy: do not advertise double-precision processing unless the product implements and tests it deliberately.

If the product supports double precision:

- Override `supportsDoublePrecisionProcessing()`.
- Implement the `juce::AudioBuffer<double>` `processBlock()` path.
- Allocate precision-dependent buffers in `prepareToPlay()` according to `getProcessingPrecision()`.
- Keep float and double paths behaviourally equivalent within defined tolerances.
- Include DSP regression, bypass, latency, state, and real-time safety tests for both paths.

Do not assume coefficient calculation in `double` means the audio processing path supports double precision.

---

## 6. Parameters, Automation, MIDI, and State Compatibility

### 6.1 Parameter Layout

Use the modern APVTS constructor with `ParameterLayout`. Do not use deprecated `createAndAddParameter()` patterns.

```cpp
juce::AudioProcessorValueTreeState::ParameterLayout createParameterLayout()
{
    std::vector<std::unique_ptr<juce::RangedAudioParameter>> params;

    params.push_back(std::make_unique<juce::AudioParameterFloat>(
        juce::ParameterID {"gain", 1},
        "Gain",
        juce::NormalisableRange<float> {-60.0f, 12.0f, 0.01f},
        0.0f,
        juce::AudioParameterFloatAttributes {}.withLabel("dB")));

    return {params.begin(), params.end()};
}
```

Parameter IDs are permanent. User-visible names may change; IDs should not.

### 6.2 Raw Parameter Access

For simple scalar parameters, cache the atomic pointer returned by `getRawParameterValue()` during construction or preparation, validate it immediately, then read it from the audio thread.

```cpp
std::atomic<float>* gainParam_ = nullptr;

gainParam_ = apvts_.getRawParameterValue("gain");
jassert(gainParam_ != nullptr);
if (gainParam_ == nullptr) {
    markProcessorInvalid("Missing required parameter: gain");
    return;
}

// relaxed: independent scalar parameter value.
const float gainDb = gainParam_->load(std::memory_order_relaxed);
```

Release builds MUST NOT allow a missing required parameter pointer to become an audio-thread null dereference. Do not throw across JUCE or host callbacks to signal missing parameters; convert to a safe invalid/bypassed state and a non-real-time diagnostic.

Use `memory_order_relaxed` for independent scalar parameter values. Use acquire/release only when a scalar flag publishes other related data.

### 6.3 Smoothing and Automation

Never apply abrupt continuous parameter changes directly to audio unless the parameter is explicitly designed to be stepped.

Use one of:

- `juce::SmoothedValue<float>` for simple linear or multiplicative smoothing;
- custom TPT/state-variable smoothing for filter coefficients;
- sub-block subdivision for sample-accurate MIDI/internal events;
- block-to-block smoothing for low-rate controls where acceptable.

```cpp
void prepareToPlay(double sampleRate, int samplesPerBlock) override
{
    gainSmoothed_.reset(sampleRate, 0.02); // 20 ms
    gainSmoothed_.setCurrentAndTargetValue(1.0f);
}

void updateAudioThreadParameters(int numSamples) noexcept
{
    // relaxed: independent scalar parameter value.
    const float gainDb = gainParam_->load(std::memory_order_relaxed);
    gainSmoothed_.setTargetValue(juce::Decibels::decibelsToGain(gainDb));
}
```

Discrete parameters, algorithm selectors, quality modes, and oversampling changes MUST NOT reallocate or rebuild DSP objects in the audio thread. Use pending-change mechanisms and apply them at non-real-time lifecycle points or via immutable prepared snapshots.

JUCE's standard parameter pipeline reads parameter values at block boundaries via cached atomic pointers. For most plug-ins, applying `SmoothedValue` ramps inside `processBlock()` is the default policy.

True sample-accurate automation at host-supplied parameter offsets is not exposed directly through JUCE's standard `AudioProcessor` API for VST3 `IParameterChanges`. Products requiring that level of granularity must accept block-level updates with smoothing, key sub-block subdivision off MIDI/internal events, or implement custom format-specific wrapper code below JUCE. The last option is rarely justified.

Sub-block processing rules:

- Build a sorted, bounded list of change offsets in `[0, numSamples]`.
- Ignore, clamp, or reject invalid offsets according to a documented policy.
- Cap the number of sub-blocks per block so worst-case CPU stays within budget.
- `makeView(buffer, startSample, sampleCount)` MUST advance channel pointers by `startSample` or carry the offset explicitly.

```cpp
int cursor = 0;
for (const auto offset : sortedBoundedOffsets) {
    if (offset <= cursor)
        continue;

    dspCore_.process(makeView(buffer, cursor, offset - cursor));
    cursor = offset;
    applyEventsAt(offset);
}

if (cursor < numSamples)
    dspCore_.process(makeView(buffer, cursor, numSamples - cursor));
```

### 6.4 MIDI Handling

MIDI processing is part of the audio callback and is subject to all real-time rules.

Rules:

- Iterate `juce::MidiBuffer` with modern range-for syntax.
- Honour `juce::MidiMessageMetadata::samplePosition` for instruments and effects that require sample-accurate response.
- Process MIDI/control events before a zero-sample audio early return.
- Define whether the plug-in passes MIDI through, filters it, transforms it, generates new MIDI, or consumes it.
- Effects that do not intentionally emit MIDI SHOULD clear the buffer after reading it.
- Instruments usually consume input MIDI and return an empty buffer unless MIDI output is a product feature.
- Do not copy events into containers that may allocate.
- Do not call `MidiBuffer::addEvent()` while iterating the same buffer.
- Avoid `MidiMessageMetadata::getMessage()` on audio hot paths unless the accepted message set is proven non-allocating. Decode fixed-length channel messages directly from `data`, `numBytes`, and `samplePosition`, or use a project-owned no-allocation parser.
- Bound per-block MIDI work.
- Sustain, sostenuto, all-notes-off, and all-sound-off semantics MUST be implemented per product contract.

### 6.5 State, Presets, and Migration

State MUST be versioned. For binary state, define a stable byte-level format rather than serialising C++ object memory directly:

```cpp
struct StateHeaderFields {
    uint32_t magic = 0x4D59504C; // MYPL
    uint32_t version = 1;
};
```

The struct documents fields only. Do not persist it with `reinterpret_cast`, object `memcpy`, or raw `sizeof(StateHeaderFields)` writes. Write each field explicitly with defined byte order, field width, and validation path.

For APVTS XML/ValueTree state, include a version property.

Do not change without explicit migration:

- parameter ID;
- normalisation mapping;
- default value;
- discrete step count;
- automatable flag;
- unit semantics.

When a parameter must be removed, keep a hidden/deprecated compatibility parameter if host automation or session recall depends on it.

Preset loading may allocate and parse, but not on the audio thread. If loading requires new DSP resources, prepare them off-thread and publish an immutable snapshot safely.

Malformed presets MUST NOT crash the plug-in.

---

## 7. Thread Safety and Memory Safety

### 7.1 Thread Roles

Assume the following threads may exist:

- **Audio thread:** calls `processBlock()`. Hard real-time constraints.
- **Message/UI thread:** constructs and updates editor components.
- **Host worker threads:** may call state, parameter, scanning, layout, and lifecycle methods.
- **Background worker threads:** project-owned analysis, preset scanning, file loading, licensing, model preparation, etc.

Do not assume only a UI thread and an audio thread. Hosts vary.

### 7.2 Atomics

Use atomics for simple scalars and flags.

```cpp
std::atomic<float> targetCutoffHz {1000.0f};

// UI/host/non-RT thread
// relaxed: independent scalar parameter value.
targetCutoffHz.store(newValue, std::memory_order_relaxed);

// audio thread
// relaxed: independent scalar parameter value.
const float cutoff = targetCutoffHz.load(std::memory_order_relaxed);
```

Memory order policy:

- `relaxed`: independent scalar parameters, counters, meters where exact cross-variable ordering is unnecessary.
- `release`/`acquire`: publish an immutable snapshot pointer or a group of related data.
- `seq_cst`: only when a clear correctness argument requires global ordering.

Each atomic load/store SHOULD carry a one-line comment at the use site naming the category. It is mandatory for non-relaxed orders and for audio-thread atomics whose correctness is not obvious.

Check lock freedom for hot atomics where relevant:

```cpp
static_assert(std::atomic<float>::is_always_lock_free,
              "Audio-thread float atomics must be lock-free on this target");
```

If the assertion fails on a supported target, redesign the communication path.

### 7.3 Immutable Snapshot Handoff

For complex state, prefer immutable snapshots prepared off the audio thread.

Pattern:

1. Build new state on a background or message thread.
2. Validate and fully allocate it.
3. Publish a pointer or index atomically.
4. Audio thread observes the new snapshot and switches at a safe boundary.
5. Retire old snapshots only after the audio thread can no longer access them.

Avoid deleting the previous snapshot on the audio thread. Use generation counters, deferred reclamation, or a small preallocated pool.

### 7.4 Lock-Free Queues

Use SPSC queues for one producer and one consumer only. Do not use an SPSC queue for multiple UI/background producers.

Queue payloads must be:

- trivially copyable or guaranteed non-allocating to copy/move;
- small enough for bounded throughput;
- free of owning pointers unless lifetime is externally guaranteed;
- versioned if interpretation can change.

Never block waiting for queue space on the audio thread. If the queue is full, drop, coalesce, or set an overflow flag according to documented policy.

Production queues SHOULD avoid false sharing between frequently written indices and payload metadata. Use documented cache-line padding for supported platforms.

### 7.5 ABA, Reclamation, and False Sharing

Avoid ABA hazards by:

- using generation counters with indices;
- avoiding raw pointer compare-exchange for reclaimable objects;
- using immutable buffers with deferred reclamation;
- using small object pools where objects are not reused until a safe epoch;
- keeping audio-thread communication SPSC where possible.

Frequently written atomics used by different threads MUST NOT share cache lines. Pad or separate queue indices, meters, flags, and per-channel worker-thread state.

### 7.6 Allocation Policy

- Allocate in constructors only for host-independent small objects.
- Allocate sample-rate/block-size resources in `prepareToPlay()`.
- Reserve vectors to maximum capacity before audio use.
- Never call `shrink_to_fit()` in real-time paths.
- Prefer `std::array` for fixed-size state.
- Use aligned buffers for SIMD.
- Use custom arenas only when preallocated and bounded.
- `std::pmr` is not automatically real-time-safe; it is safe only when backed by a suitable preallocated resource that cannot be exhausted on the audio thread.

### 7.7 Plug-in Binary Boundary and Static Initialisation

Plug-ins are loaded into a host process as DLLs, bundles, or shared objects. The host owns loading, scanning, instantiation, and teardown.

Rules:

- Avoid global non-trivial constructors that depend on JUCE globals being initialised.
- On Windows, `DllMain` MUST NOT perform file I/O, network I/O, COM/OLE initialisation, or anything that may acquire another loader lock.
- On macOS, bundle initialisation routines and Objective-C `+load` face equivalent constraints; defer non-trivial work to first use under a guarded path.
- Global mutable state shared across plug-in instances MUST be documented and synchronised.
- Singletons MUST be safe for concurrent first-use across instances. Prefer function-local statics over manual lazy-init.
- Do not rely on global destructors running in hosted plug-ins. Explicit teardown belongs in the plug-in destructor or `releaseResources()`.

---

## 8. DSP Engineering Standard

### 8.1 Denormals and Numerical Safety

Every floating-point processing block MUST prevent denormal slowdowns:

```cpp
juce::ScopedNoDenormals noDenormals;
```

DSP code MUST defend against:

- NaN and Inf propagation;
- division by zero;
- invalid square roots/logarithms;
- unstable filter coefficients;
- feedback greater than stable bounds;
- extreme sample rates;
- bad or unusual host-provided parameter values;
- denormal states after long silence.

Use targeted guards:

```cpp
inline float sanitiseSample(float x) noexcept
{
    return std::isfinite(x) ? x : 0.0f;
}
```

Do not scatter expensive checks in every sample of a hot loop unless necessary. Prefer per-block validation, coefficient validation, and debug-only assertions where sufficient.

### 8.2 Parameter and Coefficient Changes

- Smooth audible continuous parameters.
- Recalculate coefficients at a controlled rate.
- Interpolate only where the algorithm remains stable under interpolation.
- For filters, prefer topologies stable under modulation, such as TPT/SVF designs.
- Do not apply discontinuous algorithm changes in the middle of a block unless crossfaded.

### 8.3 Oversampling and Anti-Aliasing

Use oversampling for nonlinear processing when aliasing would be audible or measurable.

Rules:

- Prepare oversamplers during `prepareToPlay()`.
- Choose FIR vs IIR stages based on phase, latency, CPU, and product quality goals.
- Report latency from non-real-time paths only.
- Test aliasing at multiple sample rates and drive levels.
- Avoid changing oversampling factor in the audio thread; prepare separate chains or apply changes while inactive.
- Crossfade quality/oversampling mode changes when the output would otherwise click.

### 8.4 Latency and Tail

If processing introduces latency, report it accurately with `setLatencySamples()` from `prepareToPlay()`, configuration callbacks, or other non-real-time paths. Never call it from `processBlock()`.

Rules:

- Test reported latency with impulse tests.
- Implement tail reporting where relevant.
- Clear or preserve tails intentionally on bypass, reset, sample-rate changes, and transport changes.
- If latency changes by mode, document and test the transition policy.

### 8.5 Bypass

Bypass behaviour MUST be defined:

- hard bypass;
- smoothed/crossfaded bypass;
- latency-compensated bypass;
- tail-preserving bypass;
- host bypass parameter handling.

JUCE integration rules:

- If the plug-in exposes a bypass parameter, implement `getBypassParameter()` deliberately and keep it consistent with the internal bypass model.
- Implement and test `processBlockBypassed()` when bypassed processing needs latency compensation, tail preservation, metering, MIDI handling, or click-free crossfades.
- Do not assume host bypass means the audio callback will stop.
- Bypass code MUST still handle zero-sample blocks and MIDI/control events.
- If the plug-in has latency, bypass MUST NOT create timing discontinuities.

### 8.6 SIMD and Hot Loops

SIMD is encouraged only when guarded by correctness tests and profiling.

Options:

- `juce::FloatVectorOperations` for common vector operations.
- `juce::dsp::SIMDRegister` for portable SIMD-friendly code.
- Platform intrinsics only behind narrow abstraction layers.

Rules:

- Ensure alignment where required.
- Provide scalar fallback.
- Test tail handling for non-multiple vector lengths.
- Compare output against scalar reference within defined tolerances.
- Prefer simple loops in hot paths. Do not introduce ranges, virtual dispatch, heap-owning abstractions, callbacks, or type erasure into sample loops without profiling.

### 8.7 CPU Budget

For a block of `N` samples at sample rate `Fs`, the callback period is `N / Fs` seconds. A 64-sample buffer at 48 kHz gives about 1.33 ms total wall-clock time, and the plug-in gets only a fraction of that.

Standards:

- Profile Release builds, not Debug.
- Profile in real hosts and offline harnesses.
- Track worst-case, not only average CPU.
- Avoid first-use costs in the first audio block.
- Warm up lookup tables, FFT plans, oversamplers, resamplers, and model state outside the audio thread.
- Bound all per-block work, including MIDI/event handling and modulation matrices.

### 8.8 Algorithm-Specific DSP Rules

Nonlinear and virtual-analogue processing:

- Expect aliasing and test it.
- Consider oversampling, antiderivative anti-aliasing, band-limited nonlinear design, or model-specific anti-aliasing.
- Drive-dependent latency and quality modes MUST be documented.
- Validate at 44.1, 48, 96, and 192 kHz where supported.
- Clamp internal feedback and state variables to stable physical/algorithmic limits.

Filters and time-varying systems:

- Prefer modulation-friendly topologies for rapidly changing cutoff/resonance.
- Do not interpolate coefficients blindly for unstable structures.
- Validate near Nyquist, near zero frequency, at high resonance, and under fast automation.
- Use double precision for coefficient calculation when it improves stability.
- Document topology and known trade-offs.

Dynamics processing:

- Define detector type, ballistics, gain computer, knee shape, lookahead, sidechain filtering, stereo linking, and smoothing.
- Test gain reduction with impulses, stepped tones, sine bursts, and silence recovery.
- Avoid denormal-prone envelope tails.
- Report lookahead latency and preserve timing under bypass.

Delay, reverb, and feedback networks:

- Bound feedback coefficients.
- Clear or fade feedback buffers on reset and sample-rate changes.
- Smooth delay-time changes using interpolation or crossfading.
- Test fractional-delay interpolation error and modulation artefacts.
- Test tail length, denormal behaviour, silence detection, and bypass transitions.

Convolution and impulse responses:

- Load and parse impulse responses off the audio thread.
- Build FFT plans and partitions off the audio thread.
- Publish new convolution engines using immutable snapshot handoff.
- Crossfade IR changes.
- Report partitioning latency accurately.
- Test malformed, huge, zero-length, mono, stereo, and mismatched sample-rate IRs.

Spectral and time-frequency processing:

- Preallocate all FFT buffers and plans.
- Document window type, overlap, hop size, latency, and reconstruction assumptions.
- Test overlap-add reconstruction error.
- Handle transport jumps and reset state clearly.
- Avoid allocating when FFT size changes; prepare configurations in advance or change only while inactive.

Resampling:

- Use well-characterised polyphase or band-limited resamplers for quality-critical paths.
- Define passband, stopband, phase, latency, and CPU targets.
- Test with sweeps and impulses.
- Do not instantiate or resize resamplers in the audio callback.

Instruments, voices, and MIDI:

- Preallocate voices and event storage.
- No per-note heap allocation.
- Deterministic voice stealing.
- Sample-accurate event handling where the format/host provides offsets.
- Bounded modulation matrix work per block.
- Clear sustain, sostenuto, all-notes-off, and all-sound-off behaviour.
- Avoid dynamic graph rebuilding on note-on.

Spatial, surround, and immersive processing:

- Define supported layouts explicitly.
- Keep channel order conversions isolated and tested.
- Test mono, stereo, surround, ambisonic, and sidechain cases as applicable.
- Validate downmix/upmix behaviour.
- For HRTF or convolution spatial processing, apply the convolution rules above.

ML/model-based DSP:

- No Python, interpreter, JIT compilation, graph optimisation, file loading, or dynamic allocation in the audio callback.
- Warm up inference engines before audio starts.
- Preallocate tensors and scratch buffers.
- Provide deterministic CPU fallback if GPU/accelerator use is not host-safe.
- Bound model execution time and test worst-case CPU.
- Validate model files as untrusted input.
- Provide bypass/fallback if model preparation fails.

### 8.9 Determinism and Reproducibility

Bit-identical output across platforms is often unrealistic for floating-point DSP, especially with SIMD and different maths libraries. The standard is controlled, tested tolerance.

Rules:

- Define acceptable numerical tolerances per algorithm.
- Use deterministic random seeds where needed.
- Avoid unspecified iteration order in DSP-affecting code.
- Document platform-specific approximations.
- Use golden tests with tolerance and spectral criteria.

---

## 9. UI, Editor, and Asset Rules

### 9.1 Separation

The editor MUST NOT own DSP state. It observes processor state and writes parameters through APVTS attachments or approved safe parameter APIs.

UI controls MUST NOT directly write DSP mutable state.

### 9.2 Message Thread Only

JUCE components are message-thread objects. Do not access them from `processBlock()`.

Meters and visualisers SHOULD use one-way communication:

- audio thread writes atomics or SPSC queue entries;
- UI polls on a timer;
- UI drops frames if it cannot keep up.

### 9.3 Attachments

Attachment lifetimes MUST be explicit.

```cpp
class MyEditor final : public juce::AudioProcessorEditor {
    juce::Slider gainSlider_;
    std::unique_ptr<juce::AudioProcessorValueTreeState::SliderAttachment> gainAttachment_;
};
```

Construct controls before attachments. Destroy attachments before controls if declaration order requires it.

### 9.4 Assets

- Load fonts/images once, outside audio.
- Avoid blocking disk I/O when opening the editor; embed critical assets or preload asynchronously.
- Validate image scaling and HiDPI behaviour.
- Avoid unbounded repaint rates.

---

## 10. Error Handling and Diagnostics

### 10.1 Exceptions

Project policy:

- No exceptions from `processBlock()` or DSP hot paths.
- No exceptions across JUCE/host callbacks.
- No exceptions from destructors.
- Missing required resources, parameters, or state discovered during construction or lifecycle callbacks MUST be converted to a safe invalid/bypassed state and a non-real-time diagnostic.
- Catch at boundaries where third-party code may throw, then convert to safe state and diagnostics.

### 10.2 Assertions

Use assertions for invariants, not runtime error handling.

```cpp
jassert(sampleRate > 0.0);
```

Release builds MUST remain safe if assertions are disabled.

### 10.3 Logging

Logging is forbidden on the audio thread. Non-real-time logging MUST be bounded and optionally compiled out of release builds. For audio-thread diagnostics, write atomics/counters and read them from a non-real-time context.

---

## 11. Testing, Analysis, and Validation

### 11.1 Required Test Areas

DSP modules MUST have unit tests independent of JUCE host wrappers.

Test:

- parameter ranges;
- gain and unity cases;
- silence;
- impulse response;
- denormal-prone silence tails;
- NaN/Inf inputs;
- sample-rate changes;
- zero-length blocks;
- maximum and over-maximum block sizes;
- mono/stereo/multichannel layouts;
- bypass transitions;
- latency and tail reporting;
- state loading and migration;
- double-precision processing if supported.

### 11.2 Golden and Spectral Tests

Use golden-output tests for stable algorithms. Use spectral/aliasing tests for nonlinear processors. Define tolerances explicitly. Do not require bit identity unless the algorithm is designed for it.

### 11.3 State Migration Tests

Every supported state version MUST have fixtures. Tests MUST load old presets and verify parameter values, hidden compatibility fields, unknown-field handling where practical, and output behaviour.

### 11.4 Real-Time Safety Tests

At minimum:

- allocation counter around DSP process calls in a standalone harness;
- lock-detection wrappers for project mutexes;
- stress tests with random block sizes;
- first-audio-block tests after `prepareToPlay()`;
- repeated prepare/release cycles;
- queue overflow/coalescing tests;
- immutable snapshot handoff tests.

### 11.5 Sanitisers and Static Analysis

Required CI jobs where practical:

- MSVC `/analyze` or C++ Core Guidelines checker.
- MSVC AddressSanitizer on Windows for tests and standalone harnesses.
- Clang AddressSanitizer and UndefinedBehaviorSanitizer on at least one platform.
- Clang ThreadSanitizer on non-real-time harnesses for cross-thread code.
- Clang RealtimeSanitizer on standalone audio harnesses where the audio callback path can be annotated `[[clang::nonblocking]]`.
- `clang-tidy` for `modernize`, `bugprone`, `performance`, `readability`, and concurrency-relevant checks.

Sanitisers detect classes of bugs under tested executions; they do not prove real-time safety.

### 11.6 VST3 and Host Validation

JUCE-built VST3 release candidates MUST be run through:

- Steinberg VST3 validator;
- Tracktion `pluginval` at the project's release-candidate strictness level;
- representative DAW smoke tests for supported platforms where practical.

If a validator is unavailable for a platform or runner, the release waiver MUST record reason, affected platforms, risk, owner, expiry, and compensating host tests.

### 11.7 Project Commands

Every project MUST document canonical configure, build, test, analysis, validator, and packaging commands in a developer guide, CI workflow, or script entry point. Agents and developers MUST run the documented commands relevant to their change before claiming completion.

---

## 12. Release and Reproducibility Requirements

This standard is not a full release handbook, but coding-standard compliance depends on reproducible inputs and verifiable artefacts.

Release-candidate builds MUST record:

- Git commit and dirty-tree state.
- JUCE commit/tag and third-party dependency versions/source hashes.
- compiler, linker, SDK, CMake, and generator versions;
- configure/build invocation, preferably `cmake --preset <name>`;
- enabled plug-in formats and architectures;
- validator versions and results;
- skipped coverage with waivers.

Release builds MUST preserve symbols separately from shipped binaries with a documented retrieval path.

Byte-identical reproducibility is a goal where toolchains support it, not a universal requirement. The pipeline SHOULD produce binaries whose differences are fully explained by recorded inputs.

---

## 13. Agentic Coder Rules

When an agent modifies a JUCE plug-in project, it MUST follow these rules:

1. Never edit parameter IDs, ranges, defaults, or state migration casually. Flag any such change explicitly.
2. Never add audio-thread allocations, logging, locks, UI access, `shared_ptr` churn, or host notifications in `processBlock()` or DSP hot paths.
3. Preserve lifecycle boundaries. Allocate in `prepareToPlay()`, not in `processBlock()`.
4. Keep DSP testable. New algorithms go into `src/dsp` with tests.
5. Use atomics with documented memory-order reasoning when crossing threads.
6. Prefer simple loops in hot paths unless profiling justifies abstraction.
7. Add or update tests according to the touched risk area.
8. If a host or SDK callback thread is unclear, assume the stricter real-time/thread-safety requirement until verified.
9. Format touched files only unless requested.
10. Run documented build/test commands before claiming completion.

---

## 14. Code Review Checklist

### Real-Time Safety

- [ ] No allocation/deallocation in `processBlock()` or DSP hot paths.
- [ ] No locks, waits, sleeps, logging, file/network I/O, OS calls with unknown latency, or GUI calls in audio code.
- [ ] No `shared_ptr` ownership churn in audio code.
- [ ] No `setLatencySamples()`, `updateHostDisplay()`, state copy/replace, or other host-notifying/non-real-time calls from the audio thread.
- [ ] No exceptions escape callbacks.
- [ ] Denormal protection present.
- [ ] Variable and zero block sizes handled.
- [ ] Zero-sample blocks still process MIDI/control events before returning.

### Thread and Memory Safety

- [ ] Cross-thread data uses atomics, immutable snapshots, or bounded queues.
- [ ] Non-relaxed atomic memory orders have a correctness comment.
- [ ] Queue use matches producer/consumer assumptions.
- [ ] Object lifetimes are clear.
- [ ] No false-sharing hotspots in high-frequency atomics.
- [ ] Sanitiser/static-analysis coverage exists for new patterns.

### DSP Correctness

- [ ] Parameter smoothing is appropriate.
- [ ] Automation policy is documented per parameter.
- [ ] NaN/Inf/denormal behaviour is handled.
- [ ] Latency and tail are reported correctly from non-real-time paths.
- [ ] Bypass behaviour is click-safe and latency-safe.
- [ ] Oversampling/anti-aliasing choices are tested.
- [ ] SIMD has scalar fallback and tolerance tests.
- [ ] Algorithm-specific rules in §8.8 are addressed where applicable.

### JUCE Integration

- [ ] APVTS uses modern `ParameterLayout`.
- [ ] Parameter IDs and state formats remain compatible or have migration tests.
- [ ] Attachments have safe lifetimes.
- [ ] UI does not access DSP mutable state directly.
- [ ] `isBusesLayoutSupported()` accepts only tested layouts.
- [ ] `AudioPlayHead` reads handle missing fields and transport jumps safely.
- [ ] MIDI iteration uses range-for and honours `samplePosition`.
- [ ] MIDI input/output policy is explicit.
- [ ] MIDI hot paths avoid allocating message construction.
- [ ] `wrapperType` branches are limited to genuine format constraints.
- [ ] Double precision is either unsupported by policy or fully implemented and tested.

### Build, Test, and Release

- [ ] CMake minimum satisfies current JUCE requirement and selected preset schema.
- [ ] Toolchain and dependency versions are recorded.
- [ ] Warnings clean.
- [ ] Relevant unit, DSP, state, migration, and real-time safety tests pass.
- [ ] Static analysis and sanitisers pass where applicable.
- [ ] Steinberg VST3 validator passes for release candidates.
- [ ] Tracktion `pluginval` passes at required strictness.
- [ ] Representative host smoke tests are recorded for release candidates.
- [ ] Reproducibility inputs and symbols are archived for release builds.
