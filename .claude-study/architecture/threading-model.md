# V8 Threading Model

## Core Principle: Single-Threaded JavaScript per Isolate

V8's fundamental threading model is **single-threaded JavaScript execution per Isolate**. Each `Isolate` runs JavaScript on exactly one thread at a time. However, V8 uses background threads extensively for non-JavaScript work.

## Thread Types

### Main Thread (Mutator Thread)
- Executes JavaScript code
- Performs young-generation GC (Scavenge)
- Handles IC (inline cache) transitions
- Runs synchronous compilation (Ignition, Baseline)
- Processes microtasks

### Background Compiler Threads
- **Concurrent Maglev compilation**: Mid-tier optimization on background threads
- **Concurrent TurboFan compilation**: Full optimization on background threads
- **Concurrent parsing**: Streaming script compilation
- Managed by `src/compiler-dispatcher/`

### Background GC Threads
- **Concurrent marking**: Background threads trace the object graph
- **Concurrent sweeping**: Background threads rebuild free lists
- **Array buffer sweeper**: Cleans up ArrayBuffer backing stores
- Synchronization via marking worklists and atomic operations

### Platform Thread Pool
- Managed by `src/libplatform/default-platform.cc`
- Shared across all isolates in a process
- Executes `v8::Task` objects submitted by V8 internals

## Synchronization Mechanisms

### Mutexes and Locks
- `src/base/platform/mutex.h`: Standard mutex and recursive mutex
- `src/base/platform/condition-variable.h`: Condition variables
- `base::MutexGuard`: RAII lock guard

### Atomic Operations
- `src/base/atomic-utils.h`: Atomic counters, flags, and CAS operations
- Used extensively in GC marking bitmaps and concurrent data structures

### Write Barriers (GC Synchronization)
- Every pointer store includes a write barrier check
- Ensures concurrent marking sees all new references
- Barriers record cross-generation pointers in remembered sets

### Stack Guard
- `src/execution/stack-guard.h`: Interrupt mechanism
- Background threads request interrupts (GC, compilation) via atomic flags
- Main thread checks at loop back-edges and function calls

## Isolate Groups

**Source:** `src/init/isolate-group.h`

Multiple Isolates can share certain resources:
- Read-only heap pages (shared across all Isolates)
- Builtin code (if snapshots match)
- Platform thread pool

Each Isolate has its own:
- Heap (young gen, old gen, code space)
- Compilation state
- Execution stack
- Handle scopes

## Data Flow Between Threads

```
Main Thread                    Background Threads
─────────────                  ──────────────────
JavaScript execution
  │
  ├─ Triggers compilation ──→  TurboFan/Maglev compilation
  │    (via tiering decision)    │
  │                              ├─ Reads bytecode (immutable)
  │                              ├─ Reads feedback vector (atomic reads)
  │                              └─ Produces Code object
  │                                   │
  ├─ Installs compiled code ←────────┘
  │    (atomic pointer swap)
  │
  ├─ Allocates objects
  │    │
  │    └─ Write barrier ─────→  Concurrent marker sees new refs
  │                              │
  ├─ GC requested ──────────→  Concurrent marking starts
  │    (incremental steps)       │
  │                              └─ Marks objects (atomic bitmap ops)
  │
  ├─ Stop-the-world pause
  │    ├─ Finalize marking
  │    ├─ Scavenge (young gen)
  │    └─ Start concurrent sweep ──→ Background sweeping
  │
  └─ Resume JavaScript
```

## Key Invariants

1. **No concurrent JavaScript**: Only one thread executes JS per Isolate at any time
2. **Immutable bytecode**: BytecodeArrays are never modified after creation — safe for concurrent compilation to read
3. **Atomic feedback reads**: Optimizing compilers read FeedbackVectors using atomic operations
4. **GC safepoints**: The main thread reaches safepoints at well-defined locations (loop back-edges, function calls)
5. **Code installation is atomic**: A single pointer swap activates new compiled code
