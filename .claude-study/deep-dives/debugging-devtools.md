# Debugging and Developer Tools in V8

## Overview

V8's debugging and developer tools infrastructure spans four major subsystems:
the **Debug** module (`src/debug/`) for breakpoint management and stepping,
the **Inspector** (`src/inspector/`) implementing the Chrome DevTools Protocol (CDP),
the **Profiler** (`src/profiler/`) for CPU and heap profiling, and the
**d8** shell (`src/d8/`) providing a standalone debugging environment.
Together, these subsystems enable the rich debugging experience found in
Chrome DevTools, Node.js, and other V8 embedders.

## Architecture Overview

```
Chrome DevTools / IDE
       |
       | (WebSocket / CDP JSON messages)
       v
+------------------+
|  V8 Inspector    |  <-- src/inspector/
|  (CDP protocol)  |
+------------------+
       |
       v
+------------------+     +------------------+
|  Debug Module    |     |  Profiler Module  |
|  (breakpoints,   |     |  (CPU, heap,     |
|   stepping)      |     |   coverage)      |
+------------------+     +------------------+
       |                        |
       v                        v
+------------------------------------------+
|          V8 Isolate / Execution          |
+------------------------------------------+
```

## The Debug Module (`src/debug/`)

### Core Class: `Debug`

The `Debug` class (declared in `src/debug/debug.h`) is the central hub for
all debugging operations. It is owned by each `Isolate` and manages:

- Active breakpoints and their associated `DebugInfo` objects
- Stepping state (step-in, step-out, step-over)
- Exception break behavior (caught/uncaught)
- Script compilation events
- Live editing support

Key fields from the `Debug` class:

```cpp
class V8_EXPORT_PRIVATE Debug {
  // Debugger is active, i.e. there is a debug event listener attached.
  std::atomic<bool> is_active_;
  // Debugger needs to be notified on every new function call.
  bool hook_on_function_call_;
  // Do not trigger debug break events.
  bool break_disabled_;
  // Do not break on break points.
  bool break_points_active_;
  // Trigger debug break events for caught/uncaught exceptions.
  bool break_on_caught_exception_;
  bool break_on_uncaught_exception_;
  // List of active debug info objects.
  DebugInfoCollection debug_infos_;
  // Per-thread stepping state.
  ThreadLocal thread_local_;
};
```

### Step Actions

V8 defines three stepping modes as the `StepAction` enum:

| Action     | Value | Behavior                                      |
|------------|-------|-----------------------------------------------|
| `StepOut`  | 0     | Step out of the current function               |
| `StepOver` | 1     | Step to next statement in current function     |
| `StepInto` | 2     | Step into called functions or next statement   |

Stepping is implemented by "flooding" breakable locations with one-shot
breakpoints. When `PrepareStep()` is called, V8 instruments the bytecode
at relevant positions. The `FloodWithOneShot()` method sets temporary
breakpoints that are automatically cleared after being hit once.

### Breakpoint Management

Breakpoints are tracked through two parallel data structures in
`DebugInfoCollection`:

1. **A vector** (`std::vector<HandleLocation>`) for fast iteration
2. **An unordered map** (`std::unordered_map<SFIUniqueId, HandleLocation>`)
   for fast `SharedFunctionInfo`-to-`DebugInfo` lookups

Each `DebugInfo` object (a heap object) stores the breakpoint state for a
particular function. The `BreakLocation` class identifies specific positions
in bytecode where breaks can occur:

```cpp
enum DebugBreakType {
  NOT_DEBUG_BREAK,
  DEBUG_BREAK_AT_ENTRY,
  DEBUGGER_STATEMENT,
  DEBUG_BREAK_SLOT,
  DEBUG_BREAK_SLOT_AT_CALL,
  DEBUG_BREAK_SLOT_AT_RETURN,
  DEBUG_BREAK_SLOT_AT_SUSPEND,
};
```

The `BreakIterator` walks through a function's bytecode using a
`SourcePositionTableIterator` to enumerate all possible break locations.
This is used both for setting breakpoints and for the
`getPossibleBreakpoints` CDP command.

### Blackboxing and Ignore-Listing

V8 supports "blackboxing" scripts so the debugger skips over them during
stepping. The `IsFunctionBlackboxed()` and `IsFrameBlackboxed()` methods
check whether a function or stack frame should be ignored. This is controlled
from the inspector layer via regex patterns or explicit execution context IDs.

### Scope and Frame Inspection

- `debug-scopes.h` / `debug-scopes.cc`: `ScopeIterator` walks the scope chain
  of a paused frame, exposing local, closure, script, and global scopes.
- `debug-frames.h` / `debug-frames.cc`: Frame inspection utilities.
- `debug-stack-trace-iterator.h`: Stack trace iteration for the inspector.
- `debug-evaluate.h`: Evaluation of expressions in the context of a paused frame,
  with side-effect checking to safely evaluate watch expressions.

### Live Editing

The `LiveEdit` class (`src/debug/liveedit.h`) supports hot-patching JavaScript
source code while the program is running. The process:

1. Diff old source vs new source (using a longest-common-subsequence algorithm
   from `liveedit-diff.h`)
2. Map function literals from old to new source
3. Create a new `Script` object for the updated source
4. For unchanged functions: deoptimize, update source positions, move to new script
5. For changed functions: deoptimize, reset feedback, relink all JSFunction references
6. Swap scripts and optionally restart the bottom-most affected frame

## The Inspector (`src/inspector/`)

### Chrome DevTools Protocol Implementation

The inspector implements the Chrome DevTools Protocol (CDP), which is a
JSON-RPC protocol transported over WebSockets. The architecture follows an
agent-per-domain pattern:

| Agent Class                     | CDP Domain      | Purpose                        |
|---------------------------------|-----------------|--------------------------------|
| `V8DebuggerAgentImpl`           | `Debugger`      | Breakpoints, stepping, pausing |
| `V8RuntimeAgentImpl`            | `Runtime`       | Evaluate, object inspection    |
| `V8ProfilerAgentImpl`           | `Profiler`      | CPU profiling                  |
| `V8HeapProfilerAgentImpl`       | `HeapProfiler`  | Heap snapshots, tracking       |
| `V8ConsoleAgentImpl`            | `Console`       | Console message forwarding     |
| `V8SchemaAgentImpl`             | `Schema`        | Protocol schema queries        |

### `V8InspectorImpl`

The top-level `V8InspectorImpl` class owns:

- The `V8Debugger` instance (which wraps the internal `Debug` module)
- Maps of inspected contexts (by group ID and context ID)
- Session management (multiple concurrent debug sessions)
- Console message storage per context group
- Async task tracking for stack traces

```cpp
class V8InspectorImpl : public V8Inspector {
  v8::Isolate* m_isolate;
  V8InspectorClient* m_client;
  std::unique_ptr<V8Debugger> m_debugger;
  ContextsByGroupMap m_contexts;
  // contextGroupId -> sessionId -> session
  std::unordered_map<int, std::map<int, V8InspectorSessionImpl*>> m_sessions;
};
```

### `V8DebuggerAgentImpl`

This is the workhorse agent that maps CDP `Debugger.*` commands to V8 debug
operations. Key protocol methods it implements:

- `enable` / `disable` -- activates/deactivates debugging
- `setBreakpointByUrl` / `setBreakpoint` / `removeBreakpoint`
- `pause` / `resume`
- `stepOver` / `stepInto` / `stepOut`
- `evaluateOnCallFrame` -- expression evaluation with optional side-effect checks
- `setScriptSource` -- live editing
- `setBlackboxPatterns` -- script ignore-listing
- `setAsyncCallStackDepth` -- controls async stack trace capture depth

Breakpoint IDs in CDP are strings like `1:14:0:file.js` (scriptId:line:column).
Internally, these are mapped to `v8::debug::BreakpointId` integers through
`m_breakpointIdToDebuggerBreakpointIds` and its reverse map.

### Session Architecture

Multiple inspector sessions can be connected to a single V8 isolate
simultaneously (e.g., Chrome DevTools + a programmatic client). Each session
gets its own `V8InspectorSessionImpl` with independent agent instances.
The `V8DebuggerBarrier` synchronizes across sessions, ensuring that when one
session pauses execution, all sessions are notified.

## CPU Profiling (`src/profiler/`)

### `CpuProfiler`

The `CpuProfiler` class is a sampling CPU profiler. As its header documents:

> Sampling is done using posix signals (except on Windows). The profiling
> thread sends a signal to the main thread, based on a timer. The signal
> handler can interrupt the main thread between any arbitrary instructions.

The architecture involves several cooperating components:

1. **`CpuProfiler`** -- API entry point, manages profiling sessions
2. **`SamplingEventsProcessor`** -- background thread that processes samples
3. **`ProfilerCodeObserver`** -- listens for code creation/deletion events
4. **`Symbolizer`** -- maps instruction pointers to function names
5. **`CpuProfilesCollection`** -- stores profile trees

### Sampling Pipeline

```
Signal Handler / Thread Suspend
       |
       v
TickSample (captured on main thread)
       |
       v
SamplingCircularQueue (lock-free, 512KB)
       |
       v
SamplingEventsProcessor (background thread)
       |
       v
Symbolizer (PC -> CodeEntry mapping)
       |
       v
CpuProfilesCollection (tree of call frames)
```

The `SamplingCircularQueue` is carefully designed for the signal-handler
context: the producer (signal handler) writes `TickSampleEventRecord` entries
without locks, and the consumer (profiling thread) reads them.

### Code Event Records

Code events are tracked through a union-based `CodeEventsContainer`:

```cpp
class CodeEventsContainer {
  union {
    CodeEventRecord generic;
    CodeCreateEventRecord CodeCreateEventRecord_;
    CodeMoveEventRecord CodeMoveEventRecord_;
    CodeDisableOptEventRecord CodeDisableOptEventRecord_;
    CodeDeoptEventRecord CodeDeoptEventRecord_;
    ReportBuiltinEventRecord ReportBuiltinEventRecord_;
    CodeDeleteEventRecord CodeDeleteEventRecord_;
    NativeContextMoveEventRecord NativeContextMoveEventRecord_;
  };
};
```

