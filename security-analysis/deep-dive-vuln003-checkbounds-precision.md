# VULN-003: TurboFan CheckBounds Precision & Bounds Check Elimination Deep Dive

## Summary

The TurboFan bounds check elimination pipeline relies on IEEE 754 double-precision
arithmetic for index range computations. The `CheckBounds()` function in
`operation-typer.cc:1344-1353` computes `length.Max() - 1` to determine the
maximum valid index. For very large length values, this subtraction can lose
precision due to double-precision floating-point limitations. Additionally, the
multi-phase bounds check elimination process involves multiple code paths where
type information drives check removal decisions.

## Severity: LOW (Precision Loss) / HIGH (BCE Chain with Upstream Bug)

**Revised after deep investigation**: The precision loss at `length.Max() - 1` is
constrained by type cache limits. With sandbox: max 32GB-1 (safe). Without sandbox
on 64-bit: max 2^53-1 (exact in IEEE 754). The real severity comes from the bounds
check elimination chain being exploitable with an upstream typer bug.

## Key Code Paths

### 1. CheckBounds Type Computation (`operation-typer.cc:1344-1353`)

```cpp
Type OperationTyper::CheckBounds(Type index, Type length) {
  DCHECK(length.Is(cache_->kPositiveSafeInteger));  // Debug-only!
  if (length.Is(cache_->kSingletonZero)) return Type::None();
  Type const upper_bound = Type::Range(0.0, length.Max() - 1, zone());
  if (index.Maybe(Type::String())) return upper_bound;
  if (index.Maybe(Type::MinusZero())) {
    index = Type::Union(index, cache_->kSingletonZero, zone());
  }
  return Type::Intersect(index, upper_bound, zone());
}
```

#### Precision Analysis

The `length.Max() - 1` computation uses IEEE 754 double arithmetic. Key limits:

```
kMaxSafeInteger = 2^53 - 1 = 9007199254740991
```

For values up to `kMaxSafeInteger`, subtraction by 1 is exact because:
- IEEE 754 doubles have 53 bits of mantissa
- For integers up to 2^53, all values are exactly representable
- `(2^53 - 1) - 1 = 2^53 - 2` is exactly representable

The DCHECK on line 1345 ensures `length.Is(kPositiveSafeInteger)`, which
constrains length to `[0, 2^53 - 1]`. Within this range, `length.Max() - 1`
is always exact.

**However**: DCHECKs are disabled in release builds. If a code path manages
to produce a length type that exceeds safe integer range:

```
length.Max() = 2^53 + 2  →  length.Max() - 1 = 2^53 + 2  (no change!)
```

This would make `upper_bound = Range(0, 2^53 + 2)` instead of
`Range(0, 2^53 + 1)`, meaning the bounds check would allow one index past
the end.

### 2. Typed Optimization Phase (`typed-optimization.cc:206-220`)

```cpp
Reduction TypedOptimization::ReduceCheckBounds(Node* node) {
  CheckBoundsParameters const& p = CheckBoundsParametersOf(node->op());
  Node* const input = NodeProperties::GetValueInput(node, 0);
  Type const input_type = NodeProperties::GetType(input);
  if (p.flags() & CheckBoundsFlag::kConvertStringAndMinusZero &&
      !input_type.Maybe(Type::String()) &&
      !input_type.Maybe(Type::MinusZero())) {
    NodeProperties::ChangeOp(node, simplified()->CheckBounds(
        p.check_parameters().feedback(),
        p.flags().without(CheckBoundsFlag::kConvertStringAndMinusZero)));
    return Changed(node);
  }
  return NoChange();
}
```

This doesn't remove the bounds check itself - it only strips the
`kConvertStringAndMinusZero` flag. The actual bounds check survives. However,
removing this flag changes the semantics: the check no longer handles string
or minus-zero inputs, relying on the type system to guarantee they won't occur.

### 3. SimplifiedLowering Phase (`simplified-lowering.cc:63-84`)

The three-phase algorithm (PROPAGATE → RETYPE → LOWER) is where bounds checks
can be completely eliminated or lowered to unchecked machine operations.

