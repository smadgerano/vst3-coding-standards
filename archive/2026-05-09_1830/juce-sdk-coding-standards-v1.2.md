# Modern Professional Coding Standards for JUCE-Based VST3 Plug-ins

**Version:** 1.2  
**Generated:** 9 May 2026  
**Audience:** human developers and agentic coding systems  
**Scope:** reusable standards for professional JUCE-based audio plug-ins, with VST3 as a required target format

---

## Changes since 1.1

- §2.1: replaced the undeclared `${PROJECT_TARGET}` placeholder with a literal target name and an explicit convention note.
- §2.4: removed the redundant `PLUGIN_NAME`, replaced unconditional `juce_generate_juce_header()` with current per-module include guidance.
- §2.5: added a rationale for `JUCE_STRICT_REFCOUNTEDPOINTER=1`.
- §3.4: defined `AudioBlockView` explicitly so it is no longer an undocumented project type.
- §5.3: replaced the per-channel `processChannel` example with a multi-channel-view example so stereo-coupled DSP fits naturally.
- §5.6 (new): bus layout negotiation and `isBusesLayoutSupported`.
- §5.7 (new): `AudioPlayHead` and transport state on the audio thread.
- §5.8 (new): MIDI handling and sample-accurate event iteration.
- §5.9 (new): host detection via `wrapperType`.
- §6.3: expanded sample-accurate automation policy with sub-block subdivision guidance.
- §7.2: added `setLatencySamples()` to forbidden audio-thread operations; clarified `std::function` SBO behaviour.
- §7.10: added Clang RealtimeSanitizer (RTSan).
- §7.11 (new): plug-in DLL/bundle boundary and static initialisation rules.
- §12.5: added Tracktion `pluginval` alongside the Steinberg validator.
- §13: expanded reproducible-build mechanics (`/Brepro`, `SOURCE_DATE_EPOCH`, debug-prefix normalisation, symbol archival).
- §15: strengthened the atomic-memory-order checklist item to require a one-line cited justification at the use site.

---

## 0. Normative vocabulary

This document uses the following terms deliberately:

- **MUST** and **MUST NOT** define mandatory requirements.
- **SHOULD** and **SHOULD NOT** define strongly recommended practice. A project may depart from them only with a documented technical reason.
- **MAY** defines an allowed option.
- **PROJECT POLICY** marks a choice that each product team must make once and then apply consistently.
- **EXAMPLE** marks illustrative code or configuration. Examples must still be adapted to the pinned JUCE version, supported hosts, target platforms, and product requirements.

Code keywords, identifiers, literals, file paths, format names, compiler options, tool names, API names, and quoted external names are not localised.

---

## 1. Non-negotiable principles

These rules override convenience, style preference, and agent-generated shortcuts.

1. **The audio callback MUST be deterministic.** `AudioProcessor::processBlock()` and functions it calls MUST NOT allocate, block, wait, log, perform file/network I/O, call GUI APIs, or throw exceptions.
2. **DSP MUST be testable outside JUCE.** Put algorithms in SDK-independent classes. JUCE should adapt host buffers, parameters, state, and UI to a core DSP model.
3. **Parameter IDs and state formats are product contracts.** Do not rename, reorder, remove, or reuse parameter IDs without an explicit migration plan and tests.
4. **Thread safety is designed, not patched.** Shared mutable state between audio, UI, host, and worker threads MUST use atomics, immutable snapshots, or bounded lock-free queues with clearly documented ownership.
5. **No "probably fine" real-time code.** If an operation has unbounded duration, can call the OS, can allocate, or can wait for another thread, it is forbidden on the audio thread.
6. **Measure before optimising.** SIMD, approximations, lookup tables, oversampling, and branch removal MUST be justified by profiling and protected by tests.
7. **Generated or agent-written code MUST pass the same gates as human code.** Build, tests, static analysis, sanitiser runs, and VST3 validation are not optional.

---

## 2. Baseline toolchain and dependencies

### 2.1 Language and compiler

- Project code MUST use **C++20** or newer where the project explicitly raises the baseline.
- Windows builds MUST support **MSVC 19.44+ / Visual Studio 2022** as the required Windows compiler baseline.
- MSVC builds SHOULD enable modern conformance switches. The literal target name is shown below; substitute the actual `juce_add_plugin` target.

```cmake
target_compile_features(MyPlugin PRIVATE cxx_std_20)

target_compile_options(MyPlugin PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive- /Zc:__cplusplus /MP>
)
```

> **Convention.** Examples in this document use the literal target name `MyPlugin`. In a real project, replace it with the target produced by `juce_add_plugin()`. CMake snippets MUST NOT reference an undeclared variable such as `${PROJECT_TARGET}`; that form would silently expand to an empty string and apply the options to no target.

Recommended but project-dependent:

- `/WX` in CI after third-party warnings are isolated.
- `/analyze` or C++ Core Guidelines checks in a dedicated CI job.
- `clang-cl` or Clang builds for additional diagnostics and sanitiser coverage.

Code MUST NOT rely on compiler extensions unless they are isolated behind documented compatibility wrappers.

### 2.2 JUCE version

Projects SHOULD use the latest stable, non-preview JUCE SDK that has been verified for the product's supported hosts and platforms. The JUCE dependency MUST be pinned by release tag, immutable commit, or another reproducible dependency mechanism.

This document intentionally does not hard-code a "latest JUCE" version because that claim becomes stale. Before starting a new project or upgrading an existing one, verify the current stable non-preview JUCE release from the official JUCE release source and record the checked date, selected version, and reason for selection in dependency metadata.

Production products MUST NOT track JUCE `master`. Pin a release tag or commit and record it in dependency metadata.

### 2.3 CMake version

For JUCE projects using the current JUCE CMake API and the CMake Presets schema used in this document, require **CMake 3.25 or newer**.

Recommended:

```cmake
cmake_minimum_required(VERSION 3.25...3.30)
project(MyPlugin VERSION 1.0.0 LANGUAGES CXX)
```

Older CMake versions may be suitable for legacy non-JUCE tooling or projects using older preset schemas, but they are not the baseline for this standard.

### 2.4 Build system standards

Use CMake presets:

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "windows-msvc-debug",
      "generator": "Visual Studio 17 2022",
      "architecture": "x64",
      "binaryDir": "${sourceDir}/build/windows-msvc-debug",
      "cacheVariables": {
        "CMAKE_CONFIGURATION_TYPES": "Debug;Release;RelWithDebInfo"
      }
    }
  ]
}
```

Use `juce_add_plugin()` for plug-ins. Example skeleton:

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

Notes:

- `PLUGIN_NAME` defaults to `PRODUCT_NAME` and SHOULD be omitted when they are identical. Set it explicitly only when the host-visible plug-in name must differ from the project's product name.
- `COPY_PLUGIN_AFTER_BUILD` MAY be enabled for local developer presets only. CI SHOULD package artefacts explicitly.
- This skeleton deliberately omits `juce_generate_juce_header()`. Modern JUCE practice is to include the specific module headers needed in each translation unit (for example `#include <juce_dsp/juce_dsp.h>`). The aggregated `JuceHeader.h` pulls every linked module into every TU, slows incremental builds, and obscures dependencies. Use `juce_generate_juce_header(MyPlugin)` only when an existing codebase depends on the legacy `JuceHeader.h` include and migration has not yet been completed; track the removal in technical debt.

### 2.5 Warning and analysis policy

