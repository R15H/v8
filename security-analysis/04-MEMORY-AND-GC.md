# Memory Management & Garbage Collector Security Analysis

## 1. Heap Architecture

V8's heap is divided into multiple spaces, each with different properties:

```
Space                  │ Contents                        │ GC Strategy
───────────────────────┼─────────────────────────────────┼────────────────────
New Space (Young Gen)  │ Recently allocated objects       │ Scavenger (copying)
  Semi-space (from)    │ Current allocation space         │
  Semi-space (to)      │ Survivor space                   │
Old Space              │ Long-lived objects                │ Mark-Compact
Code Space             │ Compiled code (JIT output)       │ Mark-Compact
Large Object Space     │ Objects > kMaxRegularHeapObj      │ Mark-Compact
Read-Only Space        │ Immortal shared objects           │ Never collected
Trusted Space          │ Sandbox: outside-sandbox objects  │ Mark-Compact
Shared Space           │ Cross-isolate shared objects      │ Special
```

### Key Files
- `src/heap/heap.h` / `src/heap/heap.cc` (297KB) - Main heap class
- `src/heap/spaces.h` - Space definitions
- `src/heap/new-spaces.h` - Young generation spaces
- `src/heap/paged-spaces.h` - Old generation paged spaces
- `src/heap/large-spaces.h` - Large object space
- `src/heap/read-only-spaces.h` - Read-only space

## 2. Garbage Collector (Orinoco)

### Scavenger (Young Generation)

The scavenger uses a semi-space copying algorithm:
1. Allocate in "from" space until full
2. Copy live objects to "to" space (or promote to old space)
3. Swap "from" and "to" labels

**Security-relevant**: During copying, all references to moved objects must
be updated. If a reference is missed, it becomes a dangling pointer.

Key file: `src/heap/scavenger.cc`

### Mark-Compact (Old Generation)

Three phases:
1. **Marking**: Traverse live object graph (concurrent + incremental)
2. **Sweeping**: Free unreachable objects
3. **Compaction**: Move live objects to reduce fragmentation

**Security-relevant**: Concurrent marking runs on background threads while
the main thread mutates the heap. Write barriers ensure the marker sees all
new references. Missing write barriers → missed references → premature
collection → use-after-free.

Key files:
- `src/heap/mark-compact.cc` - Main Mark-Compact implementation
- `src/heap/concurrent-marking.cc` - Background marking thread
- `src/heap/sweeper.cc` - Sweeping phase

## 3. Write Barriers

Write barriers are the primary defense against GC-related use-after-free bugs.

### Types of Write Barriers

```
1. Generational Barrier (Young → Old reference tracking)
   When: Object in old space gets a pointer to object in young space
   Action: Record the slot in the remembered set
   Why: Scavenger only scans young space + remembered set

2. Marking Barrier (Concurrent marking support)
   When: Object is written during concurrent marking
   Action: Mark the written value and/or record the slot
   Why: Prevents the marker from missing new references

3. Shared Barrier (Cross-isolate references)
   When: Object gets a reference to shared heap object
   Action: Record in shared remembered set

4. Indirect Pointer Barrier (Sandbox pointer tables)
   When: Indirect pointer slot is written
   Action: Mark the indirectly-referenced object
```

### Implementation

Key file: `src/heap/heap-write-barrier.cc`

The write barrier has multiple fast paths:

```
WriteBarrier::ForValue()  →  Marking()  →  MarkingSlow()
                          →  Generational() → GenerationalSlow()
                          →  Shared() → SharedSlow()
```

The fast path checks simple conditions (is the target in young space?
is marking active?) and defers to slow paths for complex cases.

**Vulnerability pattern**: If a code path writes a pointer without going
through WriteBarrier::ForValue(), the barrier is skipped entirely. This
happens most often in:
- Hand-written assembler stubs (CSA / CodeStubAssembler)
- Newly added C++ code paths
- Code that writes raw memory instead of using accessor functions
- Optimized JIT code that incorrectly elides the barrier

### Write Barrier Documentation

V8 includes documentation in `src/heap/WRITE_BARRIER.md` that describes
the barrier protocol. Key quote:

> Every store to a HeapObject slot must be followed by a write barrier.
> The only exception is during object initialization, when the object
> hasn't been published yet.

## 4. Object Model Details

### Tagged Pointers

V8 uses tagged pointers (low bit = 0 for Smi, low bit = 1 for HeapObject):

```
Smi (Small Integer):
  [value (31 bits)] [0]    (on 32-bit)
  [value (32 bits)] [0000...0000]  (on 64-bit, upper 32 bits are value)

HeapObject Pointer:
  [address] [1]    (low bit set = heap object tag)
```

With pointer compression (enabled by default), heap pointers are stored
as 32-bit offsets from a cage base. The cage base is the start of the
4GB pointer compression region.

### Maps (Hidden Classes)

Every HeapObject starts with a Map pointer. The Map describes:
- Instance type (what kind of object)
- Instance size
- Property descriptors
- Element kind
- Prototype