```
Phase 1 (PROPAGATE): Determines that an index needs Word32 representation
Phase 2 (RETYPE): Recomputes types based on chosen representations
Phase 3 (LOWER): Replaces Simplified nodes with Machine nodes
  → If index type ⊆ [0, length-1], the bounds check is eliminated
  → CheckBounds becomes a plain NumberLessThan or is removed entirely
```

### 4. CanOverflowSigned32 (`simplified-lowering.cc:216-242`)

```cpp
bool CanOverflowSigned32(const Operator* op, Type left, Type right,
                         TypeCache const* type_cache, Zone* type_zone) {
  left = Type::Intersect(left, Type::Signed32(), type_zone);
  right = Type::Intersect(right, Type::Signed32(), type_zone);
  if (left.IsNone() || right.IsNone()) return false;
  switch (op->opcode()) {
    case IrOpcode::kSpeculativeSmallIntegerAdd:
      return (left.Max() + right.Max() > kMaxInt) ||
             (left.Min() + right.Min() < kMinInt);
    case IrOpcode::kSpeculativeSmallIntegerSubtract:
      return (left.Max() - right.Min() > kMaxInt) ||
             (left.Min() - right.Max() < kMinInt);
  }
}
```

**Safety analysis**: The Signed32 intersection bounds values to [-2^31, 2^31-1].
All arithmetic on these values stays within safe double range:
- Max possible add: 2^31-1 + 2^31-1 = 2^32 - 2 ≈ 4.3 billion (exact in double)
- Min possible add: -2^31 + -2^31 = -2^32 (exact in double)

This function is safe against precision loss.

### 5. CheckNotTaggedHole (`typed-optimization.cc:223-231`)

```cpp
Reduction TypedOptimization::ReduceCheckNotTaggedHole(Node* node) {
  Node* const input = NodeProperties::GetValueInput(node, 0);
  Type const input_type = NodeProperties::GetType(input);
  if (!input_type.Maybe(Type::Hole())) {
    ReplaceWithValue(node, input);
    return Replace(input);
  }
  return NoChange();
}
```

If the type says "not a hole", the hole check is completely eliminated. If the
type was wrong (e.g., due to an upstream typer bug like VULN-001), a hole value
can flow through unchecked.

## New Finding: 64-Bit Bounds Check Path for TypedArrays

### kMaxByteLength: Sandbox vs Non-Sandbox

From `src/objects/js-array-buffer.h:33-39`:

```cpp
#if V8_ENABLE_SANDBOX
  static constexpr size_t kMaxByteLength = kMaxSafeBufferSizeForSandbox;
#elif V8_HOST_ARCH_32_BIT
  static constexpr size_t kMaxByteLength = kMaxInt;
#else
  static constexpr size_t kMaxByteLength = kMaxSafeInteger;  // 2^53 - 1
#endif
```

From `include/v8-internal.h:283`:
```cpp
constexpr size_t kMaxSafeBufferSizeForSandbox = 32ULL * GB - 1;  // ~34 billion
```

| Config | kMaxByteLength | Safe Integer? | Precision Loss? |
|--------|---------------|---------------|-----------------|
| Sandbox enabled | 32GB - 1 (~3.4×10^10) | Yes (well within 2^53) | No |
| 32-bit host | kMaxInt (~2.1×10^9) | Yes | No |
| **64-bit, no sandbox** | **2^53 - 1** | **Exactly at boundary** | **No (2^53-1 is exact)** |

**Key insight**: Even on 64-bit without sandbox, `kMaxSafeInteger = 2^53 - 1` is
exactly representable in IEEE 754 double, and `(2^53 - 1) - 1 = 2^53 - 2` is also
exact. So there is NO precision loss with the defined maximums. The vulnerability
would require a code path that produces a length type EXCEEDING `kMaxSafeInteger`.

### kAllow64BitBounds Flag for TypedArrays

TypedArray accesses use a special `kAllow64BitBounds` flag that enables 64-bit
bounds checking in `SimplifiedLowering`.

From `src/compiler/js-native-context-specialization.cc:4000-4004`:
```cpp
index = effect = graph()->NewNode(
    simplified()->CheckBounds(FeedbackSource(),
                              CheckBoundsFlag::kConvertStringAndMinusZero |
                                  CheckBoundsFlag::kAllow64BitBounds),
    index, length, effect, control);
```

