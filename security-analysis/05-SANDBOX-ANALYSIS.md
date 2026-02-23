# V8 Sandbox Security Analysis

## 1. Overview

The V8 Sandbox is a hardware-enforced isolation boundary designed to contain
memory corruption inside V8's heap. It assumes an attacker has achieved
**arbitrary read/write inside the sandbox** (e.g., via a JIT type confusion
bug) and attempts to prevent them from corrupting memory outside the sandbox.

### Threat Model (from `src/sandbox/GLOSSARY.md`)

> It is assumed that an attacker can arbitrarily read and write memory inside
> the sandbox address space due to a typical security bug in V8.

This means the sandbox is a **second line of defense** - it doesn't prevent
the initial bug but contains its damage.

## 2. Sandbox Address Space (`src/sandbox/sandbox.h`)

```
Layout:
┌──────────┬────────────────┬──────────────────────────────┬──────────┐
│  Guard   │   4GB Heap     │  ArrayBuffer backing stores, │  Guard   │
│  Region  │   Region       │  WASM memories, etc.         │  Region  │
│  (32GB)  │  (compressed   │  (up to ~1TB - 4GB)          │  (32GB)  │
│          │   pointers)    │                              │          │
└──────────┴────────────────┴──────────────────────────────┴──────────┘
           ^                                                ^
           base                                             end
           <------------- size (ideally 1TB) --------------->
<----------------- reservation_size (includes guards) ------>
```

### Key Properties

1. **Size**: 1TB virtual address space (configurable)
2. **Guard Regions**: 32GB guard regions on each side to catch OOB accesses
3. **Compressed Pointers**: 4GB heap region uses 32-bit compressed pointers
4. **Smi Confusion Mitigation**: First 4GB of process address space reserved
   as inaccessible (`sandbox.h:119-125`) to catch Smi-treated-as-pointer bugs

### Partially Reserved Sandbox (`sandbox.h:72,109-113`)

```cpp
static constexpr bool kFallbackToPartiallyReservedSandboxAllowed = true;

bool is_partially_reserved() const { return reservation_size_ < size_; }
```

**WEAKNESS**: When not enough virtual memory is available, the sandbox falls
back to a "partially reserved" mode where:
- Only a portion of the 1TB is actually reserved
- Unrelated memory mappings can end up inside the sandbox address space
- Guard regions are not present
- The sandbox's security properties are significantly weakened

**Attack Vector**: On memory-constrained systems, force the sandbox into
partially-reserved mode, then map attacker-controlled data inside the
sandbox address space.

## 3. Pointer Table System

The sandbox replaces raw pointers with indices into pointer tables. This
prevents an attacker from directly controlling where pointers point.

### External Pointer Table (`src/sandbox/external-pointer-table.h`)

**Purpose**: Indirect access to objects outside the sandbox (C++ objects,
native resources, etc.)

**Structure**:
```
ExternalPointerTableEntry:
  ┌─────────────────────────────────────────────┐
  │  [marking bit] [type tag] [pointer value]    │
  │  (upper bits)  (N bits)   (lower bits)       │
  └─────────────────────────────────────────────┘
```

