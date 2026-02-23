# V8 Security: The Sandbox and Mitigations

## Overview

V8's security architecture addresses the unique challenges of running untrusted code in a high-performance JIT compiler. The centerpiece is the **V8 Sandbox**, a large virtual address space region that confines V8's objects and prevents memory corruption from escaping to the rest of the process. Surrounding this are additional mitigations including pointer compression, indirection tables, Spectre defenses, and fuzzing infrastructure.

Key source directory: `src/sandbox/`

## The V8 Sandbox

### Threat Model

The sandbox assumes that an attacker has the ability to **arbitrarily corrupt memory inside the sandbox** by exploiting a V8 vulnerability. The goal is to prevent this in-sandbox corruption from being leveraged to corrupt memory outside the sandbox or to achieve arbitrary code execution in the host process.

### Address Space Layout

The sandbox is a large, contiguous virtual address space reservation (ideally ~1 TB):

```
+-  ~~~  -+----------------------------------------  ~~~  -+-  ~~~  -+
|  32 GB  |                 (Ideally) 1 TB                 |  32 GB  |
|         |                                                |         |
| Guard   |      4 GB      :  ArrayBuffer backing stores,  | Guard   |
| Region  |    V8 Heap     :  WASM memory buffers, and     | Region  |
| (front) |     Region     :  any other sandboxed objects. | (back)  |
+-  ~~~  -+----------------+-----------------------  ~~~  -+-  ~~~  -+
          ^                                                ^
          base                                             end
```

Key components:
- **V8 Heap Region** (first 4 GB): Contains most V8 objects, using compressed (32-bit) pointers
- **Extended sandbox region**: ArrayBuffer backing stores, WASM linear memory, and other large allocations
- **Guard regions** (32 GB each side): Unmapped pages that catch out-of-bounds accesses

The `Sandbox` class (`src/sandbox/sandbox.h`) manages this address space:

```cpp
class Sandbox {
  Address base() const;
  Address end() const;
  size_t size() const;
  v8::VirtualAddressSpace* address_space() const;
  v8::PageAllocator* page_allocator() const;
  bool is_partially_reserved() const;
};
```

### Partial Sandbox Fallback

If the full 1 TB reservation cannot be obtained, V8 falls back to a **partially-reserved sandbox** with a smaller reservation backed by an `EmulatedVirtualAddressSubspace`. This provides reduced security guarantees, as unrelated mappings may end up inside the sandbox address range.

### Smi Confusion Mitigation

During initialization, the sandbox also attempts to make the first 4 GB of address space inaccessible. This mitigates **Smi/HeapObject confusion** vulnerabilities where a 32-bit Smi value is mistakenly treated as a pointer and dereferenced.

## Pointer Compression

V8's pointer compression (enabled by `V8_COMPRESS_POINTERS`) is foundational to the sandbox:

- All V8 heap object pointers are stored as **32-bit offsets** from a base address
- The base address is aligned to 4 GB, so 32-bit offsets can address a 4 GB heap region
- This halves pointer storage size and places all objects in a bounded region

Pointer compression means an attacker who corrupts a compressed pointer can only point to memory within the 4 GB heap region, not to arbitrary process memory.

## Indirection Tables

The sandbox uses several **indirection tables** to protect pointers that cross the sandbox boundary. Instead of storing raw pointers to external (out-of-sandbox) objects, V8 stores **indices** into per-isolate tables. The tables themselves are outside the sandbox (or write-protected), preventing an attacker from crafting arbitrary external pointers.

### External Pointer Table

The `ExternalPointerTable` (`src/sandbox/external-pointer-table.h`) protects external (non-V8-heap) pointers:

```cpp
struct ExternalPointerTableEntry {
  // Contains: external pointer | type tag | marking bit
  void MakeExternalPointerEntry(Address value, ExternalPointerTag tag, bool mark_as_alive);
  Address GetExternalPointer(ExternalPointerTagRange tag_range) const;
  void SetExternalPointer(Address value, ExternalPointerTag tag);
  ExternalPointerTag GetExternalPointerTag() const;
};
```

