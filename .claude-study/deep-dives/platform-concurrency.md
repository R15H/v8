# Platform Abstraction and Concurrency in V8

## Overview

V8 is designed to be embeddable in diverse environments -- Chrome, Node.js,
Deno, standalone applications, and embedded systems. To achieve this, V8
defines a **platform abstraction layer** that decouples the engine from
OS-specific primitives like threads, memory allocation, and time functions.
The key components are:

- **`v8::Platform`** (`include/v8-platform.h`): Abstract interface for embedders
- **`src/libplatform/`**: V8's default platform implementation
- **`src/base/`**: Low-level OS abstractions (mutexes, atomics, memory, CPU)
- **`src/tasks/`**: Internal task scheduling infrastructure

## The Platform API (`include/v8-platform.h`)

### Task Priorities

V8 defines three task priority levels:

```cpp
enum class TaskPriority : uint8_t {
  kBestEffort,   // Non-critical background work
  kUserVisible,  // Background compilation, GC (visible in performance)
  kUserBlocking, // Major GC, urgent tasks blocking execution
};
```

### `Task` and `IdleTask`

The fundamental work units:

```cpp
class Task {
  virtual void Run() = 0;
};

class IdleTask {
  virtual void Run(double deadline_in_seconds) = 0;
};
```

`IdleTask` receives a deadline parameter, allowing it to yield before
the embedding application needs the thread back. This is used extensively
by V8's incremental GC to perform work during idle periods.

### `TaskRunner`

The `TaskRunner` abstract class provides the scheduling interface for a
single execution context (typically associated with one isolate):

| Method                          | Purpose                                |
|---------------------------------|----------------------------------------|
| `PostTask`                      | Schedule immediate execution            |
| `PostDelayedTask`               | Schedule after a delay                  |
| `PostIdleTask`                  | Schedule for idle time                  |
| `PostNonNestableTask`           | Prevent re-entrant execution            |
| `PostNonNestableDelayedTask`    | Non-nestable with delay                 |

The non-nestable variants are critical for correctness: tasks that execute
JavaScript must be non-nestable because JS execution cannot safely nest
(e.g., inside a microtask callback that was called during another JS execution).

All TaskRunner methods delegate to corresponding `*Impl` virtual methods,
which embedders override.

### `v8::Platform`

The main platform interface that embedders must implement:

```cpp
class Platform {
  virtual int NumberOfWorkerThreads() = 0;
  virtual std::shared_ptr<TaskRunner> GetForegroundTaskRunner(
      Isolate*, TaskPriority) = 0;
  virtual void PostTaskOnWorkerThread(TaskPriority, std::unique_ptr<Task>) = 0;
  virtual void PostDelayedTaskOnWorkerThread(
      TaskPriority, std::unique_ptr<Task>, double delay) = 0;
  virtual bool IdleTasksEnabled(Isolate*) = 0;
  virtual std::unique_ptr<JobHandle> CreateJob(
      TaskPriority, std::unique_ptr<JobTask>) = 0;
  virtual double MonotonicallyIncreasingTime() = 0;
  virtual double CurrentClockTimeMillis() = 0;
  virtual TracingController* GetTracingController() = 0;
  virtual PageAllocator* GetPageAllocator() = 0;
};
```

### `JobTask` and `JobHandle`

For parallel workloads (like concurrent marking), V8 uses the Job API:

```cpp
class JobTask {
  virtual void Run(JobDelegate* delegate) = 0;
  virtual size_t GetMaxConcurrency(size_t worker_count) const = 0;
};
```

The `GetMaxConcurrency()` method allows dynamic scaling -- V8 can adjust
the number of workers based on how much work remains. The platform is
responsible for creating worker threads and calling `Run()` concurrently.

## Default Platform Implementation (`src/libplatform/`)

### `DefaultPlatform`

V8 ships a reference platform implementation suitable for most embedders:

```cpp
class DefaultPlatform : public Platform {
  base::Mutex lock_;
  const int thread_pool_size_;
  IdleTaskSupport idle_task_support_;
  std::shared_ptr<DefaultWorkerThreadsTaskRunner>
      worker_threads_task_runners_[3];  // One per priority level
  std::map<Isolate*, std::shared_ptr<DefaultForegroundTaskRunner>>
      foreground_task_runner_map_;
  std::unique_ptr<TracingController> tracing_controller_;
  std::unique_ptr<PageAllocator> page_allocator_;
  DefaultThreadIsolatedAllocator thread_isolated_allocator_;
  const PriorityMode priority_mode_;
};
```

