# Sparkplug: The Baseline Compiler

## Overview

Sparkplug (internally called the "baseline compiler") is V8's second execution tier. It compiles Ignition bytecodes directly to native machine code without performing any optimization. The goal is simple: eliminate interpreter dispatch overhead with minimal compilation cost. Sparkplug was introduced in V8 v9.1 (2021) to bridge the gap between the interpreter and the optimizing compilers.

Key source directory: `src/baseline/`

## Design Philosophy

Sparkplug is deliberately simple:

1. **No IR**: It does not build any intermediate representation. It walks bytecodes linearly, emitting machine code for each bytecode in sequence.
2. **No optimization**: There is no register allocation, no dead code elimination, no constant folding. The generated code mirrors the bytecode exactly.
3. **No register allocation**: Sparkplug uses the same register-based frame layout as the interpreter. Values live in the same stack slots, and the accumulator maps to a fixed machine register.
4. **Fast compilation**: Because there is no analysis phase, compilation is extremely fast -- roughly proportional to bytecode length.
5. **Shared frame layout**: Sparkplug code uses the same stack frame layout as Ignition, making transitions between them trivial. The on-stack profiling data (feedback vector, bytecode array) remains accessible.

## BaselineCompiler

The core class `BaselineCompiler` (`src/baseline/baseline-compiler.h`) takes a `SharedFunctionInfo` and `BytecodeArray` as input:

```cpp
class BaselineCompiler {
  explicit BaselineCompiler(LocalIsolate* local_isolate,
                            Handle<SharedFunctionInfo> shared_function_info,
                            Handle<BytecodeArray> bytecode);
  void GenerateCode();
  MaybeHandle<Code> Build();
  static int EstimateInstructionSize(Tagged<BytecodeArray> bytecode);
};
```

### Compilation Process

1. **Prologue**: `Prologue()` emits the function entry sequence, setting up the frame. `PrologueFillFrame()` initializes register slots. `PrologueHandleOptimizationState()` checks the optimization/interrupt state from the feedback vector.

2. **Bytecode Iteration**: The compiler uses a `BytecodeArrayIterator` to walk through each bytecode sequentially. For each bytecode:
   - `PreVisitSingleBytecode()` handles label binding (for jump targets) and position recording
   - `VisitSingleBytecode()` dispatches to the appropriate `Visit*` method

3. **Visitor Methods**: For every bytecode in the `BYTECODE_LIST`, there is a corresponding `Visit*` method:
   ```cpp
   #define DECLARE_VISITOR(name, ...) void Visit##name();
   BYTECODE_LIST(DECLARE_VISITOR, DECLARE_VISITOR)
   ```
   Each visitor emits machine code for that bytecode, typically by calling builtins or inlining simple operations.

4. **Code Generation**: Uses `MacroAssembler` (via the `BaselineAssembler` wrapper) for native code emission. The `BaselineAssembler` provides a platform-independent API with architecture-specific implementations in `src/baseline/arm/`, `src/baseline/x64/`, etc.

### Operand Access

The compiler provides type-safe operand accessors that read directly from the bytecode stream:
- `RegisterOperand()`, `LoadRegister()`, `StoreRegister()` -- register operands
- `Constant<Type>()`, `ConstantSmi()` -- constant pool lookups
- `Uint()`, `Int()` -- immediate values
- `FeedbackSlot()`, `ContextSlot()`, `CoverageSlot()` -- index operands
- Various `*AsTagged()` and `*AsSmi()` converters for calling conventions

### Calling Builtins

Most bytecodes are compiled as calls to shared builtins rather than fully inlined machine code:

```cpp
template <Builtin kBuiltin, typename... Args>
void CallBuiltin(Args... args);

template <typename... Args>
void CallRuntime(Runtime::FunctionId function, Args... args);

template <Builtin kBuiltin, typename... Args>
void TailCallBuiltin(Args... args);
```

This keeps baseline code compact while still faster than the interpreter, since the dispatch overhead is removed and arguments are passed in machine registers.

### Jump Handling

The compiler pre-allocates a label array indexed by bytecode offset:
```cpp
Label* labels_;
BitVector label_tags_;
```

Labels are created lazily via `EnsureLabel()`. Jump targets are tracked as both regular targets and indirect jump targets (for CFI compliance). Forward jumps create labels that are bound when the target bytecode is visited.

### Bytecode Offset Table

A `BytecodeOffsetTableBuilder` records the mapping from native code PC offsets to bytecode offsets. This is essential for:
- Deoptimization (mapping back to interpreter state)
- Debugging (setting breakpoints on bytecode positions)
- Profiling (attributing samples to source positions)

The table uses VLQ (variable-length quantity) encoding for compactness.

### Interrupt Budget and Tier-Up

