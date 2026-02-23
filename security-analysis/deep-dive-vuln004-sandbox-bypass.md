# VULN-004: V8 Sandbox Bypass Vectors Deep Dive

## Summary

The V8 Sandbox is a defense-in-depth mechanism that isolates V8's heap in a
large virtual address region (~1TB) and uses pointer indirection tables to
prevent in-sandbox memory corruption from escaping to out-of-sandbox memory.
This deep dive analyzes the sandbox architecture and identifies potential
bypass vectors.

## Severity: MEDIUM (Defense-in-Depth Weakness)

The sandbox is a second line of defense. It requires an initial vulnerability
(like VULN-001) to achieve arbitrary R/W inside the sandbox before any bypass
can be attempted.

## Sandbox Architecture

### Virtual Address Layout

```
[Guard Region 32GB] [Sandbox ~1TB] [Guard Region 32GB]
                    ^              ^
                    base           base + size

Inside the sandbox:
  [Pointer Compression Cage 4GB]  ← HeapObject pointers (compressed)
  [ArrayBuffer backing stores]    ← Raw data buffers
  [External pointer table data]   ← Referenced via EPT indices
  [Other sandbox-internal memory]
```

### Pointer Tables (Outside Sandbox)

Five indirection tables protect out-of-sandbox pointers:

| Table | Abbrev | What It Protects | Type Checked | Write Protected |
|-------|--------|------------------|--------------|-----------------|
| ExternalPointerTable | EPT | Pointers to external (C++) objects | Yes (tag-based) | No |
| CodePointerTable | CPT | Code object pointers + entrypoints | Yes (tag-based) | Yes |
| TrustedPointerTable | TPT | Pointers to trusted objects | Yes (type-based) | No |
| JSDispatchTable | JDT | JS function dispatch entries | N/A | Yes |
| CppHeapPointerTable | CHPT | Pointers to CppHeap-managed objects | Yes (tag-based) | No |

### SBXCHECK Macro (`sandbox/check.h`)

```cpp
#define SBXCHECK(condition)                                               \
  do {                                                                    \
    std::optional<v8::internal::DisallowSandboxAccess> no_sandbox_access; \
    if (!std::is_constant_evaluated()) {                                  \
      no_sandbox_access.emplace("No sandbox access during SBXCHECK");     \
    }                                                                     \
    CHECK(condition);                                                     \
  } while (false)
```

Key design: `DisallowSandboxAccess` prevents reading sandbox memory during
the check itself, mitigating TOCTOU attacks. The value to check must be
loaded BEFORE the SBXCHECK.

## Bypass Vector Analysis

### Vector 1: Partially-Reserved Sandbox Fallback

**Location**: `sandbox.h:68-72`

```cpp
static constexpr bool kFallbackToPartiallyReservedSandboxAllowed = true;
```

When the system cannot reserve the full ~1TB for the sandbox, it falls back
to a "partially-reserved" sandbox:

```cpp
// sandbox.h:106-113
// A partially-reserved sandbox is backed by a virtual address space
// reservation that is smaller than its size. It also does not have guard
// regions surrounding it. A partially-reserved sandbox is usually created if
// not enough virtual address space could be reserved for the sandbox during
// initialization. In such a configuration, unrelated memory mappings may end
// up inside the sandbox, which affects its security properties.
bool is_partially_reserved() const { return reservation_size_ < size_; }
```

**Impact**: In a partially-reserved sandbox:
- No guard regions → sandbox pointers can access adjacent memory
- Unrelated memory mappings inside the sandbox → corruption of non-V8 data
- Sandboxed pointers (40-bit offsets) can reach out-of-sandbox memory