Project code MUST build cleanly at high warning levels. Suppressions MUST be narrow, justified, and never used to hide real UB, lifetime, conversion, or truncation issues.

Recommended MSVC warnings and definitions:

```cmake
target_compile_options(MyPlugin PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive- /Zc:__cplusplus /MP>
)

target_compile_definitions(MyPlugin PRIVATE
    JUCE_MODAL_LOOPS_PERMITTED=0
    JUCE_STRICT_REFCOUNTEDPOINTER=1
)
```

Rationale:

- `JUCE_MODAL_LOOPS_PERMITTED=0` is a guardrail against UI patterns that can cause re-entrancy surprises (modal loops can pump the message thread inside callbacks).
- `JUCE_STRICT_REFCOUNTEDPOINTER=1` forces explicit conversions on `juce::ReferenceCountedObjectPtr` (no implicit raw-pointer construction or comparison). It catches a class of ownership and lifetime mistakes at compile time, at the cost of slightly more verbose call sites. Once enabled, it MUST stay enabled; turning it off later hides bugs that were previously caught.

Validate any JUCE-specific definitions against the pinned JUCE version.

---

## 3. Repository and architecture

### 3.1 Recommended directory layout

```text
.
|-- CMakeLists.txt
|-- CMakePresets.json
|-- cmake/
|-- external/
|   `-- JUCE/                    # pinned release, submodule, or fetched dependency
|-- src/
|   |-- plugin/                  # JUCE AudioProcessor/AudioProcessorEditor glue
|   |-- dsp/                     # SDK-independent DSP core
|   |-- parameters/              # parameter IDs, ranges, smoothing metadata
|   |-- state/                   # preset/state serialisation and migration
|   |-- ui/                      # components, look-and-feel, assets
|   `-- platform/                # narrow platform wrappers only
|-- tests/
|   |-- unit/
|   |-- dsp_regression/
|   |-- state_migration/
|   `-- realtime_safety/
|-- tools/
`-- docs/
```

### 3.2 Layering rule

The core DSP MUST NOT include JUCE headers except where explicitly justified. A good default is:

- `src/dsp`: standard C++20 only, plus small project utilities.
- `src/parameters`: standard C++20 data models; JUCE-specific parameter creation lives in plug-in glue.
- `src/plugin`: JUCE `AudioProcessor`, APVTS, bus layouts, host callbacks.
- `src/ui`: JUCE components and editor code.

This separation allows fast unit tests, alternate SDK wrappers, offline test rendering, and safer agent edits.

### 3.3 Processor responsibilities

`AudioProcessor` owns:

- APVTS or another approved parameter/state model;
- `prepareToPlay()` allocation and configuration;
- `processBlock()` adaptation from JUCE buffers to DSP spans;
- state save/load entry points;
- latency/tail/bypass reporting;
- bus layout negotiation;
- MIDI/event routing where applicable.

`AudioProcessor` MUST NOT contain large DSP algorithms inline. It orchestrates the DSP core.

### 3.4 DSP core responsibilities

A DSP module SHOULD expose a small interface. `AudioBlockView`, `PrepareSpec`, and `ParameterSnapshot` are project-defined value types that live in `src/dsp` (or a shared header) so the DSP core compiles without JUCE.

```cpp
// Project-defined value types in src/dsp; pure C++, no JUCE.
struct PrepareSpec {
    double sampleRate = 44100.0;
    int    maxBlockSize = 512;
    int    numChannels = 2;
};

struct AudioBlockView {
    float* const* channels = nullptr;  // non-owning array of channel pointers
    int           numChannels = 0;
    int           numSamples = 0;
};

struct ParameterSnapshot {
    float gain = 1.0f;
    // ... other parameter-derived values
};

class DspCore final {
public:
    void prepare(const PrepareSpec& spec);
    void reset() noexcept;
    void setParameters(const ParameterSnapshot& snapshot) noexcept;
    void process(AudioBlockView block) noexcept;
};
```

Rules:

- `AudioBlockView` is a non-owning view; the caller (the JUCE `AudioProcessor` glue) owns the underlying buffers and guarantees they remain valid for the duration of `process()`.
- The pointer-of-pointers form is intentional: it matches the layout returned by `juce::AudioBuffer<float>::getArrayOfWritePointers()` so adaptation is zero-copy.
- Projects MAY substitute `std::span<float* const>` paired with a sample count, or any equivalent multi-channel view, provided the type is defined in the DSP layer and not pulled from JUCE.

The DSP core SHOULD be deterministic, no-throw after `prepare()`, and directly unit-testable without a plug-in host.

---

## 4. Formatting, naming, and style

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

Agents MUST format only files they edit unless explicitly asked to reformat the project.

### 4.2 Naming

Use consistent, descriptive names. Recommended defaults:

- Types: `PascalCase` (`DspCore`, `ParameterSnapshot`).
- Functions and local variables: `camelCase` (`prepareToPlay`, `targetGain`).
- Constants: `kPascalCase` or `PascalCase`; choose one project-wide.
- Member variables: `member_` or `mMember`; choose one project-wide.
- Namespaces: short, stable, lowercase or PascalCase, but consistent.

Allow established DSP/audio abbreviations: `fft`, `ifft`, `rms`, `iir`, `fir`, `lfo`, `dc`, `midi`, `ui`, `simd`, `dB`, `LUFS`. Preserve domain-correct casing for unit-like terms (`dB`, `LUFS`, `Hz`, `kHz`).

### 4.3 Headers and includes

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
void processBlock(juce::AudioBuffer<float>&, juce::MidiBuffer&) noexcept;

