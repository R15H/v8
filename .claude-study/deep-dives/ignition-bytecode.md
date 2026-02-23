# Ignition Bytecode Interpreter

## Overview

Ignition is V8's register-based bytecode interpreter. It serves as the first execution tier for all JavaScript code, converting the AST produced by the parser into a compact bytecode representation that is then interpreted. Ignition replaced V8's earlier full-codegen baseline compiler, dramatically reducing memory consumption and providing a uniform bytecode format consumed by all higher compiler tiers.

Key source directory: `src/interpreter/`

## Architecture: Accumulator-Based Register Machine

Ignition uses a **hybrid accumulator/register model**. While it maintains a set of virtual registers (mapped to stack slots), most operations implicitly read from or write to a dedicated **accumulator register**. This design reduces bytecode size because many instructions do not need to encode a destination register.

Each bytecode declares its implicit register use via the `ImplicitRegisterUse` enum:
- `kReadAccumulator` -- the bytecode reads the accumulator
- `kWriteAccumulator` -- the bytecode writes a result to the accumulator
- `kReadWriteAccumulator` -- both reads and writes
- `kClobberAccumulator` -- the accumulator value is destroyed but not meaningfully written
- `kNone` -- no implicit accumulator use

For example, `Ldar` (Load Accumulator from Register) writes the accumulator, while `Star` (Store Accumulator to Register) reads it. The short-star variants `Star0` through `Star15` are single-byte opcodes that store the accumulator into the first 16 registers without needing an operand byte.

## Bytecode Instruction Set

Defined in `src/interpreter/bytecodes.h` via the `BYTECODE_LIST` macro, the instruction set covers:

### Loading Values
- `LdaZero`, `LdaSmi`, `LdaUndefined`, `LdaNull`, `LdaTrue`, `LdaFalse`, `LdaTheHole` -- load constants
- `LdaConstant` -- load from the constant pool
- `Ldar` -- load from a register
- `LdaGlobal`, `LdaGlobalInsideTypeof` -- global variable loads (with feedback slots)
- `LdaContextSlot`, `LdaCurrentContextSlot`, `LdaImmutableContextSlot` -- context variable loads
- `LdaLookupSlot`, `LdaLookupContextSlot`, `LdaLookupGlobalSlot` -- dynamic scope lookups

### Storing Values
- `Star`, `Star0`..`Star15` -- store accumulator to register
- `StaGlobal` -- store to global
- `StaContextSlot`, `StaCurrentContextSlot` -- store to context
- `StaLookupSlot` -- dynamic scope store

### Property Access (with IC Feedback)
- `GetNamedProperty`, `GetKeyedProperty` -- property loads via LoadIC
- `SetNamedProperty`, `SetKeyedProperty` -- property stores via StoreIC
- `DefineNamedOwnProperty`, `DefineKeyedOwnProperty` -- define-own semantics
- `GetNamedPropertyFromSuper` -- super property access
- `GetEnumeratedKeyedProperty` -- optimized for-in keyed load

### Arithmetic and Bitwise
- `Add`, `Sub`, `Mul`, `Div`, `Mod`, `Exp` -- binary arithmetic (register + accumulator)
- `AddSmi`, `SubSmi`, etc. -- immediate-operand variants for small integer operations
- `BitwiseOr`, `BitwiseAnd`, `BitwiseXor`, `ShiftLeft`, `ShiftRight`, `ShiftRightLogical`
- `Inc`, `Dec`, `Negate`, `BitwiseNot` -- unary operators

### Comparison and Test
- `TestEqual`, `TestEqualStrict`, `TestLessThan`, etc. -- comparisons with embedded feedback
- `TestReferenceEqual`, `TestUndetectable`, `TestNull`, `TestUndefined`, `TestTypeOf`

### Control Flow
- `Jump`, `JumpConstant` -- unconditional jumps
- `JumpIfTrue`, `JumpIfFalse`, `JumpIfNull`, `JumpIfUndefined`, etc. -- conditional jumps
- `JumpIfToBooleanTrue`, `JumpIfToBooleanFalse` -- jumps with implicit ToBoolean
- `JumpLoop` -- backward jump for loops (with interrupt budget and feedback slot)
- `SwitchOnSmiNoFeedback` -- table-based switch

