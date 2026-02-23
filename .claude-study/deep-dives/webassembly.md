# WebAssembly Support in V8

## Overview

V8 provides a complete WebAssembly implementation with a multi-tier compilation strategy mirroring its JavaScript pipeline. WASM modules go through decoding, validation, and compilation via two tiers: **Liftoff** (baseline) and **TurboFan** (optimizing). V8 also provides full JavaScript-WebAssembly interop, SIMD support, and advanced features like streaming compilation and tiering.

Key source directory: `src/wasm/`

## Execution Tiers

Defined in `src/wasm/wasm-tier.h`:

```cpp
enum class ExecutionTier : int8_t {
  kNone,
  kLiftoff,     // Baseline compiler (fast compilation)
  kTurbofan,    // Optimizing compiler (high-quality code)
};
```

Optionally, Drumbrake (`kInterpreter`) provides an interpreter tier when `V8_ENABLE_DRUMBRAKE` is set.

The tiering strategy is:
1. **Liftoff first**: Functions are compiled quickly by Liftoff for fast startup
2. **Background TurboFan**: Hot functions are recompiled by TurboFan on background threads
3. **Tier-up**: When TurboFan code is ready, it replaces Liftoff code seamlessly

## Module Decoding

The `ModuleDecoder` (`src/wasm/module-decoder.h`, `module-decoder-impl.h`) parses the WASM binary format:

### Decoding Process

1. **Header validation**: Checks the magic number (`\0asm`) and version (1)
2. **Section parsing**: Processes standard sections in order:
   - Type section: Function signatures
   - Import section: Imported functions, tables, memories, globals
   - Function section: Maps function indices to type indices
   - Table section: Table definitions
   - Memory section: Memory definitions
   - Global section: Global variable definitions
   - Export section: Exported names
   - Start section: Start function index
   - Element section: Table element initializers
   - Code section: Function bodies
   - Data section: Data segment initializers
   - Custom sections: Name section, debug info, etc.

3. **Function body decoding**: Each function body is decoded separately, either eagerly or lazily

### Decoding Methods

```cpp
enum class DecodingMethod {
  kSync,       // Synchronous decoding
  kAsync,      // Asynchronous decoding
  kStreaming,  // Streaming compilation
};
```

Streaming compilation (`WebAssembly.compileStreaming`) begins compilation before the entire module has been downloaded, dramatically improving page load times.

### Result Types

```cpp
using ModuleResult = Result<std::shared_ptr<WasmModule>>;
using FunctionResult = Result<std::unique_ptr<WasmFunction>>;
```

## Liftoff: Baseline WASM Compiler

Liftoff (`src/wasm/baseline/`) is V8's single-pass baseline compiler for WebAssembly. It compiles WASM bytecode directly to machine code in a single forward pass, without building any IR.

### Design

- **Single-pass**: Walks the WASM function body once, emitting code as it goes
- **No IR**: Like Sparkplug for JavaScript, Liftoff avoids intermediate representation overhead
- **Register tracking**: Maintains a `LiftoffAssembler` state that tracks which WASM locals and stack values are in registers vs. spilled to the stack
- **Stack-machine model**: WASM is a stack machine; Liftoff tracks the operand stack and maps it to registers

### Key Components

The `LiftoffCompiler` (`src/wasm/baseline/liftoff-compiler.h/.cc`) processes WASM opcodes:

```cpp
struct LiftoffOptions {
  const int func_index;
  ForDebugging for_debugging;
  // Debugging support
  base::Vector<const int> breakpoints;
  std::unique_ptr<DebugSideTable>* debug_sidetable;
};
```

The `LiftoffAssembler` (`src/wasm/baseline/liftoff-assembler.h`) extends the platform-specific assembler with WASM-aware register management. It tracks:
- The WASM operand stack mapped to physical registers and stack slots
- Spill state for register pressure management
- Parallel moves for complex register shuffling

### Bailout Reasons

Liftoff may bail out to TurboFan for complex cases:

```cpp
enum LiftoffBailoutReason : int8_t {
  kSuccess = 0,
  kDecodeError = 1,
  kUnsupportedArchitecture = 2,
  kMissingCPUFeature = 3,
  kComplexOperation = 4,
  kSimd = 5,          // Some SIMD operations
  kRefTypes = 6,      // Some reference types operations
  kGC = 13,           // GC proposal operations
  // ...
};
```

### Platform Support

Platform-specific Liftoff code lives in subdirectories:
- `src/wasm/baseline/x64/` -- x86-64
- `src/wasm/baseline/arm64/` -- ARM64
- `src/wasm/baseline/arm/` -- ARM32
- `src/wasm/baseline/ia32/` -- x86 32-bit
- `src/wasm/baseline/riscv/`, `loong64/`, `mips64/`, `ppc/`, `s390/`

## TurboFan WASM Compilation

TurboFan serves as the optimizing compiler for WASM, producing high-quality code for hot functions. The WASM pipeline differs from the JavaScript pipeline:

### Compilation Path

1. **Graph building**: WASM bytecode is translated to TurboFan's sea-of-nodes IR (no JavaScript-specific nodes)
2. **Machine lowering**: WASM types map directly to machine types (i32->Int32, f64->Float64)
3. **Optimization**: Standard TurboFan passes (dead code elimination, common subexpression elimination, loop optimization)
4. **Instruction selection**: Architecture-specific instruction selection
5. **Register allocation**: Linear scan allocation
6. **Code generation**: Machine code emission

### Key Differences from JS Pipeline

- No speculation or deoptimization (WASM types are statically known)
- No JavaScript-specific lowering phases
- Direct memory access patterns (WASM linear memory)
- Trap handling for out-of-bounds accesses
- Different calling convention

## Module Compilation

The `ModuleCompiler` (`src/wasm/module-compiler.h`) orchestrates compilation of an entire module:

```cpp
std::shared_ptr<NativeModule> CompileToNativeModule(
    Isolate* isolate, WasmEnabledFeatures enabled_features,
    WasmDetectedFeatures detected_features, CompileTimeImports compile_imports,
    ErrorThrower* thrower, std::shared_ptr<const WasmModule> module,
    base::OwnedVector<const uint8_t> wire_bytes, int compilation_id,
    v8::metrics::Recorder::ContextId context_id, ProfileInformation* pgo_info);
```

### Compilation Strategy

1. **Eager Liftoff**: All functions are compiled with Liftoff for fast startup
2. **Lazy compilation**: Functions can be compiled on first call (`CompileLazy()`)
3. **Background TurboFan**: A compilation queue processes functions for optimization on background threads
4. **PGO (Profile-Guided Optimization)**: Profile information can guide compilation decisions

### NativeModule

The `NativeModule` manages compiled WASM code:
- Owns the code space for all compiled functions
- Manages the `WasmCode` objects (function code + metadata)
- Handles code patching during tier-up
- Thread-safe for concurrent access during background compilation

## Compilation Unit

```cpp
class WasmCompilationUnit {
  int func_index_;
  ExecutionTier tier_;
  ForDebugging for_debugging_;
};
```

A `WasmCompilationResult` captures the output:
```cpp
struct WasmCompilationResult {
  CodeDesc code_desc;
  uint32_t frame_slot_count;
  base::OwnedVector<uint8_t> source_positions;
  base::OwnedVector<uint8_t> protected_instructions_data;
  base::OwnedVector<uint8_t> deopt_data;
  ExecutionTier result_tier;
};
```

## JavaScript-WebAssembly Interop

The `wasm-js.cc` / `wasm-js.h` files implement the JavaScript API for WebAssembly:

### JS API Objects

- `WebAssembly.Module`: Represents a compiled module
- `WebAssembly.Instance`: A module instantiation with imports
- `WebAssembly.Memory`: A linear memory buffer
- `WebAssembly.Table`: A function reference table
- `WebAssembly.Global`: A global variable
- `WebAssembly.Tag`/`WebAssembly.Exception`: Exception handling

