# Review of `coding-standards.md` for Professional C++20 Audio Plug-in Development

**Generated:** 2026-05-09  
**Input reviewed:** `coding-standards.md`  
**Target use:** reusable standards for professional VST3 plug-in projects using either JUCE or the raw Steinberg VST3 SDK  
**Assumed baseline:** C++20, MSVC 19.44+ / Visual Studio 2022, current non-preview JUCE SDK, current non-preview VST3 SDK, CMake-based builds

---

## 1. Executive verdict

The uploaded document is a useful foundation. It correctly emphasises the most important principle of audio plug-in engineering: code executed by the audio processing callback must be deterministic, bounded, and free of blocking operations. It also contains sensible starting points for VST3 naming, GUIDs, CMake, pre-allocation, RAII, atomics, and unit testing.

However, it should not be used as-is for new professional projects. The document is too project-specific, mixes C++17 and C++20 guidance, contains a few unsafe or incorrect recommendations, under-specifies the VST3 lifecycle, and does not provide enough modern JUCE-specific or raw VST3-specific guidance. It also omits several areas that are now standard in high-quality commercial plug-in engineering: sample-accurate automation, denormal handling, latency reporting, bypass/tail behaviour, state migration, sanitizer strategy, false sharing, lock-free ownership/lifetime discipline, host validation, cross-platform CI, reproducible builds, DSP verification, oversampling trade-offs, and agentic-coder guardrails.

The replacement standards should keep the document's strong real-time safety emphasis, but split the result into two SDK-specific standards while preserving a shared engineering philosophy.

---

## 2. What the original document does well

### 2.1 It treats real-time audio safety as critical

The strongest part of the document is its repeated warning that `process()` must not allocate, block, perform file I/O, log, sleep, use mutexes, or throw exceptions. This is correct and should remain non-negotiable.

### 2.2 It encourages modern C++ ownership and RAII

The guidance to prefer RAII, smart pointers for ownership, `constexpr` over macros, `enum class`, `nullptr`, and standard containers outside real-time code is generally sound.

### 2.3 It encourages automated formatting

Requiring `clang-format` is a strong professional practice. The exact style can be adjusted, but automated formatting is essential when human developers and agentic coding tools both contribute.

### 2.4 It recognises VST3 boundary types

The document correctly identifies important VST3 types such as `ParamID`, `ParamValue`, `Sample32`, `Sample64`, `TBool`, `tresult`, and `FUID`. Boundary code must follow SDK signatures exactly.

### 2.5 It includes testing and review checklists

The presence of test guidance and review checklists is valuable. These should be expanded into automated CI gates, real-time-safety tests, validator tests, and host-compatibility tests.

---

## 3. High-impact corrections

