# V8 Master Topic List

Comprehensive list of topics to understand about V8, organized into major areas with 10 sub-items each.

---

## 1. Parsing & Lexical Analysis
1. Scanner/Lexer token generation and Unicode handling
2. Recursive-descent parser architecture
3. Lazy parsing and pre-parser for deferred function bodies
4. Scope analysis and variable resolution
5. Arrow function and destructuring parsing edge cases
6. Template literal parsing and tagged templates
7. Module parsing and import/export resolution
8. Error recovery and error message generation
9. AST node types and the visitor pattern
10. Source position tracking for debugging and error reporting

## 2. Bytecode & Interpreter (Ignition)
1. Bytecode instruction set design (244 opcodes)
2. Register-based vs. accumulator-based execution model
3. Bytecode generation from AST (BytecodeGenerator)
4. Constant pool management
5. Exception handler table construction
6. Bytecode dispatch mechanism (threaded dispatch)
7. Feedback vector slot allocation and update
8. Bytecode flushing (reclaiming bytecode memory)
9. Source position table for debugging
10. Bytecode optimization passes (peephole, dead code elimination)

## 3. Baseline Compiler
1. Bytecode-to-machine-code translation strategy
2. Baseline assembler architecture
3. Frame layout and stack management
4. Interaction with inline caches
5. Batch compilation for multiple functions
6. Tier-up detection and triggering
7. Architecture-specific code generation
8. Performance characteristics vs. interpreter
9. Feedback collection in baseline code
10. Deoptimization support from baseline code

## 4. Maglev Compiler
1. Graph-based IR design
2. Bytecode-to-graph construction
3. Type specialization using feedback
4. Register allocation strategy
5. Phi node handling and SSA form
6. Deoptimization frame construction
7. Inlining heuristics and implementation
8. Concurrent compilation on background threads
9. Code generation and relocation
10. Interaction with the deoptimizer

## 5. TurboFan / Turboshaft Optimizing Compiler
1. Sea-of-nodes intermediate representation
2. Graph building from bytecode + feedback
3. Type system and type narrowing
4. Inlining strategy and call-site analysis
5. Escape analysis and scalar replacement
6. Load elimination and alias analysis
7. Loop optimization (unrolling, peeling, LICM)
8. Instruction selection (IR → machine instructions)
9. Register allocation (linear scan / graph coloring)
10. Turboshaft: next-gen IR design and migration strategy

## 6. Object Model & Hidden Classes
1. Tagged pointer representation (Smi vs. HeapObject)
2. Map (hidden class) structure and layout
3. Map transitions and transition trees
4. Property storage: in-object vs. PropertyArray vs. dictionary
5. Elements kinds and array backing stores
6. String representations (Seq, Cons, Sliced, Thin, External)
7. DescriptorArray and property metadata
8. Prototype chain and property lookup
9. Object creation and Map sharing
10. Map deprecation and migration

## 7. Heap & Garbage Collection
1. Generational heap layout (Young/Old/Large/Code/ReadOnly spaces)
2. Young generation Scavenger (semi-space copying)
3. Mark-Compact collector for old generation
4. Incremental marking strategy
5. Concurrent marking on background threads
6. Write barriers and remembered sets
7. Heap compaction and object migration
8. Memory pressure handling and GC scheduling
9. Weak references and weak callbacks
10. Embedder heap tracing (cppgc/Oilpan integration)

## 8. Inline Caches & Optimization Feedback
1. IC architecture and state machine
2. Monomorphic, polymorphic, megamorphic IC states
3. LoadIC and StoreIC implementation
4. FeedbackVector structure and slot kinds
5. Handler compilation and caching
6. Binary operation and comparison feedback
7. Call feedback and target prediction
8. IC miss handling and handler generation
9. Feedback-directed optimization in compilers
10. Megamorphic stub cache

## 9. Runtime & Builtins
1. Torque DSL: syntax, type system, and code generation
2. Array.prototype methods implementation
3. Object built-in functions
4. String built-in functions
5. Promise implementation and microtask queue
6. Async/await desugaring and implementation
7. Proxy and Reflect implementation
8. Iterator protocol and generator functions
9. Runtime function calling convention
10. Bootstrapping: how the global object is constructed

## 10. WebAssembly
1. Module decoding and validation
2. Liftoff baseline compiler
3. TurboFan WASM optimizing compiler
4. WASM-to-JS and JS-to-WASM call bridges
5. Memory management for WASM linear memory
6. Table and global variable implementation
7. Exception handling in WASM
8. WASM SIMD support
9. Streaming compilation
10. WASM garbage collection proposal integration

## 11. Debugging & Developer Tools
1. Chrome DevTools Protocol (CDP) implementation
2. Breakpoint setting and management
3. Step execution (into, over, out)
4. Variable inspection and scope walking
5. CPU profiling (sampling profiler)
6. Heap profiling and snapshots
7. Code coverage tracking
8. Source maps and original source mapping
9. d8 shell features and command-line flags
10. Live editing (hot code replacement)

## 12. Security & Sandboxing
1. V8 sandbox architecture
2. Pointer compression and cage
3. External pointer table
4. Code pointer table and CFI
5. Trusted/untrusted object separation
6. Write-protect code pages
7. Stack guard and overflow detection
8. ArrayBuffer and SharedArrayBuffer security
9. Spectre mitigations in generated code
10. Fuzzing infrastructure (ClusterFuzz, Fuzzilli)

## 13. Snapshots & Serialization
1. mksnapshot tool and build-time compilation
2. Heap serialization format
3. Code serialization and relocation
4. Context serialization
5. Custom startup snapshots
6. Deserialization and object reconstruction
7. Read-only space and shared objects
8. Snapshot compression
9. Profile-guided optimization for builtins
10. Startup performance and snapshot impact

## 14. Platform Abstraction & Concurrency
1. Platform API for embedders
2. Thread management and thread pools
3. Mutex and condition variable primitives
4. Atomic operations and lock-free structures
5. Task scheduling (foreground and background tasks)
6. Virtual memory management
7. CPU feature detection
8. Signal handling and trap handlers
9. Architecture-specific code paths
10. Isolate groups and shared resources

## 15. Regular Expressions
1. RegExp parsing and AST construction
2. Bytecode compilation for regexp
3. JIT compilation for regexp (Irregexp)
4. Backtracking implementation
5. Unicode property support
6. Named capture groups
7. RegExp optimization techniques
8. RE2 integration for linear-time matching
9. String search algorithms
10. RegExp test suite and conformance
