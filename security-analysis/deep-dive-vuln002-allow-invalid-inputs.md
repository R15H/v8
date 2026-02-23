# VULN-002: Turboshaft `allow_invalid_inputs()` Deep Dive

## Summary

The Turboshaft type system unconditionally accepts invalid and malformed type
inputs via `allow_invalid_inputs()` returning `true` at `typer.h:1619`. This
causes 30+ unimplemented operations to silently return `Type::Any()` instead
of crashing, masking type propagation bugs and creating a systemic weakness
in the compiler's type safety guarantees.

## Severity: HIGH (Systemic Type Safety Degradation)

## Location

- **File**: `src/compiler/turboshaft/typer.h`
- **Function**: `allow_invalid_inputs()` at line 1619
- **Guard function**: `InputIs()` at lines 1602-1615
- **Comment**: "For now we allow invalid inputs (which will then just lead to
  very generic typing). Once all operations are implemented, we are going to
  disable this."

## Mechanism

### The Guard Function

```cpp
// typer.h:1602-1615
static bool InputIs(const Type& input, Type::Kind expected) {
  if (input.IsInvalid()) {
    if (allow_invalid_inputs()) return false;  // ← silently accepts
  } else if (input.kind() == expected) {
    return true;
  } else if (input.IsAny()) {
    if (allow_invalid_inputs()) return false;  // ← silently accepts
  }

  std::stringstream s;
  s << expected;
  FATAL("Missing proper type (%s). Type is: %s", s.str().c_str(),
        input.ToString().c_str());
}
```

When `allow_invalid_inputs()` returns `true`:
1. **Invalid types** → `InputIs()` returns `false` → caller returns `Any()`
2. **Wrong-kind types** (e.g., Float64 where Word32 expected) → `FATAL` crash
3. **`Type::Any()` inputs** → `InputIs()` returns `false` → caller returns `Any()`

The key insight: `Type::Any()` is treated the same as invalid. This means if
ANY operation in the chain fails to produce a specific type and returns `Any()`,
all downstream operations also degrade to `Any()`.

### The Cascade Effect

```
Operation A (unimplemented) → returns Any()
    ↓
Operation B (implemented, calls InputIs) → InputIs returns false → returns Any()
    ↓
Operation C (implemented, calls InputIs) → InputIs returns false → returns Any()
    ↓
... entire subgraph loses type precision
```

## Unimplemented Operations (Return Any())

### Word Operations (`TypeWordBinop`, lines 1203-1231)

Only `kAdd` and `kSub` are implemented for Word32 and Word64. All others return
`Any()`:

```cpp
// typer.h:1215-1217
default:
  // TODO(nicohartmann@): Support remaining {kind}s.
  return Word32Type::Any();
```

Missing implementations:
| Operation | Kind | Status |
|-----------|------|--------|
| Multiply | `kMul` | TODO → Any() |
| BitwiseAnd | `kBitwiseAnd` | TODO → Any() |
| BitwiseOr | `kBitwiseOr` | TODO → Any() |
| BitwiseXor | `kBitwiseXor` | TODO → Any() |
| ShiftLeft | `kShiftLeft` | TODO → Any() |
| ShiftRightArithmetic | `kShiftRightArithmetic` | TODO → Any() |
| ShiftRightLogical | `kShiftRightLogical` | TODO → Any() |
| RotateRight | `kRotateRight` | TODO → Any() |
| RotateLeft | `kRotateLeft` | TODO → Any() |
| SignedDiv | `kSignedDiv` | TODO → Any() |
| UnsignedDiv | `kUnsignedDiv` | TODO → Any() |
| SignedMod | `kSignedMod` | TODO → Any() |
| UnsignedMod | `kUnsignedMod` | TODO → Any() |

### Overflow-Checked Operations (`TypeOverflowCheckedBinop`, lines 1347-1375)

Only `kSignedAdd` for Word32 is implemented. Missing:

```cpp
// typer.h:1358-1362
case OverflowCheckedBinopOp::Kind::kSignedSub:
case OverflowCheckedBinopOp::Kind::kSignedMul:
  // TODO(nicohartmann@): Support these.
  return TupleType::Tuple(Word32Type::Any(),
                          Word32Type::Set({0, 1}, zone), zone);
```

| Operation | Status |
|-----------|--------|
| Word32 SignedSub | TODO → (Any(), {0,1}) |
| Word32 SignedMul | TODO → (Any(), {0,1}) |
| Word64 SignedAdd | TODO → (Any(), {0,1}) |
| Word64 SignedSub | TODO → (Any(), {0,1}) |
| Word64 SignedMul | TODO → (Any(), {0,1}) |

### Float Comparisons (lines 1470-1505)

`kEqual` is unimplemented for both Float32 and Float64:

```cpp
// typer.h:1473-1475 (Float32)
case ComparisonOp::Kind::kEqual:
  // TODO(nicohartmann@): Support this.
  return Word32Type::Set({0, 1}, zone);

// typer.h:1492-1494 (Float64)
case ComparisonOp::Kind::kEqual:
  // TODO(nicohartmann@): Support this.
  return Word32Type::Set({0, 1}, zone);
```

