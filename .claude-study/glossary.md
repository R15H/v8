# V8 Glossary

Project-specific terminology used throughout the V8 codebase.

## Core Concepts

**Isolate** (`src/execution/isolate.h`): An independent instance of the V8 engine with its own heap, compilation state, and execution context. Each isolate is single-threaded for JavaScript execution. Multiple isolates can run in parallel on different threads.

**Context** (`include/v8-context.h`): A sandboxed execution environment within an isolate. Holds the global object and built-in functions. Multiple contexts can share an isolate (e.g., iframes in a browser).

**Handle** (`src/handles/handles.h`): A GC-safe reference to a heap object. `Handle<T>` (local, scoped), `DirectHandle<T>` (unsafe, for performance), `Global<T>` (persistent across scopes). All C++ code must reference heap objects through handles.

**HandleScope**: A stack-allocated scope that manages the lifetime of `Handle<T>` objects. When the scope exits, all handles allocated within it become invalid.

**Tagged** (`src/objects/tagged.h`): A pointer that uses its low bits to encode type information. V8 uses 1-bit tagging: bit 0 = 0 means Smi, bit 0 = 1 means HeapObject pointer.

## Object Model

**Smi** (Small Integer, `src/objects/smi.h`): An integer value encoded directly in a tagged pointer (no heap allocation). 31 bits of integer value on 64-bit systems.

**HeapObject** (`src/objects/heap-object.h`): Any object allocated on the V8 heap. The first word of every HeapObject is a pointer to its Map.

**Map** (`src/objects/map.h`): V8's name for a **hidden class** (also called "shape" or "structure" in other engines). Describes the layout of an object: which properties it has, their types, and their memory offsets. Objects with the same property names added in the same order share a Map.

**Transition**: When a property is added to an object, its Map changes to a new Map. Maps form a **transition tree** where edges are property additions. This allows objects created with the same constructor to share Maps.

**DescriptorArray** (`src/objects/descriptor-array.h`): Associated with a Map, stores property metadata (name, type, offset) for fast-mode objects.

**PropertyArray** (`src/objects/property-array.h`): Stores property values that don't fit in the object's in-object slots.

**InstanceType** (`src/objects/instance-type.h`): A 16-bit enum identifying the concrete type of every HeapObject (e.g., `JS_OBJECT_TYPE`, `JS_ARRAY_TYPE`, `SEQ_ONE_BYTE_STRING_TYPE`).

**ElementsKind** (`src/objects/elements-kind.h`): Classifies how an array's indexed elements are stored. Examples: `PACKED_SMI_ELEMENTS` (dense array of Smis), `HOLEY_DOUBLE_ELEMENTS` (sparse array of doubles), `DICTIONARY_ELEMENTS` (hash table).

**FixedArray** (`src/objects/fixed-array.h`): A fixed-size array of tagged values. The fundamental backing store for arrays, property storage, and many internal data structures.

## Compilation

**Ignition** (`src/interpreter/`): V8's bytecode interpreter. Executes bytecode instructions one at a time. Collects type feedback in FeedbackVectors for use by optimizing compilers.

**Baseline** (`src/baseline/`): Tier-1 JIT compiler. Compiles bytecode to native code without optimization. Fast compilation, moderate execution speed.

**Maglev** (`src/maglev/`): Tier-2 mid-tier optimizing compiler. Uses feedback data to generate type-specialized code. Balances compilation speed and code quality.

**TurboFan** (`src/compiler/`): Tier-3 fully optimizing compiler. Uses a sea-of-nodes IR with advanced optimizations: inlining, escape analysis, load elimination, loop optimization.

**Turboshaft** (`src/compiler/turboshaft/`): Next-generation compiler backend being developed to replace TurboFan's internal IR. Uses a more structured, maintainable graph representation.

**Sea-of-Nodes**: TurboFan's intermediate representation. A graph where nodes represent operations, with three kinds of edges: value (data flow), effect (side-effect ordering), and control (control flow).

**Bytecode** (`src/interpreter/bytecodes.h`): V8's portable instruction set (244 opcodes). Compact (~1 byte per instruction), platform-independent, and directly interpretable.

**SharedFunctionInfo** (`src/objects/shared-function-info.h`): Metadata shared across all instances of a function: bytecode, source position table, parameter count, name. One SFI per source function.

**Code** (`src/objects/code.h`): A compiled code object containing machine code. Has metadata about the compilation tier, deoptimization data, and source positions.

**Deoptimization** (`src/deoptimizer/`): The process of abandoning optimized code when a speculative assumption fails. Reconstructs the interpreter's stack frame and resumes in Ignition.

**OSR** (On-Stack Replacement): Replacing a running function's code with a differently-compiled version mid-execution. Used to optimize long-running loops without waiting for the function to be called again.

## Inline Caches