This flag routes through the 64-bit path in `simplified-lowering.cc:2087-2100`:
```cpp
} else {
  CHECK(length_type.Is(type_cache_->kPositiveSafeInteger));
  CHECK(allow_64_bit);
  // ... converts to CheckedUint64Bounds
  ChangeOp(node, simplified()->CheckedUint64Bounds(feedback, new_flags));
}
```

### Complete Bounds Check Elimination Logic

From `src/compiler/simplified-lowering.cc:2034-2045`:
```cpp
if (lower<T>()) {
  if (index_type.IsNone() || length_type.IsNone() ||
      (index_type.Min() >= 0.0 &&
       index_type.Max() < length_type.Min())) {
    // The bounds check is redundant
    if (v8_flags.turbo_typer_hardening) {
      new_flags |= CheckBoundsFlag::kAbortOnOutOfBounds;  // Keep check, but abort-style
    } else {
      DeferReplacement(node, NodeProperties::GetValueInput(node, 0));  // COMPLETELY REMOVES CHECK
      return;
    }
  }
  ChangeOp(node, simplified()->CheckedUint32Bounds(feedback, new_flags));
}
```

The check is eliminated when `index_type.Max() < length_type.Min()`. If
`--turbo-typer-hardening` is enabled (default: true), the check is kept but
converted to an abort-on-failure style. If disabled, the check is completely
removed via `DeferReplacement`.

### Memory Lowering: From Checked to Raw Access

After bounds check elimination, `memory-lowering.cc:393-402` converts the
high-level `LoadElement` to a raw machine `Load`:

```cpp
Reduction MemoryLowering::ReduceLoadElement(Node* node) {
  ElementAccess const& access = ElementAccessOf(node->op());
  Node* index = node->InputAt(1);
  node->ReplaceInput(1, ComputeIndex(access, index));  // Raw offset
  NodeProperties::ChangeOp(node, machine()->Load(type));  // Unchecked load
  return Changed(node);
}
```

### CheckBoundsFlag Definitions

From `src/compiler/simplified-operator.h`:

```cpp
enum class CheckBoundsFlag : uint8_t {
  kConvertStringAndMinusZero = 1 << 0,  // Convert string/minus-zero to 0 instead of deopt
  kAbortOnOutOfBounds = 1 << 1,         // Abort instead of deopt if input is OOB
  kAllow64BitBounds = 1 << 2,           // Bounds may exceed 32-bit range (up to 64-bit safe integers)
};
```

### Complete Enumeration of All 26 CheckBounds Callers

#### `js-call-reducer.cc` (9 callers)

| # | Line | Context | Flags | Max Bound | Notes |
|---|------|---------|-------|-----------|-------|
| 1 | 817 | `JSCallReducerAssembler::CheckBounds()` helper | Varies | - | Helper used by multiple methods |
| 2 | 6261 | `Array.prototype.pop()` | `kAbortOnOutOfBounds` | array.length | Only when `turbo_typer_hardening` |
| 3 | 6476 | `Array.prototype.shift()` | `kAbortOnOutOfBounds` | array.length | Only when `turbo_typer_hardening` |
| 4 | 6891 | for...of TypedArray iteration | `kAbortOnOutOfBounds\|kAllow64BitBounds` | 2^53-1 | **CRITICAL**: 64-bit TypedArray path |
| 5 | 7041 | `String.prototype.charAt()` | None | string.length | Default flags |
| 6 | 7253 | `String.fromCodePoint()` | `kConvertStringAndMinusZero` | 0x110000 | Unicode range check |
| 7 | 7443 | `String.prototype.concat()` | None | `String::kMaxLength+1` | |
| 8 | 8719 | DataView methods (fixed ArrayBuffer) | `kAllow64BitBounds` | buffer.byteLength | **CRITICAL**: 64-bit offset |
| 9 | 8743 | DataView methods (dynamic ArrayBuffer) | `kAllow64BitBounds` | buffer.byteLength | **CRITICAL**: 64-bit offset |

#### `js-native-context-specialization.cc` (12 callers)

