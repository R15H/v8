# VULN-003: TurboFan CheckBounds Precision & Bounds Check Elimination Deep Dive

## Summary

The TurboFan bounds check elimination pipeline relies on IEEE 754 double-precision
arithmetic for index range computations. The `CheckBounds()` function in
`operation-typer.cc:1344-1353` computes `length.Max() - 1` to determine the
maximum valid index. For very large length values, this subtraction can lose
precision due to double-precision floating-point limitations. Additionally, the
multi-phase bounds check elimination process involves multiple code paths where
type information drives check removal decisions.

## Severity: HIGH (Bounds Check Elimination Chain)

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

2. **Multi-layer checking**: Bounds checks go through multiple phases
   (TypedOptimization → SimplifiedLowering), each independently verifying
   type consistency.

3. **Deoptimization guards**: Speculative optimizations include deoptimization
   checks that trigger if runtime values don't match expected types.

4. **DCHECK as safety net**: While disabled in release, DCHECKs catch
   violations during development and testing.

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
| `type-cache.h` | 86-93 | `kSafeInteger`, `kPositiveSafeInteger` |
| `type-cache.h` | 97 | `kFixedArrayLengthType` |
| `type-cache.h` | 110 | `kJSArrayLengthType` |
| `type-cache.h` | 128-129 | `kJSTypedArrayLengthType` |

## Similar Past CVEs

- **CVE-2023-2033**: Type confusion in V8 (incorrect type for JIT-compiled code)
- **CVE-2024-0517**: OOB write in V8 via incorrect bounds check elimination
- **CVE-2023-4427**: Out of bounds memory access in V8 (type confusion → OOB)
- **CVE-2024-4947**: Type confusion in V8 (Maglev JIT)