### Function Calls and Construction
- `CallProperty`, `CallProperty0`..`CallProperty2` -- method calls
- `CallUndefinedReceiver`, `CallUndefinedReceiver0`..`CallUndefinedReceiver2` -- function calls
- `CallAnyReceiver`, `CallWithSpread` -- generic calls
- `CallRuntime`, `CallRuntimeForPair`, `CallJSRuntime` -- runtime/JS runtime calls
- `Construct`, `ConstructWithSpread`, `ConstructForwardAllArgs`
- `InvokeIntrinsic` -- interpreter intrinsics (fast-path runtime)

### Object Creation
- `CreateArrayLiteral`, `CreateObjectLiteral`, `CreateRegExpLiteral`
- `CreateClosure`, `CreateBlockContext`, `CreateFunctionContext`
- `CreateMappedArguments`, `CreateUnmappedArguments`, `CreateRestParameter`
- `CloneObject` -- structured clone for object spread

### Generators and Async
- `SwitchOnGeneratorState`, `SuspendGenerator`, `ResumeGenerator`

### Operand Encoding
Bytecodes use variable-width operands. The base encoding is single-byte, with `Wide` (16-bit) and `ExtraWide` (32-bit) prefix bytecodes that double or quadruple operand sizes. Operand types include:
- `kReg`, `kRegOut`, `kRegList`, `kRegOutPair`, `kRegOutTriple` -- register operands
- `kImm`, `kUImm` -- signed/unsigned immediates
- `kConstantPoolIndex` -- index into the function's constant pool
- `kFeedbackSlot` -- index into the FeedbackVector
- `kContextSlot` -- context slot index
- `kFlag8`, `kFlag16` -- flag operands
- `kRuntimeId`, `kIntrinsicId`, `kNativeContextIndex` -- special indices

## Bytecode Generation

The `BytecodeGenerator` class (`src/interpreter/bytecode-generator.h`) is an AST visitor that walks the parsed AST and emits bytecodes. Key aspects:

1. **AST Visitor Pattern**: Implements `AstVisitor<BytecodeGenerator>` with `Visit*` methods for every AST node type (expressions, statements, declarations).

2. **BytecodeArrayBuilder**: The generator delegates actual bytecode emission to a `BytecodeArrayBuilder`, which handles operand sizing, constant pool management, and bytecode encoding.

3. **Register Allocation**: Uses a `BytecodeRegisterAllocator` for temporary registers. The allocator provides scoped allocation via `RegisterAllocationScope`.

4. **Feedback Slot Management**: The generator creates `FeedbackVectorSpec` entries and assigns feedback slots to operations that benefit from type feedback (property accesses, calls, binary ops). A `FeedbackSlotCache` deduplicates slots for the same expression/name pairs.

5. **Expression Result Modes**: The generator has multiple visitation modes:
   - `VisitForAccumulatorValue` -- result in accumulator
   - `VisitForRegisterValue` -- result in a specific register
   - `VisitForEffect` -- discard result
   - `VisitForTest` -- result as branch condition

6. **Scope and Context Handling**: Nested `ContextScope` objects track context chain depth. `ControlScope` objects manage break/continue/return targets.

7. **Optimization**: The `BytecodeRegisterOptimizer` eliminates redundant register-to-register moves. The `BytecodeArrayWriter` performs peephole optimizations on the bytecode stream.

## Dispatch Mechanism

The interpreter uses a **dispatch table** -- an array of code entry points indexed by bytecode value and operand scale. Defined in `src/interpreter/interpreter.h`:

```
Address dispatch_table_[kDispatchTableSize];
// kDispatchTableSize = kNumberOfWideVariants * 256
```

Each bytecode handler is a small piece of generated code (a builtin). After executing a bytecode, the handler:
1. Advances the bytecode pointer by the instruction's size
2. Loads the next bytecode from the bytecode array
3. Indexes into the dispatch table
4. Tail-calls to the next handler

This is known as **threaded dispatch** (or more precisely, token-threaded interpretation). The `InterpreterAssembler` (`src/interpreter/interpreter-assembler.h`) extends `CodeStubAssembler` to provide utilities for handler generation:
- `GetAccumulator()` / `SetAccumulator()` -- access the accumulator
- `BytecodeOperandReg()`, `BytecodeOperandImm()`, etc. -- decode operands
- `Dispatch()` / `DispatchToBytecode()` -- advance and dispatch to next handler
- `LoadRegister()` / `StoreRegister()` -- access virtual registers in the frame