| Area | Original guidance | Problem | Corrected direction |
|---|---|---|---|
| C++ version | Table of contents and checklist say “Modern C++17” while requirements say C++20. | Inconsistent and confusing for agentic coders. | Use C++20 consistently. Mention C++17 only when explaining compatibility or migration. |
| CMake baseline | Says CMake 3.14+ generally. | Current JUCE CMake API documentation requires CMake 3.22 or higher. Steinberg's raw VST3 tutorial uses 3.15.0 as sufficient for a scratch plug-in. | JUCE projects: require CMake 3.22+ when using JUCE's CMake API. Raw VST3 projects: require at least CMake 3.15+ unless a project-specific reason permits lower. |
| C++20 designated initializers | Claims designated initializers are “order-independent,” then notes MSVC requires declaration order. | In C++, designated initializers must appear in declaration order. They are not order-independent as in C. | Use them only for simple aggregate config structs and keep designator order identical to member declaration order. |
| `std::span` | Claims `std::span` has bounds checking in debug builds. | The C++ standard does not require `span::operator[]` bounds checking. Some implementations/debug modes may add checks, but this is not portable. | Use `std::span` because it carries size and avoids pointer/length mismatch. Use explicit assertions, `.at()` where available, or project bounds helpers for checked access. |
| `std::atomic::wait` | Suggested for “UI ↔ Audio thread parameter communication.” | `wait()` is a blocking operation. It must never be used by the audio thread. | `wait/notify` may be used by background/UI worker threads only. The audio thread may only poll non-blockingly or consume bounded lock-free data structures. |
| Atomics | Uses acquire/release for a single scalar parameter. | Acquire/release is unnecessary for independent scalar parameters and may imply false confidence. | Use `memory_order_relaxed` for independent scalar values. Use release/acquire only when publishing related data and a happens-before relationship is required. |
| Lock-free queue | Provides a minimal SPSC queue without alignment, type constraints, cache padding, or detailed semantics. | Usable as a teaching sketch, but not production-ready. It can suffer false sharing and unsafe `T` assignment behaviour. | Use a reviewed SPSC implementation or a project-owned implementation with cache-line padding, trivially copyable payloads, non-allocating assignment, capacity policy, and tests under TSAN where applicable. |
| VST3 process assertions | Asserts `data.numSamples > 0`. | VST3 hosts may call `IAudioProcessor::process` with zero samples/no buffers to flush parameter changes. | Handle zero-sample process calls gracefully and still consume parameter queues. |
| VST3 process handling | Early exits on missing audio buffers before parameter handling. | Parameter changes can arrive even when no audio is processed. | Parse parameter changes and events before returning for zero buffers/samples. |
| VST3 buffer assumptions | Does not discuss input/output aliasing or null channel buffers. | VST3 documentation warns input and output buffer pointers may be the same, different, or null in certain cases. | Process safely for in-place, out-of-place, aliased, silent, and null-buffer cases. |
| Lifecycle allocation | Suggests allocating mostly in `initialize()`. | In VST3, sample rate and max block size arrive in `setupProcessing()`. In JUCE, they arrive in `prepareToPlay()`. | Allocate sample-rate/block-size-dependent resources in `setupProcessing()`/`prepareToPlay()` or `setActive(true)` after validating maximum size. |
| Thread model | States VST3 has two threads: UI and audio. | Hosts may call controller, processor, state, and UI methods on multiple threads. | Define roles: real-time audio thread, message/UI thread, host non-RT worker threads, background workers; assume cross-thread calls unless SDK guarantees otherwise. |
| Exceptions | Says no exceptions in audio thread, but not SDK-boundary-wide. | Exceptions crossing VST3/JUCE/host ABI boundaries are unsafe. | Never let exceptions escape plug-in callbacks, host ABI calls, destructors, or real-time code. Prefer no-throw hot paths. |
| `shared_ptr` | Recommends shared ownership generally. | `shared_ptr` can perform atomic ref-count operations and final deletion on the audio thread. | Avoid `shared_ptr` on the audio path. Prefer immutable snapshots with explicit handoff, `unique_ptr`, or raw non-owning references with clear lifetime. |
| Ranges | Presents ranges as performance-positive due to laziness. | Ranges can be excellent outside hot loops but may obscure generated code and inhibit optimisation in some toolchains. | Use ranges in non-real-time/control code. In DSP inner loops, prefer direct loops unless profiling proves otherwise. |
| File headers | Example places `#pragma once` in a `.cpp` file header example. | `#pragma once` belongs in headers, not source files. | Use `#pragma once` only in headers. Avoid stale author/date boilerplate in source files. |
| Secure CRT | Recommends `strcpy_s` as generally better. | It is MSVC-specific and not ideal for portable code. | Prefer typed buffers, `std::array`, `std::string` outside audio, and bounded formatting. Avoid C string functions on hot paths. |

---

## 4. Missing material that should be added

### 4.1 SDK-specific standards

The original document targets a raw VST3 project but is being repurposed for multiple projects, including JUCE. JUCE and raw VST3 require different lifecycle, parameter, state, build, and threading guidance. They should share the same engineering principles but be documented separately.

### 4.2 Current JUCE CMake requirements

Current JUCE CMake documentation states that all project types require CMake 3.22 or higher. Any reusable JUCE standard must reflect that. A generic “CMake 3.14+” statement is not correct for modern JUCE CMake projects.

### 4.3 VST3 lifecycle and state machine

A raw VST3 standard must specify expected behaviour in:

- `initialize()` and `terminate()`
- `setupProcessing()`
- `setProcessing()`
- `setActive()`
- `process()`
- `setState()` / `getState()`
- controller edit gestures
- bus arrangement negotiation
- precision negotiation

The original document does not describe this state machine deeply enough.

### 4.4 Parameter automation

Professional plug-ins must handle sample-accurate automation and ramps where the host provides them. A single atomic float is not sufficient for all parameter designs. Standards should distinguish:

- static parameter value caches;
- per-block target values;
- sample-accurate automation queues;
- smoothed DSP coefficients;
- discrete parameters;
- modulation-rate parameters;
- metering/output parameters.

### 4.5 DSP correctness and modern DSP practice

The original document only lightly touches DSP. Missing areas include:

- denormal/subnormal prevention;
- anti-aliasing and oversampling for nonlinear processors;
- latency reporting and compensation;
- filter topology choices and coefficient interpolation;
- numerical stability near Nyquist and at extreme parameter values;
- NaN/Inf containment;
- bounded feedback systems;
- SIMD alignment and vector width portability;
- offline rendering differences;
- spectral/golden-master testing;
- perceptual tolerances for tests;
- parameter smoothing to avoid zipper noise;
- phase, latency, and aliasing trade-offs;
- mono/stereo/surround/multibus layout handling.

### 4.6 Thread and memory safety

The original thread-safety section is too short for professional plug-ins. It should become a first-class chapter covering:

- hard audio-thread prohibitions;
- atomics and memory orders;
- lock-free queue limits;
- false sharing and cache-line padding;
- ABA hazards and reclamation;
- immutable snapshot handoff;
- object lifetime across processor/editor/host callbacks;
- RAII and deterministic teardown;
- smart pointer restrictions on the audio thread;
- custom allocators and preallocated arenas;
- sanitizer coverage and limitations;
- static analysis rules;
- data-race testing strategy.

### 4.7 State versioning and migration

Professional plug-ins must preserve sessions across product versions. Missing requirements include:

- stable parameter IDs;
- parameter rename/deprecation rules;
- versioned state blobs;
- migration tests;
- default-value compatibility;
- host automation compatibility;
- preset compatibility;
- endian/size robustness for binary state.

### 4.8 Host validation

Raw VST3 projects should run `validator` in CI. JUCE VST3 builds should also undergo VST3 validation and host smoke tests. The original document mentions testing but not formal VST3 conformance gates.

### 4.9 Agentic coder instructions

The user explicitly wants documents suitable for agentic coders. The original document has a checklist but not an agent-safe workflow. New documents should tell agents:

- what not to change without approval;
- how to preserve stable parameter IDs and state versions;
- how to run builds/tests/validators;
- how to avoid “helpful” real-time violations;
- how to report uncertainty;
- how to verify no allocation/blocking in callbacks;
- how to isolate DSP changes behind tests.

---

## 5. Section-by-section audit of the original

### 5.1 Language Standard & Toolchain

Good: C++20, MSVC 19.44+, warnings, no compiler extensions.

Needs change:

- Replace one-size-fits-all CMake 3.14+ with SDK-specific minimums.
- Add `/permissive-`, `/Zc:__cplusplus`, `/MP`, and optional `/analyze` guidance for MSVC.
- Add clang-cl/GCC/Clang sanitizer builds where possible.
- Require CMake presets and reproducible dependency version pinning.
- Avoid claiming “full C++20 support” without acknowledging compiler feature variance.

### 5.2 Code Formatting

Good: automated formatting and clear style.

Needs change:

- Add `.clang-tidy`, editorconfig, and CI enforcement.
- Do not overfit to LLVM if the project wants JUCE-like style; choose a style and enforce it.
- Include generated-file exclusions.
- Add “format only touched files” guidance for agents to avoid noisy diffs.

### 5.3 File Organization

Good: separate processor/controller/files and include order.

Needs change:

- Avoid requiring lowercase/no-separator file names universally. `AudioProcessor.cpp`, `PluginProcessor.cpp`, and `dsp/DelayLine.h` are common and readable.
- Remove stale “Created by” boilerplate.
- `#pragma once` is fine in headers but not source files.
- Add separate directories for `plugin/`, `dsp/`, `ui/`, `state/`, `tests/`, `cmake/`, and `third_party/`.

### 5.4 Naming Conventions

Good: PascalCase for types, camelCase for functions, `enum class`, `constexpr` constants.

Needs change:

- Member prefix is subjective. Use either `mMember`, `member_`, or another convention consistently.
- Allow standard DSP abbreviations: FFT, RMS, IIR, FIR, DC, LFO, MIDI, UI, GUI, SIMD, CPU, LUFS.
- Do not force VST3 naming into JUCE projects; JUCE has its own conventions.

### 5.5 VST3-Specific Patterns

Good: VST3 return codes, types, strings, GUIDs.

Needs change:

- Add lifecycle details and host edge cases.
- Explain normalized parameter values and sample-accurate queues.
- Add `IEditController` edit gesture rules.
- Add component/controller separation.
- Add bus, precision, context, events, and silence flags.
- Add `getLatencySamples()` and `getTailSamples()` requirements.

### 5.6 Real-Time Safety

