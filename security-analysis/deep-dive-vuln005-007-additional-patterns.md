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

Found **18+ call sites** of `FatalNoSecurityImpact` in the codebase, organized by function:

#### Factory Allocation (`src/heap/factory-base.cc`) — 6 sites

| Line | Function | Condition | Max Length |
|------|----------|-----------|------------|
| 240 | `FixedArray::New()` | `length > FixedArray::kMaxLength` | ~2^30-1 |
| 337 | `NewBytecodeArray()` | `length < 0 \|\| length > BytecodeArray::kMaxLength` | Negative + max |
| 1191 | `NewBigInt()` | `length > BigInt::kMaxLength` | BigInt digit limit |
| 1357 | `AllocateRawFixedArray()` | `length < 0 \|\| length > FixedArray::kMaxLength` | Low-level allocation |
| 1368 | `AllocateRawWeakArrayList()` | `capacity < 0 \|\| capacity > WeakArrayList::kMaxCapacity` | Weak array |
| 1418 | `AllocateSwissNameDictionary()` | `capacity < 0 \|\| capacity > SwissNameDictionary::MaxCapacity()` | Dictionary |

#### FixedArray Inline Allocation (`src/objects/fixed-array-inl.h`) — 10 sites (crbug.com/1201626)

| Line | Function | Type |
|------|----------|------|
| 314 | `FixedArray::New()` (basic) | `FixedArrayBase::kMaxLength` |
| 337 | `FixedArray::New()` (with callback) | `FixedArrayBase::kMaxLength` |
| 366 | `TrustedFixedArray::New()` | `TrustedFixedArray::kMaxLength` |
| 388 | `ProtectedFixedArray::New()` | `ProtectedFixedArray::kMaxLength` |
| 567 | `FixedDoubleArray::New()` (basic) | `kMaxLength` |
| 584 | `FixedDoubleArray::New()` (with callback) | `kMaxLength` |
| 726 | `TrustedWeakFixedArray::New()` | `TrustedFixedArray::kMaxLength` |
| 742 | `ProtectedWeakFixedArray::New()` | `TrustedFixedArray::kMaxLength` |
| 842 | `ByteArray::New()` | `kMaxLength` (signed-to-unsigned cast) |
| 882 | `TrustedByteArray::New()` | `kMaxLength` (signed-to-unsigned cast) |

All 10 reference `crbug.com/1201626` — a known issue where these paths were
determined to have "no security impact." Note the `ByteArray::New()` sites at
lines 842/882 use `static_cast<unsigned>(length) > kMaxLength`, which means a
negative signed `length` becomes a very large unsigned value, correctly rejected.

#### Other Sites

| File | Line | Context | Severity |
|------|------|---------|----------|
| `flag-definitions.h` | 77 | `DEFINE_REQUIREMENT` macro for flag validation | LOW (startup only) |
| `heap.cc` | 5064 | `ConfigureHeap()` flag consistency | LOW (startup only) |

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
src/regexp/regexp-compiler.cc   - RegExp compilation (recursion enforcement)
src/regexp/regexp-compiler.h    - Recursion limit definitions
src/regexp/regexp-interpreter.cc - Backtrack stack with SBXCHECK
src/regexp/regexp-parser.cc     - RegExp parsing
src/regexp/regexp-ast.h         - AST node definitions (recursive structure)
```

### Recursion Limit Mechanism

**Definition** (`regexp-compiler.h:587-594`):
```cpp
#if defined(V8_TARGET_OS_MACOS)
  static constexpr int kMaxRecursion = 50;   // macOS: 512kB stack
#else
  static constexpr int kMaxRecursion = 100;  // Others: 8MB stack