Handler code is generated at build time using the `InterpreterGenerator` (`src/interpreter/interpreter-generator.cc`), with an experimental TSA (Turboshaft Assembler) alternative for some handlers.

## Feedback Collection

Ignition is the primary feedback collector for V8's optimizing compilers. Key feedback mechanisms:

1. **FeedbackVector**: Each function has an associated `FeedbackVector` containing slots for type/shape information. Bytecodes reference slots via `kFeedbackSlot` operands.

2. **Type Feedback for Arithmetic**: Binary operations (`Add`, `Sub`, etc.) record `BinaryOperationFeedback` (SignedSmall, Number, BigInt, String, Any).

3. **IC Feedback for Property Access**: `GetNamedProperty`, `SetNamedProperty`, etc. feed into the inline cache system (LoadIC/StoreIC), which records maps, handler information, and transitions.

4. **Call Feedback**: Call bytecodes record the target function, enabling call-site specialization in Maglev/TurboFan.

5. **JumpLoop Feedback**: The `JumpLoop` bytecode carries a feedback slot used for **on-stack replacement (OSR)** tier-up decisions. Its interrupt budget parameter is decremented to trigger tier-up.

6. **Comparison Feedback**: `TestEqual`, `TestLessThan`, etc. use `kEmbeddedFeedback` -- a compact 2-byte feedback field embedded directly in the bytecode operands rather than using a separate feedback vector slot.

## Interpreter Frame Layout

When Ignition executes a function, it sets up a standard JavaScript frame:
- Return address and saved frame pointer
- The function object (JSFunction)
- The context
- The bytecode array
- The bytecode offset (program counter)
- The feedback vector (or feedback cell)
- Virtual registers (local variables and temporaries)
- The accumulator (held in a machine register during execution)

The `interpreter_entry_trampoline` builtin sets up this frame and dispatches to the first bytecode handler.

## Relationship to Other Tiers

Ignition bytecode is the single source of truth consumed by all higher tiers:
- **Sparkplug (Baseline)**: Walks bytecodes and emits native code 1:1 without optimization
- **Maglev**: Builds a graph IR from bytecodes using feedback vector data
- **TurboFan**: Builds sea-of-nodes IR from bytecodes via `BytecodeGraphBuilder`

The bytecode array, constant pool, and feedback vector together form the complete input for any recompilation. This unified bytecode representation was a key architectural decision that simplified V8's compilation pipeline.

## Key Files

| File | Purpose |
|------|---------|
| `src/interpreter/bytecodes.h` | Bytecode enum, operand types, `BYTECODE_LIST` macro |
| `src/interpreter/bytecode-generator.h/.cc` | AST-to-bytecode compilation |
| `src/interpreter/bytecode-array-builder.h/.cc` | Bytecode emission API |
| `src/interpreter/interpreter.h/.cc` | Dispatch table, initialization |
| `src/interpreter/interpreter-assembler.h/.cc` | CSA-based handler code generation |
| `src/interpreter/interpreter-generator.h/.cc` | Bytecode handler generation |
| `src/interpreter/bytecode-register-optimizer.h/.cc` | Redundant move elimination |
| `src/interpreter/bytecode-array-writer.h/.cc` | Bytecode stream writer, peephole |
| `src/interpreter/bytecode-array-iterator.h/.cc` | Bytecode stream iteration |
| `src/interpreter/constant-array-builder.h/.cc` | Constant pool construction |

## Performance Characteristics

Ignition bytecode is designed for compactness and fast interpretation:

- **Bytecode density**: The accumulator model and short-star variants keep the average bytecode size small. A typical function compiles to a few hundred bytes of bytecode.
- **Dispatch cost**: Each bytecode dispatch involves loading the next opcode, indexing the dispatch table, and an indirect jump. Modern CPUs can predict these indirect branches reasonably well due to the limited number of targets.
- **Memory savings**: Bytecode is significantly more compact than native code (roughly 25-50% the size of unoptimized native code), reducing memory pressure on memory-constrained devices.
- **Startup**: Bytecode generation is fast (roughly linear in AST size), making Ignition suitable for first-execution scenarios where compilation latency matters.
- **Feedback overhead**: The feedback slots in the FeedbackVector add per-function memory overhead, but this is amortized by enabling much more effective optimization in higher tiers.

The interpreter's performance is generally within 2-10x of optimized native code, depending on the workload. Tight numeric loops show the largest gap, while property-access-heavy code benefits from IC caching even in the interpreter.
