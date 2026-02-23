# Maglev: The Mid-Tier Optimizing Compiler

## Overview

Maglev is V8's mid-tier optimizing compiler, introduced in V8 v11.3 (2023). It occupies the space between Sparkplug (baseline) and TurboFan (top-tier optimizer) in V8's compilation pipeline. Maglev generates moderately optimized code much faster than TurboFan, providing a better speed/compilation-cost tradeoff for functions that are warm but not hot enough to justify full TurboFan optimization.

Key source directory: `src/maglev/`

## Design Goals

- **Fast compilation**: 5-10x faster than TurboFan, suitable for mid-warmth functions
- **Moderate optimization**: Type specialization, check elimination, and inlining based on feedback, but without TurboFan's heavy sea-of-nodes machinery
- **Concurrent compilation**: Compilation can happen on background threads
- **Feedback-driven**: Directly consumes type feedback from the FeedbackVector collected by Ignition/Sparkplug

## Graph IR

Maglev uses a **basic-block graph** of SSA nodes, distinct from TurboFan's sea-of-nodes. The graph structure is defined in `src/maglev/maglev-graph.h`:

```cpp
class Graph final : public ZoneObject {
  ZoneVector<BasicBlock*> blocks_;
  // Constant pools
  ZoneMap<RootIndex, RootConstant*> root_constants_;
  ZoneMap<int, SmiConstant*> smi_constants_;
  ZoneMap<int, Int32Constant*> int32_constants_;
  ZoneMap<double, Float64Constant*> float64_constants_;
  // ...
  MaglevCallSiteCandidates inlineable_calls_;
};
```

### BasicBlock

Each `BasicBlock` contains a list of `Node` objects (the body) and ends with a control node. Blocks are organized in RPO (reverse post-order). Merge points hold `MergePointInterpreterFrameState` objects that reconcile register state from multiple predecessors via phi nodes.

### Node Hierarchy

Nodes are defined in `src/maglev/maglev-ir.h` using extensive macro lists. The hierarchy:

1. **Value Nodes**: Produce SSA values
   - **Constants**: `Constant`, `SmiConstant`, `Float64Constant`, `Int32Constant`, `RootConstant`, etc.
   - **Arithmetic**: `Int32Add`, `Float64Multiply`, `CheckedSmiTagInt32`, etc.
   - **Generic Operations**: `GenericAdd`, `GenericMultiply`, etc. (fall back to runtime)
   - **Property Access**: `LoadTaggedField`, `LoadFixedArrayElement`, `LoadNamedGeneric`, `SetKeyedGeneric`, etc.
   - **Calls**: `Call`, `CallBuiltin`, `CallKnownJSFunction`, `Construct`, etc.
   - **Type Checks**: `CheckMaps`, `CheckSmi`, `CheckNumber`, `CheckString`, etc.
   - **Conversions**: `CheckedSmiUntag`, `ChangeInt32ToFloat64`, `Float64ToTagged`, etc.
   - **Allocation**: `InlinedAllocation`, `AllocationBlock`, `CreateShallowObjectLiteral`
   - **Phi**: `Phi` nodes at merge points

2. **Non-Value Nodes**: Side-effecting operations without a result
   - `StoreTaggedFieldNoWriteBarrier`, `StoreFixedArrayElementWithWriteBarrier`
   - `GeneratorStore`, `CheckMapsWithMigration`

3. **Control Nodes**: Block terminators
   - `Jump`, `JumpLoop`, `BranchIfTrue`, `BranchIfToBooleanTrue`
   - `Switch`, `Return`, `Deopt`, `JumpToInlined`, `DeferredCodeEntry`

### Type Representation System

Maglev tracks **representations** for values: Tagged, Int32, Uint32, Float64, HoleyFloat64. The `maglev-phi-representation-selector.h` pass selects optimal representations for phi nodes, enabling unboxed computation in loops. This is a key optimization: if a loop variable is always a small integer, Maglev keeps it as an Int32 without boxing/unboxing overhead.