**Exploitability**: An attacker cannot control whether the sandbox is
partially reserved (it depends on the system's virtual memory availability).
However, on memory-constrained systems (containers, embedded, 32-bit), this
fallback is more likely.

### Vector 2: External Pointer Table (EPT) Entry Swapping

**Location**: `external-pointer-table.h:39-68`

```cpp
struct ExternalPointerTableEntry {
  // Each entry contains a pointer-sized word with:
  //   [external_pointer | type_tag | marking_bit]
  inline Address GetExternalPointer(ExternalPointerTagRange tag_range) const;
  inline void SetExternalPointer(Address value, ExternalPointerTag tag);
};
```

The EPT uses type tags to ensure type safety. When loading an external pointer,
the tag in the entry is checked against the expected tag:

```
entry_value = table[index]
pointer = entry_value XOR tag  // If tag mismatch → garbled pointer
```

**Attack**: An attacker with arbitrary R/W inside the sandbox can:
1. Read the EPT index from one object (e.g., an ArrayBuffer's backing store)
2. Write this index into another object of a different type
3. Both objects now point to the same EPT entry
4. If both expect the SAME tag, the pointer is valid for both

From the GLOSSARY.md (line 106-107):
> "an attacker can swap these pointers as long as the type of the referenced
> object is the same"

This is a **designed limitation**, not a bug. But it can be used to:
- Redirect one ArrayBuffer's backing store to another's
- Create overlapping views of memory
- Bypass per-object isolation within the same type class

### Vector 3: Code Pointer Table (CPT) Entrypoint Redirection

**Location**: `code-pointer-table.h:29-57`

```cpp
struct CodePointerTableEntry {
  static constexpr bool IsWriteProtected = true;

  inline void MakeCodePointerEntry(Address code, Address entrypoint,
                                   CodeEntrypointTag tag, bool mark_as_alive);
  inline Address GetEntrypoint(CodeEntrypointTag tag) const;
  inline void SetEntrypoint(Address value, CodeEntrypointTag tag);
};
```

The CPT is **write-protected** (`IsWriteProtected = true`), meaning direct
corruption of CPT entries is prevented by hardware memory protection.

**Attack**: With write protection bypassed (e.g., by corrupting page table
mappings or disabling MPK):
1. Modify a CPT entry's entrypoint to point to arbitrary code
2. Next call through this entry → arbitrary code execution

**Mitigating factor**: Write protection makes this significantly harder. An
attacker would need to bypass the write protection first, which requires
escalation beyond the sandbox.

### Vector 4: JS Dispatch Table (JDT) Entry Manipulation

**Location**: `js-dispatch-table.h:31-46`

```cpp
struct JSDispatchEntry {
  static constexpr bool IsWriteProtected = true;

  inline void MakeJSDispatchEntry(Address object, Address entrypoint,
                                  uint16_t parameter_count, bool mark_as_alive);
  inline Address GetEntrypoint() const;
  inline Address GetCodePointer() const;
  inline void SetCodeAndEntrypointPointer(Address new_object,
                                          Address new_entrypoint);
};
```

The JDT is also write-protected. Each entry contains:
- A code object pointer
- An entrypoint (raw code address)
- A parameter count

**Attack**: If write protection is bypassed:
1. Modify entrypoint in a JDT entry → redirect function calls
2. Change parameter count → stack mismatch → potential stack corruption

### Vector 5: SBXCHECK TOCTOU Race Conditions

**Location**: `sandbox/check.h:26-31`

```
// It's unsafe to access sandbox memory during a SBXCHECK since such an access
// will be inherently racy as we need to assume an attacker can modify the value
// inside the sandbox right before and after the check.
```

V8 addresses this with `DisallowSandboxAccess` in debug builds, but:
1. This is a **debug-only** mechanism
2. Code that reads a value from the sandbox, checks it, and then reads it
   again (instead of using the checked copy) is vulnerable to TOCTOU
3. The pattern must be enforced by code review, not by the type system

**Attack pattern**:
```
Thread 1 (V8):                    Thread 2 (attacker):
  value = sandbox_memory[idx]
  SBXCHECK(value < limit)
                                    sandbox_memory[idx] = bad_value
  use(sandbox_memory[idx])          // Uses bad_value, not checked value!
```

This requires multi-threading, which limits applicability (most V8 code runs
single-threaded per isolate). But SharedArrayBuffer and worker threads create
opportunities.

### Vector 6: Smi/HeapObject Confusion Mitigation Gap

**Location**: `sandbox.h:115-125`

```cpp
// During initialization, the sandbox will also attempt to create an
// inaccessible mapping in the first four GB of the address space. This is
// useful to mitigate Smi<->HeapObject confusion issues.
bool smi_address_range_is_inaccessible() const {
  return first_four_gb_of_address_space_are_reserved_;
}
```

The sandbox tries to make the first 4GB inaccessible so that treating a Smi
(small integer with low bit 0) as a HeapObject pointer (with low bit 1) will
crash rather than access valid memory. However:

1. This reservation can fail (`first_four_gb_of_address_space_are_reserved_`
   can be false)
2. The reservation only covers 4GB, but Smis on 64-bit can represent values
   well beyond this range
3. Libraries loaded before V8 initialization may already occupy parts of the
   first 4GB

### Vector 7: Hardware Sandbox Support Limitations

**Location**: `hardware-support.h:16-80`

The hardware sandbox support uses Intel Memory Protection Keys (MPK/PKU) to
enforce write protection on pointer tables. Key limitations:

```cpp
// hardware-support.h:72-79
// Any use of this function should be considered to be a temporary exception
// and be accompanied with a comment describing how and when to get rid of it.
// Eventually, for hardware sandbox support to become robust, there should be
// no remaining uses of this function.
static void RegisterUnsafeSandboxExtensionMemory(Address addr, size_t size);
```

`RegisterUnsafeSandboxExtensionMemory` explicitly creates exceptions to the
hardware isolation. Any such extension is a potential bypass vector.

Additionally:
- MPK is not available on all platforms (ARM, older x86)
- MPK only protects 16 "keys", limiting granularity
- MPK can be bypassed via `wrpkru` instruction if code execution is achieved

### Vector 8: Sandbox Testing API

**Location**: `testing.h:80-116`

```cpp
#ifdef V8_ENABLE_MEMORY_CORRUPTION_API
  V8_EXPORT_PRIVATE static void InstallMemoryCorruptionApi(Isolate* isolate);
```

When compiled with `V8_ENABLE_MEMORY_CORRUPTION_API`, V8 exposes JavaScript
functions that allow:
- Arbitrary read/write within the sandbox
- Registration of "safe" memory regions
- Crash filtering

This is intended for testing only, but:
1. If this flag is accidentally enabled in production → full sandbox bypass
2. The crash filter can mask real sandbox violations during testing

## Exploitation Chain (Post-Initial-Bug)

After achieving arbitrary R/W inside the sandbox (e.g., via VULN-001 → OOB):

```
Step 1: Locate pointer tables in sandbox memory
  - EPT, CPT, TPT indices are stored in in-sandbox objects
  - Scan for recognizable patterns (tag bits, entry structures)

Step 2: Perform EPT entry swapping
  - Find two objects with same ExternalPointerTag
  - Swap their EPT indices
  - Object A now accesses Object B's external pointer

Step 3: Create overlapping ArrayBuffer views
  - Use EPT swap to make two ArrayBuffers share a backing store
  - One view for reading, one for writing at different offsets
  - Effective arbitrary R/W within sandbox (redundant, but useful
    for precision)

Step 4: Target write-protected tables
  - CPT/JDT are write-protected via MPK
  - If MPK is available and active: need to bypass MPK first
  - If MPK is not available: direct corruption of entries possible
  - Without MPK: modify CPT entry → redirect code execution

Step 5: Achieve out-of-sandbox access
  - Corrupted CPT entry → execute attacker-controlled code
  - Or: find an external pointer entry pointing to a useful object
    (e.g., a libc data structure) and redirect it

Step 6: Full compromise
  - With code execution outside sandbox → arbitrary system access
```

## Additional Findings (From Deep Static Analysis)

### Vector 9: Bytecode Verifier Incomplete Blocklist

**Location**: `src/sandbox/bytecode-verifier.cc:254-269`

```cpp
// static
bool BytecodeVerifier::IsAllowedRuntimeFunction(Runtime::FunctionId id) {
  // This is currently purely for fuzzer-cleanliness and so we only add
  // functions to this blocklist if we see them cause crashes during fuzzing.
  switch (id) {
#if V8_ENABLE_WEBASSEMBLY
    case Runtime::kWasmTriggerTierUp:
      return false;
#endif
    default:
      return true;  // ALL OTHER RUNTIME FUNCTIONS ALLOWED
  }
}
```

Only **1 runtime function** (`kWasmTriggerTierUp`) is blocklisted. The comment
explicitly states this is "purely for fuzzer-cleanliness" and not a thorough
security audit. An attacker who can corrupt bytecode can call any of the other
670+ runtime functions, many of which lack SBXCHECK protection (see VULN-006).

Additionally, the lightweight verification (`VerifyLight()` at line 31) only
validates jump targets, not:
- Register access bounds
- Constant pool bounds
- Feedback slot validity

### Vector 10: External Pointer Handle Bounds - Debug Only

**Location**: `src/sandbox/external-pointer-table-inl.h:315-337`

```cpp
// Line 315-318: Format check only
bool ExternalPointerTable::IsValidHandle(ExternalPointerHandle handle) {
  uint32_t index = handle >> kExternalPointerIndexShift;
  return handle == index << kExternalPointerIndexShift;  // Only format!
}

// Line 321-337: Bounds check is DCHECK only
uint32_t ExternalPointerTable::HandleToIndex(ExternalPointerHandle handle) {
  DCHECK(IsValidHandle(handle));
  uint32_t index = handle >> kExternalPointerIndexShift;
  DCHECK_LE(index, kMaxExternalPointers);  // DEBUG ONLY!
  return index;
}
```

In release builds, an attacker can craft a handle with valid format that
points past the end of the external pointer table. The `at(index)` call
at line 179 would then access out-of-bounds memory.

### Vector 11: Pointer Table Memory Ordering Issues

Multiple pointer tables use `std::memory_order_relaxed` for operations that
need stronger guarantees:

**ExternalPointerTable** (`external-pointer-table-inl.h`):
```cpp
// Line 175-179: Get() uses relaxed load
Address ExternalPointerTable::Get(...) const {
  uint32_t index = HandleToIndex(handle);
  DCHECK(index == 0 || at(index).HasExternalPointer(tag_range));  // Relaxed
  return at(index).GetExternalPointer(tag_range);  // Can race with Set()
}
```

**JSDispatchTable** (`js-dispatch-table-inl.h`):
```cpp
// Line 103-114: SBXCHECK races with SetCodeAndEntrypointPointer
void JSDispatchTable::SetCodeAndEntrypointNoWriteBarrier(...) {
  SBXCHECK(IsCompatibleCode(new_code, GetParameterCount(handle)));  // T0
  // Parameter count can be corrupted between T0 and T1 by attacker
  uint32_t index = HandleToIndex(handle);
  at(index).SetCodeAndEntrypointPointer(new_code.ptr(), new_entrypoint);  // T1
}
```

The SBXCHECK at line 105 reads the parameter count, validates it, but the
actual write operation at line 113 re-reads state that could have been
corrupted in between. However, the JSDispatchTable is write-protected
(`IsWriteProtected = true`), so this race requires bypassing write protection
first.

### Vector 12: LSan Entry Size Bypass

**Location**: `src/sandbox/external-pointer-table-inl.h:326-333`

```cpp
// When LSan is active, we use "fat" entries that also store the raw pointer.
// However, this is not secure as an attacker could reference the raw pointer
// instead of the encoded pointer in an entry, thereby bypassing the type
// checks. As such, this mode must only be used in testing environments.
```

If LSan is accidentally enabled in production, the external pointer table
entries contain unencoded raw pointers alongside the tagged pointers,
completely bypassing the type tag protection.

## Additional Findings (From Final Deep Investigation)

### Vector 13: DisallowSandboxAccess is NOP in Release Builds

**Location**: `src/sandbox/hardware-support.h:228-244`

```cpp
class V8_EXPORT_PRIVATE V8_NODISCARD V8_ALLOW_UNUSED DisallowSandboxAccess {
 public:
#if defined(DEBUG) && defined(V8_ENABLE_SANDBOX_HARDWARE_SUPPORT)
  explicit DisallowSandboxAccess(const char* reason);  // Active in debug
  ~DisallowSandboxAccess();
  // ...
#else
  explicit DisallowSandboxAccess(const char* reason) {}  // NOP in release!
#endif
};
```

The `DisallowSandboxAccess` scope used in every `SBXCHECK` (check.h:39) is a
**no-op in release builds** unless BOTH `DEBUG` AND `V8_ENABLE_SANDBOX_HARDWARE_SUPPORT`
are defined. Since production Chrome is built without `DEBUG`, every SBXCHECK
runs with full sandbox access, meaning:
- The checked value can race with sandbox memory modifications
- The TOCTOU prevention mechanism is **non-functional** in production
- SBXCHECK degrades to a plain CHECK in production (no sandbox isolation)

### Vector 14: CodePointerTable Freelist Tag Exploitation

**Location**: `src/sandbox/code-pointer-table-inl.h:62-81`

```cpp
void CodePointerTableEntry::MakeFreelistEntry(uint32_t next_entry_index) {
  Address value = kFreeEntryTag | next_entry_index;
  entrypoint_.store(value, std::memory_order_relaxed);
  code_.store(kNullAddress, std::memory_order_relaxed);
}

bool CodePointerTableEntry::IsFreelistEntry() const {
  auto entrypoint = entrypoint_.load(std::memory_order_relaxed);
  return (entrypoint & kFreeEntryTag) == kFreeEntryTag;
}
```

The CPT distinguishes live entries from freelist entries using `kFreeEntryTag`.
All `GetEntrypoint()`, `GetCodeObject()`, `SetEntrypoint()`, `SetCodeObject()`
use `DCHECK(!IsFreelistEntry())` which is **stripped in release builds**.

**Attack**: If an attacker can write `kFreeEntryTag` into a live CPT entry's
entrypoint:
1. `IsFreelistEntry()` returns true → entry appears free
2. Next allocation may reuse this "freed" entry
3. Original code still references the old handle → uses the reallocated entry
4. **Use-after-free** on the code pointer → arbitrary code execution

The CPT is write-protected, so this requires bypassing MPK first. But on
platforms without MPK (ARM, older x86), this is directly exploitable.

### Vector 15: Bytecode Verifier Missing Operand Validation

**Location**: `src/sandbox/bytecode-verifier.cc:208-219`

```cpp
case interpreter::OperandType::kFlag8:
case interpreter::OperandType::kFlag16:
case interpreter::OperandType::kEmbeddedFeedback:
case interpreter::OperandType::kIntrinsicId:
case interpreter::OperandType::kNativeContextIndex:
case interpreter::OperandType::kUImm:
case interpreter::OperandType::kImm:
case interpreter::OperandType::kFeedbackSlot:
case interpreter::OperandType::kContextSlot:
case interpreter::OperandType::kCoverageSlot:
case interpreter::OperandType::kRegCount:
  break;  // NO VALIDATION FOR ANY OF THESE!
```

The bytecode verifier's `VerifyFull()` performs **no validation** for:
- `kFeedbackSlot` - can corrupt inline caches → type confusion
- `kContextSlot` - can access arbitrary context slots → scope escape
- `kNativeContextIndex` - can reference invalid native context entries
- `kIntrinsicId` - can access undefined intrinsics

This extends Vector 9 (runtime function blocklist). Not only are almost all
runtime functions allowed, but feedback slots and context access are entirely
unchecked, giving a bytecode-corruption attacker access to V8's inline cache
and scope chain.

### Vector 16: OutsideSandbox() Weakness for Partially Reserved

**Location**: `src/sandbox/sandbox.h` (inferred from agent analysis)

The `OutsideSandbox()` function uses `ReservationContains()` for partially
reserved sandboxes:
```cpp
V8_INLINE bool OutsideSandbox(uintptr_t address) {
  Sandbox* sandbox = Sandbox::current();
  return !sandbox->ReservationContains(address);
}
```

For a partially reserved sandbox where `reservation_size_ < size_`:
- `ReservationContains()` checks `[reservation_base_, reservation_base_ + reservation_size_)`
- Gap region addresses `[base_ + reservation_size_, base_ + size_)` are NOT
  in the reservation
- These gap addresses pass `OutsideSandbox()` → treated as trusted
- But they ARE within the sandbox's virtual range
- An attacker can place corrupted pointers in the gap region that pass
  trusted pointer validation

### Vector 17: Freelist Allocation TOCTOU Race (ABA Attack)

**Location**: `src/sandbox/external-entity-table-inl.h:259-288`

```cpp
template <typename Entry, size_t size>
uint32_t ExternalEntityTable<Entry, size>::
    TryAllocateEntryFromFreelist(Space* space) {
  FreelistHead freelist = space->freelist_head_.load(std::memory_order_relaxed);
  // ...
  Entry& freelist_entry = this->at(freelist.next());
  auto maybe_next_freelist_entry = freelist_entry.GetNextFreelistEntryIndex();
  uint32_t next_freelist_entry = maybe_next_freelist_entry.value_or(0);
  FreelistHead new_freelist(next_freelist_entry, freelist.length() - 1);
  bool success = space->freelist_head_.compare_exchange_strong(
      freelist, new_freelist);
  CHECK_IMPLIES(success, maybe_next_freelist_entry.has_value());
}
```

**Attack**: Classic ABA problem on the freelist CAS loop:
1. Thread reads `freelist_entry` at index N, sees next = M
2. Attacker (or concurrent thread) allocates entry N, frees it again
3. Entry N now has a different `next` value but the CAS still succeeds
   because the freelist head hasn't changed
4. Result: freelist corruption → double allocation or use-after-free

The comments at lines 264-272 acknowledge this race window. The CHECK at
line 285 only catches the case where a non-freelist entry was read, but
cannot detect ABA corruption.

**Severity**: HIGH (requires concurrency, but leads to entry corruption)

### Vector 18: Trusted Pointer Re-Publishing Attack

**Location**: `src/sandbox/trusted-pointer-table-inl.h:139-150`

The `TrustedPointerTable` allows entries to be "unpublished" via
`kUnpublishedIndirectPointerTag` (line 58 in trusted-pointer-table.h), making
them temporarily inaccessible. When re-published, the tag can potentially be
changed, altering the perceived type of the referenced object.

```
1. Entry allocated with tag_A → points to ObjectA
2. Entry unpublished (tag = kUnpublishedIndirectPointerTag)
3. Entry re-published with tag_B → same pointer, different type
4. Code reading through tag_B sees ObjectA with wrong type → type confusion
```

**Severity**: MEDIUM (requires ability to manipulate table entries)

### Vector 19: Entry Evacuation/Compaction Race

**Location**: `src/sandbox/external-pointer-table.h:230-235`

During GC compaction, pointer table entries are evacuated (moved) between
table segments. Three evacuation marking modes exist:

```cpp
enum class EvacuateMarkMode {
  kTransferMark,    // Transfer marking bit
  kLeaveUnmarked,   // Don't mark the new entry
  kClearMark,       // Clear the mark on the new entry
};
```

During evacuation:
1. Entry is copied to new location
2. Handle locations in heap objects must be updated atomically
3. **Race window**: Between copy and handle update, two entries reference the
   same external pointer
4. If the old entry is freed before all handles are updated → dangling handle

**Severity**: MEDIUM (requires precise GC timing, but could lead to
use-after-free on pointer table entries)

### Vector 20: Hardware Sandbox Strict vs Non-Strict Mode

**Location**: `src/sandbox/hardware-support.h:46-165`

The hardware sandbox uses three Memory Protection Keys (PKEYs):
- `sandbox_pkey_` — For writable in-sandbox memory
- `out_of_sandbox_pkey_` — For out-of-sandbox memory (opt-in tracking)
- `extension_pkey_` — For out-of-sandbox memory still writable to sandboxed code

**Non-strict mode** (default) does NOT remove write access to all out-of-sandbox
memory — only memory explicitly tagged with `out_of_sandbox_pkey_`. Memory not
tagged with any PKEY remains writable. This means:
- Newly allocated memory outside the sandbox is writable by default
- Only memory explicitly registered with `RegisterOutOfSandboxMemory()` gets protection
- Attack: Target untagged out-of-sandbox memory that was never registered

**Strict mode** removes all write access to anything without `sandbox_pkey_`
or `extension_pkey_`, but has compatibility issues (line 46 comment).

Additionally, kernel signal delivery support check at line 123
(`crbug.com/429173713`) suggests signal handlers may bypass PKU protections.

### Vector 21: JSDispatchTable Compact Encoding Abuse

**Location**: `src/sandbox/js-dispatch-table-inl.h:122-130`

On 64-bit, JSDispatchTable entries encode three fields in a single word:
```
[pointer (bits 17-63)] [marking_bit (bit 16)] [parameter_count (bits 0-15)]
```

If an attacker can write to this word (requires bypassing write protection):
1. Modifying the parameter count (bits 0-15) causes stack mismatch
2. Modifying the marking bit (bit 16) disrupts GC marking
3. Modifying pointer bits (17-63) redirects code execution
4. A single corrupted word affects ALL THREE fields simultaneously

### Vector 22: Memory Corruption API Safe Crash Filtering

**Location**: `src/sandbox/testing.cc:703-1048`

The sandbox testing mode registers "safe" crash regions and filters specific
crash types. When testing mode is active:

| Address Range | Classification | Filter Reason |
|---------------|---------------|---------------|
| Pointer table memory | Safe | Lines 768-776 |
| Non-canonical addresses | Safe | Lines 914-923 |
| Kernel space addresses | Safe | Lines 925-932 |
| Nullptr (first page) | Safe | Lines 935-941 |
| First 4GB | Safe | Lines 945-953 (crbug.com/1470641) |

**Risk**: During security testing with the Memory Corruption API enabled,
crashes in these "safe" regions are silently filtered. An attacker exploiting
a pointer table corruption bug would have their crashes masked as "safe",
preventing detection.

Specific primitives exposed via the Memory Corruption API:
- `Sandbox.getAddressOf(Object)` — Get heap object address
- `Sandbox.getObjectAt(Address)` — Read object from arbitrary address
- `Sandbox.corruptObjectField(obj, offset, value)` — Direct field corruption
- `Sandbox.setFunctionCodeToBuiltin(func, id)` — Change function code
- `new Sandbox.MemoryView(offset, size)` — Raw sandbox memory access

## Recommended Mitigations

1. **Remove `kFallbackToPartiallyReservedSandboxAllowed`**: Crash instead of
   falling back to partial reservation. Security should not degrade silently.

2. **Enforce SBXCHECK TOCTOU prevention at compile time**: Use a type system
   (e.g., a wrapper type for sandbox-loaded values) to prevent reading from
   sandbox memory after a check.

3. **Audit `RegisterUnsafeSandboxExtensionMemory` usage**: Each call site is
   a potential bypass. Track and reduce these.

4. **Add EPT entry isolation**: Prevent swapping entries between objects by
   adding per-object salt to the tag computation.

5. **Strengthen Smi protection**: Reserve larger address range or use guard
   pages more aggressively.

## Key File References

| File | Line | Component |
|------|------|-----------|
| `sandbox/sandbox.h` | 72 | `kFallbackToPartiallyReservedSandboxAllowed` |
| `sandbox/sandbox.h` | 113 | `is_partially_reserved()` |
| `sandbox/sandbox.h` | 119-125 | Smi address range protection |
| `sandbox/sandbox.cc` | 294-355 | `InitializeAsPartiallyReservedSandbox()` |
| `sandbox/check.h` | 26-44 | SBXCHECK with DisallowSandboxAccess |
| `sandbox/external-pointer-table.h` | 39-68 | EPT entry structure |
| `sandbox/external-pointer-table-inl.h` | 175-180 | EPT Get() relaxed memory ordering |
| `sandbox/external-pointer-table-inl.h` | 315-337 | Handle validation (format only + debug bounds) |
| `sandbox/code-pointer-table.h` | 29-57 | CPT entry structure (write-protected) |
| `sandbox/js-dispatch-table.h` | 31-46 | JDT entry structure (write-protected) |
| `sandbox/js-dispatch-table-inl.h` | 103-114 | SBXCHECK + parameter count TOCTOU |
| `sandbox/bytecode-verifier.cc` | 254-269 | `IsAllowedRuntimeFunction()` (1 blocklisted) |
| `sandbox/bytecode-verifier.cc` | 31-82 | `VerifyLight()` (jump targets only) |
| `sandbox/hardware-support.h` | 16-80 | MPK-based hardware protection |
| `sandbox/testing.h` | 80-86 | Memory Corruption API |
| `sandbox/hardware-support.h` | 228-244 | **DisallowSandboxAccess NOP in release** |
| `sandbox/code-pointer-table-inl.h` | 62-81 | **Freelist tag exploitation (CPT)** |
| `sandbox/bytecode-verifier.cc` | 208-219 | **Missing operand validation (11 types)** |
| `sandbox/GLOSSARY.md` | 106-107 | EPT swap attack documentation |
| `sandbox/external-entity-table-inl.h` | 259-288 | **Freelist CAS ABA race (Vector 17)** |
| `sandbox/trusted-pointer-table-inl.h` | 139-150 | **Entry re-publishing attack (Vector 18)** |
| `sandbox/trusted-pointer-table.h` | 58 | `kUnpublishedIndirectPointerTag` |
| `sandbox/external-pointer-table.h` | 230-235 | **Entry evacuation/compaction race (Vector 19)** |
| `sandbox/hardware-support.h` | 46 | **Strict vs non-strict mode (Vector 20)** |
| `sandbox/hardware-support.h` | 149-165 | Three PKEY allocation |
| `sandbox/js-dispatch-table-inl.h` | 122-130 | **Compact encoding abuse (Vector 21)** |
| `sandbox/testing.cc` | 703-1048 | **Safe crash filtering (Vector 22)** |
| `sandbox/testing.cc` | 85-125 | `Sandbox.MemoryView` raw access |
| `sandbox/testing.cc` | 510-562 | `corruptObjectField()` |