// We use a TPT one-pole smoother so coefficient changes remain stable at high cutoff values.
```

Bad comments:

```cpp
sample++; // increment sample
```

---

## 5. JUCE lifecycle rules

### 5.1 Constructor

Allowed:

- Define static parameter layout.
- Construct APVTS.
- Initialise small value types.
- Register immutable metadata.

Avoid:

- Heavy allocation.
- Host-dependent work.
- File I/O.
- Assumptions about sample rate, block size, channel count, or active bus layout.

### 5.2 `prepareToPlay(double sampleRate, int samplesPerBlock)`

This is the primary place to allocate sample-rate/block-size-dependent resources.

Required:

- Validate `sampleRate > 0` and `samplesPerBlock >= 0`.
- Prepare all DSP modules.
- Preallocate temporary buffers for the maximum block size expected.
- Prepare oversamplers, delay lines, filters, smoothers, FFT plans, and SIMD buffers.
- Reset denormal-prone state.
- Compute/report latency when it changes (see §8.5; latency reporting MUST happen here or in another non-real-time configuration call, never in `processBlock`).

Do not assume the host will keep the same block size forever. `samplesPerBlock` is a hint/maximum in many contexts. `processBlock()` MUST tolerate smaller, larger, and zero-length buffers. If a block exceeds the prepared maximum, use a pre-agreed safe path: split processing into chunks, resize only outside the audio thread if the host allows, or bypass safely and flag a diagnostic.

### 5.3 `processBlock()`

Required at the top of every floating-point `processBlock()` implementation:

```cpp
void MyProcessor::processBlock(juce::AudioBuffer<float>& buffer,
                               juce::MidiBuffer& midiMessages)
{
    juce::ScopedNoDenormals noDenormals;

    // No allocation, no locks, no waits, no logging, no GUI calls, no exceptions.
    // ...
}
```

Rules:

- Do not allocate or free memory.
- Do not call `MessageManagerLock`, `callAsync` patterns that may allocate, modal UI, file/network APIs, logging APIs, or OS APIs with unknown latency.
- Do not mutate APVTS `ValueTree` state.
- Do not call `copyState()` or `replaceState()`.
- Do not call `setLatencySamples()`, `updateHostDisplay()`, or any other host-notifying lifecycle call from the audio thread.
- Do not construct or destroy objects whose constructors/destructors allocate, lock, log, or notify.
- Do not use `shared_ptr` ownership churn.
- Do not access UI components.
- Do not assume channel count or block size is constant.
- Handle bypass, silence, zero samples, and more output channels than input channels.

A multi-channel pattern that hands a view to the DSP core (preferred over per-channel loops, because it lets the DSP core implement stereo-coupled, mid/side, sidechain-aware, or surround processing without the glue layer enforcing a channel-independent shape):

```cpp
void MyProcessor::processBlock(juce::AudioBuffer<float>& buffer,
                               juce::MidiBuffer& midi)
{
    juce::ScopedNoDenormals noDenormals;

    const int numSamples = buffer.getNumSamples();
    if (numSamples == 0)
        return;

    const int totalInputChannels  = getTotalNumInputChannels();
    const int totalOutputChannels = getTotalNumOutputChannels();

    // Clear any output-only channels to avoid garbage propagation.
    for (int ch = totalInputChannels; ch < totalOutputChannels; ++ch)
        buffer.clear(ch, 0, numSamples);

    updateAudioThreadParameters(numSamples); // atomics/smoothing only

    handleMidi(midi); // see §5.8 for MIDI iteration rules

    AudioBlockView view {
        buffer.getArrayOfWritePointers(),
        std::min(totalInputChannels, totalOutputChannels),
        numSamples
    };
    dspCore_.process(view);
}
```

Real products with sidechains, multiple buses, mono-to-stereo processing, surround layouts, or instruments MUST define and test their own bus/channel mapping policy. The view in the example above carries the active-channel count (the smaller of input/output for an effect); plug-ins with a sidechain bus or an unequal input/output topology will need a richer view, typically one view per bus.

### 5.4 `releaseResources()`

Allowed:

- Release large buffers if the processor is no longer active.
- Reset prepared flags.

Do not assume `releaseResources()` is the only teardown path. Destructors must still be safe.

### 5.5 State methods

`getStateInformation()` and `setStateInformation()` are not audio callbacks. They may lock or allocate if necessary, but MUST be robust against host threading and malformed data.

Rules:

- Use versioned state.
- Validate all input data.
- Never trust preset files.
- Never deserialise raw pointers or platform-dependent layouts.
- Preserve backwards compatibility.
- Do not call audio-thread-only methods.
- Treat APVTS `copyState()` and `replaceState()` as non-real-time operations. They MUST NOT be used on the audio thread. Verify the threading and locking behaviour against the pinned JUCE version and guard calls so they cannot contend with the audio callback in product-specific state flows.

### 5.6 Bus layouts and `isBusesLayoutSupported`

Bus layout negotiation is part of the plug-in's contract with the host. Hosts call `isBusesLayoutSupported` while probing channel configurations and call it again whenever the user changes routing.

Rules:

- Declare initial/default buses in the constructor via `BusesProperties`. Add a sidechain bus only if the product supports one.
- Implement `isBusesLayoutSupported` to accept exactly the layouts the DSP and tests cover. Do not return `true` indiscriminately.
- Reject unequal main input and output channel counts for effects unless the plug-in is explicitly designed for mono-to-stereo or upmix/downmix.
- Allow disabled buses where the format permits (input or output `disabled()`); do not treat a disabled bus as an error condition.
- Re-evaluate prepared resources when the layout changes; the host will call `prepareToPlay` again.
- Test mono, stereo, mono-to-stereo, sidechain (where supported), and any surround/ambisonic configurations advertised by the product.

EXAMPLE for a stereo-or-mono effect with optional sidechain:

```cpp
MyProcessor::MyProcessor()
    : AudioProcessor(BusesProperties()
        .withInput  ("Input",     juce::AudioChannelSet::stereo(), true)
        .withOutput ("Output",    juce::AudioChannelSet::stereo(), true)
        .withInput  ("Sidechain", juce::AudioChannelSet::stereo(), false))
{ /* ... */ }

bool MyProcessor::isBusesLayoutSupported(const BusesLayout& layouts) const
{
    const auto& mainOut = layouts.getMainOutputChannelSet();
    const auto& mainIn  = layouts.getMainInputChannelSet();

    if (mainOut != juce::AudioChannelSet::mono()
        && mainOut != juce::AudioChannelSet::stereo())
        return false;

    if (mainIn != mainOut)
        return false;

    // Sidechain (optional): allow disabled, mono, or stereo.
    if (layouts.inputBuses.size() > 1) {
        const auto& sc = layouts.getChannelSet(true, 1);
        if (! sc.isDisabled()
            && sc != juce::AudioChannelSet::mono()
            && sc != juce::AudioChannelSet::stereo())
            return false;
    }

    return true;
}
```

### 5.7 `AudioPlayHead` and transport state

Reading the host's playhead is allowed on the audio thread, but the API is fallible and host-dependent.

Rules:

- Call `getPlayHead()` once per `processBlock` call; the returned pointer may be `nullptr` (offline render, certain hosts).
- Use the modern `getPosition()` API, which returns `std::optional<PositionInfo>`. The deprecated `CurrentPositionInfo` struct MUST NOT be used in new code.
- Treat every field as optional. `getBpm`, `getTimeSignature`, `getPpqPosition`, `getTimeInSamples`, and friends each return `std::optional<...>`. Provide safe fallbacks rather than asserting presence.
- Detect transport jumps and looping: a non-monotonic change in `getTimeInSamples` or `getPpqPosition` between blocks indicates a seek/loop and any time-dependent DSP state (LFO phase, tempo-synced delay reads, granular schedulers) MUST be reset or smoothly bridged.
- Do not retain pointers or references to `PositionInfo` or any sub-object beyond the current block; copy out the values you need.
- Do not block, allocate, or call back into the host while reading the playhead.

EXAMPLE:

```cpp
void MyProcessor::readTransport() noexcept
{
    if (auto* ph = getPlayHead()) {
        if (auto pos = ph->getPosition()) {
            const double bpm        = pos->getBpm().orFallback(120.0);
            const bool   isPlaying  = pos->getIsPlaying();
            const auto   sampleTime = pos->getTimeInSamples().orFallback(0);

            const auto previous = lastSampleTime_;
            lastSampleTime_ = sampleTime;

            if (isPlaying && sampleTime < previous) {
                // Seek/loop: bridge or reset time-dependent state.
                resetTimeDependentState();
            }

            currentBpm_.store(static_cast<float>(bpm), std::memory_order_relaxed);
        }
    }
}
```

### 5.8 MIDI handling on the audio thread

MIDI processing is part of the audio callback and is subject to all real-time rules.

Rules:

- Iterate `juce::MidiBuffer` with the modern range-for syntax. The legacy `juce::MidiBuffer::Iterator` is deprecated and MUST NOT be used in new code.
- `juce::MidiMessageMetadata::samplePosition` is the sample-accurate offset within the current block. Honour it for instruments and for any effect whose response should be sample-accurate.
- Do not copy events into containers that may allocate. If event reordering or filtering is needed, use a preallocated, fixed-capacity buffer.
- Do not call `MidiBuffer::addEvent` while iterating the same buffer; collect output into a separate preallocated buffer and swap or merge after iteration.
- Bound per-block MIDI work. Hosts can deliver dense bursts (CC sweeps, all-notes-off cascades); the worst case MUST fit the CPU budget.
- Sustain, sostenuto, all-notes-off, and all-sound-off semantics MUST be implemented per the product's instrument/effect contract.
- For instruments: dispatch note-on/off via a preallocated voice pool. Dynamic graph rebuilding on note-on is forbidden.
- For MPE/MIDI 2.0 features advertised by the product, validate the host actually supports the feature; fall back gracefully.

EXAMPLE:

```cpp
void MyProcessor::handleMidi(const juce::MidiBuffer& midi) noexcept
{
    for (const auto meta : midi) {
        const auto& msg    = meta.getMessage();
        const int   offset = meta.samplePosition;

        if (msg.isNoteOn())
            voicePool_.startNote(msg.getNoteNumber(), msg.getFloatVelocity(), offset);
        else if (msg.isNoteOff())
            voicePool_.stopNote(msg.getNoteNumber(), offset);
        else if (msg.isAllNotesOff() || msg.isAllSoundOff())
            voicePool_.releaseAll(offset);
        // ...
    }
}
```

### 5.9 Host detection via `wrapperType`

`AudioProcessor::wrapperType` identifies the wrapper format the plug-in is currently running under (`wrapperType_VST3`, `wrapperType_AudioUnit`, `wrapperType_AAX`, `wrapperType_Standalone`, etc.).

Rules:

- Use `wrapperType` only to gate format-specific behaviour that cannot be expressed through the JUCE abstractions: AAX-specific bypass routing, AU-specific MIDI handling quirks, standalone-only file-system access, etc.
- Do not branch on `wrapperType` to select algorithms or to differentiate user-facing behaviour. Plug-ins MUST behave consistently across formats unless the format itself imposes a constraint.
- Read it on the message thread or during construction. Do not branch on it inside `processBlock` for performance-relevant code paths.

---

## 6. Parameters and automation in JUCE

### 6.1 Parameter layout

Use the modern APVTS constructor with `ParameterLayout`. Do not use deprecated `createAndAddParameter()` patterns.

```cpp
juce::AudioProcessorValueTreeState::ParameterLayout createParameterLayout()
{
    std::vector<std::unique_ptr<juce::RangedAudioParameter>> params;

    params.push_back(std::make_unique<juce::AudioParameterFloat>(
        juce::ParameterID{"gain", 1},
        "Gain",
        juce::NormalisableRange<float>{-60.0f, 12.0f, 0.01f},
        0.0f,
        juce::AudioParameterFloatAttributes{}.withLabel("dB")));

    return {params.begin(), params.end()};
}
```

Parameter IDs are permanent. Names shown to users may change; IDs should not.

### 6.2 Raw parameter access

For simple scalar parameters, cache the atomic pointer returned by `getRawParameterValue()` during construction or preparation, validate it immediately, then read it from the audio thread.

```cpp
std::atomic<float>* gainParam_ = nullptr;