| # | Line | Context | Flags | Max Bound | Notes |
|---|------|---------|-------|-----------|-------|
| 10 | 3501 | Generic object element store | `kConvertStringAndMinusZero` | `Smi::kMaxValue` | |
| 11 | 3507 | Generic object element load | `kConvertStringAndMinusZero` | object.length | |
| 12 | 3562 | Object load with OOB→undefined | `kConvertStringAndMinusZero\|kAbortOnOutOfBounds` | object.length | Only when `turbo_typer_hardening` |
| 13 | 3688 | Object element has() check | `kConvertStringAndMinusZero` | object.length | |
| 14 | 3797 | JSArray store with growth | `kConvertStringAndMinusZero` | array.length+1 | May add 1 for growth |
| 15 | 4001 | **TypedArray element load** | `kConvertStringAndMinusZero\|kAllow64BitBounds` | 2^53-1 | **PRIMARY EXPLOIT TARGET** |
| 16 | 4029 | TypedArray load OOB (hardened) | `kConvertStringAndMinusZero\|kAbortOnOutOfBounds\|kAllow64BitBounds` | 2^53-1 | Only when `turbo_typer_hardening` |
| 17 | 4112 | TypedArray store OOB (hardened) | `kConvertStringAndMinusZero\|kAbortOnOutOfBounds\|kAllow64BitBounds` | 2^53-1 | Only when `turbo_typer_hardening` |
| 18 | 4178 | String character access (protector) | `kConvertStringAndMinusZero` | `String::kMaxLength` | |
| 19 | 4197 | String access (hardened) | `kConvertStringAndMinusZero\|kAbortOnOutOfBounds` | string.length | Only when `turbo_typer_hardening` |
| 20 | 4217 | String access (generic) | `kConvertStringAndMinusZero` | string.length | |

#### Other files (4 callers)

| # | File | Line | Context | Flags | Max Bound |
|---|------|------|---------|-------|-----------|
| 21 | `js-create-lowering.cc` | 495 | JSArray constructor | None | `kInitialMaxFastElementArray` |
| 22 | `js-typed-lowering.cc` | 658 | String concat length | None | `String::kMaxLength+1` |
| 23 | `typed-optimization.cc` | 192 | LoadElement optimization | `kAbortOnOutOfBounds` | array.length |
| 24 | `typed-optimization.cc` | 215 | CheckBounds flag cleanup | None | - |

### Flag Usage Summary by Caller Type

| Caller Type | kConvertStringAndMinusZero | kAbortOnOutOfBounds | kAllow64BitBounds |
|-------------|:-:|:-:|:-:|
| String operations | Yes | No | No |
| String operations (hardened) | Yes | Yes | No |
| **TypedArray (main path)** | **Yes** | **No** | **Yes** |
| **TypedArray (hardened)** | **Yes** | **Yes** | **Yes** |
| **DataView** | **No** | **No** | **Yes** |
| Array operations | No | Yes | No |
| Generic objects | Yes | No | No |

The TypedArray paths at `js-native-context-specialization.cc:4001-4033` and the
DataView paths at `js-call-reducer.cc:8719-8745` are the most interesting because
they use `kAllow64BitBounds`, routing through the 64-bit path in SimplifiedLowering.

### Complete IR Operation Sequence (TypedArray Access)

For a TypedArray access with length near 2^53-1, the full pipeline is:

```
1. TypedArrayLength(receiver) → Type::Range(0, 2^53-1)
   Location: js-native-context-specialization.cc:3911

2. CheckBounds(index, length)
   Flags: kConvertStringAndMinusZero | kAllow64BitBounds
   Location: js-native-context-specialization.cc:4001-4004

3. OperationTyper::CheckBounds() type inference
   Location: operation-typer.cc:1344-1353
   Computes: upper_bound = Type::Range(0.0, length.Max() - 1, zone())
   → With possible precision loss at line 1347

4. Type narrowing via loop variable optimization or other inference

5. SimplifiedLowering::VisitCheckBounds() elimination decision
   Location: simplified-lowering.cc:2034-2045
   If index_type.Max() < length_type.Min(): ELIMINATE VIA DeferReplacement()

6. Memory lowering: LoadElement/StoreElement → raw machine ops
   Location: memory-lowering.cc:393-402

7. Raw unchecked memory load/store executes
   → Out-of-bounds access possible if types were wrong
```

### turbo_typer_hardening Check Locations

The `v8_flags.turbo_typer_hardening` flag (default: true, defined at
`flag-definitions.h:1646`) is checked at 8 locations before inserting
hardening bounds checks:

| Location | Context |
|----------|---------|
| `simplified-lowering.cc:2040` | 32-bit bounds check elimination |
| `js-native-context-specialization.cc:4027` | TypedArray load OOB |
| `js-native-context-specialization.cc:4110` | TypedArray store OOB |
| `js-native-context-specialization.cc:4195` | String character access |
| `js-call-reducer.cc:6259` | `Array.prototype.pop()` |
| `js-call-reducer.cc:6474` | `Array.prototype.shift()` |
| `js-call-reducer.cc:6889` | for...of TypedArray iteration |
| `typed-optimization.cc:190` | LoadElement optimization |

When `turbo_typer_hardening` is disabled (`--no-turbo-typer-hardening`), ALL of
these hardening checks are skipped, and bounds checks that appear redundant based
on type information are completely eliminated. This flag is the single most
important mitigation against type-system-driven BCE exploits.

### turbo_loop_variable Interaction

The `--turbo-loop-variable` flag (default: true, `flag-definitions.h:1632`)
enables loop variable induction analysis that can narrow type ranges of loop
indices. This is relevant because narrowed loop indices may cause bounds checks
to appear redundant.

Notable comment at `js-call-reducer.cc:6442-6448`:
```cpp
// When disable v8_flags.turbo_loop_variable, typer cannot infer index
// is in [1, kMaxCopyElements-1], and will break in representing
// kRepFloat64 (Range(1, inf)) to kRepWord64 when converting
// input for kLoadElement. So we need to add type guard here.
```

This shows the tight coupling between loop variable optimization and bounds
check elimination — incorrect loop variable typing can cascade into incorrect
bounds check elimination.

## Exploitation Scenarios

### Scenario 1: DCHECK Bypass for Precision Loss

```javascript
// THEORETICAL: Requires finding a path that creates a length type
// exceeding kMaxSafeInteger in release builds.

// In release builds, DCHECKs are disabled. If we could make the
// compiler believe length is beyond 2^53:
//
// length.Max() = 2^53 + 2
// upper_bound = Range(0, 2^53 + 2)  // Should be (0, 2^53 + 1)
// Bounds check allows index = 2^53 + 2 (one past end)
//
// However, in practice, array lengths are constrained to:
// - JSArray: uint32 (max ~4.2 billion, well within safe integer)
// - TypedArray: kMaxByteLength (platform-dependent, typically safe)
// - FixedArray: kMaxLength (< 2^30)
```

The type cache enforces these limits:
```cpp
// type-cache.h:110
Type const kJSArrayLengthType = Type::Unsigned32();
// type-cache.h:128-129
Type const kJSTypedArrayLengthType = CreateRange(0.0, JSTypedArray::kMaxByteLength);
// type-cache.h:97
Type const kFixedArrayLengthType = CreateRange(0.0, FixedArray::kMaxLength);
```

**Verdict**: The DCHECK is an extra safety net, but the actual length types are
constrained well within safe integer range by the type cache. Precision loss in
`CheckBounds` is **not currently exploitable**.

### Scenario 2: Upstream Type Bug → Bounds Check Bypass

This is the more realistic attack vector. If an upstream typer bug produces
an incorrect narrow type for an array index:

```javascript
// Pattern: typer says index ∈ [0, 5] but actual value is 1000
function exploit(arr, idx) {
  // If typer incorrectly computes idx type as Range[0, 5]
  // and arr.length type is Range[6, 6]
  // Then CheckBounds(idx, length) → Range[0, 5]
  // SimplifiedLowering sees [0, 5] ⊆ [0, 5] → eliminates check
  return arr[idx];  // No bounds check → OOB access
}
```

This is the classic V8 JIT exploitation pattern (CVE-2023-2033, etc.):
1. Find a typer bug that produces an overly narrow type
2. Use the narrow type to eliminate a bounds check
3. Supply a value outside the expected range at runtime
4. OOB read/write on the array's backing store

### Scenario 3: Hole Check Elimination via Type Bug

```javascript
function exploit(arr) {
  // If typer says element type doesn't include Hole
  // but the array is holey:
  let x = arr[100];  // Hole check eliminated
  // x is actually "the_hole" value
  // Treated as a valid HeapObject → type confusion
  return x + 1;      // Adds 1 to a hole sentinel → undefined behavior
}
```

