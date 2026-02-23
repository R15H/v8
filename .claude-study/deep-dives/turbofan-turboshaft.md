# TurboFan and Turboshaft: Top-Tier Optimizing Compilers

## Overview

TurboFan is V8's top-tier optimizing compiler, producing the highest quality machine code for hot JavaScript functions and WebAssembly. It uses a **sea-of-nodes** intermediate representation and performs aggressive optimizations including inlining, escape analysis, loop optimization, and advanced register allocation. **Turboshaft** is the next-generation IR and compiler framework that is progressively replacing TurboFan's sea-of-nodes backend while preserving its optimization capabilities.

Key source directory: `src/compiler/`

## TurboFan Architecture

### Sea-of-Nodes IR

TurboFan's central abstraction is the **sea-of-nodes** graph, where nodes represent both values and effects, connected by three kinds of edges:

1. **Value edges**: Data dependencies (node A uses the output of node B)
2. **Effect edges**: Ordering constraints for side-effecting operations
3. **Control edges**: Control flow dependencies

The `Node` class (`src/compiler/node.h`) is compact:

```cpp
class Node final {
  const Operator* op_;     // Defines the node's operation
  // Input edges stored inline (small nodes) or outline (large nodes)
  // Use list for tracking reverse edges
};
```

Each node carries an `Operator` that defines its opcode, properties (commutative, associative, no-throw, etc.), and input/output counts. Opcodes are defined in `src/compiler/opcodes.h` and cover:
- **JS-level operations**: `JSCall`, `JSLoadProperty`, `JSAdd`, etc.
- **Simplified operations**: `NumberAdd`, `StringConcat`, `LoadField`, etc.
- **Machine operations**: `Int32Add`, `Float64Mul`, `Load`, `Store`, etc.
- **Common operations**: `Branch`, `Merge`, `Phi`, `FrameState`, etc.

### Compilation Pipeline

The `Pipeline` class (`src/compiler/pipeline.h`) orchestrates TurboFan compilation through a series of phases:

1. **Graph Building** (`BytecodeGraphBuilder`): Walks bytecodes and feedback to build the initial JS-level graph
2. **Inlining** (`JSInliningHeuristic`): Inlines hot call targets based on feedback
3. **Type Narrowing**: Refines types based on runtime feedback
4. **Typed Lowering** (`JSTypedLowering`): Replaces JS operations with Simplified operations using type information
5. **Escape Analysis** (`EscapeAnalysis`): Detects non-escaping allocations for scalar replacement
6. **Load Elimination** (`LoadElimination`): Eliminates redundant loads
7. **Simplified Lowering** (`SimplifiedLowering`): Lowers Simplified operations to Machine operations, selecting representations (tagged, int32, float64)
8. **Generic Lowering** (`JSGenericLowering`): Replaces remaining JS operations with runtime calls
9. **Effect/Control Linearization**: Linearizes the sea-of-nodes into a scheduled form
10. **Late Optimization**: `MachineOperatorReducer`, `DeadCodeElimination`, `BranchElimination`
11. **Instruction Selection** (`InstructionSelector`): Maps machine nodes to architecture-specific instructions
12. **Register Allocation** (`LinearScanAllocator` or `GreedyAllocator`): Assigns physical registers
13. **Code Generation** (`CodeGenerator`): Emits native machine code

### Key Optimization Passes

**JS Call Reducer** (`src/compiler/js-call-reducer.cc`): One of TurboFan's most impactful passes. It recognizes and specializes calls to known built-in functions:
- `Array.prototype.map/filter/forEach` can be inlined as loops
- `Math.abs/floor/ceil` become machine-level operations
- `Object.is` becomes a reference comparison
- String and Promise operations are specialized

**Escape Analysis** (`src/compiler/escape-analysis.cc`): Detects objects that are allocated but never escape the current function scope. These allocations are eliminated ("scalar replaced"), with their fields stored in registers/stack slots instead.

