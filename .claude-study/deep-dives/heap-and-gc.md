# Deep Dive: V8 Heap & Garbage Collection

## Overview

V8's heap is a managed memory system that automatically allocates and deallocates JavaScript objects. It uses a **generational garbage collector** with two main collection strategies: a fast **Scavenger** for the young generation (minor GC) and a **Mark-Compact** collector for the old generation (major GC). Both support incremental and concurrent operation to minimize pause times.

Key files: `src/heap/heap.h`, `src/heap/scavenger.h`, `src/heap/mark-compact.h`, `src/heap/sweeper.h`, `src/heap/concurrent-marking.h`

---

## 1. Generational Heap Layout

V8's heap is divided into several memory spaces:

```
V8 Heap Layout:
┌─────────────────────────────────────────────────────┐
│                    YOUNG GENERATION                   │
│  ┌──────────────────┐  ┌──────────────────┐          │
│  │   From-Space      │  │   To-Space        │          │
│  │ (active alloc)    │  │ (copy target)     │          │
│  └──────────────────┘  └──────────────────┘          │
├─────────────────────────────────────────────────────┤
│                    OLD GENERATION                      │
│  ┌──────────────────┐  ┌──────────────────┐          │
│  │   Old Space       │  │   Code Space      │          │
│  │ (long-lived objs) │  │ (executable code) │          │
│  └──────────────────┘  └──────────────────┘          │
│  ┌──────────────────┐  ┌──────────────────┐          │
│  │  Map Space        │  │ Large Object Space│          │
│  │  (Maps only)      │  │ (>page size objs) │          │
│  └──────────────────┘  └──────────────────┘          │
├─────────────────────────────────────────────────────┤
│  ┌──────────────────┐                                │
│  │  Read-Only Space  │  (immutable roots, builtins)  │
│  └──────────────────┘                                │
└─────────────────────────────────────────────────────┘
```

### Memory Spaces

| Space | Purpose | GC Algorithm |
|-------|---------|-------------|
| **New Space** (Young Gen) | Newly allocated objects | Scavenger (semi-space copy) |
| **Old Space** | Objects surviving young gen GC | Mark-Compact |
| **Code Space** | JIT-compiled machine code | Mark-Compact (special handling for executable pages) |
| **Large Object Space** | Objects exceeding page size | Mark-Compact (never moved) |
| **Map Space** | Map objects (hidden classes) | Mark-Compact |
| **Read-Only Space** | Immutable root objects, builtins | Never collected |

### Page-Based Memory Management

**Source:** `src/heap/base-page.h`

V8 organizes heap memory into **pages** (typically 256 KB or 512 KB). Each page belongs to a specific space and contains:
- Page header with metadata (space owner, flags, marking bitmap)
- Object area for actual heap objects
- Slot sets for remembered set tracking

---

## 2. Young Generation Scavenger (Semi-Space Copying GC)

**Source:** `src/heap/scavenger.h`, `src/heap/scavenger.cc`

The young generation uses a **semi-space** design with two equally-sized spaces:

### Algorithm

```
1. Allocate new objects in From-Space (bump pointer allocation)
2. When From-Space is full, trigger minor GC (Scavenge)
3. Scan roots: stack, handles, global handles, remembered sets
4. For each live object in From-Space:
   a. If it has survived enough scavenges → promote to Old Space
   b. Otherwise → copy to To-Space
5. Update all pointers to point to new locations
6. Swap From-Space and To-Space labels
7. From-Space (now empty) is available for new allocations
```

### Allocation

Young generation allocation is extremely fast — just a bump pointer:

```cpp
// Pseudocode for young gen allocation
Address Allocate(int size) {
  Address result = allocation_top_;
  allocation_top_ += size;
  if (allocation_top_ > allocation_limit_) {
    // Trigger GC or request new page
    return SlowAllocate(size);
  }
  return result;
}
```

### Promotion Policy

Objects are promoted to old generation based on:
- **Age**: Objects that survive a threshold number of scavenges
- **Size**: Large objects may be promoted immediately
- **Pretenuring**: Allocation sites that consistently produce long-lived objects allocate directly in old space

**Source:** `src/heap/pretenuring-handler.h`

---

## 3. Mark-Compact Collector (Old Generation)

**Source:** `src/heap/mark-compact.h`, `src/heap/mark-compact.cc`

The old generation uses a **mark-sweep-compact** algorithm with three phases:

### Phase 1: Marking

Identifies all reachable (live) objects by traversing the object graph from roots.

```
Marking Algorithm:
1. Push all root objects onto marking worklist
2. While worklist is not empty:
   a. Pop object
   b. If not already marked, mark it in the marking bitmap
   c. Scan object's fields for more heap pointers
   d. Push unmarked referenced objects onto worklist
3. All unmarked objects are garbage
```