Good: the core prohibitions are correct.

Needs change:

- Explicitly include GUI calls, `MessageManagerLock`, `AsyncUpdater` assumptions, Objective-C/COM calls, memory-mapped I/O, plugin scanning side effects, and any API with unbounded latency.
- Add “no destructor-triggered work” in process: no local objects whose destructors may deallocate, lock, log, or notify.
- Add “no `shared_ptr` ownership churn” in process.
- Add block-size variability and zero-sample handling.

### 5.7 Modern C++20

Good: `std::span`, concepts, `std::optional`, structured bindings, `constexpr`.

Needs change:

- Correct `std::span` bounds claims.
- Correct designated initializers.
- Restrict ranges in hot loops unless measured.
- Do not use `std::atomic::wait` on the audio thread.
- Add `std::pmr` guidance only for non-real-time/preallocated use; polymorphic allocators do not magically make code real-time-safe.

### 5.8 Memory Management

Good: RAII and preallocation.

Needs change:

- Allocate at lifecycle points that know sample rate and block size.
- Avoid `shared_ptr` on audio path.
- Require clear ownership annotations for raw pointers.
- Add alignment and SIMD buffers.
- Add memory-safety tooling.

### 5.9 Error Handling

Good: use `tresult` at VST3 boundaries and avoid exceptions from `process()`.

Needs change:

- Never allow exceptions to cross host ABI boundaries.
- Add no-throw destructors and `noexcept` for hot DSP methods where possible.
- Add safe failure states for allocation failures during preparation.
- Add host-visible diagnostic strategy outside real-time code.

### 5.10 Thread Safety

Good: mentions atomics and lock-free queues.

Needs change:

- Treat thread safety as a first-class chapter.
- Use relaxed memory order for independent scalar params.
- Discuss immutable snapshot handoff, SPSC queues, MPSC restrictions, false sharing, ABA, and reclamation.
- State that lock-free does not automatically mean real-time-safe.
- Add sanitizer and static analysis workflow.

### 5.11 Performance

Good: cache locality, branch hoisting, avoiding virtual calls, lookup tables.

Needs change:

- Require measurement before optimization.
- Add CPU budget calculations.
- Add SIMD and alignment policy.
- Add denormal policy.
- Add profiling across Debug/Release and real DAWs.
- Distinguish offline rendering from real-time rendering.

### 5.12 Testing

Good: unit tests and edge cases.

Needs change:

- Add property tests, fuzzing for state loading, golden DSP regression tests, spectral/aliasing tests, latency/tail tests, automation tests, and host validation.
- Add CI matrix and sanitizer builds.
- Add VST3 validator for raw VST3 and JUCE VST3 outputs.
- Add tests for no allocation/no locks in processing.

---

## 6. Recommended replacement structure

The replacement standards should be split into:

1. `juce-vst3-modern-coding-standards.md`  
   For JUCE-based plug-ins. Focus on `AudioProcessor`, `prepareToPlay()`, `processBlock()`, APVTS, JUCE DSP, JUCE CMake, JUCE state, UI/editor safety, and VST3 output validation.

2. `vst3-sdk-modern-coding-standards.md`  
   For raw Steinberg VST3 SDK plug-ins. Focus on `IComponent`, `IAudioProcessor`, `IEditController`, parameter queues, `ProcessData`, `IBStream`, CMake `smtg_*` functions, VST3 validator, and SDK-specific host contracts.

Both should share:

- C++20 style and safety philosophy;
- real-time audio non-negotiables;
- thread/memory safety policy;
- DSP verification strategy;
- CI and testing requirements;
- agentic coder rules;
- stable state/parameter migration rules.

---

## 7. Practical migration guidance

When adopting the new standards in an existing project:

1. **Freeze public compatibility first.** Record existing plug-in IDs, parameter IDs, parameter names, default values, ranges, state format versions, and latency/tail behaviour.
2. **Separate DSP from host glue.** Extract algorithms into SDK-independent classes with explicit `prepare()`, `reset()`, and `process()` methods.
3. **Move allocations out of processing.** Allocate at `prepareToPlay()` for JUCE or `setupProcessing()`/`setActive(true)` for raw VST3.
4. **Add state migration tests.** Load old presets and sessions before changing parameter layouts.
5. **Add automation tests.** Validate parameter ramps, abrupt changes, discrete parameters, bypass, and sample-rate changes.
6. **Add validator and sanitizer gates.** CI should build, test, run static analysis, run sanitizer builds, and run VST3 validation where applicable.
7. **Review every cross-thread edge.** Replace ad hoc shared mutable state with atomics, immutable snapshots, or bounded queues.
8. **Only then optimise.** Profile in realistic DAW sessions before adding SIMD, approximations, or lookup tables.

