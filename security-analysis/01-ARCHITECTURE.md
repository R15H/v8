# V8 Architecture & Data Flows (Security Perspective)

## Compilation Pipeline

V8 uses a multi-tier compilation architecture. Each tier is a potential attack
surface because bugs in any tier's type system or optimization passes can lead
to incorrect code generation.

```
JavaScript Source Code
        │
        ▼
┌─────────────────────┐
│  Parser / Scanner    │  src/parsing/ (32 files)
│  AST Construction    │  Key: parser.cc, scanner.cc
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Ignition           │  src/interpreter/ (52 files)
│  (Bytecode          │  Key: bytecode-generator.cc, interpreter.cc
│   Interpreter)      │  Generates & executes bytecode; collects type feedback
└─────────┬───────────┘
          │  Type feedback (IC = Inline Cache)
          ▼
┌─────────────────────┐
│  Sparkplug           │  src/baseline/ (28 files)
│  (Baseline Compiler) │  Key: baseline-compiler.cc
│  No optimization     │  1:1 bytecode-to-machine-code, no type speculation
└─────────┬───────────┘
          │  Hot function detected
          ▼
┌─────────────────────┐
│  Maglev             │  src/maglev/ (86 files)
│  (Mid-Tier JIT)     │  Key: maglev-compiler.cc, maglev-graph-builder.cc
│  SSA, type feedback  │  Uses int64 ranges (safer than TurboFan's doubles)
│  Speculative opts    │  maglev-range-analysis.h for range analysis
└─────────┬───────────┘
          │  Very hot function
          ▼
┌─────────────────────────────────────────────────────┐
│  TurboFan / Turboshaft  (Full Optimizing JIT)       │
│                                                     │
│  TurboFan: src/compiler/ (497 files)                │
│    Key: pipeline.cc (orchestrator)                   │
│    Key: turbofan-typer.cc (type inference)           │
│    Key: operation-typer.cc (semantic operations)     │
│    Key: simplified-lowering.cc (representation)      │
│    Key: typed-optimization.cc (bounds check elim)    │
│                                                     │
│  Turboshaft: src/compiler/turboshaft/ (116 files)    │
│    Key: typer.h (type system)                        │
│    Key: type-inference-reducer.h                     │
│    Key: machine-lowering-reducer-inl.h               │
│                                                     │
│  [!] PRIMARY ATTACK SURFACE FOR TYPE CONFUSION       │
└─────────┬───────────────────────────────────────────┘
          │
          ▼
┌─────────────────────┐
│  Code Generation     │  src/codegen/ (246 files)
│  Register Allocator  │  Architecture-specific backends
│  Machine Code        │  x64, arm64, arm, ia32, riscv, loong64, mips64, s390, ppc
└─────────────────────┘
```

## TurboFan Optimization Pipeline (Detailed)

The pipeline is defined in `src/compiler/pipeline.cc:1991-2099`
(`PipelineImpl::OptimizeTurbofanGraph`):

```
Phase                          │ File                              │ Security Relevance
───────────────────────────────┼───────────────────────────────────┼──────────────────────
EarlyGraphTrimming             │ graph-trimming.cc                 │ LOW - removes dead nodes
TyperPhase                     │ turbofan-typer.cc                 │ CRITICAL - type inference
TypedLoweringPhase             │ js-typed-lowering.cc              │ HIGH - JS→Simplified ops
LoopPeeling / LoopExitElim     │ loop-peeling.cc                   │ MEDIUM - loop transforms
LoadEliminationPhase           │ load-elimination.cc               │ HIGH - can remove checks
EscapeAnalysisPhase            │ escape-analysis.cc                │ HIGH - object elimination
SimplifiedLoweringPhase        │ simplified-lowering.cc            │ CRITICAL - representation
  (3 phases: PROPAGATE/RETYPE/LOWER)
GenericLoweringPhase           │ generic-lowering.cc               │ MEDIUM
EarlyOptimizationPhase         │ machine-operator-reducer.cc       │ MEDIUM
SchedulePhase                  │ scheduler.cc                      │ LOW
```

### Critical observation (pipeline.cc:2066-2073):
After SimplifiedLowering, **types are removed from nodes in debug builds**.
This means type information used for optimization is no longer verifiable
after this point. If the typer produced incorrect types, the damage is done
and invisible.

## Memory Model

```
V8 Heap Layout
├── New Space (Young Generation)     - Scavenger GC
│   ├── From-Space
│   └── To-Space
├── Old Space (Old Generation)       - Mark-Compact GC
├── Code Space                       - Executable code
│   └── (Protected by W^X)
├── Large Object Space               - Objects > kMaxRegularHeapObjectSize
├── Read-Only Space                  - Immortal objects (maps, constants)
├── Trusted Space                    - Objects outside sandbox
│   └── (Code, BytecodeArray, etc.)
└── Shared Space                     - Cross-isolate shared objects
```

### Key Files:
- `src/heap/heap.cc` (297KB) - Main heap implementation
- `src/heap/heap-write-barrier.cc` (25KB) - Write barrier logic
- `src/heap/scavenger.cc` - Young generation GC
- `src/heap/mark-compact.cc` - Old generation GC
- `src/heap/concurrent-marking.cc` - Concurrent marking
- `src/heap/WRITE_BARRIER.md` - Write barrier documentation