**IC** (Inline Cache, `src/ic/`): A mechanism that speeds up property accesses by caching the result of previous lookups. Each property access site in bytecode has an IC.

**FeedbackVector** (`src/objects/feedback-vector.h`): Per-function array of feedback slots. Each slot records type information observed at a particular bytecode location (which Maps were seen, which functions were called, etc.).

**Monomorphic**: An IC state where only one Map has been observed. Enables the fastest property access path.

**Polymorphic**: An IC state where 2-4 different Maps have been observed. Uses a linear search through cached handlers.

**Megamorphic**: An IC state where too many Maps have been observed. Falls back to a generic (slow) lookup.

## Memory Management

**Young Generation** (New Space): The heap region for newly allocated objects. Uses semi-space copying GC (Scavenger). Objects that survive a collection are promoted to the Old Generation.

**Old Generation** (Old Space): The heap region for long-lived objects. Collected by Mark-Compact GC, which can run incrementally and concurrently.

**Scavenger**: The minor GC algorithm for the Young Generation. Copies live objects from one semi-space to the other (or promotes to Old Generation). Fast but requires copying.

**Mark-Compact**: The major GC algorithm for the Old Generation. Marks reachable objects (concurrent), then sweeps/compacts (partially concurrent).

**Write Barrier**: Code inserted at every pointer store that records cross-generation or cross-space references. Ensures the GC can find all pointers from Old → Young generation without scanning the entire old space.

**Remembered Set**: A data structure recording which old-generation pages contain pointers to young-generation objects. Scanned during minor GC.

**Zone** (`src/zone/zone.h`): A fast bump allocator used for temporary data during compilation. All memory in a zone is freed at once when the zone is destroyed. No individual deallocation.

**Oilpan** / **cppgc** (`src/heap/cppgc/`): A garbage collector for C++ objects (not JavaScript objects). Used for DOM objects in Blink and other embedder-managed objects.

**Pointer Compression** (`src/common/ptr-compr.h`): On 64-bit platforms, heap pointers are stored as 32-bit offsets from the isolate's base address. Halves pointer memory usage at the cost of a 4 GB heap limit per isolate.

## Builtins & Runtime

**Builtin**: A pre-compiled function implementing JavaScript standard library behavior (e.g., `Array.prototype.map`, `Object.keys`). Stored in the snapshot.

**Torque** (`src/torque/`): V8's domain-specific language for writing builtins. Compiles to C++ code. Provides type-safe access to V8 internals with a TypeScript-like syntax.

**Runtime Function** (`src/runtime/`): A C++ function callable from generated code via a `Runtime::kFunctionName` call. Used for operations too complex to inline.

**Snapshot** (`src/snapshot/`): A serialized heap image built at compile time. Contains all builtins, prototypes, and the global object. Deserialized at startup for fast initialization.

**CodeStubAssembler** (CSA, `src/codegen/code-stub-assembler.h`): A C++ API for generating machine code programmatically. Used to write builtins and IC handlers in a platform-independent way.

## Strings

**SeqString**: A string with characters stored inline (sequentially in memory).

**ConsString**: A rope-like string formed by concatenation. Stores pointers to left and right substrings without copying. Flattened lazily.

**SlicedString**: A substring referencing a parent string with an offset and length. Avoids copying.

**ThinString**: A forwarding pointer from a non-internalized string to its internalized (canonical) copy.

**ExternalString**: A string whose character data is stored outside the V8 heap (provided by the embedder).

**Internalized String**: A canonicalized string. Two internalized strings with the same content are guaranteed to be the same object, enabling identity comparison.

## WebAssembly

**Liftoff**: WebAssembly baseline compiler. Generates code quickly with minimal optimization.

**WASM Module**: A compiled WebAssembly binary. Contains function definitions, type signatures, memory definitions, and import/export declarations.

## Debugging

**Inspector** (`src/inspector/`): V8's implementation of the Chrome DevTools Protocol (CDP). Enables remote debugging via WebSocket.

**d8** (`src/d8/`): V8's standalone developer shell. A command-line JavaScript REPL and script runner.

## Build System

**GN**: Generate Ninja. V8's primary build configuration system. Reads `.gn` and `BUILD.gn` files, produces Ninja build files.

**Ninja**: The actual build executor. Reads build files generated by GN and runs compilation commands.

**Siso**: A faster build executor that can replace Ninja. Enabled by default in V8.

**DEPS**: Chromium-style dependency specification file. Lists all external dependencies and their exact versions.

**gclient**: Google's dependency management tool. Reads the DEPS file and fetches/syncs all dependencies.

**mksnapshot**: Build-time tool that initializes V8, creates builtins, and serializes the heap state to a snapshot blob.

**PGO** (Profile-Guided Optimization): Using runtime profiles to guide compiler optimizations. V8 uses PGO for builtin functions.