---

## 8. Source-backed notes used in the replacement documents

- Current JUCE GitHub releases list JUCE 8.0.12 as the latest release in the observed release page.
- Current JUCE CMake API documentation states that all project types require CMake 3.22 or higher.
- Steinberg's VST 3 change history lists VST SDK 3.8.0 dated 2025-10-20.
- Steinberg's VST3 SDK README describes VST SDK 3.8.x, states that the SDK is under the MIT license, and documents Visual Studio 17 2022 CMake build examples.
- Steinberg's VST3 API documentation states that `IAudioProcessor::process` receives all data required for processing through `ProcessData`, that block sizes may vary up to `maxSamplesPerBlock`, that hosts may call process with zero samples/no buffers to flush parameters, and that parameter changes are transmitted through `IParameterChanges`/`IParamValueQueue`.
- Steinberg's Data Exchange documentation explicitly warns that UI and audio processing should not directly share data because mutexes have no defined running time and can block the audio thread.
- Steinberg documents the VST3 validator as suitable for automatic build server integration.
- JUCE APVTS documentation states `getRawParameterValue()` returns an atomic float pointer that real-time processing can read, and that `copyState()`/`replaceState()` are thread-safe but use locks and are not real-time-safe.
- PortAudio's callback documentation and long-standing real-time audio engineering guidance warn against allocation, file I/O, mutexes, sleeps, and unpredictable OS/library calls in real-time callbacks.
- Clang ThreadSanitizer and UndefinedBehaviorSanitizer documentation supports the sanitizer guidance in the new standards; Microsoft documents MSVC `/fsanitize=address` support and Visual Studio C++ Core Guidelines checkers.

---

## 9. References

These are the principal public sources consulted while producing the replacement standards:

- JUCE CMake API: https://github.com/juce-framework/JUCE/blob/master/docs/CMake%20API.md
- JUCE releases: https://github.com/juce-framework/JUCE/releases
- JUCE `AudioProcessorValueTreeState`: https://docs.juce.com/master/classjuce_1_1AudioProcessorValueTreeState.html
- JUCE APVTS tutorial: https://juce.com/tutorials/tutorial_audio_processor_value_tree_state/
- JUCE `dsp::Oversampling`: https://docs.juce.com/master/classdsp_1_1Oversampling.html
- Steinberg VST3 SDK: https://github.com/steinbergmedia/vst3sdk
- VST3 change history: https://steinbergmedia.github.io/vst3_dev_portal/pages/Versions/Index.html
- VST3 API documentation: https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical%2BDocumentation/API%2BDocumentation/Index.html
- VST3 parameters and automation: https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical%2BDocumentation/Parameters%2BAutomation/Index.html
- VST3 Data Exchange: https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical%2BDocumentation/Data%2BExchange/Index.html
- VST3 validator: https://steinbergmedia.github.io/vst3_dev_portal/pages/What%2Bis%2Bthe%2BVST%2B3%2BSDK/Validator.html
- VST3 CMake tutorial: https://steinbergmedia.github.io/vst3_dev_portal/pages/Tutorials/Creating%2Ba%2Bplug-in%2Bfrom%2Bscratch.html
- CMake 3.14 release notes: https://cmake.org/cmake/help/v3.29/release/3.14.html
- Microsoft Visual Studio 2022 17.14 release notes: https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes/
- Microsoft C++ conformance improvements: https://learn.microsoft.com/cpp/overview/cpp-conformance-improvements
- Microsoft `/fsanitize`: https://learn.microsoft.com/cpp/build/reference/fsanitize
- Microsoft C++ Core Guidelines checker: https://learn.microsoft.com/cpp/code-quality/using-the-cpp-core-guidelines-checkers
- C++ Core Guidelines: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines
- Clang ThreadSanitizer: https://clang.llvm.org/docs/ThreadSanitizer.html
- Clang UndefinedBehaviorSanitizer: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html
- PortAudio callback documentation: https://files.portaudio.com/docs/v19-doxydocs-dev/portaudio_8h.html
- Ross Bencina, “Real-time audio programming 101”: https://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing
- DAFX: Digital Audio Effects, Second Edition: https://www.dafx.de/DAFX_Book_Page_2nd_edition/index.html