## Object Model

All JavaScript objects in V8 are represented as `HeapObject` instances with
a `Map` (hidden class) that describes their layout.

```
HeapObject Layout:
┌─────────────────────────┐
│  Map pointer (tagged)    │  → Points to Map describing object layout
├─────────────────────────┤
│  Properties (tagged)     │  → PropertyArray or NameDictionary
├─────────────────────────┤
│  Elements (tagged)       │  → FixedArray, FixedDoubleArray, etc.
├─────────────────────────┤
│  In-object properties    │  → Stored directly in the object
└─────────────────────────┘

Map (Hidden Class):
┌─────────────────────────┐
│  instance_type           │  → What kind of object this is
│  instance_size           │  → Size of instances
│  elements_kind           │  → How elements are stored (30+ kinds)
│  prototype               │  → Prototype chain link
│  transitions             │  → Map transition tree
│  descriptors             │  → Property descriptors
│  back_pointer            │  → Previous map in transition chain
└─────────────────────────┘
```

### Element Kinds (`src/objects/elements-kind.h`):

Element kind transitions are a major source of type confusion bugs. The kinds
form a lattice with transitions that must preserve invariants:

```
PACKED_SMI_ELEMENTS          (all elements are small integers)
     │
     ├──→ HOLEY_SMI_ELEMENTS       (SMIs with holes)
     │         │
     ▼         ▼
PACKED_DOUBLE_ELEMENTS       HOLEY_DOUBLE_ELEMENTS
     │                            │
     ▼                            ▼
PACKED_ELEMENTS (any)        HOLEY_ELEMENTS (any with holes)

TypedArray Element Kinds:
UINT8_ELEMENTS, INT8_ELEMENTS, UINT16_ELEMENTS, ...
FLOAT32_ELEMENTS, FLOAT64_ELEMENTS, FLOAT16_ELEMENTS

RAB/GSAB (Resizable) TypedArray kinds:
RAB_GSAB_UINT8_ELEMENTS, RAB_GSAB_INT8_ELEMENTS, ...
(These are security-sensitive: backing store can be resized/detached)
```

**Security-critical invariant**: Once an array transitions from PACKED to HOLEY,
it must never go back. If it does, the engine will skip hole checks and read
uninitialized memory.

## Sandbox Architecture

```
V8 Sandbox (1 TB virtual address space)
┌──────────┬──────────────────────────────────────────────┬──────────┐
│  Guard   │                                              │  Guard   │
│  Region  │   4GB Heap Region  │  ArrayBuffer stores,    │  Region  │
│  (32GB)  │   (compressed ptrs)│  WASM memories, etc.    │  (32GB)  │
└──────────┴──────────────────────────────────────────────┴──────────┘
                                 ▲
    Pointer Tables              │
    ┌──────────────────┐        │   Pointer indirection prevents
    │ External Ptr Tbl │────────┘   direct pointer manipulation
    │ Code Ptr Table   │            from inside the sandbox
    │ Trusted Ptr Tbl  │
    │ CppHeap Ptr Tbl  │
    │ JS Dispatch Tbl  │
    └──────────────────┘
```

### Key sandbox files:
- `src/sandbox/sandbox.h` - Main sandbox class
- `src/sandbox/check.h` - SBXCHECK macro (security-critical checks)
- `src/sandbox/external-pointer-table.h` - External pointer indirection
- `src/sandbox/code-pointer-table.h` - Code pointer indirection
- `src/sandbox/trusted-pointer-table.h` - Trusted pointer indirection
- `src/sandbox/testing.h` - Memory Corruption API for testing
- `src/sandbox/bytecode-verifier.cc` - Bytecode validation

## Deoptimization

When speculative optimizations fail (type feedback was wrong), V8 must
"deoptimize" - transfer execution back to unoptimized code. This is
security-critical because:

1. If deopt doesn't fire when it should, speculative code runs on
   unexpected types → type confusion
2. The deopt mechanism itself must correctly reconstruct the frame state

Key files:
- `src/deoptimizer/deoptimizer.cc` (23 files in `src/deoptimizer/`)
- `src/compiler/common-operator.cc` - DeoptimizeIf, DeoptimizeUnless nodes

## WebAssembly

```
WASM Compilation:
  .wasm module
       │
       ▼
┌─────────────────┐
│ Module Decoder   │  src/wasm/module-decoder.cc
│ (Validation)     │  Validates structure + types
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌──────────────┐
│ Liftoff │ │ TurboFan/    │
│ (Base)  │ │ Turboshaft   │
│         │ │ (Optimizing) │
└────────┘ └──────────────┘
```

Key files:
- `src/wasm/` (161 files)
- `src/wasm/module-decoder.cc` - Module validation
- `src/wasm/function-body-decoder.cc` - Function body validation
- `src/wasm/wasm-engine.cc` - Engine management
- `src/wasm/wasm-js.cc` - JS ↔ WASM boundary
- `src/trap-handler/` - Hardware trap handling for bounds checks