**Marking Bitmap:** Each page has a bitmap where each bit represents one word of heap memory. A set bit means the word is the start of a live object.

**Source:** `src/heap/marking-state.h`

### Phase 2: Sweeping

Reclaims memory from dead (unmarked) objects.

**Source:** `src/heap/sweeper.h`

```
Sweeping:
1. Iterate through each page
2. Find gaps between live objects (dead object regions)
3. Add gaps to the page's free list
4. Free lists are used for subsequent allocations
```

### Phase 3: Compaction (Optional)

Moves live objects to reduce fragmentation.

```
Compaction:
1. Select fragmented pages for compaction
2. Copy live objects to less-fragmented pages
3. Update all pointers to moved objects
4. Release fully evacuated pages
```

Compaction is only done for heavily fragmented pages to minimize the cost.

---

## 4. Incremental Marking

**Source:** `src/heap/incremental-marking.h`, `src/heap/incremental-marking.cc`

To avoid long pause times, marking can be done **incrementally** — interleaved with JavaScript execution:

```
Incremental Marking:
1. Start marking (push roots to worklist)
2. Do a small amount of marking work (e.g., mark N objects)
3. Return control to JavaScript execution
4. Repeat steps 2-3 until marking is complete
5. Do final pause for sweeping/compaction
```

The marking work is triggered by the **StackGuard** interrupt mechanism — V8 checks for pending marking work at loop back-edges and function calls.

### Tri-Color Marking

Incremental marking uses a tri-color scheme:
- **White**: Not yet discovered (potentially garbage)
- **Grey**: Discovered but not fully scanned (on the worklist)
- **Black**: Fully scanned (live, all references processed)

The **write barrier** ensures correctness: when JavaScript writes a pointer from a black object to a white object, the barrier marks the white object grey (pushes it onto the worklist).

---

## 5. Concurrent Marking

**Source:** `src/heap/concurrent-marking.h`, `src/heap/concurrent-marking.cc`

Marking work can also run on **background threads** concurrently with JavaScript execution:

```
Main Thread:                Background Thread(s):
┌──────────┐               ┌──────────────────┐
│ JS code  │               │ Mark objects      │
│ executes │  ←─shared─→   │ from worklist     │
│          │   worklist     │                  │
│ Write    │               │ Scan object       │
│ barriers │               │ fields            │
│ protect  │               │                  │
│ invariant│               │ Mark referenced   │
└──────────┘               │ objects           │
                           └──────────────────┘
```

### Synchronization

- The **marking worklist** is a concurrent data structure shared between the main thread and background markers
- **Write barriers** on the main thread ensure objects modified during concurrent marking are properly handled
- At the end, a brief **stop-the-world** pause finalizes marking

---

## 6. Write Barriers and Remembered Sets

### Write Barriers

Every pointer store in V8 goes through a **write barrier** that maintains GC invariants:

```cpp
// Pseudocode for write barrier
void WriteBarrier(HeapObject host, ObjectSlot slot, Object value) {
  // Generational barrier: old → young pointer
  if (InYoungGeneration(value) && !InYoungGeneration(host)) {
    RecordSlotInRememberedSet(host, slot);
  }

  // Marking barrier: ensure marking invariant during incremental/concurrent marking
  if (IsMarking() && IsBlack(host) && IsWhite(value)) {
    MarkGrey(value);  // Push to marking worklist
  }
}
```

### Remembered Sets

**Purpose:** Track pointers from old generation to young generation, so the Scavenger doesn't need to scan the entire old generation.

**Implementation:** Per-page **slot sets** that record which slots in old-generation pages point to young-generation objects.

```
Old Generation Page:
┌────────────────────────────────┐
│ Object A                       │
│   field1 → [young gen obj] ────┼── recorded in remembered set
│   field2 → [old gen obj]       │   (not recorded — same gen)
│ Object B                       │
│   field3 → [young gen obj] ────┼── recorded in remembered set
└────────────────────────────────┘
```

During Scavenge, only the remembered set entries need to be scanned, not all old-generation objects.

---

## 7. Heap Compaction and Object Migration

### When Compaction Occurs

Compaction is triggered when:
- Fragmentation exceeds a threshold
- After many sweeping cycles without compaction
- Memory pressure requires reclaiming maximum space

### Evacuation Process

```
1. Select "evacuation candidates" — highly fragmented pages
2. For each live object on an evacuation candidate:
   a. Allocate space on a target page
   b. Copy the object
   c. Install a forwarding pointer at the old location
3. Update all references (scan entire heap for pointers to moved objects)
4. Free the evacuated pages
```

### Pointer Update

After objects are moved, all pointers to them must be updated. V8 does this by:
- Scanning the entire heap (or using recorded slots for optimization)
- Checking if each pointer target was on an evacuated page
- If so, following the forwarding pointer to the new location