### Scenario 4: VULN-001 → Bounds Check Chain

Combining the Turboshaft Multiply NaN bug with bounds check elimination:

```javascript
function exploit(a, b, arr) {
  let x = a * b;      // VULN-001: typed as [-inf, inf] without NaN
  let idx = x | 0;    // NaN | 0 = 0 in JS
                       // But typer thinks x is a number (not NaN)
                       // So typer may compute a different range for idx

  // If idx is typed as a constant or narrow range, bounds check
  // could be eliminated:
  return arr[idx];
}
```

**Current assessment**: This specific chain doesn't work because `x | 0`
truncates to int32 regardless of NaN, and the typer for bitwise OR produces
a range based on the full [-inf, inf] input, which would be Word32::Any().
The bounds check would NOT be eliminated for an Any() index.

## Mitigating Factors

1. **Type cache constraints**: Array length types are well-defined and within
   safe integer range, preventing precision loss in `CheckBounds`.

2. **`--turbo-typer-hardening` (default: true)**: When enabled, bounds checks
   that could be eliminated are instead converted to abort-on-failure checks
   (`kAbortOnOutOfBounds`). This means even if types say the check is redundant,
   a runtime check still exists. This is the **primary mitigation** against
   type-system-driven BCE attacks.

3. **Multi-layer checking**: Bounds checks go through multiple phases
   (TypedOptimization → SimplifiedLowering), each independently verifying
   type consistency.

4. **Deoptimization guards**: Speculative optimizations include deoptimization
   checks that trigger if runtime values don't match expected types.

5. **DCHECK as safety net**: While disabled in release, DCHECKs catch
   violations during development and testing.

6. **Sandbox buffer size limit**: With sandbox enabled, `kMaxByteLength` is
   capped at 32GB-1, well within safe integer range for precision.

## Verdict

**Current exploitability: LOW** for the precision loss specifically.
**Chain exploitability: HIGH** when combined with an upstream typer bug
(like VULN-001) that produces an incorrect narrow type for an array index.

The bounds check elimination pipeline is correct given correct input types.
The vulnerability is in the type system that feeds it, not in the bounds
check elimination logic itself.

## Key File References

| File | Line | Component |
|------|------|-----------|
| `operation-typer.cc` | 1344-1353 | `CheckBounds()` type computation |
| `typed-optimization.cc` | 206-220 | `ReduceCheckBounds()` flag reduction |
| `typed-optimization.cc` | 223-231 | `ReduceCheckNotTaggedHole()` |
| `simplified-lowering.cc` | 63-84 | Three-phase PROPAGATE/RETYPE/LOWER |
| `simplified-lowering.cc` | 216-242 | `CanOverflowSigned32()` |
| `simplified-lowering.cc` | 2034-2045 | **Bounds check elimination decision** |
| `simplified-lowering.cc` | 2087-2100 | **64-bit bounds check path (TypedArrays)** |
| `js-native-context-specialization.cc` | 4001-4004 | **kAllow64BitBounds for TypedArrays** |
| `memory-lowering.cc` | 393-402 | **Conversion to raw machine loads** |
| `type-cache.h` | 86-93 | `kSafeInteger`, `kPositiveSafeInteger` |
| `type-cache.h` | 97 | `kFixedArrayLengthType` |
| `type-cache.h` | 110 | `kJSArrayLengthType` |
| `type-cache.h` | 128-129 | `kJSTypedArrayLengthType` |
| `js-array-buffer.h` | 33-39 | `kMaxByteLength` (sandbox vs non-sandbox) |
| `v8-internal.h` | 283 | `kMaxSafeBufferSizeForSandbox = 32GB - 1` |
| `flag-definitions.h` | 1646 | `--turbo-typer-hardening` (default: true) |

## Similar Past CVEs

- **CVE-2023-2033**: Type confusion in V8 (incorrect type for JIT-compiled code)
- **CVE-2024-0517**: OOB write in V8 via incorrect bounds check elimination
- **CVE-2023-4427**: Out of bounds memory access in V8 (type confusion → OOB)
- **CVE-2024-4947**: Type confusion in V8 (Maglev JIT)