gainParam_ = apvts_.getRawParameterValue("gain");
jassert(gainParam_ != nullptr);
if (gainParam_ == nullptr)
    throw std::logic_error{"Missing required parameter: gain"};

const float gainDb = gainParam_->load(std::memory_order_relaxed);
```

The construction-time failure policy is PROJECT POLICY. Release builds MUST NOT allow a missing required parameter pointer to become an audio-thread null dereference.

Use `memory_order_relaxed` for independent scalar parameter values. Use acquire/release only when a scalar flag publishes other related data.

### 6.3 Smoothing and sample-accurate automation

Never apply abrupt continuous parameter changes directly to audio unless the parameter is explicitly designed to be stepped.

Use one of:

- `juce::SmoothedValue<float>` for simple linear or multiplicative smoothing;
- custom TPT/state-variable smoothing for filter coefficients;
- sub-block subdivision for sample-accurate automation (see below);
- block-to-block smoothing for low-rate controls where acceptable.

Example (block-level smoothing — the default and sufficient for most plug-ins):

```cpp
void prepareToPlay(double sampleRate, int samplesPerBlock) override
{
    gainSmoothed_.reset(sampleRate, 0.02); // 20 ms
    gainSmoothed_.setCurrentAndTargetValue(1.0f);
}

void updateAudioThreadParameters(int numSamples) noexcept
{
    const float gainDb = gainParam_->load(std::memory_order_relaxed);
    gainSmoothed_.setTargetValue(juce::Decibels::decibelsToGain(gainDb));
}
```

Discrete parameters, algorithm selectors, quality modes, and oversampling changes MUST NOT reallocate or rebuild DSP objects in the audio thread. Use safe pending-change mechanisms and apply them at non-real-time lifecycle points or via immutable prepared snapshots.

#### Sample-accurate automation policy

JUCE's standard parameter pipeline reads parameter values at block boundaries via cached atomic pointers. For most plug-ins, applying `SmoothedValue` ramps inside `processBlock` is audibly indistinguishable from per-sample updates and is the default policy.

True sample-accurate automation — where the parameter changes at exact host-supplied sample offsets within the block — requires sub-block processing:

1. Build a sorted, bounded list of change offsets within the block. Sources include MIDI events (CC, pitch bend, MPE per-note expression) iterated from the `MidiBuffer`, or product-specific scheduling.
2. Process audio in sub-blocks bounded by those offsets.
3. Update the relevant parameter(s) at each sub-block boundary, then continue.

```cpp
// Sketch: sub-block processing keyed off MIDI offsets.
int cursor = 0;
for (const auto meta : midi) {
    const int offset = meta.samplePosition;
    if (offset > cursor) {
        dspCore_.process(makeView(buffer, cursor, offset - cursor));
        cursor = offset;
    }
    applyMidiEvent(meta.getMessage()); // updates the snapshot atomically
}
if (cursor < numSamples)
    dspCore_.process(makeView(buffer, cursor, numSamples - cursor));
```

Rules:

- Sub-block paths MUST be bounded. Cap the number of sub-blocks per block (for example by coalescing changes that occur within N samples of each other) so worst-case CPU stays within budget.
- Document the chosen policy per parameter: block-level smoothed, sub-block, or stepped.
- JUCE does not expose VST3's per-sample `IParameterChanges` queue directly through the standard `AudioProcessor` API. Plug-ins requiring that level of granularity must either accept block-level updates with smoothing, key sub-block subdivision off MIDI/internal events, or implement custom format-specific wrapper code below the JUCE abstraction. The third option is rarely justified.

### 6.4 Attachments and UI

Use APVTS attachments for standard controls. Attachment lifetimes MUST be shorter than both the control and APVTS. Prefer member declaration order that destroys attachments before controls if needed.

UI controls MUST NOT directly write DSP state. They write parameters; the processor consumes parameter values safely.

---

## 7. Dedicated thread-safety and memory-safety standard

### 7.1 Thread roles

Assume the following threads may exist:

- **Audio thread:** calls `processBlock()`. Hard real-time constraints.
- **Message/UI thread:** constructs and updates editor components.
- **Host worker threads:** may call state, parameter, scanning, layout, and lifecycle methods.
- **Background worker threads:** project-owned analysis, preset scanning, file loading, licensing, ML/model preparation, etc.

Do not assume only "UI thread" and "audio thread". Hosts vary.

### 7.2 Audio-thread forbidden operations

The audio thread MUST NOT:

- allocate or deallocate heap memory;
- grow `std::vector`, `std::string`, `juce::Array`, `juce::String`, `std::function`, or any container;
- assign a non-empty target to a `std::function` whose capture exceeds the implementation's small-buffer optimisation. The SBO threshold is implementation-specific (commonly two or three pointers, but not guaranteed); treat any non-trivial capture as a potential heap allocation. If a callable is needed in audio-thread code, use a stateless function pointer, a small `inplace_function`-style type with documented capacity, or a hand-rolled tagged union;
- lock or unlock mutexes;
- call `std::atomic::wait`, condition variables, semaphores, futures, joins, sleeps, or yields;
- perform file, console, network, registry, Objective-C, COM, Win32, CoreFoundation, or OS calls with unknown latency;
- log;
- interact with UI components;
- call `MessageManagerLock`;
- call `setLatencySamples()`, `updateHostDisplay()`, `setStateInformation()`, `getStateInformation()`, or any other host-notifying or non-real-time JUCE API. `setLatencySamples()` in particular triggers host plug-in delay compensation recomputation and may cause the host to flush or rebuild its plug-in graph;
- throw exceptions or allow exceptions to escape;
- create/destroy objects with non-trivial destructors that may allocate, lock, notify, or deallocate;
- perform unbounded loops dependent on user files, presets, or host state.

### 7.3 Atomics

Use atomics for simple scalars and flags.

```cpp
std::atomic<float> targetCutoffHz {1000.0f};

