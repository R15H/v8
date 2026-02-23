# V8's Memory Model

## Overview

V8's memory model encompasses how JavaScript objects are allocated, referenced,
protected, and reclaimed. The key subsystems are:

- **Pointer compression** (`src/common/ptr-compr.h`): 32-bit tagged pointers in a 4GB cage
- **Handle system** (`src/handles/`): GC-safe references to heap objects
- **Zone allocator** (`src/zone/`): Fast bump-pointer allocation for temporary data
- **Heap spaces** (`src/heap/`): Generational heap with multiple specialized spaces
- **Write barriers** (`src/heap/heap-write-barrier.h`): Tracking inter-generation references
- **Pointer tagging**: Distinguishing SMIs from heap object pointers

## Pointer Tagging

V8 uses the lowest bit of every tagged value to distinguish between:

- **SMI (Small Integer)**: bit 0 = 0, value stored shifted left by 1
- **HeapObject pointer**: bit 0 = 1 (strong ref) or bit 0 = 1 + bit 1 = 1 (weak ref)

On 64-bit platforms with pointer compression enabled (`V8_COMPRESS_POINTERS`),
tagged values are stored as 32-bit values but represent full 64-bit addresses
through compression.

```
SMI:        [...value...0]  (lowest bit 0)
Strong ref: [...addr....1]  (lowest bit 1, bit 1 = 0)
Weak ref:   [...addr...11]  (lowest bits 11)
```

The `Tagged<T>` template class (`src/objects/tagged.h`) provides type-safe
access to tagged values, with the tag bit handled transparently.

## Pointer Compression

### The Problem

On 64-bit systems, every tagged pointer consumes 8 bytes. For a JavaScript
engine where objects are pointer-heavy (arrays of pointers, property chains,
prototype references), this doubles memory usage compared to 32-bit.

### The Solution: 4GB Cage

V8's pointer compression stores only the lower 32 bits of heap object
addresses. The upper 32 bits are reconstructed from a "cage base" -- a fixed
base address for a 4GB virtual memory reservation called the "pointer
compression cage."

```cpp
// From src/common/ptr-compr.h
template <typename Cage>
class V8HeapCompressionSchemeImpl {
  static Tagged_t CompressObject(Address tagged);   // 64->32 bit
  static Address DecompressTagged(Tagged_t raw);     // 32->64 bit
  static void InitBase(Address base);                // Set cage base
  static Address base();                             // Get cage base
};
```

Compression is a simple truncation: `compressed = (uint32_t)address`.
Decompression adds the cage base: `address = base | (uint64_t)compressed`
(for heap objects; SMIs are sign-extended).

### Cage Variants

V8 supports multiple compression cage configurations:

| Cage            | Class                     | Purpose                              |
|-----------------|---------------------------|--------------------------------------|
| Main cage       | `MainCage`                | Most heap objects                    |
| Trusted cage    | `TrustedCage`             | Trusted objects outside sandbox      |
| Code cage       | `ExternalCodeCompressionScheme` | InstructionStream objects       |

The main cage base is stored as either:
- A process-wide global (`V8_COMPRESS_POINTERS_IN_SHARED_CAGE`) -- all isolates
  share one cage
- A thread-local variable (`V8_COMPRESS_POINTERS_IN_MULTIPLE_CAGES`) -- each
  isolate group has its own cage

```cpp
class MainCage : public AllStatic {
#ifdef V8_COMPRESS_POINTERS_IN_SHARED_CAGE
  static V8_EXPORT_PRIVATE uintptr_t base_;           // Global
#else
  static thread_local uintptr_t base_;                 // Per-thread
#endif
};
```

### Code Cage Special Handling

The `ExternalCodeCompressionScheme` allows the code range to cross a 4GB
boundary. Decompression is slightly more complex:

```
If compressed_cage_base <= compressed_value:
    result = upper_32_of_base | compressed_value
Else:
    result = (upper_32_of_base | compressed_value) + 4GB
```

This flexibility allows the code range to be placed closer to the `.text`
section in memory, improving relative call performance.

### `PtrComprCageAccessScope`

When multiple cages are enabled, switching between isolates requires updating
the cage base. `PtrComprCageAccessScope` is an RAII scope that saves and
restores cage bases:

```cpp
class PtrComprCageAccessScope final {
  const Address cage_base_;
  const Address code_cage_base_;
  IsolateGroup* saved_current_isolate_group_;
};
```

## The Handle System

### Why Handles Exist

V8's garbage collector can move objects (compacting GC). Raw pointers to heap
objects would become dangling after a GC. Handles provide an indirection layer
that the GC can update.

### Handle Types

V8 has evolved multiple handle types:

| Type               | Storage    | Purpose                                   |
|--------------------|------------|-------------------------------------------|
| `Handle<T>`        | Indirect   | Traditional GC-safe handle via HandleScope |
| `DirectHandle<T>`  | Direct     | Lighter handle for conservative stack scan |
| `MaybeHandle<T>`   | Indirect   | Nullable handle (can be empty)             |
| `GlobalHandle`     | Global     | Persistent cross-scope references          |
| `PersistentHandle` | Persistent | Survives HandleScope destruction           |
| `TracedHandle`     | Traced     | For C++ pointers traced by CppGC           |

### `Handle<T>` (Indirect Handle)

The traditional handle is a pointer to a pointer (`Address* location_`)
stored in `HandleScope` storage. The GC updates `*location_` when it moves
an object, so the handle transparently tracks the new address.

`HandleScope` manages a contiguous block of handle slots. Handle allocation
is a fast bump-pointer operation. When a scope is destroyed, all its handles
are invalidated. `CloseAndEscape()` promotes one handle to the parent scope.

### `DirectHandle<T>` (Direct Handle)

With conservative stack scanning, V8 can use direct handles that store the
tagged pointer directly (`Address obj_`). These are lighter weight but depend
on the GC scanning the stack for heap-pointer-like values to keep objects
alive. Requires `v8_flags.conservative_stack_scanning`.

### Global and Persistent Handles

For references that must outlive a `HandleScope`:

- **`GlobalHandles`**: Process-wide storage with weak callback support
- **`PersistentHandles`**: Transferable between threads (background compilation)
- **`TracedHandles`**: Integration with CppGC for C++ -> JS references

## Zone Allocator

### Design

The `Zone` class (`src/zone/zone.h`) provides ultra-fast bump-pointer
allocation for temporary data structures -- particularly the AST during
parsing and compilation. All memory is freed at once when the zone is destroyed.

The zone maintains a linked list of segments (8KB-32KB each). Allocation is
a simple pointer bump with 8-byte alignment:

```cpp
void* Allocate(size_t size) {
  size = RoundUp(size, kAlignmentInBytes);
  if (size > limit_ - position_) Expand(size);
  void* result = reinterpret_cast<void*>(position_);
  position_ += size;
  return result;
}
```

The zone is not thread-safe by design. Individual `Delete()` only zaps memory
in debug mode; real freeing happens on zone destruction.

Classes intended for zone allocation inherit from `ZoneObject` and use
`zone->New<T>()` for construction. `ZoneAllocationPolicy` adapts zone
allocation for STL containers (`ZoneVector<T>`, `ZoneMap<K,V>`).
`ZoneScope` provides RAII-based snapshot/restore for nested allocations.

## Heap Spaces

### Space Architecture

V8's heap is divided into multiple spaces, each with different allocation
and GC characteristics:

```
Heap
  |-- NewSpace (young generation)
  |     |-- SemiSpaceNewSpace (Scavenger) OR PagedNewSpace (Minor Mark-Sweep)
  |
  |-- OldSpace (old generation, regular objects)
  |-- CodeSpace (executable code objects)
  |-- LargeObjectSpace (objects > ~128KB)
  |-- CodeLargeObjectSpace (large code objects)
  |-- ReadOnlySpace (immutable objects, shared across isolates)
  |-- TrustedSpace (sandbox: trusted objects)
  |-- SharedSpace (cross-isolate shared objects)
  |-- StickySpace (optional: sticky-bit mark-sweep young gen)
```

### Space Class Hierarchy

```
BaseSpace
  |-- ReadOnlySpace (sealed after startup)
  |-- Space (mutable spaces)
        |-- SpaceWithLinearArea
        |     |-- PagedSpace (OldSpace, CodeSpace, SharedSpace, TrustedSpace)
        |     |-- NewSpace (SemiSpaceNewSpace, PagedNewSpace)
        |
        |-- LargeObjectSpace
              |-- CodeLargeObjectSpace
              |-- SharedLargeObjectSpace
              |-- TrustedLargeObjectSpace
```

### Page Structure

Memory is organized in pages. Each `MutablePage` (typically 256KB-512KB)
contains:
- A header with metadata (owner space, marking bitmaps, slot sets)
- An allocation area for objects
- Optionally, a `FreeList` for fragmented space

The `FreeList` tracks available memory within pages using size-bucketed
linked lists, enabling fast allocation of appropriately-sized objects.

### Allocation Flow

For regular objects:
1. **Fast path**: Bump pointer in the `LinearAllocationArea` (pointer bump)
2. **Slow path**: Refill from `FreeList` in the owning space
3. **GC path**: If no memory available, trigger garbage collection

The `MainAllocator` (`src/heap/main-allocator.h`) manages this flow,
with per-space `AllocatorPolicy` implementations.

## Write Barriers

### Purpose

Write barriers are essential for generational and concurrent garbage
collection. When a pointer field in an object is updated, the write barrier
ensures the GC is aware of the new reference.

### Barrier Types

V8 implements several write barrier variants:

| Barrier              | Purpose                                          |
|----------------------|--------------------------------------------------|
| Generational barrier | Tracks old-to-new references for minor GC        |
| Marking barrier      | Notifies concurrent marking of pointer updates   |
| Shared barrier       | Tracks cross-isolate shared-heap references       |
| Ephemeron barrier    | Special handling for WeakMap key references       |

