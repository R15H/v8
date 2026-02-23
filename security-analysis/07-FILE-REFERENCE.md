# Security-Critical File Reference

Quick-reference index of files most relevant to V8 security research.
Files are organized by attack surface and annotated with specific line
references for critical functions.

## JIT Compiler - Type System

| File | Size | Key Functions / Lines | Vuln Ref |
|------|------|-----------------------|----------|
| `src/compiler/turboshaft/typer.h` | 1,625 lines | `Multiply()` **:613-642** (BUG at :620), `Add()`, `Subtract()`, `Divide()`, `allow_invalid_inputs()` **:1619** | VULN-001, VULN-002 |
| `src/compiler/turbofan-typer.cc` | 2,787 lines | `TypeNode()`, type visitors for all operations | - |
| `src/compiler/operation-typer.cc` | ~1,500 lines | `CheckBounds()` **:1344-1353**, `NumberMultiply()` **:760-774** (correct version), `WeakenRange()` **:47-123** | VULN-003 |
| `src/compiler/turbofan-types.cc` | 1,310 lines | `Range()`, `Union()`, `Intersect()`, `Min()`, `Max()` | - |
| `src/compiler/turbofan-types.h` | ~800 lines | Type bitset hierarchy, type class definitions | - |
| `src/compiler/type-cache.h` | ~200 lines | `kPositiveSafeInteger`, `kZeroish`, `kSingletonZero` | - |

## JIT Compiler - Optimization Passes

| File | Size | Key Functions / Lines | Vuln Ref |
|------|------|-----------------------|----------|
| `src/compiler/typed-optimization.cc` | 1,006 lines | `ReduceCheckBounds()` **:206-220**, `ReduceCheckNotTaggedHole()` **:223-231**, `ReduceCheckMaps()` **:233+** | - |
| `src/compiler/simplified-lowering.cc` | ~6,000 lines | 3-phase algorithm **:63-84**, `CanOverflowSigned32()` **:216-242**, representation selection | - |
| `src/compiler/escape-analysis.cc` | ~800 lines | Escape analysis, object materialization | - |
| `src/compiler/load-elimination.cc` | large | Redundant load elimination (can remove checks) | - |

## JIT Compiler - Pipeline

| File | Size | Key Functions / Lines |
|------|------|-----------------------|
| `src/compiler/pipeline.cc` | very large | `OptimizeTurbofanGraph()` **:1991-2099** (phase ordering), type removal **:2066-2077** |
| `src/compiler/common-operator.cc` | | `DeoptimizeIf`, `DeoptimizeUnless` node creation |
| `src/deoptimizer/deoptimizer.cc` | large | Deoptimization frame materialization |
| `src/deoptimizer/deoptimize-reason.h` | | Deoptimization reason enum |

## Maglev Compiler

| File | Size | Key Functions / Lines |
|------|------|-----------------------|
| `src/maglev/maglev-compiler.cc` | | Compiler entry point |
| `src/maglev/maglev-graph-builder.cc` | | Graph construction from bytecode + feedback |
| `src/maglev/maglev-range-analysis.h` | | `NodeRanges` class **:47-100**, int64 range analysis |
| `src/maglev/maglev-range.h` | | Range class (int64-based) |

## Turboshaft Compiler

| File | Size | Key Functions / Lines |
|------|------|-----------------------|
| `src/compiler/turboshaft/typer.h` | 1,625 lines | `WordOperationTyper` **:50-403**, `FloatOperationTyper` **:405-1140** |
| `src/compiler/turboshaft/type-inference-reducer.h` | | Type propagation during Turboshaft optimization |
| `src/compiler/turboshaft/machine-lowering-reducer-inl.h` | | Machine operation lowering |
| `src/compiler/turboshaft/simplified-lowering-reducer-inl.h` | | Simplified operation lowering |

## Sandbox

| File | Size | Key Functions / Lines | Vuln Ref |
|------|------|-----------------------|----------|
| `src/sandbox/sandbox.h` | 379 lines | `Sandbox` class, `Contains()`, `is_partially_reserved()` **:113**, `kFallbackToPartiallyReservedSandboxAllowed` **:72** | VULN-004 |
| `src/sandbox/sandbox.cc` | | `Initialize()`, `FinishInitialization()`, partial reservation **:126** | VULN-004 |
| `src/sandbox/check.h` | 76 lines | `SBXCHECK` macro **:37-44**, TOCTOU warning **:26-31** | - |
| `src/sandbox/external-pointer-table.h` | | `ExternalPointerTableEntry`, type-tagged entries | - |
| `src/sandbox/code-pointer-table.h` | | Code pointer indirection | - |
| `src/sandbox/trusted-pointer-table.h` | | Trusted pointer indirection | - |
| `src/sandbox/js-dispatch-table.h` | | JS function dispatch indirection | - |
| `src/sandbox/testing.h` | 148 lines | `MemoryCorruptionApi`, crash filter, testing modes | - |
| `src/sandbox/bytecode-verifier.cc` | 12KB | Bytecode security validation | - |
| `src/sandbox/GLOSSARY.md` | 141 lines | Security properties of pointer types | - |
| `src/sandbox/hardware-support.h` | | Hardware sandbox support, **:252** (review needed) | - |