When `PriorityMode::kDontApply` is set, all priorities collapse to a
single thread pool. When priorities are applied, three separate
`DefaultWorkerThreadsTaskRunner` instances are created, each with threads
at the corresponding OS priority level.

### `DefaultWorkerThreadsTaskRunner`

Manages a pool of worker threads:

```cpp
class DefaultWorkerThreadsTaskRunner : public TaskRunner {
  std::vector<std::unique_ptr<WorkerThread>> thread_pool_;
  std::vector<WorkerThread*> idle_threads_;  // LIFO for locality
  DelayedTaskQueue queue_;
  std::queue<std::unique_ptr<Task>> task_queue_;
  base::Mutex lock_;
};
```

Worker threads are managed using a LIFO idle-thread stack -- the most
recently active thread is reactivated first, improving CPU cache locality.
Each worker thread runs a `GetNext()` loop, blocking on a condition
variable when no tasks are available.

### `DefaultForegroundTaskRunner`

Handles per-isolate foreground tasks. These run on the embedder's main
thread via `PumpMessageLoop()`:

```cpp
bool DefaultPlatform::PumpMessageLoop(
    Isolate* isolate,
    MessageLoopBehavior behavior = MessageLoopBehavior::kDoNotWait);
```

Embedders call this in their event loop to drain V8's foreground task queue.

### `DefaultJob`

The `DefaultJob` (`src/libplatform/default-job.h`) implements the Job API
by spawning worker tasks that call `JobTask::Run()`. It manages concurrency
by checking `GetMaxConcurrency()` and only scheduling as many tasks as needed.

### `DelayedTaskQueue`

The `DelayedTaskQueue` (`src/libplatform/delayed-task-queue.h`) manages
time-delayed tasks using a sorted queue. Tasks are held until their trigger
time, then moved to the immediate execution queue.

## Base Layer (`src/base/`)

### Threading Primitives

`src/base/platform/` provides portable threading abstractions:

| Class              | File                    | Purpose                       |
|--------------------|-------------------------|-------------------------------|
| `Thread`           | `platform.h`            | OS thread wrapper             |
| `Mutex`            | `mutex.h`               | Non-recursive mutex           |
| `RecursiveMutex`   | `mutex.h`               | Recursive mutex               |
| `ConditionVariable`| `condition-variable.h`  | Condition variable            |
| `Semaphore`        | `semaphore.h`           | Counting semaphore            |
| `SharedMutex`      | `mutex.h`               | Reader-writer lock            |

All of these wrap OS primitives (pthreads on POSIX, Win32 on Windows).
The `Thread` class supports setting priority levels:

```cpp
enum class Priority {
  kBestEffort,
  kUserVisible,
  kUserBlocking,
  kDefault,
};
```

### Atomic Utilities (`src/base/atomic-utils.h`)

V8 provides its own atomic utility classes for common patterns:

- **`AtomicValue<T>`** -- atomic load/store wrapper
- **`AtomicWord`** -- platform word-sized atomic operations
- **`AsAtomicWord`** / **`AsAtomic32`** -- cast-based atomic access for
  non-atomic fields (used extensively for concurrent GC)
- **`Relaxed_Load` / `Relaxed_Store`** -- memory-order-relaxed operations

These are crucial for V8's concurrent garbage collector, which reads object
fields concurrently with the mutator thread.

### Virtual Memory (`src/base/virtual-address-space.h`)

V8's virtual memory abstraction handles:

- **`VirtualAddressSpace`** -- process-wide virtual address space
- **`VirtualAddressSubspace`** -- reserved ranges within the address space
- **`EmulatedVirtualAddressSubspace`** -- for platforms without OS-level subspaces

The `PageAllocator` interface (`include/v8-platform.h`) provides:
- `AllocatePages()` / `FreePages()` -- page-aligned allocation
- `SetPermissions()` -- change page protection (R/W/X)
- `AllocateSharedPages()` -- shared memory allocation
- `GetRandomMmapAddr()` -- ASLR-compatible random addresses

V8's `BoundedPageAllocator` (`src/base/bounded-page-allocator.h`) wraps a
`PageAllocator` to constrain allocations within a specific address range,
which is essential for pointer compression cages.

### CPU Detection (`src/base/cpu.h`)

The `CPU` class detects hardware features at runtime:

```cpp
class CPU final {
  // x86 features
  bool has_sse2(), has_sse3(), has_ssse3(), has_sse41(), has_sse42();
  bool has_avx(), has_avx2(), has_avx_vnni(), has_avx_vnni_int8();
  bool has_fma3(), has_bmi1(), has_bmi2(), has_lzcnt(), has_popcnt();
  // ARM features
  int implementer(), part(), architecture();
  // General
  int icache_line_size(), dcache_line_size();
  bool has_fpu();
};
```

This information drives code generation decisions -- for example, whether
to emit AVX instructions, which ARM NEON operations are available, or what
the cache line size is for alignment optimization.

### Region Allocator (`src/base/region-allocator.h`)

The `RegionAllocator` manages a contiguous virtual address range, tracking
allocated and free regions. It is used by `BoundedPageAllocator` to manage
the pointer compression cage's address space.

## Task Scheduling in V8

### Internal Task Types

V8 uses the platform's task infrastructure for many internal operations:

| Task Type                    | Purpose                                |
|------------------------------|----------------------------------------|
| Concurrent marking tasks     | Parallel GC marking                    |
| Sweeper tasks                | Background memory sweeping             |
| Compilation tasks            | Background JS/Wasm compilation         |
| Minor GC job                 | Young-generation collection scheduling |
| Incremental marking job      | Idle-time incremental GC               |
| Array buffer sweeper tasks   | Freeing detached ArrayBuffer memory    |
| Code serializer tasks        | Background code cache serialization    |

### Foreground vs Background

V8 strictly separates foreground (main-thread) and background work:

- **Foreground tasks** access the isolate and heap directly. They are
  posted via `GetForegroundTaskRunner()` and run during `PumpMessageLoop()`.
- **Background tasks** run on worker threads. They cannot directly access
  the heap and must use `LocalHeap` / `LocalIsolate` for GC-safe access.

The `ParkedScope` / `UnparkedScope` mechanism coordinates between background
threads and the main thread during safepoints (GC pauses).

### Safepoints and Parking

When the GC needs to stop all threads (a "safepoint"), it uses the
`IsolateSafepoint` mechanism:

1. The main thread requests a safepoint
2. Background threads with `LocalHeap` instances "park" (block at
   safepoint check)
3. GC work proceeds on the main thread
4. Background threads are unparked and resume

## Isolate Groups

An `IsolateGroup` is a collection of isolates that share:
- A pointer compression cage (same 4GB address range)
- A code range (for near-call optimization)
- Read-only space (shared immutable objects)
- Possibly a shared heap (for cross-isolate shared objects)

This is the mechanism that enables memory sharing in Chrome's site isolation
architecture: multiple V8 isolates (one per renderer) share a single pointer
compression cage and read-only heap.

## Thread-Isolated Allocator

The `DefaultThreadIsolatedAllocator` (`src/libplatform/default-thread-isolated-allocator.h`)
provides thread-isolated memory allocation, used for security-sensitive data
like code pages. On supported platforms (Linux with PKU/MPK), it uses
memory protection keys to ensure that only the allocating thread can write
to certain memory regions, protecting JIT code from corruption.

## Tracing

V8 integrates with the embedder's tracing infrastructure through
`TracingController`. The `src/libplatform/tracing/` directory provides
a default implementation that supports:
- Chrome's trace event format (compatible with `chrome://tracing`)
- Perfetto integration for modern Chrome builds
- Trace categories for V8-specific subsystems (GC, compiler, etc.)

## Summary of Key Design Principles

1. **Embedder control**: V8 never creates threads or allocates memory directly.
   All such operations go through the `Platform` interface, giving embedders
   full control over resource usage.

2. **Priority awareness**: The three-tier priority system (best-effort,
   user-visible, user-blocking) allows embedders to schedule V8's background
   work appropriately for their use case.

3. **Non-nestable task safety**: The distinction between nestable and
   non-nestable tasks prevents re-entrant JavaScript execution bugs that
   would otherwise be extremely difficult to diagnose.

4. **Portable abstractions**: The `src/base/` layer provides a uniform API
   across all supported platforms (Linux, macOS, Windows, Android, iOS, etc.)
   while allowing platform-specific optimizations.

5. **Concurrent GC support**: The atomics, safepoints, and local heap
   mechanisms enable V8's concurrent and incremental garbage collector to
   run marking and sweeping work on background threads safely.