The `WriteBarrier` class dispatches to the appropriate barrier:

```cpp
class WriteBarrier final {
  // Called from generated code:
  static int MarkingFromCode(Address raw_host, Address raw_slot);
  static int SharedMarkingFromCode(Address raw_host, Address raw_slot);
  static int SharedFromCode(Address raw_host, Address raw_slot);
  static void EphemeronKeyWriteBarrierFromCode(
      Address raw_object, Address key_slot, Isolate* isolate);
};
```

### Remembered Sets and Slot Sets

The **remembered set** tracks cross-generation pointers (old -> new).
It is implemented using **slot sets** (`src/heap/slot-set.h`), which are
bitmap-based data structures recording which slots in a page contain
pointers into the young generation.

When the minor GC runs, it only needs to scan the remembered set
(rather than the entire old generation) to find roots into the young
generation.

### `MemoryChunk` Flags

Each memory chunk (page) has flags that control write barrier behavior:

- `IS_IN_YOUNG_GENERATION`: Object is in young space (no barrier needed for
  young-to-young writes)
- `INCREMENTAL_MARKING`: Incremental marking is active
- `IS_EXECUTABLE`: Page contains executable code

The write barrier fast path checks these flags to skip unnecessary work.

## Read-Only Space

The `ReadOnlySpace` (`src/heap/read-only-spaces.h`) contains immutable
objects that are shared across isolates when `V8_SHARED_RO_HEAP` is enabled:

- All built-in `Map` objects
- Oddball values: `undefined`, `null`, `true`, `false`, `the_hole`
- Empty collections: `empty_fixed_array`, `empty_byte_array`
- The `empty_string`
- Root constants used by generated code

Pages in read-only space are sealed after initialization:
```cpp
void ReadOnlyPage::MakeHeaderRelocatableAndMarkAsSealed();
```

With static roots, the addresses of read-only objects are known at compile
time, enabling direct constant comparisons in generated machine code
rather than root-table loads.

## The `Heap` Class

The `Heap` class (`src/heap/heap.h`) is the central orchestrator:

```cpp
class Heap {
  // Spaces
  std::unique_ptr<NewSpace> new_space_;
  std::unique_ptr<OldSpace> old_space_;
  std::unique_ptr<CodeSpace> code_space_;
  std::unique_ptr<ReadOnlySpace> read_only_space_;
  std::unique_ptr<LargeObjectSpace> lo_space_;
  // ... more spaces

  // GC infrastructure
  std::unique_ptr<IncrementalMarking> incremental_marking_;
  std::unique_ptr<ConcurrentMarking> concurrent_marking_;
  std::unique_ptr<GCTracer> gc_tracer_;
  std::unique_ptr<MemoryAllocator> memory_allocator_;
  std::unique_ptr<Sweeper> sweeper_;

  // Allocation
  HeapAllocator heap_allocator_;

  // Roots
  RootsTable roots_table_;
};
```

The Heap class has an enormous API surface (the header alone is thousands
of lines) covering allocation, GC triggering, space management, statistics,
iteration, and embedder integration.

## Memory Layout Summary

```
Process Address Space:
|------------|---------|---------|---------|---------|---------|
| V8 binary  | Main    | Code    | Trusted | External|  Stack  |
| (.text)    | Cage    | Range   | Cage    | memory  |         |
|            | (4GB)   |         |         |         |         |
|------------|---------|---------|---------|---------|---------|

Main Cage (4GB):
|------------|---------|---------|---------|---------|---------|
| ReadOnly   | New     | Old     | Code    | Large   | Shared  |
| Space      | Space   | Space   | Space   | Objects | Space   |
|------------|---------|---------|---------|---------|---------|
      ^
      |-- Pointer compression base (cage base)
```

All tagged pointers within the main cage are stored as 32-bit offsets from
the cage base, cutting memory usage nearly in half for pointer-dense objects.

## Key Design Tradeoffs

1. **Pointer compression vs. address space**: Limits the heap to 4GB but
   saves ~30% memory. V8 compensates by using 64-bit pointers for external
   (C++) references.

2. **Handles vs. raw pointers**: Handles add indirection overhead but enable
   a moving GC. Direct handles with conservative stack scanning reduce this
   overhead at the cost of potential false retention.

3. **Zone allocator vs. general allocator**: Zones are extremely fast but
   cannot free individual objects. This is ideal for compiler temporaries
   but unsuitable for long-lived data.

4. **Generational GC vs. write barriers**: The young generation enables fast
   minor GCs but requires write barriers on every pointer store. The barrier
   fast path is carefully optimized (a single flag check) to minimize overhead.

5. **Read-only sharing vs. flexibility**: Sharing the read-only heap saves
   memory but means all isolates must agree on the same built-in object layout.
   Static roots further constrain this but enable faster comparisons.
