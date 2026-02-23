# V8 Module Map

This document maps every major directory in V8's source tree to its responsibility and key files.

## Source Directory (`src/`)

### Core Execution

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/execution/` | Isolate management, call stack, runtime control | `isolate.h`, `frames.h`, `execution.h`, `stack-guard.h` |
| `src/init/` | Engine bootstrapping, global object setup | `bootstrapper.cc` (336 KB), `heap-symbols.h` (74 KB) |
| `src/api/` | Public C++ API implementation (implements `include/v8*.h`) | `api.cc`, `api-inl.h` |
| `src/d8/` | Standalone JavaScript shell (REPL) | `d8.cc`, `d8-console.cc` |

### Parsing & AST

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/parsing/` | Recursive-descent JavaScript parser, scanner | `parser.cc/h`, `preparser.h`, `scanner.h` |
| `src/ast/` | Abstract Syntax Tree node definitions | `ast.h`, `scopes.h`, `ast-value-factory.h` |

### Compilation Pipeline

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/interpreter/` | Ignition bytecode interpreter and bytecode generator | `bytecode-generator.cc/h`, `bytecodes.h`, `interpreter.cc` |
| `src/baseline/` | Tier-1 non-optimizing JIT compiler | `baseline-compiler.cc/h`, `baseline-assembler.h` |
| `src/maglev/` | Tier-2 mid-tier optimizing compiler | `maglev-compiler.cc/h`, `maglev-graph-builder.cc/h` |
| `src/compiler/` | TurboFan optimizing compiler (sea-of-nodes IR) | `pipeline.h`, `bytecode-graph-builder.h`, `js-operator.h` |
| `src/compiler/turboshaft/` | Next-gen compiler backend (replacing TurboFan internals) | `graph.h`, `operations.h`, `assembler.h` |
| `src/compiler-dispatcher/` | Concurrent compilation scheduling | `compiler-dispatcher.cc/h` |
| `src/codegen/` | Low-level code generation primitives | `assembler.h`, `code-stub-assembler.cc/h`, `code-factory.cc` |
| `src/deoptimizer/` | Bailout from optimized code to interpreter | `deoptimizer.cc/h`, `deopt-data.h` |

### Object Model

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/objects/` | All internal object representations (~250+ files) | `map.h`, `js-objects.h`, `heap-object.h`, `string.h`, `fixed-array.h` |
| `src/handles/` | GC-safe reference management | `handles.h`, `global-handles.h`, `local-handles.h` |
| `src/roots/` | Heap root object definitions | `roots.h`, `roots-table.h` |
| `src/ic/` | Inline caches for property access optimization | `ic.cc/h`, `accessor-assembler.cc/h`, `stub-cache.h` |

### Memory Management

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/heap/` | Garbage collection and heap management | `heap.h`, `mark-compact.cc`, `scavenger.cc`, `sweeper.h` |
| `src/heap/cppgc/` | C++ garbage collector (Oilpan) | `heap.h`, `marker.h`, `sweeper.h` |
| `src/zone/` | Bump allocator for temporary compilation data | `zone.h`, `zone-allocator.h` |

### Runtime & Standard Library

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/builtins/` | JavaScript built-in function implementations (Torque + C++) | 244 `.tq` files, `builtins-definitions.h` |
| `src/runtime/` | Runtime support functions callable from generated code | `runtime-array.cc`, `runtime-compiler.cc`, `runtime-object.cc` |
| `src/torque/` | Torque DSL compiler (`.tq` → C++) | `torque-compiler.cc`, `torque-parser.cc` |

### Strings & Numbers

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/strings/` | String interning, utilities, Unicode | `string-builder.h`, `unicode.h` |
| `src/numbers/` | Number-to-string conversion, math utilities | `conversions.h`, `dtoa.cc` |
| `src/bigint/` | Arbitrary-precision integer support | `bigint.h` |
| `src/date/` | Date/time implementation | `date.h` |
| `src/json/` | JSON parsing and serialization | `json-parser.cc`, `json-stringifier.cc` |

### Regular Expressions

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/regexp/` | Regular expression engine (bytecode + JIT) | `regexp-compiler.cc`, `regexp-interpreter.cc` |

### WebAssembly

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/wasm/` | WebAssembly compilation, execution, and JS interop | `c-api.cc`, `function-body-decoder.h`, `wasm-module.h` |
| `src/trap-handler/` | WASM trap signal handling | `trap-handler.h` |
| `src/asmjs/` | Legacy asm.js support | `asm-js.cc` |

### Debugging & Profiling

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/debug/` | Breakpoints, stepping, debug API | `debug-interface.cc/h`, `debug-scopes.cc` |
| `src/inspector/` | Chrome DevTools Protocol implementation | `v8-inspector-impl.cc` |
| `src/profiler/` | CPU and heap profiling | `cpu-profiler.cc`, `heap-profiler.cc` |
| `src/diagnostics/` | Diagnostic tools (disassembly, object printing) | `disassembler.cc`, `objects-printer.cc` |
| `src/logging/` | Event logging and counters | `counters.h`, `log.cc` |
| `src/tracing/` | Perfetto trace integration | `tracing-category-observer.cc` |
| `src/libsampler/` | Stack sampling for profiling | `sampler.cc` |

