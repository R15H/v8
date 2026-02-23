# Runtime Functions and Builtins

## Overview

V8's runtime layer consists of three interrelated subsystems: **builtins** (pre-compiled code implementing core language operations), **runtime functions** (C++ functions callable from generated code), and the **Torque DSL** (a domain-specific language for writing builtins). Together, these implement the JavaScript specification, from Array methods to Promise resolution, and provide the glue between compiled JavaScript code and V8's C++ internals.

Key source directories: `src/builtins/`, `src/runtime/`, `src/torque/`

## Builtins

Builtins are pre-compiled code objects that implement fundamental operations in V8. They are compiled at build time (during `mksnapshot`) and embedded in the V8 binary.

### Types of Builtins

Builtins are defined via several mechanisms:

1. **Torque builtins** (`.tq` files): Written in V8's Torque DSL, these are the preferred approach for new builtins. They compile to CSA (CodeStubAssembler) code at build time.

2. **CSA builtins** (`builtins-*-gen.cc`): Written using the `CodeStubAssembler` API, which provides a high-level interface over the TurboFan/Turboshaft graph builder.

3. **C++ builtins** (`builtins-*.cc`): Plain C++ implementations that are called through the runtime call mechanism. Used for complex operations where CSA/Torque would be impractical.

4. **ASM builtins** (`builtins-*-asm.cc`): Hand-written assembly for performance-critical low-level operations (e.g., function entry trampolines, deoptimizer entry points).

### Builtin Categories

The `src/builtins/` directory contains builtins organized by JavaScript feature:

**Array builtins** (extensive Torque coverage):
- `array-map.tq`, `array-filter.tq`, `array-forEach.tq` -- Array iteration methods
- `array-splice.tq`, `array-shift.tq`, `array-unshift.tq` -- Array mutation
- `array-join.tq`, `array-slice.tq`, `array-from.tq` -- Array conversion
- `array-to-sorted.tq`, `array-to-reversed.tq` -- New immutable methods
- `array-at.tq`, `array-find.tq`, `array-findindex.tq` -- Element access

**String builtins**:
- String iteration, search, replacement, case conversion
- Template literal handling

**Promise builtins**:
- Promise construction, resolution, rejection
- Async/await machinery
- Microtask queue interaction

**Object builtins**:
- Property definition, deletion, enumeration
- Prototype chain operations
- `Object.keys`, `Object.assign`, `Object.create`

**TypedArray and ArrayBuffer builtins**:
- Buffer allocation and detachment
- Typed array construction and element access

**Generator and Async builtins**:
- Generator function entry and resume
- Async function and async generator machinery

### Builtin Registration

Builtins are registered in `src/builtins/builtins-definitions.h` using macro lists. Each builtin has a unique `Builtin` enum value, a name, and a kind (TorqueBuiltin, CodeStubAssemblerBuiltin, CppBuiltin, etc.).

### Builtin Code Objects

At runtime, each builtin is represented as a `Code` object stored in the builtins table. Code is embedded in the V8 snapshot and mapped read-only into memory. The `Builtins` class provides access:

```cpp
// Access a builtin by its enum value
Tagged<Code> code = isolate->builtins()->code(Builtin::kArrayMap);
```

## Torque DSL

Torque is V8's domain-specific language for writing builtins in a type-safe, readable way. It compiles to CSA code at build time.

### Language Features

```torque
// Example Torque code (simplified)
transitioning javascript builtin ArrayForEach(
    js-implicit context: NativeContext, receiver: JSAny)(
    ...arguments): Undefined {
  const o = ToObject_Inline(context, receiver);
  const len = GetLengthProperty(o);
  const callbackfn = Cast<Callable>(arguments[0])
      otherwise ThrowCalledNonCallable(arguments[0]);
  const thisArg = arguments[1];

  for (let k: Smi = 0; k < len; k++) {
    if (HasProperty_Inline(o, k) == True) {
      const value = GetProperty(o, k);
      Call(context, callbackfn, thisArg, value, k, o);
    }
  }
  return Undefined;
}
```

Key Torque features:
- **Strong typing**: Types map to V8 internal types (`Smi`, `HeapNumber`, `JSArray`, `Map`, etc.)
- **Transitioning/non-transitioning**: Indicates whether the builtin can trigger GC
- **Labels**: Used for multi-return / exception-like control flow (`otherwise` clauses)
- **Implicit parameters**: `js-implicit context`, `receiver`, etc.
- **Type assertions**: `Cast<T>(x) otherwise label` -- type-checked downcasts
- **Macros**: Reusable code fragments
- **Structs**: Value types for passing multiple values

### Torque Compiler

The Torque compiler (`src/torque/`) parses `.tq` files and generates:
- C++ CSA code (`*-tq-csa.cc/.h`)
- Class definitions and verifiers
- Type system metadata

The generated code uses the `CodeStubAssembler` API to build TurboFan graphs that are compiled to machine code during snapshot creation.

## Runtime Functions

Runtime functions are C++ functions that can be called from generated code (bytecode handlers, builtins, and optimized code). They provide access to V8's full C++ API and handle complex operations that cannot be efficiently expressed in generated code.

### Declaration

Runtime functions are declared in `src/runtime/runtime.h` using macro lists:

```cpp
#define FOR_EACH_INTRINSIC_ARRAY(F, I) \
  F(ArrayIncludes_Slow, 3, 1)          \
  F(ArrayIndexOf, 3, 1)                \
  F(GrowArrayElements, 2, 1)           \
  F(NewArray, -1 /* >= 3 */, 1)        \
  // ...

#define FOR_EACH_INTRINSIC_COMPILER(F, I) \
  F(CompileOptimized, 1, 1)              \
  F(HealOptimizedCodeSlot, 1, 1)         \
  // ...
```