---

## 8. Memory Pressure Handling and GC Scheduling

**Source:** `src/heap/heap.h`

V8 uses heuristics to decide when to collect:

### GC Triggers

| Trigger | Action |
|---------|--------|
| Young generation full | Minor GC (Scavenge) |
| Old generation allocation limit reached | Major GC (Mark-Compact) |
| Memory pressure from OS | Aggressive GC + memory reduction |
| Idle notification from embedder | Incremental marking steps |
| Explicit `gc()` call (d8 debug) | Full GC |

### Heap Growing Strategy

The heap limit grows dynamically based on:
- Live object size after last GC
- A growth factor (decreasing as heap grows)
- Hard limits set by the embedder (`--max-heap-size`)

```
new_limit = live_size_after_gc * growth_factor + slack
```

---

## 9. Weak References and Weak Callbacks

### Weak References

V8 supports several weak reference mechanisms:

- **WeakFixedArray**: Array of weak pointers; entries cleared when targets are collected
- **WeakCell**: Single weak reference with a callback
- **EphemeronHashTable**: Hash table where entries are kept alive only if the key is alive (used for WeakMap/WeakSet)

### Weak Callbacks

The embedder API (`include/v8-weak-callback-info.h`) provides weak persistent handles:

```cpp
// Register a weak callback on a Global handle
Global<Object> persistent(isolate, object);
persistent.SetWeak(parameter, weak_callback, WeakCallbackType::kParameter);
```

When the referenced object is collected, the callback is invoked, allowing the embedder to clean up associated native resources.

### Weak Processing Order

During GC, weak references are processed in a specific order:
1. Mark all strongly reachable objects
2. Process ephemerons (WeakMap/WeakSet entries)
3. Clear weak references to unreachable objects
4. Invoke weak callbacks
5. Clear finalization registry entries

---

## 10. Embedder Heap Tracing (cppgc / Oilpan Integration)

**Source:** `src/heap/cppgc/`, `include/cppgc/`

V8 includes a separate garbage collector for C++ objects called **cppgc** (also known as Oilpan, originally from Blink):

### Purpose

- Manages C++ objects that participate in the GC object graph
- Supports tracing across the V8 ↔ C++ boundary
- Used by Chromium for DOM objects that reference JavaScript objects and vice versa

### Architecture

```
V8 Heap (JavaScript objects)     cppgc Heap (C++ objects)
┌──────────────────────┐        ┌──────────────────────┐
│ JSObject             │        │ C++ DOMNode          │
│   → C++ wrapper ─────┼────→   │   → JSObject ────────┼──→
│                      │        │                      │
│ GC traces into ──────┼────→   │ GC traces into ──────┼──→
│ cppgc objects        │        │ V8 objects            │
└──────────────────────┘        └──────────────────────┘
```

### Unified Heap

The **unified heap** mode synchronizes V8's GC with cppgc's GC:
- Both heaps are traced together during marking
- Cross-heap references are properly tracked
- Prevents premature collection of objects referenced only from the other heap

### cppgc GC Algorithm

cppgc uses its own mark-sweep collector:
- **Marking**: Traces C++ object graph using `Trace()` methods
- **Sweeping**: Frees unreachable C++ objects, calls destructors
- Supports concurrent marking and sweeping
- Integrates with V8's GC scheduling

### Key cppgc Types

```cpp
// cppgc managed object
class MyObject : public cppgc::GarbageCollected<MyObject> {
  void Trace(cppgc::Visitor* visitor) const {
    visitor->Trace(member_);  // Trace member references
  }
  cppgc::Member<OtherObject> member_;
};

// Persistent handle (prevent collection)
cppgc::Persistent<MyObject> persistent;

// Weak member (cleared when target collected)
cppgc::WeakMember<MyObject> weak;
```

---

## GC Performance Characteristics

| GC Type | Pause Time | Frequency | Scope |
|---------|-----------|-----------|-------|
| Minor GC (Scavenge) | ~1-10 ms | Frequent | Young generation only |
| Major GC (Mark-Compact) | ~10-100 ms | Infrequent | Entire heap |
| Incremental Marking | ~0.5-5 ms per step | Many small steps | Old generation |
| Concurrent Marking | Near zero main-thread pause | Background | Old generation |

### Key Optimizations

1. **Generational hypothesis**: Most objects die young → frequent cheap young gen GC
2. **Bump pointer allocation**: Young gen allocation is just incrementing a pointer
3. **Concurrent marking**: Background threads do most marking work
4. **Incremental marking**: Spreads marking work over many small pauses
5. **Concurrent sweeping**: Background threads rebuild free lists
6. **Pretenuring**: Allocate long-lived objects directly in old space
7. **Page-based remembered sets**: Efficient tracking of cross-generation pointers