**Security Properties**:
- Type tag ensures the pointer is used for its intended purpose
- Tag mismatch produces an invalid pointer that crashes on dereference
- BUT: An attacker can **swap entries with the same type tag** (line 106-107
  in GLOSSARY.md: "an attacker can swap these pointers as long as the type
  of the referenced object is the same")

**Attack Vector**: If two objects of the same type have external pointers,
swap their EPT indices to redirect operations to a different object.

### Code Pointer Table (`src/sandbox/code-pointer-table.h`)

**Purpose**: Indirect access to compiled code (JIT output, builtins).
Contains both the code object pointer and the code entrypoint.

**Security Properties**:
- Code pointer swaps can only redirect to other Code objects
- The entrypoint must match the code object
- Type tags enforce Code vs non-Code distinction

**Attack Vector**: Swap code pointers to redirect function execution to a
different compiled function. This is constrained by type tags but could
be useful for ROP-like gadget chaining within V8's code space.

### Trusted Pointer Table (`src/sandbox/trusted-pointer-table.h`)

**Purpose**: Access to TrustedObjects in trusted space (outside sandbox).

**Security Properties**:
- Points only to valid TrustedObjects
- Type-checked on access
- Subject to same swap attacks as External Pointers

### JS Dispatch Table (`src/sandbox/js-dispatch-table.h`)

**Purpose**: Function dispatch indirection. Maps function IDs to
code entrypoints.

**Security Properties**:
- Controls which code runs when a JS function is called
- Corrupting dispatch table entries = hijacking function execution
- Must be outside the sandbox to be secure

## 4. SBXCHECK Macro (`src/sandbox/check.h`)

```cpp
#define SBXCHECK(condition)
  do {
    std::optional<v8::internal::DisallowSandboxAccess> no_sandbox_access;
    if (!std::is_constant_evaluated()) {
      no_sandbox_access.emplace("No sandbox access during SBXCHECK");
    }
    CHECK(condition);
  } while (false)
```

**Purpose**: Security-critical invariant checks that prevent sandbox bypass.
Unlike DCHECK (debug-only), SBXCHECK is enabled in **all builds** including
release.

**Key insight** (`check.h:26-31`):
> It's unsafe to access sandbox memory during a SBXCHECK since such an access
> will be inherently racy as we need to assume an attacker can modify the
> value inside the sandbox right before and after the check.

This means SBXCHECK has a TOCTOU (time-of-check-time-of-use) limitation.
If the check reads a value from sandbox memory, the attacker could modify
it between the check and the subsequent use. V8 addresses this with
`DisallowSandboxAccess` guards in debug builds.

**Audit target**: Find SBXCHECKs that read sandbox memory and don't copy
the value to a local variable first.

## 5. Bytecode Verifier (`src/sandbox/bytecode-verifier.cc`)

The bytecode verifier validates Ignition bytecode to ensure it satisfies
certain security properties. This is important because bytecode is stored
inside the sandbox and could be corrupted by an attacker.

Key file: `src/sandbox/bytecode-verifier.cc` (12KB)

**Purpose**: After an attacker achieves sandbox memory corruption, they might
try to modify bytecode to execute arbitrary operations. The verifier checks
that bytecode:
- References valid constant pool entries
- Has valid jump targets
- Uses valid register indices
- Doesn't access out-of-bounds handler table entries

## 6. Sandbox Testing Infrastructure (`src/sandbox/testing.h`)

### Memory Corruption API

```cpp
#ifdef V8_ENABLE_MEMORY_CORRUPTION_API
V8_EXPORT_PRIVATE static void InstallMemoryCorruptionApi(Isolate* isolate);
```

When compiled with `V8_ENABLE_MEMORY_CORRUPTION_API`, V8 provides a JavaScript
API for simulating memory corruption. This is used for:
- Testing sandbox security
- Developing fuzzers
- Writing regression tests for sandbox bypasses

**Contains**:
- `Sandbox.getAddressOf(obj)` - Get the in-sandbox address of an object
- `Sandbox.getFieldOffset(type, field)` - Get field offsets
- `Sandbox.read/write*(addr)` - Read/write sandbox memory
- Access to InstanceType mappings

### Sandbox Testing Modes

```cpp
enum class Mode {
  kDisabled,     // Normal operation
  kForTesting,   // Crash filter active, exit 0 on safe crash
  kForFuzzing,   // Crash filter active, non-zero exit on safe crash
};
```

### Crash Filter

The crash filter (`testing.h:46-54`) is a signal handler that distinguishes
between:
- "Safe" crashes: memory access violations inside the sandbox (expected)
- "Unsafe" crashes: violations outside the sandbox (= sandbox bypass)

Only unsafe crashes are reported as bugs.

## 7. Sandbox Bypass Vectors

### Vector 1: Pointer Table Entry Corruption

```
1. Achieve arbitrary R/W inside sandbox
2. Locate External Pointer Table entries in sandbox metadata
3. Corrupt an entry: change the pointer value (staying within same type tag)
4. Redirect the external pointer to a controlled buffer outside sandbox
5. Next use of the external pointer reads/writes attacker-controlled memory
```

**Limitation**: Type tags restrict what pointers can be swapped to. An
attacker can only redirect to another object of the same type.

### Vector 2: Code Pointer Redirection

```
1. Corrupt Code Pointer Table entry
2. Redirect to a ROP gadget or different Code object
3. When the function is called, attacker's code executes
```

**Limitation**: Only within V8's code space. Need to chain gadgets.

### Vector 3: JS Dispatch Table Hijacking

```
1. Corrupt JS Dispatch Table entry for a function
2. Point it to a different function's code
3. Call the original function → different code executes
4. If the different code has different parameter types, type confusion
```

### Vector 4: Trusted Object Field Corruption

```
1. Find a TrustedObject that writes data based on sandbox-controlled input
2. Corrupt the sandbox-side data that influences the write
3. TrustedObject writes corrupted data to trusted space
4. Use corrupted trusted data for further exploitation
```

**Key concern from GLOSSARY.md line 33**:
> one must be particularly careful when writing to [TrustedObjects] to avoid
> any memory corruption in trusted space.

### Vector 5: Partially-Reserved Sandbox Exploitation

```
1. Force sandbox into partially-reserved mode (memory pressure)
2. Map attacker-controlled memory into the sandbox address space
3. Attacker can now read/write their own mappings from sandbox code
4. If these mappings overlap with sensitive data, direct corruption
```

### Vector 6: TOCTOU on SBXCHECK

```
1. SBXCHECK reads a value from sandbox memory and validates it
2. If the value isn't copied to a stack variable before use...
3. Attacker (from another thread) modifies the value after check
4. Code uses the now-corrupted value despite the check passing
```

## 8. Audit Checklist for Sandbox Security

- [ ] All raw pointers in sandbox objects are replaced with sandboxed types
- [ ] All SBXCHECKs copy sandbox values before checking
- [ ] TrustedObject write paths validate all sandbox-sourced inputs
- [ ] No code paths exist that store External Pointer Table indices in a way
      that allows type confusion (different EPT types in the same field)
- [ ] Code Pointer Table entries correctly pair code objects with entrypoints
- [ ] Partially-reserved sandbox mode handles unrelated mappings safely
- [ ] Bytecode verifier catches all forms of bytecode corruption
- [ ] No raw pointers leak from trusted space back into the sandbox