// UI/host/non-RT thread
targetCutoffHz.store(newValue, std::memory_order_relaxed);

// audio thread
const float cutoff = targetCutoffHz.load(std::memory_order_relaxed);
```

Memory order policy:

- `relaxed`: independent scalar parameters, counters, meters where exact cross-variable ordering is unnecessary.
- `release`/`acquire`: publish an immutable snapshot pointer or a group of related data.
- `seq_cst`: only when a clear correctness argument requires global ordering.

Each atomic load/store SHOULD carry a one-line comment at the use site naming which of those three categories applies. This makes review tractable and prevents drift when code is refactored.

Check lock freedom for hot atomics if using unusual types:

```cpp
static_assert(std::atomic<float>::is_always_lock_free,
              "Audio-thread float atomics must be lock-free on this target");
```

If the assertion fails on a supported target, redesign the communication path.

### 7.4 Immutable snapshot handoff

For complex state, prefer immutable snapshots prepared off the audio thread.

Pattern:

1. Build new state on a background or message thread.
2. Validate and fully allocate it.
3. Publish a pointer/index atomically.
4. Audio thread observes the new snapshot and switches at a safe boundary.
5. Retire old snapshots only after the audio thread can no longer access them.

Avoid deleting the previous snapshot on the audio thread. Use generation counters, deferred reclamation, or a small preallocated pool.

### 7.5 Lock-free queues

Use SPSC queues for one producer and one consumer only. Do not use an SPSC queue for multiple UI/background producers.

Queue payloads must be:

- trivially copyable or guaranteed non-allocating to copy/move;
- small enough for bounded throughput;
- free of owning pointers unless lifetime is externally guaranteed;
- versioned if interpretation can change.

Production queues SHOULD avoid false sharing between frequently written indices and payload metadata. Use a documented cache-line padding strategy for the supported platforms. `alignas(64)` is a common baseline, but projects SHOULD prefer `std::hardware_destructive_interference_size` where available and validated, or a platform abstraction where a larger value is required.

```cpp
struct alignas(64) AtomicIndex {
    std::atomic<size_t> value {0};
};
```

Never block waiting for queue space on the audio thread. If the queue is full, drop, coalesce, or set an overflow flag according to the documented policy.

### 7.6 ABA and reclamation

ABA hazards occur when a pointer/index changes from A to B and back to A, and a consumer mistakes the final A for the original A. Avoid ABA by:

- using generation counters with indices;
- avoiding raw pointer compare-exchange for reclaimable objects;
- using immutable buffers with deferred reclamation;
- using small object pools where objects are not reused until a safe epoch;
- keeping audio-thread communication SPSC where possible.

### 7.7 False sharing

Frequently written atomics used by different threads MUST NOT share cache lines. Pad or separate:

- queue read/write indices;
- meters written by audio and read by UI;
- flags updated at high frequency;
- per-channel DSP state processed by worker threads.

### 7.8 Ownership and lifetime

Rules:

- Use `std::unique_ptr` for unique ownership outside hot loops.
- Use raw pointers or references as non-owning views only when lifetime is obvious and documented.
- Avoid `std::shared_ptr` and `std::weak_ptr` on the audio path. Reference count increments/decrements are atomic operations and final release may delete on the wrong thread.
- Do not capture `this` in async callbacks unless cancellation/lifetime is guaranteed.
- Editors must tolerate processor destruction ordering and host editor recreation.
- Use `juce::Component::SafePointer` for UI callbacks targeting components, not for audio-thread communication.

### 7.9 Memory allocation policy

- Allocate in constructors only for host-independent small objects.
- Allocate sample-rate/block-size resources in `prepareToPlay()`.
- Reserve vectors to maximum capacity before audio use.
- Never call `shrink_to_fit()` in real-time paths.
- Prefer `std::array` for fixed-size state.
- Use aligned buffers for SIMD.
- Use custom arenas only when they are preallocated and their allocation/deallocation operations are bounded and non-blocking.
- `std::pmr` is not automatically real-time-safe; it is only safe if backed by a suitable preallocated resource and never exhausted on the audio thread.

### 7.10 Sanitisers and static analysis

Required CI jobs where practical:

- MSVC `/analyze` or C++ Core Guidelines checker.
- MSVC AddressSanitizer on Windows for tests and standalone harnesses.
- Clang AddressSanitizer and UndefinedBehaviorSanitizer on at least one platform.
- Clang ThreadSanitizer on non-real-time test harnesses for cross-thread code.
- Clang **RealtimeSanitizer (RTSan)** on standalone audio harnesses where the audio callback path is annotated `[[clang::nonblocking]]`. RTSan instruments forbidden operations (allocation, mutex acquisition, syscalls, blocking primitives) inside non-blocking regions and is the closest available approximation to a real-time correctness checker. Failures in RTSan indicate a real-time rule violation, not a flake.
- `clang-tidy` for modernize, bugprone, performance, readability, and concurrency-relevant checks.

Sanitisers are not a substitute for design. They detect classes of bugs under tested executions; they do not prove real-time safety.

### 7.11 Plug-in DLL/bundle boundary and static initialisation

Plug-ins are loaded into a host process as DLLs (Windows), bundles (macOS), or shared objects (Linux). The host owns the loader, may instantiate multiple plug-in instances concurrently in a single process, and may rescan repeatedly.

Standards:

- Avoid global non-trivial constructors that depend on JUCE globals being initialised. Static initialisation order across translation units is not defined. JUCE plug-in wrappers initialise the framework on the host's terms; user globals cannot assume they are running afterwards.
- On Windows, `DllMain` MUST NOT perform file I/O, network I/O, COM/OLE initialisation, or anything that may acquire another loader lock. Windows holds the loader lock during `DllMain` and any blocking operation can deadlock the host.
- On macOS, bundle initialisation routines (`__attribute__((constructor))`, Objective-C `+load`) face equivalent constraints; defer non-trivial work to first use under a guarded path.
- Global mutable state shared across plug-in instances MUST be documented and synchronised. Multiple instances in the same process see the same statics; an instance counter or per-instance storage is usually safer than a singleton.
- Singletons used by the plug-in MUST be safe for concurrent first-use across instances. Prefer Meyers singletons (function-local statics) over manual lazy-init schemes.
- Validate scan-time behaviour. Hosts often instantiate, query, and destruct plug-ins many times during a scan. Constructors and destructors MUST be cheap, side-effect-free at process scope, and exception-safe.
- Static destruction order at process exit is unreliable in hosted DLLs. Do not rely on global destructors running; explicit teardown belongs in the plug-in's destructor or `releaseResources()`, not in a global destructor.

---

## 8. DSP engineering standards

### 8.1 Denormals/subnormals

Every floating-point processing block MUST prevent denormal slowdowns.

JUCE default:

```cpp
juce::ScopedNoDenormals noDenormals;
```

Additionally:

- Reset feedback/filter states when they become non-finite or subnormal-prone.
- Use tiny DC/noise injection only when musically and technically justified.
- Prefer explicit state sanitisation for feedback paths.

### 8.2 Numerical safety

DSP code MUST defend against:

- NaN and Inf propagation;
- division by zero;
- invalid square roots/logarithms;
- unstable filter coefficients;
- feedback greater than stable bounds;
- extreme sample rates;
- host-provided bad or unusual parameter values;
- denormal states after long silence.

Use targeted guards:

```cpp
inline float sanitiseSample(float x) noexcept
{
    return std::isfinite(x) ? x : 0.0f;
}
```

Do not scatter expensive checks in every sample of a hot loop unless necessary. Prefer per-block validation, coefficient validation, and debug-only assertions where sufficient.

### 8.3 Parameter smoothing and coefficient updates

- Smooth audible continuous parameters.
- Recalculate coefficients at a controlled rate.
- Interpolate safely where filters can tolerate interpolation.
- For filters, prefer topologies stable under modulation, such as TPT/SVF designs, for fast-changing cutoff/resonance.
- Do not apply discontinuous algorithm changes in the middle of a block unless crossfaded.

### 8.4 Oversampling and anti-aliasing

Use oversampling for nonlinear processing when aliasing would be audible or measurable.

With `juce::dsp::Oversampling`:

- Prepare during `prepareToPlay()`.
- Choose FIR vs IIR stages based on phase/latency trade-off.
- Report latency via `setLatencySamples()` when oversampling latency changes (from `prepareToPlay` or another non-real-time call; never from `processBlock`).
- Test aliasing at multiple sample rates and drive levels.
- Avoid changing oversampling factor in the audio thread; prepare separate chains or apply changes when inactive.

### 8.5 Latency and tail

If processing introduces latency, report it accurately:

```cpp
setLatencySamples(computedLatencySamples);
```

Rules:

- `setLatencySamples()` MUST NOT be called from the audio thread (see §7.2). It triggers host PDC recomputation and may cause the host to flush or rebuild its plug-in graph. Update latency only from `prepareToPlay()`, configuration callbacks, or other non-real-time paths.
- Test reported latency with impulse tests.
- Implement tail reporting where relevant.
- Clear or preserve tails intentionally on bypass, reset, and transport changes.

### 8.6 Bypass

Bypass behaviour MUST be defined:

- hard bypass;
- smoothed/crossfaded bypass;
- latency-compensated bypass;
- tail-preserving bypass;
- host bypass parameter handling.

Avoid clicks on bypass transitions. If the plug-in has latency, bypass MUST NOT create timing discontinuities.

### 8.7 SIMD

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

### 8.8 CPU budget

For a block of `N` samples at sample rate `Fs`, the callback period is `N / Fs` seconds. A 64-sample buffer at 48 kHz gives about 1.33 ms total wall-clock time, and the plug-in only gets a fraction of that.

Standards:

- Profile Release builds, not Debug.
- Profile in real hosts and offline harnesses.
- Track worst-case, not only average CPU.
- Avoid first-use costs in the first audio block.
- Warm up lookup tables, FFT plans, oversamplers, and model state outside the audio thread.

### 8.9 Advanced DSP research and implementation checklist

Use this section when designing new effects or instruments. It is intentionally broader than a narrow coding-style guide because many plug-in bugs are algorithm-design bugs expressed as code.

#### 8.9.1 Nonlinear and virtual-analogue processing

Nonlinear processors include saturation, clipping, waveshaping, compressors with nonlinear detectors, filters with nonlinear feedback, virtual analogue circuits, exciters, tape models, and amp/cabinet models.

Standards:

- Expect aliasing. Test it rather than assuming oversampling is enough.
- Consider oversampling, antiderivative anti-aliasing, band-limited nonlinear design, or model-specific anti-aliasing depending on the algorithm.
- Drive-dependent latency and quality modes MUST be documented.
- Crossfade when switching quality/oversampling modes.
- Validate at 44.1, 48, 96, and 192 kHz where supported.
- Test high-frequency sine sweeps and high-drive impulses for foldback.
- Clamp internal feedback and state variables to stable physical/algorithmic limits.

#### 8.9.2 Filters and time-varying systems

Filters used in musical plug-ins are often modulated. A filter that is stable for static coefficients can click, zipper, or become unstable under modulation.

Standards:

- Prefer modulation-friendly topologies for rapidly changing cutoff/resonance.
- Smooth parameter values or coefficients, but do not interpolate coefficients blindly for unstable structures.
- Validate behaviour near Nyquist, near zero frequency, at high resonance, and under fast automation.
- Use double precision for coefficient calculation when it improves stability, even if audio samples are float.
- Document filter topology and known trade-offs.

#### 8.9.3 Dynamics processing

Dynamics processors MUST define their detector, ballistics, gain computer, and smoothing precisely.

Standards:

- Document peak/RMS/LUFS-like detector choices.
- Define attack/release curves, knee shape, lookahead latency, sidechain filtering, and stereo linking.
- Test gain reduction with impulses, stepped tones, sine bursts, and silence recovery.
- Avoid denormal-prone envelope tails.
- Report lookahead latency and preserve timing under bypass.

#### 8.9.4 Delay, reverb, and feedback networks

Feedback systems require explicit stability policy.

Standards:

- Bound feedback coefficients.
- Clear or fade feedback buffers on reset and sample-rate changes.
- Smooth delay-time changes using interpolation or crossfading.
- For fractional delays, test interpolation error and modulation artefacts.
- For reverbs, test tail length, denormal behaviour, silence detection, and bypass transitions.

#### 8.9.5 Convolution and impulse responses

Convolution processors are high risk for real-time allocation and latency mistakes.

Standards:

- Load and parse impulse responses off the audio thread.
- Build FFT plans and partitions off the audio thread.
- Publish new convolution engines using immutable snapshot handoff.
- Crossfade IR changes.
- Report partitioning latency accurately.
- Test malformed, huge, zero-length, mono, stereo, and mismatched sample-rate IRs.

#### 8.9.6 Spectral and time-frequency processing

STFT/phase-vocoder/spectral processors MUST treat windowing, hop size, phase, and latency as part of the product design.

Standards:

- Preallocate all FFT buffers and plans.
- Document window type, overlap, hop size, latency, and reconstruction assumptions.
- Test overlap-add reconstruction error.
- Handle transport jumps and reset state clearly.
- Avoid allocating when the FFT size changes; prepare configurations in advance or change only while inactive.

#### 8.9.7 Resampling

Resampling appears in oversampling, sample playback, IR loading, granular processing, and pitch/time effects.

Standards:

- Use well-characterised polyphase or band-limited resamplers for quality-critical paths.
- Define passband, stopband, phase, latency, and CPU targets.
- Test with sweeps and impulses.
- Do not instantiate or resize resamplers in the audio callback.

#### 8.9.8 Instruments, voices, and MIDI

Instrument plug-ins MUST preallocate voices and event storage.

Standards:

- No per-note heap allocation.
- Deterministic voice stealing.
- Sample-accurate event handling where the format/host provides offsets.
- Bounded modulation matrix work per block.
- Clear sustain/sostenuto/all-notes-off/all-sound-off behaviour.
- Avoid dynamic graph rebuilding on note-on.

#### 8.9.9 Spatial, surround, and immersive processing

For spatial plug-ins, channel semantics are part of correctness.

Standards:

- Define supported layouts explicitly.
- Keep channel order conversions isolated and tested.
- Test mono, stereo, surround, ambisonic, and sidechain cases as applicable.
- Validate downmix/upmix behaviour.
- For HRTF or convolution spatializers, use the convolution safety rules above.

#### 8.9.10 ML/model-based DSP

Machine-learning or model-inference DSP is permitted only with strict real-time controls.

Standards:

- No Python, interpreter, JIT compilation, graph optimisation, file loading, or dynamic allocation in the audio callback.
- Warm up inference engines before audio starts.
- Preallocate tensors and scratch buffers.
- Provide deterministic CPU fallback if GPU/accelerator use is not host-safe.
- Bound model execution time and test worst-case CPU.
- Validate model files as untrusted input.
- Provide bypass/fallback if model preparation fails.

#### 8.9.11 Determinism and reproducibility

Bit-identical output across platforms is often unrealistic for floating-point DSP, especially with SIMD and different maths libraries. The standard is controlled, tested tolerance, not false precision.

- Define acceptable numerical tolerances per algorithm.
- Use deterministic random seeds where needed.
- Avoid unspecified iteration order in DSP-affecting code.
- Document any platform-specific approximations.
- Use golden tests with tolerance and spectral criteria.

---

## 9. State, presets, and migration

### 9.1 State model

State MUST be versioned. For binary state, define a stable byte-level format rather than serialising C++ object memory directly:

```cpp
struct StateHeaderFields {
    uint32_t magic = 0x4D59504C; // MYPL
    uint32_t version = 1;
};
```

The struct above documents fields only. Do not persist it with `reinterpret_cast`, `memcpy` of the object, or raw `sizeof(StateHeaderFields)` writes. Write each field explicitly with a defined byte order, field width, and validation path.

For APVTS XML/ValueTree state, include a version property.

Rules:

- Never serialise raw pointers.
- Validate all values on load.
- Clamp to safe ranges.
- Preserve unknown future fields where practical.
- Add migration tests for every state version.

### 9.2 Parameter compatibility

Do not change without explicit migration:

- parameter ID;
- normalisation mapping;
- default value;
- discrete step count;
- automatable flag;
- unit semantics.

When a parameter must be removed, keep a hidden/deprecated compatibility parameter if host automation or session recall depends on it.

### 9.3 Preset loading

Preset loading may allocate and parse, but not on the audio thread. If loading requires new DSP resources, prepare them off-thread and publish an immutable snapshot safely.

Malformed presets MUST NOT crash the plug-in.

---

## 10. UI standards

### 10.1 Separation

The editor MUST NOT own DSP state. It observes processor state and writes parameters through APVTS attachments or safe parameter APIs.

### 10.2 Message thread only

JUCE components are message-thread objects. Do not access them from `processBlock()`.

Meters and visualisers SHOULD use one-way communication:

- audio thread writes atomics or SPSC queue entries;
- UI polls on a timer;
- UI drops frames if it cannot keep up.

### 10.3 Attachments

Attachment lifetimes MUST be explicit. A common pattern:

```cpp
class MyEditor final : public juce::AudioProcessorEditor {
    juce::Slider gainSlider_;
    std::unique_ptr<juce::AudioProcessorValueTreeState::SliderAttachment> gainAttachment_;
};
```

Construct controls before attachments. Destroy attachments before controls if declaration order requires it.

### 10.4 Assets

- Load fonts/images once, outside audio.
- Avoid blocking disk I/O when opening the editor; embed critical assets or preload asynchronously.
- Validate image scaling and HiDPI behaviour.
- Avoid unbounded repaint rates.

---

## 11. Error handling and diagnostics

### 11.1 Exceptions

Project policy:

- No exceptions from `processBlock()` or DSP hot paths.
- No exceptions across JUCE/host callbacks.
- No exceptions from destructors.
- Catch at boundaries where third-party code may throw, then convert to safe state and diagnostics.

### 11.2 Assertions

Use assertions for invariants, not runtime error handling.

```cpp
jassert(sampleRate > 0.0);
```

Release builds MUST remain safe if assertions are disabled.

### 11.3 Logging

Logging is forbidden on the audio thread. Non-real-time logging MUST be bounded and optionally compiled out of release builds. For audio-thread diagnostics, write atomics/counters and read them from a non-real-time context.

---

## 12. Testing requirements

### 12.1 Unit tests

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
- maximum block sizes;
- mono/stereo/multichannel layouts;
- bypass transitions.

### 12.2 Golden and spectral tests

Use golden-output tests for stable algorithms. Use spectral/aliasing tests for nonlinear processors. Define tolerances explicitly; do not require bit identity unless the algorithm is designed for it.

### 12.3 State migration tests

Every supported state version MUST have fixtures. Tests MUST load old presets and verify parameter values, hidden compatibility fields, and output behaviour.

### 12.4 Real-time safety tests

At minimum:

- allocation counter around DSP process calls in a standalone harness;
- lock-detection wrappers for project mutexes;
- stress tests with random block sizes;
- test first audio block after `prepareToPlay()`;
- repeated prepare/release cycles.

### 12.5 VST3 validation

JUCE-built VST3 plug-ins SHOULD be run through both validators in CI where practical:

- **Steinberg VST3 validator** — the official conformance check, focused on API correctness.
- **Tracktion `pluginval`** (https://github.com/Tracktion/pluginval) — a more aggressive harness that exercises threading, parameter, state, lifecycle, and bus-layout paths under stress. Widely used by JUCE plug-in vendors and surfaces classes of bugs the Steinberg validator does not. Run it at increasing strictness levels (typically a baseline level on every PR and a higher strictness level nightly or on release branches).

Add smoke tests in representative DAWs before release.

### 12.6 Sanitiser tests

Run sanitiser tests on standalone/unit harnesses, not necessarily inside every DAW. Use:

- ASAN for use-after-free, buffer overflows, and allocation bugs;
- UBSAN for undefined behaviour;
- TSAN for data races in non-real-time harnesses;
- MSVC ASAN on Windows where supported;
- Clang RTSan for real-time-rule violations on annotated audio paths (see §7.10).

---

## 13. CI and release gates

A production CI pipeline SHOULD include:

1. Configure with CMake presets.
2. Build Debug and Release/RelWithDebInfo.
3. Run unit tests.
4. Run DSP regression tests.
5. Run state migration tests.
6. Run static analysis / clang-tidy.
7. Run sanitiser jobs (ASan, UBSan, TSan, RTSan as applicable).
8. Build VST3 artefact.
9. Run Steinberg VST3 validator.
10. Run Tracktion `pluginval` (baseline strictness on every build, higher strictness on release branches).
11. Package artefact.
12. Optional: smoke-test in a plug-in host.
13. Sign/notarise where applicable.

### 13.1 Reproducible builds

Release builds MUST be reproducible enough to debug, and SHOULD aim for byte-level reproducibility where the toolchain supports it.

Required:

- Preserve symbols (PDB on Windows, dSYM on macOS, separate `.debug` files on Linux). Archive symbols separately from shipped binaries with a documented retrieval path (symbol server or artefact store).
- Record dependency versions: JUCE commit/tag, VST3 SDK version where bundled, third-party library versions and source hashes.
- Record toolchain versions: compiler, linker, Windows SDK / Xcode / glibc as applicable.
- Record the configure and build invocation: prefer `cmake --preset <name>` so the configuration is captured, not loose flag combinations.
- Record the Git commit and dirty-tree state of the source.

Recommended toolchain settings for byte-level reproducibility:

- **MSVC:** `/Brepro` to omit timestamps from object files and the PE header; `/PDBALTPATH:%_PDB%` to normalise embedded PDB paths; pin `_MSC_VER` and Windows SDK version in CI.
- **Clang / Apple Clang:** `-fdebug-prefix-map=<build-root>=.` (and equivalents for `-fmacro-prefix-map`, `-ffile-prefix-map`) to normalise paths in debug info; `-Werror=date-time` to catch accidental `__DATE__`/`__TIME__` use.
- **Honour `SOURCE_DATE_EPOCH`** for tooling that respects it (some archivers, packagers, doc generators).
- **Linker:** prefer deterministic linker modes (`/Brepro` on `link.exe`; `-Wl,--build-id=sha1` and modern lld defaults on ELF) and pin the linker version.

Bit-identical reproducibility across heterogeneous CI runners is not always feasible. The pipeline SHOULD produce binaries whose differences are fully explained by recorded inputs.

---

## 14. Agentic coder rules

When an agent modifies a JUCE plug-in project, it MUST follow these rules:

1. **Never edit parameter IDs casually.** If a change touches parameter IDs, ranges, defaults, or state migration, flag it explicitly.
2. **Never add audio-thread allocations.** Avoid containers, strings, function wrappers, `shared_ptr`, logging, locks, and UI calls in `processBlock()` or DSP calls.
3. **Preserve lifecycle boundaries.** Allocate in `prepareToPlay()`, not in `processBlock()`.
4. **Keep DSP testable.** New algorithms go into `src/dsp` with tests.
5. **Use atomics correctly.** Use relaxed loads for independent scalars; do not use `atomic::wait` on the audio thread; cite the chosen memory order at the call site.
6. **Prefer simple loops in hot paths.** Do not introduce ranges, virtual dispatch, heap-owning abstractions, or callbacks into sample loops without profiling.
7. **Add or update tests.** DSP changes require DSP tests. State changes require migration tests. Thread communication changes require stress/sanitiser tests.
8. **Report uncertainty.** If a host or SDK callback thread is unclear, assume the stricter real-time/thread-safety requirement until verified.
9. **Do not make broad formatting diffs.** Format touched files only unless requested.
10. **Run the documented build/test commands before claiming completion.**

---

## 15. Code review checklist

### Real-time safety

- [ ] No allocation/deallocation in `processBlock()` or DSP hot path.
- [ ] No locks, waits, sleeps, logging, file/network I/O, or GUI calls in audio code.
- [ ] No `shared_ptr` ownership churn in audio code.
- [ ] No `setLatencySamples()`, `updateHostDisplay()`, or other host-notifying calls from the audio thread.
- [ ] No exceptions escape callbacks.
- [ ] Denormal protection present.
- [ ] Variable and zero block sizes handled.

### Thread and memory safety

- [ ] Cross-thread data uses atomics, immutable snapshots, or bounded queues.
- [ ] Each atomic load/store cites its memory order at the call site with a one-line justification (independent scalar / publishes related data / required total order).
- [ ] Queue use matches producer/consumer assumptions.
- [ ] Object lifetimes are clear.
- [ ] No false-sharing hotspots in high-frequency atomics.
- [ ] Sanitiser/static-analysis coverage exists for new patterns (ASan/UBSan/TSan/RTSan as applicable).

### DSP correctness

- [ ] Parameter smoothing is appropriate.
- [ ] Sample-accurate vs block-level automation policy is documented per parameter.
- [ ] NaN/Inf/denormal behaviour is handled.
- [ ] Latency and tail are reported correctly, from non-real-time paths only.
- [ ] Oversampling/anti-aliasing choices are tested.
- [ ] SIMD has scalar fallback and tolerance tests.

### JUCE integration

- [ ] APVTS uses modern `ParameterLayout`.
- [ ] APVTS `copyState()`/`replaceState()` not used on audio thread.
- [ ] Attachments have safe lifetimes.
- [ ] UI does not access DSP mutable state directly.
- [ ] `isBusesLayoutSupported` covers and rejects layouts the DSP does not handle.
- [ ] `AudioPlayHead` reads handle missing fields and transport jumps.
- [ ] MIDI iteration uses the modern range-for API and honours `samplePosition`.
- [ ] `wrapperType` branches are limited to genuinely format-specific behaviour.

### Build and release

- [ ] CMake minimum satisfies current JUCE requirement and the selected preset schema.
- [ ] Warnings clean.
- [ ] Tests pass.
- [ ] Steinberg VST3 validator passes.
- [ ] Tracktion `pluginval` passes at the project's required strictness level.
- [ ] Dependency versions and toolchain versions recorded.
- [ ] Reproducible-build flags applied where supported.

---

## 16. References

Prefer references that match the pinned JUCE version. Links to moving branches or live documentation are informational only and MUST be checked against the selected SDK version during project setup or upgrade.

- JUCE CMake API: use the CMake API documentation from the pinned JUCE tag or commit, for example `external/JUCE/docs/CMake API.md` in the checked-out dependency.
- JUCE releases: https://github.com/juce-framework/JUCE/releases
- JUCE `AudioProcessorValueTreeState`: use the API reference for the pinned JUCE version.
- JUCE APVTS tutorial: https://juce.com/tutorials/tutorial_audio_processor_value_tree_state/
- JUCE `dsp::Oversampling`: use the API reference for the pinned JUCE version.
- Steinberg VST3 API documentation: https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical%2BDocumentation/API%2BDocumentation/Index.html
- Steinberg VST3 validator: https://steinbergmedia.github.io/vst3_dev_portal/pages/What%2Bis%2Bthe%2BVST%2B3%2BSDK/Validator.html
- Tracktion `pluginval`: https://github.com/Tracktion/pluginval
- C++ Core Guidelines: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines
- Microsoft C++ Core Guidelines checker: https://learn.microsoft.com/cpp/code-quality/using-the-cpp-core-guidelines-checkers
- Microsoft `/fsanitize`: https://learn.microsoft.com/cpp/build/reference/fsanitize
- Microsoft `/Brepro` (deterministic builds): https://learn.microsoft.com/cpp/build/reference/brepro-use-repro-build
- Clang ThreadSanitizer: https://clang.llvm.org/docs/ThreadSanitizer.html
- Clang UndefinedBehaviorSanitizer: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html
- Clang RealtimeSanitizer (RTSan): https://clang.llvm.org/docs/RealtimeSanitizer.html
- PortAudio callback documentation: https://files.portaudio.com/docs/v19-doxydocs-dev/portaudio_8h.html
- Ross Bencina, "Real-time audio programming 101": https://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing
- DAFX: Digital Audio Effects, Second Edition: https://www.dafx.de/DAFX_Book_Page_2nd_edition/index.html