Each entry stores:
- The external pointer value
- A **type tag** in the unused upper bits
- A **marking bit** for GC

When loading an external pointer, the tag is verified against the expected tag range. If there is a mismatch (e.g., an attacker swapped entries), the resulting pointer will be invalid and cannot be meaningfully dereferenced.

Entry types include freelist entries (for allocation) and evacuation entries (for table compaction during GC).

### Code Pointer Table

The `CodePointerTable` (`src/sandbox/code-pointer-table.h`) protects pointers to executable `Code` objects:

```cpp
struct CodePointerTableEntry {
  static constexpr bool IsWriteProtected = true;

  void MakeCodePointerEntry(Address code, Address entrypoint,
                            CodeEntrypointTag tag, bool mark_as_alive);
  Address GetEntrypoint(CodeEntrypointTag tag) const;
  Address GetCodeObject() const;
};
```

Each entry stores both:
- A pointer to the `Code` heap object
- The code's **entrypoint address** (for direct calls)
- A `CodeEntrypointTag` for entrypoint verification

The table is **write-protected** on platforms that support it, providing forward-edge **control flow integrity (CFI)**. An attacker cannot modify code pointers to redirect execution to arbitrary addresses.

### JS Dispatch Table

The `JSDispatchTable` (`src/sandbox/js-dispatch-table.h`) is specialized for JavaScript function dispatch:

```cpp
struct JSDispatchEntry {
  static constexpr bool IsWriteProtected = true;

  Address GetEntrypoint() const;
  Address GetCodePointer() const;
  Tagged<Code> GetCode() const;
  uint16_t GetParameterCount() const;
  void SetCodeAndEntrypointPointer(Address new_object, Address new_entrypoint);
};
```

Each entry contains:
- The function's entrypoint
- A pointer to the Code object
- The parameter count (serving as a lightweight signature check)

This enables seamless tiering: when a function is recompiled (e.g., Ignition to Sparkplug to Maglev), only the dispatch table entry is updated. All call sites automatically pick up the new code.

### Trusted Pointer Table

The trusted pointer table stores pointers to "trusted" objects -- objects that live outside the main sandbox heap and cannot be corrupted by an attacker. Each entry includes a 15-bit type tag:

```
+------------+----------+-----------------+
| 15-bit tag | mark bit | 48-bit payload  |
+------------+----------+-----------------+
```

The `IndirectPointerTag` enum defines valid tags for different trusted object types (WASM instance data, bytecode arrays, etc.).

### CppHeap Pointer Table

The `CppHeapPointerTable` (`src/sandbox/cppheap-pointer-table.h`) protects pointers to C++ heap objects managed by the embedder's `CppHeap` (used by Blink for DOM objects, etc.).

## SBXCHECK: Sandbox-Critical Assertions

The `SBXCHECK` macro (`src/sandbox/check.h`) marks security-critical invariant checks:

```cpp
#define SBXCHECK(condition)
#define SBXCHECK_EQ(lhs, rhs)
#define SBXCHECK_BOUNDS(index, limit)
```

These behave like `CHECK` but serve as documentation that the check prevents a sandbox bypass. Key properties:
- They remain enabled in release builds (like CHECK, unlike DCHECK)
- In debug builds with hardware sandbox support, they temporarily block sandbox memory access during the check to prevent TOCTOU races
- They document security-critical code paths for auditing

## Hardware Support

`hardware-support.h/.cc` provides platform-specific hardware features for sandbox enforcement:
- Memory protection keys (MPK) on x86-64 for efficient write-protection
- `DisallowSandboxAccess` scope for debugging TOCTOU issues

## Sandboxed Pointer Types

### Sandboxed Pointers

`sandboxed-pointer.h` defines pointers that are stored as offsets from the sandbox base. Any sandboxed pointer value, even if corrupted, can only reference memory inside the sandbox.