#endif
```
Reference: `crbug.com/408820921`

**Tracking** (`regexp-compiler.h:595-597`):
```cpp
inline int recursion_depth() { return recursion_depth_; }
inline void IncrementRecursionDepth() { recursion_depth_++; }
inline void DecrementRecursionDepth() { recursion_depth_--; }
```

**Enforcement Point 1** (`regexp-compiler.cc:1521-1524`):
```cpp
bool RegExpNode::KeepRecursing(RegExpCompiler* compiler) {
  return !compiler->limiting_recursion() &&
         compiler->recursion_depth() <= RegExpCompiler::kMaxRecursion;
}
```

**Enforcement Point 2** (`regexp-compiler.cc:2583-2586`) — loop length analysis:
```cpp
int recursion_depth = 0;
while (node != this) {
  if (recursion_depth++ > RegExpCompiler::kMaxRecursion) {
    return kNodeIsTooComplexForFixedLengthLoops;
  }
```

### Stack Overflow Check — "Super Hacky"

`regexp-compiler.h:621-630`:
```cpp
// The recursive nature of ToNode node generation means we may run into stack
// overflow issues. We introduce periodic checks to detect these, and the
// tick counter helps limit overhead of these checks.
// TODO(jgruber): This is super hacky and should be replaced by an abort
// mechanism or iterative node generation.
void ToNodeMaybeCheckForStackOverflow() {
  if ((to_node_overflow_check_ticks_++ % 64 == 0)) {
    ToNodeCheckForStackOverflow();
  }
}
```

**Concern**: The check only runs every 64 calls. If a deeply nested pattern
causes >64 levels of recursion between checks, the stack could overflow before
detection. The `% 64` interval means up to 63 recursive calls happen unchecked.

### Backtracking Stack Protection

`regexp-interpreter.cc:126-150`:
```cpp
class BacktrackStack {
  V8_WARN_UNUSED_RESULT bool push(int v) {
    data_.emplace_back(v);
    return (static_cast<int>(data_.size()) <= kMaxSize);
  }
  int peek() const {
    SBXCHECK(!data_.empty());  // Sandbox-hardened check
    return data_.back();
  }
```

The `SBXCHECK` on `peek()` is a sandbox-hardened check (survives release builds),
preventing underflow on the backtracking stack.

### Additional RegExp Concerns

**Experimental compiler** (`experimental-compiler.cc:480`):
```cpp
// TODO(v8:10765): Handle stack overflow instead of passing unlimited max
```
Experimental path has NO stack overflow handling.

**Bytecode generator** (`regexp-bytecode-generator.cc:428,441`):
```cpp
// TODO(pthier): This is super hacky. We could still check for 4 characters
```
Multiple hacks acknowledged in bytecode generation.

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

### Impact

- **Primary**: Denial of service (process crash via stack overflow)
- **Mitigated by**: kMaxRecursion limits (50/100), periodic stack checks
- **Gaps**: Stack check granularity (every 64 calls), experimental compiler
  unlimited stack, platform-dependent stack sizes

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

## Additional Pattern: StoreNoWriteBarrier in Keyed Store Generic

### Severity: LOW (Correct Usage) / AUDIT TARGET

### Analysis (`src/ic/keyed-store-generic.cc`)

9 specific `StoreNoWriteBarrier` call sites in keyed element store operations:

| Line | Value Stored | Representation | Safety |
|------|-------------|----------------|--------|
| 468 | SMI value | `kTaggedSigned` | **SAFE** — SMIs never require write barriers |
| 509 | `kUndefinedNanInt64` | `kWord64` | **SAFE** — Raw double bit pattern |
| 514 | `kUndefinedNanLower32` | `kWord32` | **SAFE** — Raw bits |
| 517 | `kUndefinedNanLower32` (upper) | `kWord32` | **SAFE** — Raw bits |
| 545 | `double_value` | `kFloat64` | **SAFE** — Primitive double |
| 611 | `double_value` | `kFloat64` | **SAFE** — Primitive for PACKED_DOUBLE_ELEMENTS |
| 623 | `kUndefinedNanInt64` | `kWord64` | **SAFE** — Raw bit pattern |
| 627 | `kUndefinedNanLower32` | `kWord32` | **SAFE** — Raw bits |
| 629 | `kUndefinedNanLower32` (upper) | `kWord32` | **SAFE** — Raw bits |

**Assessment**: All uses in keyed-store-generic are safe because:
1. SMI values are verified via `TaggedIsSmi(value)` before store
2. Double values are primitive bit patterns, not heap object references
3. All paths verify element kind before selecting the store path

**Residual Risk**: If element kind checking fails or gets corrupted (e.g., via
sandbox memory corruption), what the code assumes is a double could actually
store a tagged pointer without write barrier. However, element kind transitions
are guarded by the type feedback system.

---

## Additional Pattern: UNREACHABLE() Elimination Assumptions

### Severity: MEDIUM (Type Confusion Indicator)

### Location: `src/compiler/js-generic-lowering.cc`

21+ sites assume specific optimization phases eliminate operations before
generic lowering. Each follows the pattern:

```cpp
void JSGenericLowering::LowerJSXxx(Node* node) {
  UNREACHABLE();  // Eliminated in typed lowering.
}
```

### Complete List

| Line | Operation | Expected Eliminator |
|------|-----------|-------------------|
| 567 | `JSHasContextExtension` | Typed lowering |
| 571 | `JSLoadContextNoCell` | Typed lowering |
| 575 | `JSLoadContext` | Typed lowering |
| 579 | `JSStoreContextNoCell` | Typed lowering |
| 583 | `JSStoreContext` | Context specialization |
| 633 | `JSCreateArrayIterator` | Typed lowering |
| 637 | `JSCreateAsyncFunctionObject` | Typed lowering |
| 641 | `JSCreateCollectionIterator` | Typed lowering |
| 645 | `JSCreateBoundFunction` | Typed lowering |
| 649 | `JSObjectIsArray` | Typed lowering |
| 657 | `JSCreateStringWrapper` | Typed lowering |
| 717+ | 10+ additional operations | Various phases |

### Security Implications

1. **Single point of failure**: If typed lowering fails to eliminate an operation,
   `UNREACHABLE()` triggers an abort rather than a graceful fallback
2. **Type confusion indicator**: A type confusion bug that prevents proper
   elimination would trigger these crashes — useful as canary signals
3. **Not defensive**: Unlike `CHECK()`, `UNREACHABLE()` provides no diagnostic
   information about what went wrong or recovery path
4. **Potential masking**: If reached via crafted input, the crash may not be
   flagged as security-relevant by fuzzers (depends on crash classification)

### Attack Relevance

An attacker who can prevent the typed lowering phase from eliminating a specific
operation (e.g., via malformed feedback data or type system confusion) could
force execution to reach `UNREACHABLE()`. While this is "just" a crash, it
indicates the compiler entered an invalid state, which may have other
exploitable consequences before the crash occurs.

---

## Additional Pattern: Integer Overflow Protection

### Severity: LOW (Well-Protected)

### BigInt Addition Overflow (`src/objects/bigint.cc:1201-1211`)

```cpp
uint32_t result_length = input_length + will_overflow;  // will_overflow is 0 or 1
```

**Safe**: `BigInt::kMaxLength` is well below `UINT32_MAX`, so `+1` cannot wrap.

### SignedMulOverflow32 Checks (`src/objects/fixed-array-inl.h`)

5 allocation paths use `base::bits::SignedMulOverflow32()` to prevent
`length * sizeof(T)` overflow:

| Line | Function |
|------|----------|
| 928 | `FixedIntegerArrayBase` template |
| 983 | `PodArray::New()` |
| 993 | `PodArray::New()` (LocalIsolate) |
| 1003 | `TrustedPodArray::New()` |
| 1013 | `TrustedPodArray::New()` (LocalIsolate) |

All use the same pattern:
```cpp
int byte_length;
CHECK(!base::bits::SignedMulOverflow32(length, sizeof(T), &byte_length));
```

**Assessment**: Integer overflow detection is comprehensive for array allocation
paths. The `CHECK` (not `DCHECK`) ensures this runs in release builds.

---

## Additional Pattern: Write Barrier Mode Caching

### Severity: LOW (Theoretical)

**Location**: `src/heap/factory-base.cc:440-446`

```cpp
WriteBarrierMode write_barrier_mode = allocation == AllocationType::kYoung
                                          ? SKIP_WRITE_BARRIER
                                          : UPDATE_WRITE_BARRIER;
result->set_context(*context, write_barrier_mode);
result->set_arguments(*arguments, write_barrier_mode);
```

Write barrier mode is determined at allocation time. For `kYoung` allocation,
barriers are skipped. This is safe because young space objects are evacuated
(copied) during scavenge, not promoted in place. The GC root set handles the
newly allocated object correctly.

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
| FatalNoSecurityImpact (factory) | MEDIUM | 6 sites confirmed | Audit sandbox-reachable paths |
| FatalNoSecurityImpact (FixedArray) | MEDIUM | 10 sites, crbug.com/1201626 | Verify "no security impact" classification |
| RegExp recursion limits | LOW | 50/100 limits enforced | Stack check granularity (every 64 calls) |
| RegExp experimental compiler | LOW | No stack limits | Experimental-only, not production |
| StoreNoWriteBarrier (CSA) | HIGH (if wrong) | 45 occurrences | Priority: lines 3649, 4138, 4207 |
| StoreNoWriteBarrier (keyed-store) | LOW | 9 sites, all safe | SMI/double verified before store |
| UnsafeStoreNoWriteBarrier | HIGH (if wrong) | 3+ explicit unsafe | Audit lines 4140, 4210, 5197 |
| UNREACHABLE() elimination | MEDIUM | 21+ sites | Type confusion indicator/canary |
| Integer overflow (array alloc) | LOW | 5 sites, well-protected | `SignedMulOverflow32` CHECK |
| Write barrier mode caching | LOW | Theoretical only | Safe due to scavenge semantics |
| Concurrent marking races | HIGH (if wrong) | Systemic risk | Barrier completeness audit |