## Heap Profiling

### `HeapProfiler`

The `HeapProfiler` class supports two profiling modes:

1. **Heap Snapshots** -- full graph of all live objects with edges
2. **Sampling Heap Profiler** -- statistical sampling of allocations

Key capabilities:
- `TakeSnapshot()` -- captures the complete object graph
- `StartSamplingHeapProfiler()` / `StopSamplingHeapProfiler()`
- Object tracking with `HeapObjectsMap` for stable object IDs across snapshots
- Embedder graph integration via `BuildEmbedderGraphCallback`
- Detection of detached DOM nodes via `GetDetachednessCallback`

### Heap Snapshot Generation

The `HeapSnapshotGenerator` (`src/profiler/heap-snapshot-generator.h`) walks
the entire heap and builds a graph representation with:
- **Nodes**: each heap object becomes a node with type, name, size
- **Edges**: references between objects (property, element, internal, etc.)

The snapshot format is compatible with the Chrome DevTools "Memory" panel.

## Code Coverage (`src/debug/debug-coverage.h`)

V8's code coverage system supports multiple modes:

- **Best-effort**: Uses invocation counts already tracked by the feedback system
- **Precise count**: Exact per-function invocation counts (resets on collection)
- **Precise binary**: Records whether each function was executed at least once

```cpp
struct CoverageFunction {
  int start;
  int end;
  uint32_t count;
  Handle<String> name;
  std::vector<CoverageBlock> blocks;  // For block-level coverage
  bool has_block_coverage;
};

struct CoverageScript {
  Handle<Script> script;
  std::vector<CoverageFunction> functions;
};
```

Block-level coverage uses `CoverageInfo` objects attached to `DebugInfo`,
which are populated during bytecode execution. The blocks are sorted by
start position, from outer to inner.

## The d8 Shell (`src/d8/`)

The `d8` binary is V8's standalone JavaScript shell, used for testing,
benchmarking, and development. Key files:

| File              | Purpose                                              |
|-------------------|------------------------------------------------------|
| `d8.cc` / `d8.h` | Main shell implementation, REPL, file I/O            |
| `d8-console.cc`   | `console.log()` and friends                          |
| `d8-platforms.cc`  | Custom platform implementations for testing          |
| `d8-test.cc`      | Test-specific builtins (`%OptimizeFunctionOnNextCall`)|
| `d8-posix.cc`     | POSIX-specific OS interaction                        |

d8 exposes V8's internal testing functions through the `%` prefix syntax
(enabled via `--allow-natives-syntax`), which is extensively used in V8's
test suite to control optimization, GC, and debugging behavior.

## How Debugging Interacts with Compilation

When debugging is active, V8 must ensure that breakpoints can be hit.
This affects the compilation pipeline:

1. **Baseline/Sparkplug code is discarded** via `DiscardBaselineCode()` when
   breakpoints are set, forcing execution back to the interpreter where
   breakpoints can be precisely controlled.

2. **TurboFan-optimized code is deoptimized** via `DeoptimizeFunction()`,
   because optimized code may have eliminated the bytecode offsets needed
   for break locations.

3. **Debug break trampolines** are installed by `InstallDebugBreakTrampoline()`
   to intercept function entry for functions with break-at-entry breakpoints.

4. The `PrepareFunctionForDebugExecution()` method ensures a function's
   `SharedFunctionInfo` has a `DebugInfo` object and that the bytecode is
   instrumented for debugging.

## Side-Effect Checking

When evaluating expressions in the debugger (e.g., hover-to-inspect or
watch expressions), V8 enters a special "side-effect check" mode:

```cpp
void Debug::StartSideEffectCheckMode();
void Debug::StopSideEffectCheckMode();
```

In this mode, V8 intercepts each bytecode operation and checks whether it
could produce observable side effects. If a side effect is detected, the
evaluation is terminated. This allows safe "preview" evaluation of
expressions without changing program state.

## Key Relationships

- The **Inspector** depends on the **Debug** module for all execution control
- The **Profiler** is largely independent but shares the `Isolate`
- **CPU profiling** uses signal-based sampling, independent of the debug system
- **Heap profiling** triggers GC and walks the heap, coordinating with the GC
- **Code coverage** piggybacks on the debug infrastructure (`DebugInfo` / `CoverageInfo`)
- **d8** provides a thin shell around all these capabilities for testing

## Summary

V8's debugging infrastructure is a layered system: the `Debug` class provides
low-level breakpoint and stepping mechanics, the Inspector translates CDP
commands into debug operations, and the profiler modules provide independent
CPU and heap analysis. The design carefully manages the tension between
debugging capability and execution performance -- debugging features are
activated on demand and deactivated when not needed, minimizing overhead
for non-debugged code.