### Platform & Utilities

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/base/` | Platform abstraction (threads, mutexes, atomics, memory) | `platform.h`, `mutex.h`, `atomic-utils.h`, `bits.h` |
| `src/common/` | Global constants, types, assert macros | `globals.h` (100 KB), `checks.h`, `ptr-compr.h` |
| `src/flags/` | Command-line flag definitions and parsing | `flags.h`, `flag-definitions.h` |
| `src/utils/` | General utilities (vectors, hash maps) | `utils.h`, `bit-vector.h` |
| `src/tasks/` | Task scheduling for background work | `cancelable-task.h` |
| `src/libplatform/` | Default platform implementation | `default-platform.cc` |
| `src/extensions/` | V8 extension mechanism | `statistics-extension.cc` |

### Security

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/sandbox/` | Pointer compression, sandboxing, security hardening | `sandbox.h`, `external-pointer-table.h` |

### Snapshots & Serialization

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/snapshot/` | Heap serialization/deserialization for fast startup | `mksnapshot.cc`, `serializer.cc`, `deserializer.cc` |

### Fuzzing

| Directory | Responsibility | Key Files |
|-----------|---------------|-----------|
| `src/fuzzilli/` | Fuzzilli fuzzer integration | `fuzzilli.cc` |
| `src/dumpling/` | Core dump support | `dumpling.cc` |

## Public API (`include/`)

| Directory/File | Responsibility |
|---------------|---------------|
| `include/v8.h` | Master include (pulls in all `v8-*.h` headers) |
| `include/v8-isolate.h` | Isolate creation and management |
| `include/v8-context.h` | Context (global scope) creation |
| `include/v8-script.h` | Script compilation and execution |
| `include/v8-value.h` | Base value type for all JS values |
| `include/v8-object.h` | Object property access |
| `include/v8-function.h` | Function call and construction |
| `include/v8-exception.h` | Exception handling (TryCatch) |
| `include/v8-template.h` | ObjectTemplate and FunctionTemplate for native bindings |
| `include/v8-platform.h` | Platform abstraction for embedders |
| `include/v8-wasm.h` | WebAssembly API |
| `include/v8-profiler.h` | Profiling API |
| `include/v8-inspector.h` | DevTools debugging protocol API |
| `include/cppgc/` | C++ garbage collection (Oilpan) public API |
| `include/libplatform/` | Default platform implementation API |

## Test Infrastructure (`test/`)

| Directory | Type | Framework |
|-----------|------|-----------|
| `test/mjsunit/` | JavaScript unit tests | Custom JS test runner |
| `test/cctest/` | C++ component tests | Custom C++ framework |
| `test/unittests/` | C++ unit tests | Google Test (gtest) |
| `test/test262/` | ECMAScript conformance | Test262 harness |
| `test/inspector/` | DevTools protocol tests | Inspector test harness |
| `test/wasm-spec-tests/` | WASM specification compliance | WASM test harness |
| `test/wasm-js/` | WASM-JS interop tests | JS test runner |
| `test/js-perf-test/` | Performance benchmarks | Custom benchmark runner |
| `test/intl/` | Internationalization tests | JS test runner |
| `test/fuzzer/` | Fuzzing harnesses | LibFuzzer / custom |
| `test/message/` | Error message verification | Message comparison |
| `test/webkit/` | WebKit compatibility | JS test runner |
| `test/mozilla/` | Mozilla compatibility | JS test runner |

## Tools (`tools/`)

| Tool | Purpose |
|------|---------|
| `tools/callstats.py` | Call frequency analysis |
| `tools/builtins-pgo/` | Profile-guided optimization for builtins |
| `tools/debug_helper/` | Debugging helper library |
| `tools/clusterfuzz/` | ClusterFuzz fuzzing integration |
| `tools/android-build.sh` | Android cross-compilation |
| `tools/bash-completion.sh` | Shell completion for d8 |

## Third-Party Dependencies (`third_party/`)

| Dependency | Purpose |
|------------|---------|
| `abseil-cpp/` | Google Abseil C++ utilities |
| `icu/` | International Components for Unicode |
| `googletest/` | Google Test framework |
| `inspector_protocol/` | Chrome DevTools Protocol generator |
| `perfetto/` | Performance tracing |
| `re2/` | Regular expression library |
| `zlib/` | Compression |
| `highway/` | SIMD library |
| `llvm-libc/` | LLVM C library components |
