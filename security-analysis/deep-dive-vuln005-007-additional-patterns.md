# VULN-005/007: FatalNoSecurityImpact, Write Barriers, RegExp & Additional Patterns

## Overview

This document covers several additional vulnerability patterns discovered
during deep analysis of the V8 codebase:

1. **VULN-005**: `FatalNoSecurityImpact` crash classification concerns
2. **VULN-007**: RegExp stack overflow patterns
3. **Additional**: `StoreNoWriteBarrier` in CodeStubAssembler
4. **Additional**: UnsafeStoreNoWriteBarrier patterns

---

## VULN-005: FatalNoSecurityImpact Crash Classification

### Severity: MEDIUM (Security Process Gap)

### Location
- **Definition**: `src/base/logging.h:119` and `src/base/logging.cc:101-119`
- **Macro**: `CHECK_NO_SECURITY_IMPACT` at `src/base/logging.h:146-151`

### Mechanism

```cpp
// logging.cc:101-119
void FatalNoSecurityImpact(const char* format, ...) {
  OS::PrintError("\n\n#\n# Fatal error with no security impact:\n# ");
  // ... format and print the error ...
  if (FatalErrorsWithNoSecurityImpactShouldExit()) {
    OS::ExitProcess(-1);   // Clean exit (not a crash signal)
  } else {
    OS::Abort();           // Normal abort
  }
}
```

From the header comment:
```cpp
// logging.h:111-118
// A variant of Fatal that makes it clear that the failure does not have any
// security impact. This is useful for automatic vulnerability discover systems
// (e.g. fuzzers) to ignore or discard such crashes.
//
// USE WITH CARE! Using this function means that fuzzers will *not* report
// situations in which the function is reached.
```

### Call Sites Analysis

Found **20+ call sites** of `FatalNoSecurityImpact` in the codebase:

#### Factory Allocation (`src/heap/factory-base.cc`)
```
Line 240:  FatalNoSecurityImpact("Invalid FixedArray size %d", length);
Line 337:  FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 1191: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 1357: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 1368: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 1418: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
```

#### FixedArray Access (`src/objects/fixed-array-inl.h`)
```
Line 314: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 337: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 366: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 388: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 567: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 584: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 726: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 742: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 842: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
Line 882: FatalNoSecurityImpact("Fatal JavaScript invalid size error %d", ...);
```

### Security Concern

The core issue: **"invalid size" errors in FixedArray operations** are
classified as having "no security impact". But:

1. **FixedArray sizes directly control memory allocation**: An incorrect size
   can lead to buffer under/overflow during element access
2. **These checks are reachable from user JavaScript**: Creating large arrays,
   using Array methods with crafted arguments
3. **Fuzzers are told to IGNORE these crashes**: Any fuzzer using the crash
   classification will skip these entirely

### Risk Assessment

If an attacker can manipulate the size parameter reaching `factory-base.cc:240`
(e.g., through type confusion making the compiler pass an incorrect size), the
`FatalNoSecurityImpact` call prevents the fuzzer from ever discovering it:

```
Attacker input → type confusion → incorrect size → FatalNoSecurityImpact
                                                    ↑
                                           Fuzzer stops here (ignores crash)
                                           Real vulnerability goes unfound
```

However, the `FatalNoSecurityImpact` is a CHECK - it terminates the process.
It doesn't continue execution with the bad size. So the direct security
impact is denial-of-service at worst. The real concern is the fuzzer coverage
gap.

### Verdict

**Direct impact: LOW** (crash, not corruption)
**Indirect impact: MEDIUM** (masks reachability of these code paths from fuzzers)

---

## VULN-007: RegExp Stack Overflow Patterns

### Severity: LOW (DoS, Limited Code Execution Risk)

### Evidence

Recent commit `b07860d0` fixed a stack overflow in the regexp AST visitor,
indicating this class of bug recurs in the regexp subsystem. The key concern
is recursive visitors over deeply nested regexp ASTs.

### Key Files

```
src/regexp/regexp-compiler.cc   - RegExp compilation
src/regexp/regexp-interpreter.cc - RegExp interpretation
src/regexp/regexp-parser.cc     - RegExp parsing
src/regexp/regexp-ast.h         - AST node definitions (recursive structure)
```

### Attack Pattern

```javascript
// Create a deeply nested regular expression
let pattern = "";
for (let i = 0; i < 100000; i++) {
  pattern = "(" + pattern + ")?";
}
// Compile the regex - triggers recursive AST traversal
new RegExp(pattern);
```

The AST visitor recursion depth is bounded by the nesting depth of the regex.
V8 has stack depth checks, but:
1. Check granularity matters - if checks are per-N-levels instead of per-level
2. Platform stack sizes vary
3. Stack guard checks may not be present in all visitor paths

### Impact

- **Primary**: Denial of service (process crash via stack overflow)
- **Theoretical**: If stack overflow corrupts return addresses or local
  variables before being caught, could enable control flow hijacking. In
  practice, modern systems have guard pages that make this extremely difficult.

---

## Additional Pattern: StoreNoWriteBarrier in CodeStubAssembler

### Severity: HIGH (If Incorrect) / AUDIT TARGET

### Quantification