### Node Types

The `NodeType` system (`maglev-node-type.h`) tracks type information more precisely than representations:
- Smi, HeapNumber, Number, String, Symbol, Boolean, JSReceiver, etc.
- This enables check elimination: if a node is known to be a Smi, a `CheckSmi` can be removed

## Graph Building

The `MaglevGraphBuilder` (`src/maglev/maglev-graph-builder.h`) walks the bytecode array and builds the graph:

1. **Bytecode Iteration**: Processes each bytecode in order, maintaining an `InterpreterFrameState` that maps interpreter registers to graph nodes.

2. **Feedback Consumption**: For each bytecode with a feedback slot, the builder queries the `FeedbackVector` via the `JSHeapBroker` to determine observed types, maps, and call targets.

3. **Specialization**: Based on feedback, the builder emits specialized nodes:
   - If a property load always sees the same map, emit `CheckMaps` + `LoadTaggedField` instead of `LoadNamedGeneric`
   - If addition always produces small integers, emit `Int32AddWithOverflow` instead of `GenericAdd`
   - If a call always targets the same function, emit `CallKnownJSFunction` with potential inlining

4. **Known Node Aspects** (`maglev-known-node-aspects.h`): Tracks type information, stable maps, and available loads across the graph. This enables redundant check elimination and load elimination during graph building.

5. **Merge Points**: At join points (after if/else, loop headers), the builder creates `MergePointInterpreterFrameState` objects that insert phi nodes for values that differ between predecessors.

### Inlining

Maglev supports inlining (`src/maglev/maglev-inlining.h`), controlled by multiple budget parameters in `CompilationFlags`:
- `max_eager_inlined_bytecode`: Size limit for eagerly inlined callees
- `max_inlined_bytecode_size_cumulative`: Total inlined bytecode budget
- `max_inline_depth`: Maximum nesting depth

The `MaglevCallSiteCandidates` priority queue ranks call sites by score for inlining. After graph building, top candidates are inlined by recursively building their graphs and splicing them into the caller's graph.

## Optimization Passes

After graph building, Maglev runs several optimization passes:

1. **Graph Optimizer** (`maglev-graph-optimizer.h`): Generic optimization pass that processes nodes using a graph processor pattern.

2. **Phi Representation Selection** (`maglev-phi-representation-selector.h`): Selects unboxed representations (Int32, Float64) for phi nodes when all inputs and uses are compatible.

3. **Post-Hoc Optimizations** (`maglev-post-hoc-optimizations-processors.h`): Additional simplifications after the main optimization pass.

4. **Escape Analysis**: Tracks allocations that do not escape the function, enabling scalar replacement. Traced via `--trace-maglev-escape-analysis`.

5. **Range Analysis** (`maglev-range-analysis.h`): Computes integer value ranges to eliminate bounds checks and overflow checks.

6. **KNA (Known Node Aspects) Processing** (`maglev-kna-processor.h`): Propagates known type information and eliminates redundant checks.

## Register Allocation

Maglev uses a **linear scan register allocator** defined across `maglev-regalloc*.h/.cc` files. Key aspects:

- Processes blocks in reverse post-order
- Allocates physical registers for virtual registers
- Handles spilling to stack slots when registers are exhausted
- Resolves parallel moves at block boundaries
- Special handling for fixed register constraints (e.g., call arguments)

The `MaglevVregAllocationState` assigns virtual registers, and `maglev-pre-regalloc-codegen-processors.h` runs pre-allocation preparation.

## Code Generation

The `MaglevCodeGenerator` (`src/maglev/maglev-code-generator.h/.cc`) emits machine code from the graph:

1. Walks blocks in order, emitting code for each node
2. Each node class has a `GenerateCode(MaglevAssembler*, ProcessingState&)` method
3. Uses `MaglevAssembler` (extends `MacroAssembler`) with platform-specific implementations in `src/maglev/arm64/`, `src/maglev/x64/`, etc.
4. Handles deoptimization metadata generation
5. Creates the final `Code` object with safepoint information

## Concurrent Compilation

Maglev supports concurrent compilation via `MaglevConcurrentDispatcher`:

```cpp
class MaglevConcurrentDispatcher {
  void EnqueueJob(std::unique_ptr<MaglevCompilationJob>&& job);
  void FinalizeFinishedJobs();
  QueueT incoming_queue_;
  QueueT outgoing_queue_;
};
```

The compilation pipeline has three phases:
1. **PrepareJob** (main thread): Collects handles, serializes heap data via `JSHeapBroker`
2. **ExecuteJob** (background thread): Runs graph building, optimization, register allocation, and code generation
3. **FinalizeJob** (main thread): Installs the generated code, updates the JSFunction

The `MaglevCompilationJob` (`src/maglev/maglev-concurrent-dispatcher.h`) implements the `OptimizedCompilationJob` interface with these three phases.

## Compilation Entry Point

The `MaglevCompiler` class provides the static entry points:

```cpp
class MaglevCompiler : public AllStatic {
  static bool Compile(LocalIsolate* local_isolate,
                      MaglevCompilationInfo* compilation_info);
  static std::pair<MaybeHandle<Code>, BailoutReason> GenerateCode(
      Isolate* isolate, MaglevCompilationInfo* compilation_info);
};
```

`Compile()` can run on any thread (including background). `GenerateCode()` must run on the main thread to finalize the Code object.

## Turbolev: Maglev as TurboFan Frontend

The `TURBOLEV_VALUE_NODE_LIST` and `TURBOLEV_NON_VALUE_NODE_LIST` macros define nodes that exist in "Turbolev" mode, where Maglev serves as a faster frontend to TurboFan's backend. This experimental path builds a Maglev graph and then converts it to Turboshaft IR for further optimization.

## Platform Support

Maglev has architecture-specific code in subdirectories:

| Directory | Architecture |
|-----------|-------------|
| `src/maglev/x64/` | x86-64 |
| `src/maglev/arm64/` | ARM64 |
| `src/maglev/arm/` | ARM32 |
| `src/maglev/loong64/` | LoongArch64 |

## Deoptimization

Maglev generates deoptimization metadata for every speculative operation. When a type check fails at runtime, execution transfers to the deoptimizer, which reconstructs the interpreter frame state from the deopt frame data and resumes in Ignition or Sparkplug. The `DeoptFrame` structure records:
- The bytecode offset to resume at
- The values of interpreter registers (as graph node references)
- The closure and context

## Key Files

| File | Purpose |
|------|---------|
| `src/maglev/maglev-compiler.h/.cc` | Top-level compilation orchestration |
| `src/maglev/maglev-graph-builder.h/.cc` | Bytecode-to-graph translation |
| `src/maglev/maglev-ir.h/.cc` | Node definitions (value, control, check nodes) |
| `src/maglev/maglev-graph.h/.cc` | Graph container with block list and constants |
| `src/maglev/maglev-code-generator.h/.cc` | Graph-to-machine-code emission |
| `src/maglev/maglev-concurrent-dispatcher.h/.cc` | Background compilation dispatch |
| `src/maglev/maglev-compilation-info.h/.cc` | Compilation metadata and flags |
| `src/maglev/maglev-known-node-aspects.h/.cc` | Type/map tracking during building |
| `src/maglev/maglev-phi-representation-selector.h/.cc` | Phi unboxing optimization |
| `src/maglev/maglev-inlining.h/.cc` | Function inlining decisions |
| `src/maglev/maglev-graph-optimizer.h/.cc` | Post-build graph optimization |
| `src/maglev/maglev-assembler.h/.cc` | Platform-independent code generation API |