**Branch Elimination** (`src/compiler/branch-elimination.cc`): Propagates branch conditions to eliminate redundant checks. If a branch confirms `x instanceof Foo`, subsequent instanceof checks on `x` can be eliminated.

**Redundancy Elimination** (`src/compiler/redundancy-elimination.cc`): Removes redundant bounds checks, map checks, and type guards when they have already been verified.

### Frame States and Deoptimization

TurboFan aggressively speculates but must be able to "undo" speculation via deoptimization. `FrameState` nodes capture the interpreter state at specific points, enabling the deoptimizer to reconstruct a valid interpreter frame if a speculation fails. This includes:
- Bytecode offset
- Register values (possibly computed lazily)
- Context chain
- Inlined frame stack

## Turboshaft

Turboshaft (`src/compiler/turboshaft/`) is V8's next-generation compiler backend, designed to replace TurboFan's sea-of-nodes IR with a more structured approach.

### Motivation

TurboFan's sea-of-nodes has drawbacks:
- Difficult to debug (no linear order during most of compilation)
- Expensive graph operations (modifying use lists)
- Hard to reason about memory layout
- Node identity changes during optimization can cause subtle bugs

### Turboshaft IR

Turboshaft uses an **operation buffer** -- a flat, append-only array of operations:

```cpp
class OperationBuffer {
  OperationStorageSlot* begin_;
  OperationStorageSlot* end_;
  uint16_t* operation_sizes_;  // For bidirectional iteration
};
```

Operations are referenced by `OpIndex` (an offset into the buffer) rather than pointers. This provides:
- **Cache-friendly iteration**: Operations are contiguous in memory
- **Stable identifiers**: OpIndex values do not change
- **Efficient allocation**: Append-only, no use-list maintenance

### Operations

Operations (`src/compiler/turboshaft/operations.h`) are defined as structs deriving from `OperationT<>` or `FixedArityOperationT<>`:

```cpp
struct FooOp : FixedArityOperationT<2, FooOp> {
  // Input accessors
  OpIndex left() const { return Base::input(0); }
  OpIndex right() const { return Base::input(1); }
  // Options
  auto options() const { return std::tuple{...}; }
};
```

The operation set covers similar ground to TurboFan but with a cleaner structure. Operations are categorized into:
- Block terminators (Branch, Return, Goto, Switch, Deoptimize)
- Simplified operations (arithmetic, comparisons, conversions)
- Machine operations (loads, stores, atomic operations)
- JS operations (calls, property access)
- WASM operations

### Blocks and Graph

The Turboshaft `Graph` (`src/compiler/turboshaft/graph.h`) contains:
- An `OperationBuffer` with all operations
- A `ZoneVector<Block>` organizing operations into basic blocks
- Type information side-tables
- Source position tables

Blocks contain operations in linear order. The last operation in each block is always a terminator.

### Reducer Pipeline

Turboshaft uses a **reducer** architecture for optimization. Reducers are composable classes that intercept operation creation:

```cpp
template <template<typename> typename... Reducers>
class Assembler;
```

When an operation is emitted through the assembler, each reducer in the chain gets a chance to modify, replace, or eliminate it. This is similar to TurboFan's `Reducer` pattern but more type-safe and composable.

Key reducers include:
- `BranchEliminationReducer` -- removes redundant branches
- `DeadCodeEliminationReducer` -- removes unreachable operations
- `DuplicationOptimizationReducer` -- CSE and deduplication
- `DataviewLoweringReducer` -- lowers DataView operations
- `FastApiCallLoweringReducer` -- optimizes Fast API calls
- `GrowableStacksReducer` -- handles growable stack frames
- `DecompressionOptimization` -- optimizes pointer decompression

### Phases

Turboshaft compilation is organized into phases:
- `BuildGraphPhase` -- converts TurboFan graph to Turboshaft graph (migration bridge)
- `CodeEliminationAndSimplificationPhase`
- `DecompressionOptimizationPhase`
- `BlockInstrumentationPhase` -- for code coverage
- `DebugFeatureLoweringPhase`
- CSA-specific phases for builtin compilation

### Migration Strategy