```
StoreNoWriteBarrier occurrences:
  src/codegen/code-stub-assembler.cc:    45 occurrences
  src/codegen/code-stub-assembler.h:      2 occurrences
  src/compiler/code-assembler.cc:          7 occurrences
  src/compiler/code-assembler.h:           5 occurrences
  Total:                                 ~59 occurrences
```

### What It Does

`StoreNoWriteBarrier` writes a pointer to a HeapObject field WITHOUT recording
the reference in the remembered set or notifying the concurrent marker. This
is ONLY safe when:

1. **Object initialization**: The object hasn't been published yet, so no GC
   can see it
2. **Non-pointer values**: Writing Smis, raw integers, or non-heap values
3. **Same-space references**: Both source and target are in old space and
   marking is not active
4. **Trusted space**: Objects in trusted space don't need barriers for
   within-trusted-space references

### Security-Critical Examples

From `code-stub-assembler.cc`:

```cpp
// Line 1576: Updating allocation top pointer (non-heap, safe)
StoreNoWriteBarrier(MachineType::PointerRepresentation(), top_address, ...);

// Line 1586: Initializing newly allocated object (not yet published, safe)
StoreNoWriteBarrier(MachineRepresentation::kTagged, top, ...);

// Line 3649: Writing to Context slot
StoreNoWriteBarrier(MachineRepresentation::kTagged, context, ...);
// ← NEEDS AUDIT: Is this always during initialization?

// Line 4138: Conditional no-barrier store
StoreNoWriteBarrier(MachineRepresentation::kTagged, object, offset, value);
// ← NEEDS AUDIT: What ensures this is safe?

// Line 4207: Writing to feedback vector
StoreNoWriteBarrier(MachineRepresentation::kTagged, feedback_vector, offset, ...);
// ← NEEDS AUDIT: Feedback vectors live in old space, written value might be young
```

### The UnsafeStoreNoWriteBarrier Variant

Additionally, there is `UnsafeStoreNoWriteBarrier` which is even more
dangerous:

```cpp
// Line 4140: UnsafeStoreNoWriteBarrier for tagged stores
UnsafeStoreNoWriteBarrier(MachineRepresentation::kTagged, object, offset, value);

// Line 4210: UnsafeStoreNoWriteBarrier for feedback vector
UnsafeStoreNoWriteBarrier(MachineRepresentation::kTagged, feedback_vector, offset, ...);

// Line 5197: UnsafeStoreNoWriteBarrier in array operations
UnsafeStoreNoWriteBarrier(MachineRepresentation::kTagged, current, ...);
```

The "Unsafe" prefix suggests these are known to be risky and should be
reviewed with extra care.

### Exploitation Pattern

If ANY of these `StoreNoWriteBarrier` calls incorrectly skip the barrier:

```
1. Object A (old space) gets reference to Object B (young space)
2. No write barrier → B not in remembered set
3. Scavenge triggered → B is moved to new address B'
4. A still points to old address B (now freed memory)
5. New object C allocated at B's old address
6. A accesses C through dangling pointer → type confusion/UAF
```

### Audit Recommendation

Each `StoreNoWriteBarrier` call should be verified for:
- [ ] Is the stored value guaranteed to be a non-pointer (Smi, raw)?
- [ ] Is the target object guaranteed to be unpublished (during init)?
- [ ] Is there a comment explaining why the barrier is safe to skip?
- [ ] Could any code path reach this with unexpected object lifetimes?

Priority audit targets (tagged pointer stores with no obvious safety comment):
1. `code-stub-assembler.cc:3649` - Context slot write
2. `code-stub-assembler.cc:4138` - Conditional tagged store
3. `code-stub-assembler.cc:4207-4210` - Feedback vector writes
4. `code-stub-assembler.cc:5197` - Array operation write

---

## Additional Pattern: Concurrent Marking Write Barrier Races

### Mechanism

V8's concurrent marking runs on background threads while the main thread
mutates the heap. The marking barrier ensures the marker sees all new
references:

```
Main thread:                      Marking thread:
  obj.field = new_value
  WriteBarrier(obj, field)        Scanning obj → sees new_value ✓
```

If the WriteBarrier is missing:

```
Main thread:                      Marking thread:
  obj.field = new_value           Already scanned obj → doesn't see new_value ✗
  (no barrier)                    new_value is not marked → freed!
```

### Key Files

```
src/heap/marking-barrier.cc       - Marking barrier implementation
src/heap/concurrent-marking.cc    - Background marking thread
src/heap/heap-write-barrier.cc    - Main write barrier dispatcher
src/heap/heap-write-barrier-inl.h - Inline fast paths
```

### Known Safe Patterns

V8 documents specific cases where barriers can be skipped:
- Object initialization before publishing
- Storing Smis (non-pointer values)
- Stores in read-only space (never collected)
- Stores between trusted objects (separate barrier protocol)

---

## Summary Table

| Pattern | Severity | Status | Action |
|---------|----------|--------|--------|
| FatalNoSecurityImpact | MEDIUM | Confirmed | Audit all 20+ call sites |
| RegExp stack overflow | LOW | Known pattern | Stack guards in all visitors |
| StoreNoWriteBarrier (CSA) | HIGH (if wrong) | Needs audit | 45 occurrences in CSA |
| UnsafeStoreNoWriteBarrier | HIGH (if wrong) | Needs audit | 3+ explicit unsafe stores |
| Concurrent marking races | HIGH (if wrong) | Systemic risk | Barrier completeness audit |