**Security-critical**: If an attacker can corrupt the Map pointer, they can
change how V8 interprets the object's memory layout. This is the basis of
"type confusion" attacks.

Key file: `src/objects/map.h` (line 650+ for security-related properties)

### Element Kinds Transitions

Element kind transitions form a lattice:

```
                    PACKED_SMI_ELEMENTS
                    /                \
        HOLEY_SMI_ELEMENTS    PACKED_DOUBLE_ELEMENTS
                    \                /
                HOLEY_DOUBLE_ELEMENTS
                    /                \
            PACKED_ELEMENTS    (further transitions)
                    \                /
                HOLEY_ELEMENTS
```

Key invariants:
1. Transitions only go "down" the lattice (more general)
2. Once HOLEY, always HOLEY
3. Once DOUBLE, elements stored as raw doubles (not tagged)
4. TypedArray element kinds are fixed and never transition

**Vulnerability**: If an element kind transition is incorrectly applied:
- PACKED_DOUBLE → PACKED: engine reads raw doubles as tagged pointers
- PACKED → PACKED_DOUBLE: engine reads tagged pointers as raw doubles
- Both can be used for addrof/fakeobj exploit primitives

Key files:
- `src/objects/elements-kind.h` - Element kind enum and transitions
- `src/objects/elements.cc` - Element accessor implementations
- `src/objects/elements-kind.cc` - Transition logic

## 5. ArrayBuffer and TypedArray

### ArrayBuffer

ArrayBuffers store raw binary data in a "backing store" allocated outside
the V8 heap (but inside the sandbox when enabled).

```
JSArrayBuffer object (on V8 heap):
  ├── Map pointer
  ├── backing_store pointer → raw memory (sandbox or external)
  ├── byte_length
  ├── max_byte_length (for resizable)
  └── bit_field (detached flag, shared flag, resizable flag)
```

**Security-critical fields**:
- `backing_store`: If corrupted → arbitrary memory access
- `byte_length`: If corrupted → OOB access
- `bit_field.is_detached`: If corrupted → access detached buffer

### Resizable ArrayBuffers (RAB) and Growable SharedArrayBuffers (GSAB)

Added by TC39 proposal. These allow buffers to be resized after creation.

**Security complexity**: TypedArray views over RAB/GSAB must recheck their
bounds on every access because the buffer size can change at any time.
All RAB/GSAB TypedArrays have separate element kinds:
`RAB_GSAB_UINT8_ELEMENTS`, `RAB_GSAB_FLOAT64_ELEMENTS`, etc.

If the JIT compiler caches the buffer length and doesn't recheck after
a potential resize point (GC, function call), it may use a stale length
→ OOB access.

Key files:
- `src/objects/js-array-buffer.h` - ArrayBuffer definition
- `src/objects/js-array-buffer-inl.h` - Inline accessors
- `src/builtins/builtins-typed-array.cc` - TypedArray builtins

## 6. Factory and Object Allocation

Objects are allocated through the Factory class:

```
Factory::NewJSArray()
Factory::NewJSArrayBuffer()
Factory::NewMap()
Factory::NewFixedArray()
...
```

**Security-relevant**: If allocation fails (OOM), V8 may throw an exception
or trigger GC. Code that doesn't handle allocation failure correctly may
use uninitialized or freed memory.

Key file: `src/heap/factory.cc` (214KB)

## 7. GC Attack Patterns

### Pattern 1: Missing Write Barrier → UAF

```
1. Create object A in old space with reference to object B in young space
2. Modify A to point to new object C in young space (without write barrier)
3. Trigger scavenge: B is moved, C is not found (not in remembered set)
4. C is freed because scavenger thinks it's unreachable
5. Access through A → use-after-free on C's memory
```

### Pattern 2: Concurrent Marking Race

```
1. Start concurrent marking
2. On main thread, allocate new object and store pointer to it
3. If marking barrier is missing, the new object may not be marked
4. Mark-Compact frees the unmarked object
5. Next access → use-after-free
```

### Pattern 3: Object Movement During Compaction

```
1. Object at address X contains a reference to object Y
2. During compaction, Y is moved to address Y'
3. If the reference X→Y is not updated to X→Y', X has dangling pointer
4. Access through X → accesses freed memory at Y
```

### Pattern 4: TypedArray Detachment

```
1. Create ArrayBuffer + TypedArray view
2. Detach (or resize) the ArrayBuffer via another reference
3. Access TypedArray → if detachment check was optimized away, OOB access
```

## 8. Verification Checklist

To find GC-related vulnerabilities, audit these code patterns:

- [ ] Any code that writes to HeapObject fields without using accessor
      functions (bypasses write barrier)
- [ ] CSA (CodeStubAssembler) code that does StoreNoWriteBarrier
- [ ] JIT-generated code that stores references without barriers
- [ ] Code that caches pointers across GC-safe points (function calls,
      allocation, deoptimization)
- [ ] TypedArray code that caches buffer length across resize points
- [ ] Code that accesses objects during GC (incremental marking callbacks)