### Import/Export Wrappers

When JavaScript calls a WASM function (or vice versa), wrapper code handles the transition:

- **JS-to-WASM wrappers**: Convert JavaScript values to WASM types, set up the WASM stack frame, call the WASM function, convert return values back
- **WASM-to-JS wrappers**: Convert WASM values to JavaScript objects, call the JavaScript function, convert return values to WASM types

Import wrappers are cached in the `WasmImportWrapperCache` to avoid redundant compilation.

### Module Instantiation

`module-instantiate.cc` handles creating instances from compiled modules:
1. Allocate memory, tables, and globals
2. Link imports to their implementations
3. Initialize data segments and element segments
4. Run the start function (if present)

## SIMD Support

V8 supports the WebAssembly SIMD proposal (128-bit SIMD):

### Implementation

- **Liftoff**: Direct register-to-register SIMD operations on architectures that support it (x86 SSE/AVX, ARM NEON)
- **TurboFan**: SIMD operations map to machine-level vector instructions during instruction selection

### SIMD Shuffle

The `simd-shuffle.cc/.h` files handle SIMD shuffle operations, which rearrange vector lanes. The shuffle canonicalization identifies common patterns (e.g., blend, zip, unzip) that can be mapped to efficient machine instructions.

### Architecture-Specific SIMD

SIMD support varies by architecture:
- **x86-64**: SSE2/SSE4.1/AVX/AVX2 instructions
- **ARM64**: NEON instructions
- **ARM32**: NEON (limited SIMD support)
- Other architectures may bail out of Liftoff for SIMD-heavy code

## Trap Handling

WASM traps (division by zero, out-of-bounds memory access) are handled through:
1. **Signal handlers**: On supported platforms, out-of-bounds memory accesses trigger SIGSEGV/SIGBUS, caught by V8's trap handler (`src/trap-handler/`)
2. **Explicit bounds checks**: On other platforms, bounds checks are inserted before every memory access
3. **Protected instructions**: Metadata records which instructions may trap, enabling the signal handler to map the fault address to the correct WASM trap

## Effect Handlers

The `effect-handler.h` file defines effect handler support for structured concurrency proposals, tracking try/catch regions in WASM functions.

## Key Data Structures

### WasmModule
```cpp
struct WasmModule {
  std::vector<WasmFunction> functions;
  std::vector<FunctionSig*> types;
  std::vector<WasmImport> import_table;
  std::vector<WasmExport> export_table;
  std::vector<WasmTable> tables;
  std::vector<WasmMemory> memories;
  std::vector<WasmGlobal> globals;
};
```

### WasmCode
Each compiled function is a `WasmCode` object containing:
- Machine code instructions
- Source position table
- Protected instruction metadata
- Stack map information
- Relocation information

## Key Files

| File | Purpose |
|------|---------|
| `src/wasm/module-decoder.h/.cc` | WASM binary format parsing |
| `src/wasm/module-compiler.h/.cc` | Module-level compilation orchestration |
| `src/wasm/function-compiler.h/.cc` | Per-function compilation |
| `src/wasm/baseline/liftoff-compiler.h/.cc` | Liftoff baseline compiler |
| `src/wasm/baseline/liftoff-assembler.h/.cc` | Liftoff assembler with register tracking |
| `src/wasm/wasm-js.h/.cc` | JavaScript API implementation |
| `src/wasm/module-instantiate.h/.cc` | Module instantiation logic |
| `src/wasm/wasm-objects.h/.cc` | WASM heap object definitions |
| `src/wasm/wasm-code-manager.h/.cc` | Compiled code management |
| `src/wasm/simd-shuffle.h/.cc` | SIMD shuffle canonicalization |
| `src/wasm/compilation-environment.h` | Compilation configuration |
| `src/wasm/wasm-tier.h` | Execution tier definitions |
| `src/wasm/canonical-types.h/.cc` | Cross-module type canonicalization |
| `src/wasm/pgo.h/.cc` | Profile-guided optimization |