The TurboFan-to-Turboshaft migration is gradual:
1. **Backend first**: Turboshaft initially replaced TurboFan's backend (instruction selection onwards)
2. **Graph builder bridge**: `graph-builder.cc` converts TurboFan sea-of-nodes to Turboshaft IR
3. **Progressive frontend migration**: Optimization passes are being rewritten as Turboshaft reducers
4. **Builtin compilation**: The TSA (Turboshaft Assembler) provides a `CodeAssembler`-like API for writing builtins directly in Turboshaft (`src/compiler/turboshaft/builtin-compiler.h`)

### Turboshaft Assembler (TSA)

The Turboshaft Assembler provides a typed API for constructing Turboshaft graphs, analogous to TurboFan's `CodeStubAssembler`:

```cpp
// Example: Turboshaft-based bytecode handler
// Uses define-assembler-macros.inc for convenient operation emission
```

Some bytecode handlers are being rewritten using TSA, controlled by the `V8_ENABLE_EXPERIMENTAL_TSA_BUILTINS` flag. The `V_TSA` variant in `BYTECODE_LIST` marks handlers with Turboshaft alternative implementations.

## Instruction Selection

Both TurboFan and Turboshaft share the instruction selection backend (`src/compiler/backend/`):

- `InstructionSelector` maps IR operations to architecture-specific `Instruction` objects
- Architecture-specific instruction selectors in `src/compiler/backend/x64/`, `src/compiler/backend/arm64/`, etc.
- The `InstructionSequence` is the output, containing machine-level instructions with virtual registers

## Register Allocation

The register allocator (`src/compiler/backend/register-allocator.h`) operates on the `InstructionSequence`:

- **Live range analysis**: Computes the live interval of each virtual register
- **Linear scan allocation**: Fast allocation by processing intervals in order
- **Spill slot assignment**: When registers are exhausted, values spill to stack
- **Move resolution**: Inserts register-to-register and stack moves at block boundaries
- **Parallel moves**: Uses `GapInstruction` slots for move resolution

## WebAssembly Compilation

TurboFan also serves as the optimizing compiler for WebAssembly (see the WebAssembly deep-dive). The pipeline is adapted:
- No JavaScript-specific phases
- Direct WASM-to-machine lowering
- SIMD support through machine-level operations
- Different calling conventions

## Relationship to Other Tiers

```
Ignition -> Sparkplug -> Maglev -> TurboFan/Turboshaft
                                        ^
                                        |
                                   WASM Liftoff -> TurboFan/Turboshaft
```

TurboFan compiles only the hottest functions. Tier-up is triggered by:
- Maglev's interrupt budget checks on loop back-edges
- Explicit tier-up requests from the runtime
- OSR (on-stack replacement) for long-running loops

## Key Files

| File | Purpose |
|------|---------|
| `src/compiler/pipeline.h/.cc` | Compilation pipeline orchestration |
| `src/compiler/node.h` | TurboFan sea-of-nodes Node class |
| `src/compiler/opcodes.h` | TurboFan opcode definitions |
| `src/compiler/bytecode-graph-builder.h/.cc` | Bytecode-to-TF graph building |
| `src/compiler/js-call-reducer.h/.cc` | Built-in call specialization |
| `src/compiler/escape-analysis.h/.cc` | Escape analysis and scalar replacement |
| `src/compiler/simplified-lowering.cc` | Representation selection and lowering |
| `src/compiler/turboshaft/graph.h/.cc` | Turboshaft graph and operation buffer |
| `src/compiler/turboshaft/operations.h` | Turboshaft operation definitions |
| `src/compiler/turboshaft/assembler.h` | Typed graph construction API |
| `src/compiler/turboshaft/copying-phase.h/.cc` | Reducer-based optimization framework |
| `src/compiler/turboshaft/graph-builder.h/.cc` | TF-to-Turboshaft bridge |
| `src/compiler/backend/instruction-selector.h` | IR-to-machine instruction mapping |
| `src/compiler/backend/register-allocator.h` | Linear scan register allocation |