### Bounded Size

`bounded-size.h` defines size fields with maximum bounds, preventing an attacker from corrupting a size field to cause an out-of-bounds access beyond the sandbox.

## Spectre Mitigations

V8 includes several mitigations for Spectre-class speculative execution attacks:

### Branch Poisoning

On Spectre-vulnerable architectures, V8 can "poison" speculative values by masking them with a condition-dependent value that is all-ones on the correct path and all-zeros on the speculative (wrong) path.

### Site Isolation

V8 relies on Chromium's site isolation to ensure that different security origins run in different processes, limiting the impact of Spectre attacks.

### Constant-Time Operations

For cryptographic and security-sensitive operations, V8 avoids data-dependent branching that could leak information through timing side channels.

### ArrayBuffer and TypedArray Bounds

ArrayBuffer accesses include bounds checks that are designed to be non-speculative, preventing Spectre gadgets from reading out-of-bounds memory.

## Bytecode Verification

`bytecode-verifier.h/.cc` verifies the integrity of bytecode arrays, ensuring that corrupted bytecode cannot lead to sandbox escapes. This is particularly important because bytecode controls interpreter execution and could be a target for corruption.

## Code Sandboxing

`code-sandboxing-mode.h` defines modes for code object protection:
- Code objects may be placed in read-execute-only memory
- The code pointer table provides indirection to prevent direct code pointer manipulation
- Code entrypoint tags enable type-safe indirect calls

## Fuzzing

V8 has extensive fuzzing infrastructure for finding security vulnerabilities:

### Fuzzers

V8 includes fuzzers for:
- JavaScript parsing and execution (`test/fuzzer/`)
- WebAssembly module decoding and compilation
- Regular expression compilation
- JSON parsing
- The sandbox itself (corruption simulation)

### ClusterFuzz

V8 is continuously fuzzed by Google's ClusterFuzz infrastructure, which runs thousands of fuzzer instances and automatically reports crashes and security issues.

### Differential Testing

V8 uses differential testing between tiers (Ignition vs. Sparkplug vs. Maglev vs. TurboFan) to find correctness bugs that could have security implications.

## Stack Guard

V8's stack guard mechanism (`src/execution/stack-guard.h`) protects against stack overflow and provides interrupt handling:
- Stack limit checks in function prologues prevent stack overflow
- The stack guard also serves as an interrupt mechanism for GC, debugging, and tier-up
- JavaScript stack frames are bounded, preventing unbounded recursion from consuming process memory

## Key Files

| File | Purpose |
|------|---------|
| `src/sandbox/sandbox.h/.cc` | Sandbox address space management |
| `src/sandbox/external-pointer-table.h/.cc` | External pointer indirection table |
| `src/sandbox/code-pointer-table.h/.cc` | Code pointer indirection with write-protection |
| `src/sandbox/js-dispatch-table.h/.cc` | JS function dispatch table |
| `src/sandbox/trusted-pointer-table.h/.cc` | Trusted object pointer table |
| `src/sandbox/cppheap-pointer-table.h/.cc` | C++ heap pointer table |
| `src/sandbox/indirect-pointer-tag.h` | Trusted pointer type tags |
| `src/sandbox/check.h` | SBXCHECK security assertion macros |
| `src/sandbox/hardware-support.h/.cc` | Hardware-assisted sandbox enforcement |
| `src/sandbox/bounded-size.h` | Size fields with maximum bounds |
| `src/sandbox/sandboxed-pointer.h` | Sandbox-relative pointer type |
| `src/sandbox/bytecode-verifier.h/.cc` | Bytecode integrity verification |
| `src/sandbox/code-entrypoint-tag.h` | Code entrypoint type tags |
| `src/sandbox/code-sandboxing-mode.h` | Code protection modes |
| `src/sandbox/external-entity-table.h` | Base class for all indirection tables |
| `src/sandbox/testing.h/.cc` | Sandbox testing utilities |
