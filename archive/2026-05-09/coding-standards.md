# YourPlugin VST3 Plugin - Coding Standards

---

## Overview

This document defines the coding standards for the YourPlugin VST3 plugin project. These standards ensure:

1. **Real-time audio safety** - Code that runs without glitches or dropouts
2. **VST3 SDK compliance** - Proper integration with the Steinberg VST3 SDK
3. **Modern C++20 practices** - Leveraging current language features
4. **Maintainability** - Code that is clear, consistent, and testable
5. **Windows/MSVC compatibility** - Platform-specific best practices

**Critical**: VST3 audio plugins have strict real-time constraints. Violating these rules will cause audio glitches, crashes, or host incompatibilities.

---

## Table of Contents

1. [Language Standard & Toolchain](#1-language-standard--toolchain)
2. [Code Formatting (Automated)](#2-code-formatting-automated)
3. [File Organization](#3-file-organization)
4. [Naming Conventions](#4-naming-conventions)
5. [VST3-Specific Patterns](#5-vst3-specific-patterns)
6. [Real-Time Safety Rules (CRITICAL)](#6-real-time-safety-rules-critical)
7. [Modern C++17 Practices](#7-modern-c17-practices)
8. [Memory Management](#8-memory-management)
9. [Error Handling](#9-error-handling)
10. [Comments and Documentation](#10-comments-and-documentation)
11. [Thread Safety](#11-thread-safety)
12. [Performance Considerations](#12-performance-considerations)
13. [Testing Requirements](#13-testing-requirements)
14. [Platform Considerations (Windows/MSVC)](#14-platform-considerations-windowsmsvc)
15. [Code Review Checklist](#15-code-review-checklist)

---

## 1. Language Standard & Toolchain

### Requirements

- **C++ Standard**: C++20 (minimum)
- **Compiler**: MSVC 19.44+ (Visual Studio 2022)
- **VST3 SDK**: v3.8.0_build_66
- **Build System**: CMake 3.14+
- **Warning Level**: `/W4` (MSVC) - treat warnings seriously

### Compiler Settings

```cmake
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(MSVC)
    target_compile_options(yourplugin PRIVATE /W4)
endif()
```

### Rationale

- **C++20**: Modern features (std::span, concepts, ranges), VST3 SDK compatible (C++17 minimum satisfied)
- **MSVC**: Windows VST3 development standard, optimized for x64 architecture, full C++20 support
- **No extensions**: Ensures portable, standard-compliant code

---

## 2. Code Formatting (Automated)

### Automatic Formatting

All code **must** be formatted with `clang-format` before commit.

**Configuration** (`.clang-format`):

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
ColumnLimit: 100
PointerAlignment: Left
NamespaceIndentation: All
```

### Format Code

**Automatic Formatting** (Recommended):

- VS Code with `xaver.clang-format` extension formats on save automatically
- Configured in `.vscode/settings.json` with `"editor.formatOnSave": true`

**Manual Formatting** (Command Line):

```powershell
# Format single file
clang-format -i src/processor.cpp

# Format all source files
Get-ChildItem src -Recurse -Include *.cpp,*.h | ForEach-Object { clang-format -i $_.FullName }
```

**VS Code Command Palette**:

- Press `Shift+Alt+F` to format current file
- Or: `Ctrl+Shift+P` → "Format Document"

### Key Rules

- **Indentation**: 4 spaces (never tabs)
- **Line Length**: 100 characters maximum
- **Braces**: Opening brace on same line (LLVM style)
- **Pointer/Reference**: `int* ptr` not `int *ptr`
- **Namespaces**: All indented

### Example

```cpp
namespace Steinberg {
    namespace YourPlugin {
        class AudioProcessor {
        public:
            tresult PLUGIN_API process(ProcessData& data) {
                if (data.numInputs == 0) {
                    return kResultOk;
                }
                return kResultOk;
            }
        };
    }
}
```

**Important**: Do not manually fight `clang-format`. If formatting looks wrong, fix the configuration, not the code.

---

## 3. File Organization

### File Naming

- **Lowercase, no separators**: `processor.h`, `controller.cpp`, `pluginids.h`
- **Match class name**: Class `YourPluginProcessor` → files `processor.h`/`processor.cpp`
- **Header extension**: `.h` (not `.hpp`)
- **Source extension**: `.cpp`

**Examples**:

```
src/processor.h       // YourPluginProcessor class declaration
src/processor.cpp     // YourPluginProcessor class implementation
src/controller.h      // YourPluginController class declaration
src/controller.cpp    // YourPluginController class implementation
```

### Header Guards

Use `#pragma once` (not traditional include guards).

**Correct**:

```cpp
#pragma once

#include "pluginterfaces/vst/ivstaudioprocessor.h"

namespace Steinberg {
namespace YourPlugin {

class YourPluginProcessor {
    // ...
};

}
}
```

**Rationale**: `#pragma once` is:

- Simpler and less error-prone
- Supported by all modern compilers (MSVC, GCC, Clang)
- Used throughout VST3 SDK

### Include Order

1. Corresponding header (for `.cpp` files)
2. VST3 SDK headers
3. Standard library headers
4. Project headers

**Example** (`processor.cpp`):

```cpp
#include "processor.h"

#include "public.sdk/source/vst/vstaudioeffect.h"
#include "pluginterfaces/vst/ivstparameterchanges.h"

#include <algorithm>
#include <cmath>

#include "controller.h"
#include "pluginids.h"
```

### File Header Comments

Each file should have a header comment:

```cpp
//-----------------------------------------------------------------------------
// Project     : YourPlugin VST3 plugin
// Filename    : src/processor.cpp
// Created by  : [Author Name]
// Description : Audio processor implementation
//-----------------------------------------------------------------------------

#pragma once
```

**Optional but recommended** - helps with documentation generation.

---

## 4. Naming Conventions

### Classes and Structs

**PascalCase** - capitalize first letter of each word.

```cpp
class YourPluginProcessor : public AudioEffect { };
class ParameterManager { };
struct AudioBuffer { };
```

**VST3 Base Classes**: Match SDK naming exactly:

- Inherit from `AudioEffect`, not `audioEffect`
- Inherit from `EditController`, not `EditCtrl`

### Functions and Methods

**camelCase** - lowercase first letter, capitalize subsequent words.

```cpp
void processAudio(ProcessData& data);
float calculateGain(float input);
bool isParameterValid(ParamID id);
```

**VST3 Interface Methods**: Match SDK signatures exactly:

```cpp
tresult PLUGIN_API initialize(FUnknown* context) SMTG_OVERRIDE;
tresult PLUGIN_API process(ProcessData& data) SMTG_OVERRIDE;
tresult PLUGIN_API setActive(TBool state) SMTG_OVERRIDE;
```

### Variables

**camelCase** for local variables and parameters:

```cpp
int numChannels = 2;
float sampleRate = 44100.0f;
Sample32* inputBuffer = nullptr;
```

**Member Variables** - prefix with `m`:

```cpp
class YourPluginProcessor {
private:
    float mGain = 1.0f;
    int32 mSampleRate = 44100;
    std::vector<float> mDelayBuffer;
};
```

**Rationale**: `m` prefix clearly distinguishes members from locals, used throughout VST3 SDK.

### Constants

**constexpr PascalCase** (preferred over `#define`):

```cpp
constexpr int32 MaxChannels = 8;
constexpr float DefaultGain = 1.0f;
constexpr ParamID GainParamId = 100;
```

**Avoid `#define` for constants** - use `constexpr` for type safety.

**Exception**: SDK macros like `SMTG_OVERRIDE`, `PLUGIN_API` are required.

### Enumerations

**enum class** (strongly typed):

```cpp
enum class ProcessorState {
    Inactive,
    Active,
    Processing
};

ProcessorState mState = ProcessorState::Inactive;
```

**Never** use old-style `enum` - always `enum class` for type safety.

### Namespaces

**PascalCase**, two-level structure:

```cpp
namespace Steinberg {
    namespace YourPlugin {

        class YourPluginProcessor { };

    }  // namespace YourPlugin
}  // namespace Steinberg
```

**Close namespaces with comments** for clarity in large files.

### Avoid

- **Homophones**: `cnt`, `count1`, `count2` (use descriptive names)
- **Single letters**: `i`, `j`, `k` (except loop indices)
- **Hungarian notation**: `iCount`, `fGain` (type prefixes unnecessary in modern C++)
- **Abbreviations**: `buf`, `tmp`, `mgr` (write full words: `buffer`, `temporary`, `manager`)

---

## 5. VST3-Specific Patterns

### Interface Implementation

Always use VST3 macros and patterns:

```cpp
class YourPluginProcessor : public AudioEffect {
public:
    tresult PLUGIN_API initialize(FUnknown* context) SMTG_OVERRIDE;
    tresult PLUGIN_API terminate() SMTG_OVERRIDE;
    tresult PLUGIN_API setActive(TBool state) SMTG_OVERRIDE;
    tresult PLUGIN_API process(ProcessData& data) SMTG_OVERRIDE;

    static FUnknown* createInstance(void*) {
        return (IAudioProcessor*)new YourPluginProcessor;
    }
};
```

**Required**:

- `PLUGIN_API` before VST3 interface methods
- `SMTG_OVERRIDE` to mark overridden virtual methods
- Static `createInstance()` factory method
- Cast to interface pointer in factory

### Return Values

Use VST3 `tresult` codes, never `bool` or `int`:

```cpp
tresult PLUGIN_API YourPluginProcessor::initialize(FUnknown* context) {
    tresult result = AudioEffect::initialize(context);
    if (result != kResultOk) {
        return result;
    }

    // Plugin-specific initialization...

    return kResultOk;
}
```

**Common `tresult` values**:

- `kResultOk` - Success
- `kResultFalse` - Operation failed (not an error)
- `kInvalidArgument` - Invalid parameter
- `kNotImplemented` - Feature not implemented

### VST3 Type System

Use VST3 types for plugin code:

```cpp
ParamID gainParamId = 100;
ParamValue normalizedValue = 0.5;  // Always 0.0 to 1.0
Sample32* buffer32 = data.inputs[0].channelBuffers32[0];
Sample64* buffer64 = data.inputs[0].channelBuffers64[0];
int32 numSamples = data.numSamples;
TBool isActive = true;
```

**Do not** use `int`, `bool`, `float` where VST3 has specific types.

### String Handling

VST3 uses UTF-16 strings (`TChar` = `char16_t`):

```cpp
#include "pluginterfaces/base/ustring.h"

// ✅ CORRECT - Converting ASCII to UTF-16
void setParameterName(Vst::String128 name) {
    Steinberg::UString(name, 128).assign(STR16("Gain"));
}

// ✅ CORRECT - Converting UTF-16 to ASCII
char text[128];
snprintf(text, 128, "%.2f", value);
Steinberg::UString(outputString, 128).fromAscii(text);
```

**Important String Conversion Rules**:

- `UString` - Base class for wrapping existing buffers
- `UString128` - Typedef for `UStringBuffer<128>` (has internal buffer)
- **Always use `Steinberg::UString(buffer, size)` to wrap existing buffers**
- **Never use `UString128(buffer, size)` - creates temporary that is destroyed immediately**

```cpp
// ❌ WRONG - Creates temporary object
UString128(string, 128).fromAscii(text);  // Bug: temporary destroyed, string unchanged

// ✅ CORRECT - Wraps buffer reference
Steinberg::UString(string, 128).fromAscii(text);  // Works: writes to buffer
```

Use `STR16()` macro for string literals.

### Plugin IDs

Generate unique GUIDs, **never** use zeros:

```cpp
// pluginids.h
#pragma once

#include "pluginterfaces/base/funknown.h"

namespace Steinberg {
namespace YourPlugin {

// Generate with online GUID tool or guidgen.exe
static const FUID ProcessorUID(0x12345678, 0x9ABCDEF0, 0x12345678, 0x9ABCDEF0);
static const FUID ControllerUID(0x87654321, 0x0FEDCBA9, 0x87654321, 0x0FEDCBA9);

}
}
```

**Critical**: Use unique GUIDs for every plugin to avoid conflicts.

---

## 6. Real-Time Safety Rules (CRITICAL)

### The Golden Rule

**NEVER allocate, deallocate, or block in the `process()` method.**

The audio `process()` callback runs on a real-time thread with strict timing deadlines. Any violation causes:

- Audio glitches and dropouts
- Host instability or crashes
- Plugin blacklisting by DAW vendors

### Forbidden in `process()` Method

#### ❌ Memory Allocation/Deallocation

```cpp
// ❌ WRONG - causes audio glitches
tresult PLUGIN_API process(ProcessData& data) {
    auto buffer = new float[1024];              // ❌ Heap allocation
    std::vector<float> temp;
    temp.push_back(1.0f);                       // ❌ May allocate
    delete[] buffer;                            // ❌ Deallocation
    return kResultOk;
}
```

```cpp
// ✅ CORRECT - pre-allocated buffers
class YourPluginProcessor {
private:
    std::vector<float> mTempBuffer;  // Pre-allocated in initialize()

public:
    tresult PLUGIN_API initialize(FUnknown* context) SMTG_OVERRIDE {
        mTempBuffer.resize(8192);  // ✅ Allocate here
        return AudioEffect::initialize(context);
    }

    tresult PLUGIN_API process(ProcessData& data) SMTG_OVERRIDE {
        // ✅ Use pre-allocated buffer
        float* temp = mTempBuffer.data();
        return kResultOk;
    }
};
```

#### ❌ File I/O

```cpp
// ❌ WRONG - file I/O is blocking
tresult PLUGIN_API process(ProcessData& data) {
    std::ofstream log("debug.txt");  // ❌ File I/O
    log << "Processing...";          // ❌ Write to file
    fprintf(stderr, "Debug\n");      // ❌ Console output
    return kResultOk;
}
```

**Never** use: `std::cout`, `printf`, `fprintf`, file operations, Windows `OutputDebugString` in `process()`.

#### ❌ Locks and Mutexes

```cpp
// ❌ WRONG - mutex can block
class YourPluginProcessor {
private:
    std::mutex mLock;
    float mGain;

public:
    tresult PLUGIN_API process(ProcessData& data) SMTG_OVERRIDE {
        std::lock_guard<std::mutex> guard(mLock);  // ❌ Blocking lock
        float gain = mGain;
        return kResultOk;
    }
};
```

**Use lock-free alternatives** (see Thread Safety section).

#### ❌ System Calls

```cpp
// ❌ WRONG - system calls
tresult PLUGIN_API process(ProcessData& data) {
    Sleep(1);                    // ❌ Thread sleep
    HANDLE h = CreateFile(...);  // ❌ OS call
    return kResultOk;
}
```

#### ❌ Exceptions

```cpp
// ❌ WRONG - exceptions in audio thread
tresult PLUGIN_API process(ProcessData& data) {
    if (data.numInputs == 0) {
        throw std::runtime_error("No inputs");  // ❌ Exception
    }
    return kResultOk;
}
```

**Never throw exceptions from `process()`** - use error flags or return codes.

### Allowed in `process()`

#### ✅ Stack Allocations

```cpp
tresult PLUGIN_API process(ProcessData& data) {
    float tempValue = 0.0f;          // ✅ Stack allocation
    float samples[128];              // ✅ Fixed-size stack array
    return kResultOk;
}
```

**Warning**: Keep stack usage reasonable (<16KB) to avoid stack overflow.

#### ✅ Pre-Allocated Buffers

```cpp
tresult PLUGIN_API process(ProcessData& data) {
    float* buffer = mPreAllocatedBuffer.data();  // ✅ Pre-allocated
    return kResultOk;
}
```

#### ✅ Atomic Operations

```cpp
tresult PLUGIN_API process(ProcessData& data) {
    float gain = mGainAtomic.load(std::memory_order_relaxed);  // ✅ Lock-free
    return kResultOk;
}
```

#### ✅ Simple Arithmetic and Logic

```cpp
tresult PLUGIN_API process(ProcessData& data) {
    for (int32 i = 0; i < data.numSamples; ++i) {
        output[i] = input[i] * gain;  // ✅ Math operations
    }
    return kResultOk;
}
```

#### ✅ Const Data Access

```cpp
const float* lookupTable = mWavetable.data();  // ✅ Read-only access
```

### Where to Allocate Resources

```cpp
class YourPluginProcessor {
private:
    std::vector<float> mDelayBuffer;

public:
    // ✅ Allocate in initialize()
    tresult PLUGIN_API initialize(FUnknown* context) SMTG_OVERRIDE {
        tresult result = AudioEffect::initialize(context);
        if (result != kResultOk)
            return result;

        mDelayBuffer.resize(88200);  // ✅ Pre-allocate 2 seconds @ 44.1kHz

        addAudioInput(STR16("Stereo In"), SpeakerArr::kStereo);
        addAudioOutput(STR16("Stereo Out"), SpeakerArr::kStereo);

        return kResultOk;
    }

    // ✅ Use in process()
    tresult PLUGIN_API process(ProcessData& data) SMTG_OVERRIDE {
        float* delayBuffer = mDelayBuffer.data();  // ✅ No allocation
        // Use buffer...
        return kResultOk;
    }

    // ✅ Deallocate in terminate()
    tresult PLUGIN_API terminate() SMTG_OVERRIDE {
        mDelayBuffer.clear();  // ✅ Safe to deallocate here
        return AudioEffect::terminate();
    }
};
```

---

## 7. Modern C++20 Practices

### Use `nullptr` (not `NULL`)

```cpp
// ❌ Old style
Sample32* buffer = NULL;
if (buffer == 0) { }

// ✅ C++17 style
Sample32* buffer = nullptr;
if (buffer == nullptr) { }
```

### Smart Pointers

Prefer smart pointers to raw pointers for ownership:

```cpp
// ✅ Unique ownership
std::unique_ptr<AudioBuffer> buffer = std::make_unique<AudioBuffer>(size);

// ✅ Shared ownership
std::shared_ptr<Parameter> param = std::make_shared<Parameter>(id, name);

// ❌ Avoid raw pointers for ownership
AudioBuffer* buffer = new AudioBuffer(size);  // Who deletes this?
```

**Exception**: VST3 SDK uses raw pointers in interfaces - follow SDK patterns there.

### `auto` Type Deduction

Use `auto` for complex types, but keep code readable:

```cpp
// ✅ Good use of auto
auto* bus = FCast<AudioBus>(audioInputs.at(0));
auto speakerArr = SpeakerArr::kStereo;

// ❌ Over-use obscures type
auto x = 42;  // Just write: int32 x = 42;

// ✅ Clear intent
int32 channelCount = 2;
```

### Range-Based For Loops

```cpp
// ✅ Modern C++17
for (const auto& parameter : mParameters) {
    parameter.update();
}

// ❌ Old style (but sometimes necessary)
for (size_t i = 0; i < mParameters.size(); ++i) {
    mParameters[i].update();
}
```

### `constexpr` Constants

```cpp
// ✅ Type-safe compile-time constant
constexpr int32 MaxChannels = 8;
constexpr float Pi = 3.14159265359f;
constexpr double DefaultSampleRate = 44100.0;

// ❌ Avoid preprocessor macros
#define MAX_CHANNELS 8  // No type safety
```

### `enum class` (Strongly Typed)

```cpp
// ✅ Strongly typed
enum class FilterType {
    LowPass,
    HighPass,
    BandPass
};

FilterType type = FilterType::LowPass;

// ❌ Old-style enum
enum FilterType {
    LOWPASS,   // Pollutes global namespace
    HIGHPASS
};
```

### Structured Bindings (C++17)

```cpp
// ✅ Clean unpacking
auto [min, max] = getParameterRange(paramId);

// ❌ Verbose alternative
std::pair<float, float> range = getParameterRange(paramId);
float min = range.first;
float max = range.second;
```

### `std::optional` for Optional Values

```cpp
#include <optional>

std::optional<float> findParameterValue(ParamID id) {
    if (isValidParameter(id)) {
        return mParameterValues[id];
    }
    return std::nullopt;
}

// Usage
if (auto value = findParameterValue(id)) {
    processValue(*value);
}
```

### Lambda Functions

```cpp
// ✅ Useful for callbacks and STL algorithms
std::sort(mParameters.begin(), mParameters.end(),
    [](const Parameter& a, const Parameter& b) {
        return a.getId() < b.getId();
    });

// ✅ Capture by reference for local scope
float gain = 2.0f;
auto applyGain = [&gain](float sample) { return sample * gain; };
```

**Warning**: Be careful with captures in audio callbacks - ensure thread safety.

---

## 7A. C++20 Features (New)

### `std::span` for Buffer Safety

**Always use `std::span` for array-like parameters**:

```cpp
#include <span>

// ✅ Type-safe, bounds-aware
void processAudio(std::span<const float> input,
                  std::span<float> output) {
    assert(input.size() == output.size());

    for (size_t i = 0; i < input.size(); ++i) {
        output[i] = input[i] * mGain;
    }
}

// ✅ Range-based iteration
void processAudio(std::span<float> buffer) {
    for (float& sample : buffer) {  // Safe iteration
        sample *= mGain;
    }
}

// ❌ Old style - error-prone
void processAudio(const float* input, float* output, int numSamples);
```

**Benefits**:

- Size bundled with data (prevents buffer overruns)
- Bounds checking in debug builds
- Zero overhead in release builds
- Iterator support for range-based for loops

**Use for**: All audio processing, modulation matrix operations, oversampling buffers

### Concepts for Template Constraints

**Use concepts for better error messages**:

```cpp
#include <concepts>

// ✅ Self-documenting, clear errors
template<std::floating_point T>
T linearInterpolate(T a, T b, T t) {
    return a + (b - a) * t;
}

// ✅ Custom concepts
template<typename T>
concept AudioSample = std::floating_point<T> && sizeof(T) <= 8;

template<AudioSample T>
void processSamples(std::span<T> buffer) {
    // Compiler enforces that T is float or double only
}

// ✅ Constraining template parameters
template<typename T>
requires std::totally_ordered<T>
inline T clamp(T value, T min, T max) {
    return std::clamp(value, min, max);
}
```

**Benefits**:

- Better compile error messages
- Self-documenting type requirements
- Compile-time validation

**Use for**: Parameter conversion templates, DSP utility functions

### Designated Initializers

**Use for clear struct initialization**:

```cpp
// ✅ C++20 - Self-documenting
struct OversampleConfig {
    int factor;
    float cutoffFreq;
    int filterOrder;
    bool enabled;
};

OversampleConfig config{
    .factor = 4,
    .cutoffFreq = 0.45f,
    .filterOrder = 6,
    .enabled = true
};  // Clear, self-documenting

**Important**: MSVC requires designated initializers to appear in the same order as member declarations.

// ✅ For routing structures
ModulationRouting routing{
    .sourceIndex = 0,           // LFO 1
    .targetParamId = kGainParam,
    .amount = 0.75f,
    .enabled = true
};

// ❌ Old style - unclear
OversampleConfig config{4, 0.45f, 6, true};  // What do these mean?
```

**Benefits**:

- Self-documenting initialization
- Order-independent (safer refactoring)
- Compile-time checking of field names

**Use for**: All configuration structs, modulation routing, effect parameters

### Ranges Library

**Use for expressive transformations**:

```cpp
#include <ranges>

// ✅ Composable, lazy evaluation
auto activeRoutings = mRoutings
    | std::views::filter([](auto& r) { return r.enabled; })
    | std::views::transform([](auto& r) { return r.calculate(); });

for (float modValue : activeRoutings) {
    applyModulation(modValue);
}

// ✅ Multiple operations chained
auto processedValues = inputBuffer
    | std::views::transform([](float s) { return s * 2.0f; })
    | std::views::filter([](float s) { return std::abs(s) > 0.001f; })
    | std::views::take(100);
```

**Benefits**:

- Lazy evaluation (performance)
- Composable operations
- More readable than nested loops

**Use for**: Modulation matrix filtering, LFO calculations, effect chain processing

### Atomic Wait/Notify

**Use for efficient lock-free signaling**:

```cpp
#include <atomic>

class ParameterQueue {
    std::atomic<bool> mHasData{false};

    void producer() {
        // Write data...
        mHasData.store(true, std::memory_order_release);
        mHasData.notify_one();  // ✅ C++20 - Wake waiting thread
    }

    void consumer() {
        mHasData.wait(false, std::memory_order_acquire);  // ✅ Efficient wait
        // Process data...
    }
};
```

**Benefits**:

- OS-level thread blocking (not busy-waiting)
- Lower CPU usage
- Cleaner lock-free patterns

**Use for**: UI ↔ Audio thread parameter communication

---

## 8. Memory Management

### RAII Principle

**Resource Acquisition Is Initialization** - resources are tied to object lifetime.

```cpp
class AudioBuffer {
private:
    std::vector<float> mData;

public:
    AudioBuffer(size_t size) : mData(size) {
        // ✅ Constructor allocates
    }

    ~AudioBuffer() {
        // ✅ Destructor automatically deallocates (vector does this)
    }

    // ✅ No manual memory management needed
};
```

### Prefer Standard Containers

```cpp
// ✅ Use standard containers
std::vector<float> buffer;
std::array<Sample32, 1024> fixedBuffer;
std::unique_ptr<float[]> rawArray = std::make_unique<float[]>(size);

// ❌ Avoid manual allocation
float* buffer = new float[size];  // Must remember to delete[]
```

### Pre-Allocation Strategy

For real-time code, allocate in `initialize()`, use in `process()`:

```cpp
class YourPluginProcessor {
private:
    std::vector<float> mDelayBuffer;
    std::vector<float> mTempBuffer;

public:
    tresult PLUGIN_API initialize(FUnknown* context) SMTG_OVERRIDE {
        AudioEffect::initialize(context);

        // ✅ Pre-allocate maximum required size
        mDelayBuffer.resize(192000);  // 2 sec @ 96kHz
        mTempBuffer.resize(8192);     // Max block size

        return kResultOk;
    }

    tresult PLUGIN_API setActive(TBool state) SMTG_OVERRIDE {
        if (state) {
            // ✅ Clear buffers when activated
            std::fill(mDelayBuffer.begin(), mDelayBuffer.end(), 0.0f);
        }
        return AudioEffect::setActive(state);
    }
};
```

### Avoid Memory Leaks

```cpp
// ❌ Potential leak
void processAudio() {
    float* buffer = new float[1024];
    // ... processing ...
    // Forgot to delete[]! Memory leak!
}

// ✅ Automatic cleanup
void processAudio() {
    std::vector<float> buffer(1024);
    // ... processing ...
    // Automatic cleanup when function exits
}

// ✅ Or use unique_ptr for raw arrays
void processAudio() {
    auto buffer = std::make_unique<float[]>(1024);
    // ... processing ...
    // Automatic cleanup
}
```

---

## 9. Error Handling

### VST3 Return Codes

Always return `tresult` from VST3 interface methods:

```cpp
tresult PLUGIN_API YourPluginProcessor::initialize(FUnknown* context) {
    tresult result = AudioEffect::initialize(context);
    if (result != kResultOk) {
        return result;  // ✅ Propagate error
    }

    if (!allocateBuffers()) {
        return kResultFalse;  // ✅ Indicate failure
    }

    return kResultOk;  // ✅ Success
}
```

### Check Parameters

```cpp
tresult PLUGIN_API YourPluginProcessor::setBusArrangements(
    SpeakerArrangement* inputs, int32 numIns,
    SpeakerArrangement* outputs, int32 numOuts) {

    // ✅ Validate inputs
    if (numIns != 1 || numOuts != 1) {
        return kResultFalse;
    }

    if (inputs[0] != SpeakerArr::kStereo) {
        return kResultFalse;
    }

    return AudioEffect::setBusArrangements(inputs, numIns, outputs, numOuts);
}
```

### Assertions for Development

Use `SMTG_ASSERT` for debug-time checks:

```cpp
#include "base/source/fassert.h"

tresult PLUGIN_API process(ProcessData& data) {
    // ✅ Assert invariants in debug builds
    SMTG_ASSERT(data.numSamples > 0);
    SMTG_ASSERT(data.numSamples <= 8192);

    // Production code...
    return kResultOk;
}
```

**Note**: Assertions compile out in release builds - don't use for error handling!

### No Exceptions in Audio Thread

```cpp
// ❌ NEVER throw from process()
tresult PLUGIN_API process(ProcessData& data) {
    throw std::runtime_error("Error");  // ❌ Crashes host!
}

// ✅ Use error flags
class YourPluginProcessor {
private:
    std::atomic<bool> mErrorOccurred{false};

public:
    tresult PLUGIN_API process(ProcessData& data) SMTG_OVERRIDE {
        if (data.numInputs == 0) {
            mErrorOccurred.store(true);  // ✅ Set flag
            return kResultFalse;
        }
        return kResultOk;
    }
};
```

### Logging (Non-Real-Time Only)

```cpp
// ✅ Log in initialize(), not process()
tresult PLUGIN_API initialize(FUnknown* context) {
    #ifdef DEBUG
        OutputDebugStringA("YourPluginProcessor::initialize()\n");  // ✅ Debug only
    #endif

    return AudioEffect::initialize(context);
}

// ❌ NEVER log in process()
tresult PLUGIN_API process(ProcessData& data) {
    OutputDebugStringA("Processing...\n");  // ❌ File I/O!
    return kResultOk;
}
```

---

## 10. Comments and Documentation

### Doxygen Format

Use Doxygen-style comments for API documentation:

```cpp
/**
 * Audio processor for the YourPlugin VST3 plugin.
 *
 * Implements a stereo audio effect with real-time parameter modulation.
 */
class YourPluginProcessor : public AudioEffect {
public:
    /**
     * Initialize the processor and allocate resources.
     *
     * @param context Host application context
     * @return kResultOk on success, error code otherwise
     */
    tresult PLUGIN_API initialize(FUnknown* context) SMTG_OVERRIDE;

    /**
     * Process audio samples.
     *
     * @param data Contains input/output buffers and parameter changes
     * @return kResultOk on success
     *
     * @note This method runs in real-time thread - no allocations allowed!
     */
    tresult PLUGIN_API process(ProcessData& data) SMTG_OVERRIDE;
};
```

### File Headers

```cpp
//-----------------------------------------------------------------------------
// Project     : YourPlugin VST3 plugin
// Filename    : src/processor.cpp
// Created by  : [Author Name]
// Description : Audio processor implementation with gain and filter
//-----------------------------------------------------------------------------

#pragma once

#include "public.sdk/source/vst/vstaudioeffect.h"
```

### Inline Comments

```cpp
tresult PLUGIN_API process(ProcessData& data) {
    // Early exit for bypassed or empty input
    if (data.numInputs == 0 || data.numOutputs == 0) {
        return kResultOk;
    }

    // Process parameter changes from host
    if (data.inputParameterChanges) {
        int32 numChanges = data.inputParameterChanges->getParameterCount();
        for (int32 i = 0; i < numChanges; ++i) {
            // Handle each parameter change queue
        }
    }

    // Audio processing loop
    int32 numChannels = data.inputs[0].numChannels;
    for (int32 ch = 0; ch < numChannels; ++ch) {
        Sample32* input = data.inputs[0].channelBuffers32[ch];
        Sample32* output = data.outputs[0].channelBuffers32[ch];

        // Apply gain to each sample
        for (int32 s = 0; s < data.numSamples; ++s) {
            output[s] = input[s] * mGain;
        }
    }

    return kResultOk;
}
```

### Comment Style Rules

1. **Explain WHY, not WHAT**

   ```cpp
   // ❌ Bad: documents the obvious
   i++;  // increment i

   // ✅ Good: explains the reason
   // Advance to next sample, skipping DC offset at index 0
   i++;
   ```

2. **Document complex algorithms**

   ```cpp
   // Biquad filter implementation using Direct Form II
   // Reference: "Introduction to Digital Filters" - Julius O. Smith III
   // https://ccrma.stanford.edu/~jos/filters/
   ```

3. **Mark TODOs and FIXMEs**

   ```cpp
   // TODO: Implement SIMD optimization for multi-channel processing
   // FIXME: Handle edge case when sample rate changes during processing
   ```

4. **Use `//` for single-line, `/** \*/` for Doxygen\*\*

   ```cpp
   // Short inline comment

   /**
    * Multi-line Doxygen comment
    * for documentation generation
    */
   ```

---

## 11. Thread Safety

### Parameter Updates (Cross-Thread Communication)

VST3 plugins have two threads:

- **UI Thread**: Parameter changes from controller
- **Audio Thread**: `process()` method

**Use atomics for simple values**:

```cpp
class YourPluginProcessor {
private:
    std::atomic<float> mGainParameter{1.0f};
    float mGainSmoothed = 1.0f;

public:
    // ✅ Called from UI thread
    void setGainParameter(float gain) {
        mGainParameter.store(gain, std::memory_order_release);
    }

    // ✅ Called from audio thread
    tresult PLUGIN_API process(ProcessData& data) SMTG_OVERRIDE {
        // Read parameter atomically
        float targetGain = mGainParameter.load(std::memory_order_acquire);

        // Smooth parameter changes to avoid clicks
        const float smoothingFactor = 0.001f;
        mGainSmoothed += (targetGain - mGainSmoothed) * smoothingFactor;

        // Use smoothed value for processing
        applyGain(mGainSmoothed);

        return kResultOk;
    }
};
```

### Lock-Free Data Structures

For complex data, use lock-free queues:

```cpp
#include <atomic>
#include <array>

// Simple single-producer, single-consumer lock-free queue
template<typename T, size_t Size>
class LockFreeQueue {
private:
    std::array<T, Size> mBuffer;
    std::atomic<size_t> mWriteIndex{0};
    std::atomic<size_t> mReadIndex{0};

public:
    bool push(const T& item) {
        size_t currentWrite = mWriteIndex.load(std::memory_order_relaxed);
        size_t nextWrite = (currentWrite + 1) % Size;

        if (nextWrite == mReadIndex.load(std::memory_order_acquire)) {
            return false;  // Queue full
        }

        mBuffer[currentWrite] = item;
        mWriteIndex.store(nextWrite, std::memory_order_release);
        return true;
    }

    bool pop(T& item) {
        size_t currentRead = mReadIndex.load(std::memory_order_relaxed);

        if (currentRead == mWriteIndex.load(std::memory_order_acquire)) {
            return false;  // Queue empty
        }

        item = mBuffer[currentRead];
        mReadIndex.store((currentRead + 1) % Size, std::memory_order_release);
        return true;
    }
};
```

### Memory Order Guidelines

- **`memory_order_relaxed`**: No synchronization, just atomic operation
- **`memory_order_acquire`**: Read operation, synchronize with release
- **`memory_order_release`**: Write operation, synchronize with acquire
- **`memory_order_seq_cst`**: Sequential consistency (default, safest, slowest)

**For audio code**: Use `relaxed` for performance, `acquire/release` for correctness.

---

## 12. Performance Considerations

### Cache-Friendly Data Structures

```cpp
// ✅ Cache-friendly: contiguous memory
std::vector<float> samples(1024);

// ❌ Cache-unfriendly: pointer chasing
std::list<float> samples;  // Linked list, bad cache locality
```

### Minimize Branches in Hot Loops

```cpp
// ❌ Branch inside loop (slow)
for (int32 i = 0; i < numSamples; ++i) {
    if (mBypass) {
        output[i] = input[i];
    } else {
        output[i] = processEffect(input[i]);
    }
}

// ✅ Branch outside loop (fast)
if (mBypass) {
    std::copy(input, input + numSamples, output);
} else {
    for (int32 i = 0; i < numSamples; ++i) {
        output[i] = processEffect(input[i]);
    }
}
```

### Lookup Tables for Non-Linear Functions

```cpp
class WaveShaper {
private:
    static constexpr size_t TableSize = 1024;
    std::array<float, TableSize> mLookupTable;

public:
    WaveShaper() {
        // ✅ Pre-calculate expensive function
        for (size_t i = 0; i < TableSize; ++i) {
            float x = static_cast<float>(i) / TableSize * 2.0f - 1.0f;
            mLookupTable[i] = std::tanh(x);  // Expensive calculation once
        }
    }

    float process(float input) {
        // ✅ Fast table lookup instead of std::tanh()
        float normalized = (input + 1.0f) * 0.5f;  // Map to 0..1
        size_t index = static_cast<size_t>(normalized * (TableSize - 1));
        return mLookupTable[index];
    }
};
```

### Inline Small Functions

```cpp
// ✅ Inline for frequently-called small functions
inline float linearInterpolate(float a, float b, float t) {
    return a + (b - a) * t;
}

inline float clamp(float value, float min, float max) {
    return std::max(min, std::min(max, value));
}
```

### Avoid Virtual Calls in Inner Loops

```cpp
// ❌ Virtual call per sample (slow)
for (int32 i = 0; i < numSamples; ++i) {
    output[i] = virtualProcess(input[i]);  // Virtual dispatch overhead
}

// ✅ Resolve virtual call once
auto processFunc = getProcessFunction();
for (int32 i = 0; i < numSamples; ++i) {
    output[i] = processFunc(input[i]);  // Direct call
}
```

---

## 13. Testing Requirements

### Unit Tests for DSP Code

Test all audio processing algorithms:

```cpp
#include <gtest/gtest.h>
#include "processor.h"

TEST(YourPluginProcessorTest, GainProcessing) {
    YourPluginProcessor processor;

    // Setup
    float input[128];
    float output[128];
    std::fill(std::begin(input), std::end(input), 1.0f);

    float gain = 0.5f;
    processor.setGain(gain);

    // Process
    processor.processBlock(input, output, 128);

    // Verify
    for (int i = 0; i < 128; ++i) {
        EXPECT_FLOAT_EQ(output[i], input[i] * gain);
    }
}

TEST(YourPluginProcessorTest, ParameterRanges) {
    YourPluginProcessor processor;

    // Test parameter limits
    EXPECT_TRUE(processor.setGain(0.0f));    // Minimum
    EXPECT_TRUE(processor.setGain(1.0f));    // Maximum
    EXPECT_FALSE(processor.setGain(-0.1f));  // Below minimum
    EXPECT_FALSE(processor.setGain(1.1f));   // Above maximum
}
```

### Real-Time Safety Tests

Verify no allocations in `process()`:

```cpp
TEST(YourPluginProcessorTest, NoAllocationsInProcess) {
    YourPluginProcessor processor;
    ProcessData data;
    // Setup data...

    // Track allocations
    size_t allocsBefore = getAllocationCount();

    processor.process(data);

    size_t allocsAfter = getAllocationCount();

    EXPECT_EQ(allocsBefore, allocsAfter) << "process() must not allocate";
}
```

### Edge Case Testing

```cpp
TEST(YourPluginProcessorTest, EdgeCases) {
    YourPluginProcessor processor;

    // Zero samples
    processor.process(createEmptyData(0));

    // Maximum samples
    processor.process(createEmptyData(8192));

    // Silent input
    auto result = processor.process(createSilentData(128));
    EXPECT_TRUE(isSilent(result));

    // Full-scale input
    processor.process(createFullScaleData(128));
}
```

### Test Coverage Target

- **DSP algorithms**: >90% coverage
- **Parameter handling**: 100% coverage
- **Edge cases**: All identified cases tested

---

## 14. Platform Considerations (Windows/MSVC)

### Windows-Specific Code

```cpp
#ifdef _WIN32
    #include <windows.h>

    // Use Windows APIs sparingly, only when necessary
    void debugLog(const char* message) {
        #ifdef DEBUG
            OutputDebugStringA(message);  // ✅ Debug builds only
        #endif
    }
#endif
```

### MSVC Warning Level

Project uses `/W4` - address all warnings:

```cpp
// ❌ Warning C4100: unreferenced formal parameter
void process(ProcessData& data) {
    // data not used - generates warning
}

// ✅ Fix: Mark unused parameter
void process(ProcessData& /*data*/) {
    // Clearly indicates intentional non-use
}

// ✅ Or use [[maybe_unused]] (C++17)
void process([[maybe_unused]] ProcessData& data) {
    // Attribute marks parameter as potentially unused
}
```

### Secure CRT Functions

MSVC recommends secure versions of C functions:

```cpp
// ❌ MSVC warning: unsafe function
strcpy(dest, src);

// ✅ Use secure version
strcpy_s(dest, destSize, src);

// ✅ Better: Use C++ strings
std::string dest = src;
```

### UTF-16 String Handling

Windows uses UTF-16 for wide strings:

```cpp
// VST3 uses TChar (char16_t)
Vst::String128 name;  // UTF-16 string

// Convert from UTF-8 if needed
#include "pluginterfaces/base/ustring.h"

void setName(const char* utf8Name) {
    UString128 ustr;
    ustr.fromAscii(utf8Name);
    // Use ustr...
}
```

---

## 15. Code Review Checklist

Before submitting code for review, verify:

### Formatting and Style

- [ ] Code formatted with `clang-format`
- [ ] No compiler warnings (`/W4` clean)
- [ ] Naming conventions followed (camelCase, PascalCase)
- [ ] File organization correct (includes, namespaces)

### VST3 Compliance

- [ ] VST3 interface methods use `PLUGIN_API` and `SMTG_OVERRIDE`
- [ ] Return `tresult` codes (not `bool` or `int`)
- [ ] VST3 types used (`ParamID`, `Sample32`, etc.)
- [ ] Unique GUIDs generated (not zeros)

### Real-Time Safety

- [ ] **No allocations in `process()` method**
- [ ] **No file I/O in audio thread**
- [ ] **No locks/mutexes in `process()`**
- [ ] Buffers pre-allocated in `initialize()`
- [ ] No exceptions thrown from `process()`

### Modern C++17

- [ ] Uses `nullptr` (not `NULL`)
- [ ] Uses `constexpr` (not `#define` for constants)
- [ ] Uses `enum class` (not plain `enum`)
- [ ] Smart pointers for ownership (`unique_ptr`, `shared_ptr`)
- [ ] RAII for resource management

### Thread Safety

- [ ] Parameter updates use atomics or lock-free structures
- [ ] Cross-thread communication properly synchronized
- [ ] No data races

### Documentation

- [ ] Doxygen comments for public API
- [ ] File headers present
- [ ] Complex algorithms documented
- [ ] Non-obvious code explained

### Testing

- [ ] Unit tests written for new DSP code
- [ ] Edge cases tested
- [ ] Parameter ranges validated
- [ ] No allocations verified in tests

### Performance

- [ ] No obvious performance issues
- [ ] Cache-friendly data structures used
- [ ] Branches minimized in hot loops

---

## Summary

These coding standards ensure the YourPlugin VST3 plugin is:

1. **Real-time safe** - No audio glitches or dropouts
2. **VST3 compliant** - Proper host integration
3. **Modern C++20** - Leveraging language features (span, concepts, ranges)
4. **Maintainable** - Code that is clear, consistent, and testable
5. **Performant** - Efficient audio processing

**Most Critical Rule**: Never allocate, block, or perform I/O in the `process()` method.

**Migration Note**: Project migrated from C++17 to C++20 on 2025-10-20 (ADR-010). All C++17 code remains valid in C++20.

**Questions?** Refer to:

- VST3 SDK documentation: `external/vst3sdk/doc`
- VST3 SDK examples: `external/vst3sdk/public.sdk/samples/vst/again`
- Project architecture: `oc/docs/00-project-master.md`

---

**End of Coding Standards Document**