**This is directly security-relevant**: When Float64Equal IS typed, the
VULN-001 Multiply NaN bug becomes immediately exploitable. If `x === x` is
typed as `Constant(1)` when `!x.has_nan()`, then `x !== x` branches are
eliminated. Combined with the Multiply bug that incorrectly excludes NaN,
this creates a dead-code-elimination exploit.

### Word Comparisons (lines 1427-1464)

`kEqual` is also unimplemented for Word32 and Word64:

```cpp
// typer.h:1435-1437
case ComparisonOp::Kind::kEqual:
  // TODO(nicohartmann@): Support this.
  return Word32Type::Set({0, 1}, zone);
```

## InputIs() Call Sites (40+)

The `InputIs()` function is called throughout the typer. Key call sites where
invalid inputs are silently accepted:

```
typer.h:1249-1251: TypeWord64Add - InputIs(lhs, Word64) and InputIs(rhs, Word64)
typer.h:1261-1262: TypeWord64Sub - same pattern
typer.h:TruncateWord32Input (line 1521-1522) - InputIs implicit via IsAny() check
```

Each call site that returns `Any()` on failure can cascade to make all
downstream type computations imprecise.

## Exploitation Analysis

### Current Impact: MEDIUM

The `Any()` fallback is actually SAFE in isolation - it's the widest possible
type, so no optimization will make incorrect assumptions. However:

1. **Precision loss cascade**: One `Any()` result propagates through the entire
   computation subgraph, potentially causing optimizations to miss opportunities
   OR (critically) causing imprecise types to interact with precise types in
   unexpected ways at merge points.

2. **Masking of real bugs**: Because invalid inputs don't crash, type system
   bugs go undetected. The VULN-001 Multiply bug was only found by manual
   code review, not by type system invariant violations.

3. **Future danger**: When `allow_invalid_inputs()` is finally set to `false`,
   any code that currently relies on the silent degradation will crash. This
   means there may be latent bugs hidden behind this flag that produce
   incorrect results in edge cases.

### Interaction with VULN-001

The `allow_invalid_inputs()` behavior is directly relevant to VULN-001's
future exploitability:

```
1. VULN-001 produces incorrect type (missing NaN flag)
2. Float64Equal typing is currently TODO (returns {0, 1})
3. When Float64Equal is implemented, it will check has_nan()
4. If has_nan() is false (due to VULN-001), Float64Equal(x, x) → Constant(1)
5. Branch x !== x is eliminated as dead code
6. NaN flows into "dead" code path → type confusion
```

The timeline for this becoming exploitable is when Float64Equal typing is
implemented (the TODO at typer.h:1492-1494).

### Attack Scenario: Imprecision-Induced Bug

```javascript
// If a WordBinop (e.g., BitwiseAnd) returns Any() due to missing
// implementation, and this Any() flows into a typed operation that
// normally narrows the range, the narrowing may interact incorrectly
// with the Any() type at phi merge points.

function trigger(x) {
  let a = x | 0;      // Word32 truncation
  let b = a & 0xFF;   // BitwiseAnd → Word32Type::Any() (unimplemented!)
  // TypedOptimization sees b as Any() instead of Range[0, 255]
  // If b is used as an array index, bounds check CANNOT be eliminated
  // (safe, but imprecise)

  // However, at phi merge points:
  if (x > 0) {
    b = x & 0xFF;     // Also Any()
  }
  // Phi merge of two Any() values = Any()
  // But the INTENDED type is Range[0, 255]
  // Any downstream optimization that uses this will be imprecise
  return arr[b];       // Bounds check NOT eliminated (safe)
}
```

Currently this produces imprecise but safe results. The danger is if future
passes ADD optimizations that interact with `Any()` in ways that produce
incorrect narrowings.

## Recommended Fixes

### Short-term (High Priority)
1. **Implement Float64Equal/Float32Equal typing** - this is needed independently
   and its absence is what currently prevents VULN-001 from being exploitable
2. **Add logging/metrics** when `allow_invalid_inputs()` triggers - to track
   how often this fallback path is taken in practice

### Medium-term
3. **Implement remaining WordBinop kinds** - especially BitwiseAnd/Or which
   are very common in JavaScript and produce narrow, well-defined ranges
4. **Implement OverflowCheckedBinop** for Sub and Mul

### Long-term
5. **Set `allow_invalid_inputs()` to `false`** once all operations are typed
6. **Add fuzzing** specifically targeting Turboshaft type inference with
   complex operation chains to catch cascade failures

## Related Code References

| File | Line | Description |
|------|------|-------------|
| `typer.h` | 1619 | `allow_invalid_inputs()` definition |
| `typer.h` | 1602-1615 | `InputIs()` guard function |
| `typer.h` | 1203-1231 | `TypeWordBinop()` with unimplemented kinds |
| `typer.h` | 1347-1375 | `TypeOverflowCheckedBinop()` with TODOs |
| `typer.h` | 1473-1475 | Float32 kEqual TODO |
| `typer.h` | 1492-1494 | Float64 kEqual TODO |
| `typer.h` | 1435-1437 | Word32 kEqual TODO |
| `typer.h` | 1453-1455 | Word64 kEqual TODO |
| `type-inference-reducer.h` | 139-151 | Generic operation fallback |
| `typed-optimizations-reducer.h` | 108-118 | Optimization that checks has_nan() |