## Garbage Collector

| File | Size | Key Functions / Lines |
|------|------|-----------------------|
| `src/heap/heap.cc` | 297KB | Main heap implementation |
| `src/heap/heap-write-barrier.cc` | 25KB | Write barrier implementation |
| `src/heap/heap-write-barrier-inl.h` | | Inline fast-path barriers |
| `src/heap/WRITE_BARRIER.md` | | Write barrier design documentation |
| `src/heap/scavenger.cc` | | Young generation (copying) GC |
| `src/heap/mark-compact.cc` | | Old generation GC |
| `src/heap/concurrent-marking.cc` | | Background concurrent marking |
| `src/heap/marking-barrier.cc` | | Write barriers for marking phase |

## Object Model

| File | Size | Key Functions / Lines |
|------|------|-----------------------|
| `src/objects/elements-kind.h` | ~200 lines | Element kind enum (30+ kinds), transitions |
| `src/objects/elements.cc` | | Element accessor implementations |
| `src/objects/map.h` | large | Map (hidden class), **:650** security token |
| `src/objects/js-array.h` | | JSArray definition |
| `src/objects/js-array-buffer.h` | | ArrayBuffer, SharedArrayBuffer |
| `src/objects/js-typed-array.h` | | TypedArray definitions |
| `src/objects/js-objects.h` | | **:1195** JSGlobalProxy security note |
| `src/heap/factory.cc` | 214KB | Object allocation |

## Runtime Functions

| File | Role | Vuln Ref |
|------|------|----------|
| `src/runtime/runtime-array.cc` | Array.prototype methods | VULN-006 |
| `src/runtime/runtime-object.cc` | Object operations | VULN-006 |
| `src/runtime/runtime-strings.cc` | String operations | VULN-006 |
| `src/runtime/runtime-typedarray.cc` | TypedArray operations | VULN-006 |
| `src/runtime/runtime-wasm.cc` | WASM support | VULN-006 |

## Error Handling

| File | Key Functions / Lines | Vuln Ref |
|------|-----------------------|----------|
| `src/base/logging.cc` | `FatalNoSecurityImpact()` **:101**, abort logic **:94** | VULN-005 |
| `src/base/abort-mode.h` | `AbortMode::kExitIfNoSecurityImpact` **:34**, `FatalErrorsWithNoSecurityImpactShouldExit()` **:56** | VULN-005 |

## WebAssembly

| File | Role |
|------|------|
| `src/wasm/module-decoder.cc` | Module structure validation |
| `src/wasm/function-body-decoder.cc` | Function body validation |
| `src/wasm/wasm-js.cc` | JS ↔ WASM boundary |
| `src/wasm/wasm-engine.cc` | Engine management |
| `src/trap-handler/trap-handler.h` | Hardware trap handling |

## RegExp

| File | Role | Vuln Ref |
|------|------|----------|
| `src/regexp/regexp-compiler.cc` | RegExp to native code | VULN-007 |
| `src/regexp/regexp-interpreter.cc` | RegExp bytecode interpreter | VULN-007 |
| `src/regexp/regexp-parser.cc` | RegExp pattern parser | VULN-007 |

## Builtins (CodeStubAssembler / Torque)

| File | Role |
|------|------|
| `src/builtins/builtins-array.cc` | Array builtins |
| `src/builtins/builtins-typed-array.cc` | TypedArray builtins |
| `src/builtins/builtins-string.cc` | String builtins |
| `src/builtins/builtins-collections-gen.cc` | Map/Set builtins, **:2111,3148,3262** CSA_HOLE_SECURITY_CHECK |
| `src/builtins/accessors.cc` | Property accessors, **:552** security token check |
| `src/codegen/code-stub-assembler.cc` | CSA base (barrier emission, etc.) |

## Security Infrastructure

| File | Role |
|------|------|
| `src/ic/accessor-assembler.cc` | Inline cache handling, **:1368-1372** security token comparison |
| `src/objects/contexts.h` | **:318** SECURITY_TOKEN_INDEX, **:649** `HasSameSecurityTokenAs()` |
| `src/common/code-memory-access.h` | **:78** JIT entitlement (macOS) |
| `src/common/code-memory-access.cc` | **:334** CFI data untrusted |
| `src/common/segmented-table-inl.h` | **:111** attacker-controlled data warning |

## Useful V8 Flags for Security Research

```
--trace-turbo              # Dump TurboFan IR to turbo-*.json
--trace-turbo-types        # Show types in TurboFan IR dumps
--trace-opt                # Log optimization events
--trace-deopt              # Log deoptimization events
--print-bytecode           # Print Ignition bytecode
--allow-natives-syntax     # Enable %DebugPrint, %OptimizeFunctionOnNextCall, etc.
--turboshaft               # Force Turboshaft (may be default in 14.7)
--no-turboshaft            # Disable Turboshaft, use classic TurboFan
--maglev                   # Enable Maglev
--no-maglev                # Disable Maglev
--jit-fuzzing              # Enable JIT fuzzing mode
--sandbox-testing           # Enable sandbox testing mode
--sandbox-fuzzing          # Enable sandbox fuzzing mode
```