The format is `F(name, arg_count, return_count)`. A `-1` arg count means variadic.

### Organization by Category

Runtime functions are implemented in separate files by category:

| File | Category |
|------|----------|
| `runtime-array.cc` | Array operations |
| `runtime-atomics.cc` | SharedArrayBuffer atomics |
| `runtime-bigint.cc` | BigInt operations |
| `runtime-classes.cc` | Class definition, super access |
| `runtime-collections.cc` | Map, Set, WeakMap, WeakSet |
| `runtime-compiler.cc` | Tier-up, optimization, deoptimization |
| `runtime-date.cc` | Date parsing and formatting |
| `runtime-debug.cc` | Debugger support |
| `runtime-forin.cc` | For-in enumeration |
| `runtime-function.cc` | Function.prototype operations |
| `runtime-generator.cc` | Generator/async machinery |
| `runtime-internal.cc` | V8-internal operations (throw, stack checks) |
| `runtime-literals.cc` | Object/array literal creation |
| `runtime-module.cc` | ES modules |
| `runtime-numbers.cc` | Number conversion and operations |
| `runtime-object.cc` | Object.prototype methods |
| `runtime-promise.cc` | Promise resolution |
| `runtime-regexp.cc` | RegExp execution |
| `runtime-scopes.cc` | Scope and variable operations |
| `runtime-strings.cc` | String operations |
| `runtime-typedarray.cc` | TypedArray and DataView |
| `runtime-wasm.cc` | WebAssembly support |
| `runtime-weak-refs.cc` | WeakRef and FinalizationRegistry |

### Calling Convention

From generated code, runtime functions are invoked via:
- `CallRuntime` bytecode (Ignition)
- `CallRuntime` macro (CSA/Torque builtins)
- `Runtime::kFunctionName` enum (from optimized code)

The calling convention passes arguments via registers/stack, with the runtime function ID identifying which C++ function to call. The `CEntry` stub handles the transition from generated code to C++.

### Inline vs. Runtime Intrinsics

Some runtime functions have both forms:
- `%FunctionName()` -- always a runtime call
- `%_FunctionName()` -- can be inlined by the compiler

The `I` macro in the intrinsic lists declares inline-capable variants that the optimizing compilers can potentially replace with inline code.

## Bootstrapping

V8's bootstrapping process sets up the initial JavaScript environment:

1. **Snapshot deserialization**: The pre-built snapshot contains builtins, the global object template, and built-in prototypes. This is the fast path for normal V8 startup.

2. **Genesis** (`src/init/bootstrapper.cc`): When no snapshot is available (or during `mksnapshot`), `Genesis` creates the initial environment from scratch:
   - Creates the global object and global proxy
   - Sets up `Object.prototype`, `Function.prototype`, `Array.prototype`, etc.
   - Installs built-in functions on prototypes
   - Creates the `Math`, `JSON`, `Reflect`, `Atomics` objects
   - Sets up the `Promise` infrastructure
   - Installs Intl (internationalization) support

3. **Lazy deserialization**: Some builtins are deserialized lazily on first use to reduce memory.

## CodeStubAssembler (CSA)

The `CodeStubAssembler` (`src/codegen/code-stub-assembler.h`) is the workhorse API for writing builtins in C++. It provides a high-level interface over graph construction:

```cpp
class CodeStubAssembler : public compiler::CodeAssembler {
  // Type operations
  TNode<Smi> SmiAdd(TNode<Smi> a, TNode<Smi> b);
  TNode<Number> NumberAdd(TNode<Number> a, TNode<Number> b);

  // Object access
  TNode<Object> LoadObjectField(TNode<HeapObject> object, int offset);
  void StoreObjectField(TNode<HeapObject> object, int offset, TNode<Object> value);

  // Type checks
  TNode<BoolT> IsJSArray(TNode<HeapObject> object);
  TNode<BoolT> IsSmi(TNode<Object> object);

  // Control flow
  void GotoIf(TNode<BoolT> condition, Label* label);
  void Branch(TNode<BoolT> condition, Label* if_true, Label* if_false);
};
```

CSA code is compiled to a TurboFan graph during `mksnapshot`, then optimized and turned into machine code that becomes part of the V8 binary.

## Performance Considerations

### Fast Paths vs. Slow Paths

Most builtins implement a fast path for common cases and fall back to slow runtime functions for edge cases:
- Array methods check if the receiver is a packed `JSArray` with fast elements
- Property operations check for fast-mode objects before handling dictionary mode
- String operations check for flat one-byte strings

### Builtin Inlining

TurboFan's `JSCallReducer` can inline many builtins. When it recognizes a call to `Array.prototype.map`, it can replace the call with an inline loop, eliminating function call overhead and enabling further optimization of the callback.

## Key Files

| File | Purpose |
|------|---------|
| `src/builtins/builtins-definitions.h` | Builtin enum and registration macros |
| `src/builtins/builtins.h/.cc` | Builtin table and access |
| `src/builtins/array.tq` | Core Array builtin implementations |
| `src/builtins/base.tq` | Torque standard library |
| `src/builtins/builtins-array-gen.cc` | CSA Array builtins |
| `src/runtime/runtime.h` | Runtime function declarations |
| `src/runtime/runtime-compiler.cc` | Tier-up and optimization runtime |
| `src/runtime/runtime-object.cc` | Object operation runtime |
| `src/torque/` | Torque compiler |
| `src/codegen/code-stub-assembler.h` | CSA API for builtin authoring |
| `src/init/bootstrapper.cc` | JavaScript environment initialization |