Sparkplug code checks the interrupt budget on backward branches (loop headers). When the budget is exhausted, it triggers a tier-up to Maglev or TurboFan. The `UpdateInterruptBudgetAndJumpToLabel()` method handles this:
- Decrements the interrupt budget counter
- If exhausted, calls into the runtime for potential tier-up
- Otherwise, continues to the loop body

## Batch Compilation

The `BaselineBatchCompiler` (`src/baseline/baseline-batch-compiler.h`) amortizes compilation overhead by batching multiple functions:

```cpp
class BaselineBatchCompiler {
  void EnqueueFunction(DirectHandle<JSFunction> function);
  void EnqueueSFI(Tagged<SharedFunctionInfo> shared);
  void InstallBatch();
  bool ShouldCompileBatch(Tagged<SharedFunctionInfo> shared);
};
```

### How Batching Works

1. **Enqueueing**: When a function becomes hot enough for baseline compilation, its `SharedFunctionInfo` is enqueued in a `WeakFixedArray` compilation queue.

2. **Threshold Check**: `ShouldCompileBatch()` checks whether the accumulated estimated instruction size exceeds a threshold. This groups multiple small functions into a single compilation batch.

3. **Batch Execution**: When the threshold is exceeded, `CompileBatch()` compiles all queued functions. This amortizes the overhead of entering the compiler.

4. **Concurrent Compilation**: Via `ConcurrentBaselineCompiler`, batch compilation can be offloaded to a background thread. `CompileBatchConcurrent()` dispatches to the background, while `InstallBatch()` installs completed code on the main thread.

5. **Weak References**: The queue uses weak references to `SharedFunctionInfo` objects. If a function's bytecode is flushed before compilation, the entry is silently skipped.

## Platform-Specific Code

Each target architecture has its own `BaselineAssembler` implementation:

| Directory | Architecture |
|-----------|-------------|
| `src/baseline/x64/` | x86-64 |
| `src/baseline/arm64/` | ARM64 / AArch64 |
| `src/baseline/arm/` | ARM32 |
| `src/baseline/ia32/` | x86 (32-bit) |
| `src/baseline/riscv/` | RISC-V |
| `src/baseline/loong64/` | LoongArch64 |
| `src/baseline/ppc/` | PowerPC |
| `src/baseline/s390/` | IBM System/390 |
| `src/baseline/mips64/` | MIPS64 |

The `baseline-assembler-inl.h` header selects the correct platform implementation.

## Debug and Verification

In debug builds, the `EffectState` tracking system verifies that:
- The accumulator is not read after a potential deoptimization point without being reloaded
- No unsafe operations occur while the accumulator is saved on the stack
- Bytecodes that may deopt are properly handled

The `SaveAccumulatorScope` RAII class saves/restores the accumulator across operations that might clobber it.

## Sparkplug+ (Extended Baseline)

The `allow_sparkplug_plus_` flag enables "Sparkplug+" mode, which allows slightly more aggressive code generation for certain patterns. This is controlled by the `--sparkplug-plus` flag.

## Relationship to Other Tiers

```
Source Code -> Parser -> AST -> Ignition (Bytecodes)
                                    |
                                    v
                              Sparkplug (Baseline native code)
                                    |
                                    v
                              Maglev / TurboFan (Optimized code)
```

Sparkplug sits between Ignition and the optimizing compilers. Key interactions:
- **Tier-up from Ignition**: Functions are baseline-compiled after they become warm (based on call counts or loop iterations)
- **Tier-up to Maglev/TurboFan**: Sparkplug code triggers tier-up through interrupt budget checks
- **Deoptimization**: When optimized code deoptimizes, it can land back in either Sparkplug or Ignition code
- **OSR (On-Stack Replacement)**: Loops can be replaced mid-execution, transitioning from Sparkplug to optimized code

## Performance Characteristics

- **Compilation speed**: ~100x faster than TurboFan, suitable for eager compilation
- **Code quality**: ~2-3x faster than interpretation, but far from optimized
- **Memory**: Code is larger than bytecode but metadata is shared with the interpreter
- **Startup impact**: Reduces "jank" from interpreter overhead without the latency of optimization

## Key Files

| File | Purpose |
|------|---------|
| `src/baseline/baseline-compiler.h/.cc` | Core compilation logic |
| `src/baseline/baseline-batch-compiler.h/.cc` | Batch and concurrent compilation |
| `src/baseline/baseline-assembler.h` | Platform-independent assembler API |
| `src/baseline/baseline-assembler-inl.h` | Platform dispatch header |
| `src/baseline/baseline.h/.cc` | Entry points and utilities |
| `src/baseline/bytecode-offset-iterator.h/.cc` | PC-to-bytecode mapping iteration |
| `src/baseline/x64/baseline-assembler-x64-inl.h` | x64-specific code generation |
