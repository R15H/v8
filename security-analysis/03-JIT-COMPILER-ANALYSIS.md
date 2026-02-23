# JIT Compiler Security Analysis

## Overview

The JIT compiler is V8's most security-critical component. Type confusion bugs
in the JIT type system are the most common and most exploitable class of V8
vulnerabilities. This document provides a detailed analysis of the type systems
in TurboFan, Turboshaft, and Maglev.

## 1. TurboFan Type System

### Architecture

TurboFan uses a **bitset + range** type system:
- **Bitset types**: Represent categories like `Number`, `String`, `Boolean`
- **Range types**: `Range(min, max)` using IEEE 754 doubles
- **Union types**: Combine multiple type components

Defined in:
- `src/compiler/turbofan-types.h` - Type class hierarchy
- `src/compiler/turbofan-types.cc` - Type operations
- `src/compiler/type-cache.h` - Cached type constants

### The Typer (`src/compiler/turbofan-typer.cc`)

The typer traverses the TurboFan IR graph and computes types for each node.
It uses the `OperationTyper` for semantic operations.

**Security-critical flow**:
```
turbofan-typer.cc::TypeNode()
  → operation-typer.cc::NumberAdd/Multiply/etc.
  → turbofan-types.cc::Type::Range(min, max)
  → Result used in typed-optimization.cc and simplified-lowering.cc
```

### OperationTyper (`src/compiler/operation-typer.cc`)

This is the heart of the type system. It computes result types for every
operation. Key functions:

#### CheckBounds (line 1344-1353)
```cpp
Type OperationTyper::CheckBounds(Type index, Type length) {
  DCHECK(length.Is(cache_->kPositiveSafeInteger));
  if (length.Is(cache_->kSingletonZero)) return Type::None();
  Type const upper_bound = Type::Range(0.0, length.Max() - 1, zone());
  if (index.Maybe(Type::String())) return upper_bound;
  if (index.Maybe(Type::MinusZero())) {
    index = Type::Union(index, cache_->kSingletonZero, zone());
  }
  return Type::Intersect(index, upper_bound, zone());
}
```

**Analysis**:
- `length.Max() - 1` computes the maximum valid index
- The DCHECK constrains length to safe integers (prevents precision loss)
- BUT DCHECKs are disabled in release builds
- If length somehow exceeds safe integer range, `length.Max() - 1` could
  equal `length.Max()` due to double precision, allowing OOB access
- **Verdict**: Protected by DCHECK in debug, potentially exploitable if
  the DCHECK precondition is violated in release

#### NumberMultiply (line 760-774)
```cpp
Type OperationTyper::NumberMultiply(Type lhs, Type rhs) {
  ...
  bool maybe_nan = lhs.Maybe(Type::NaN()) || rhs.Maybe(Type::NaN()) ||
                   (lhs.Maybe(cache_->kZeroish) &&
                    (rhs.Min() == -V8_INFINITY || rhs.Max() == V8_INFINITY)) ||
                   (rhs.Maybe(cache_->kZeroish) &&
                    (lhs.Min() == -V8_INFINITY || lhs.Max() == V8_INFINITY));
```
This is the **correct** version. Compare with Turboshaft below.

#### WeakenRange (line 47-123)
```cpp
Type OperationTyper::WeakenRange(Type previous_range, Type current_range) {
  static const double kWeakenMinLimits[] = {0.0, -1073741824.0, ...};
  static const double kWeakenMaxLimits[] = {0.0, 1073741823.0, ...};
  ...
}
```

Range weakening is used during fixpoint iteration to ensure convergence.
It snaps type range bounds to predefined limits. If the weakening limits
are too aggressive, the resulting type can be overly wide, which is safe
(but imprecise). If they're too narrow, the type can be wrong, which is
dangerous. The current limits go up to `kMaxAdditiveSafeInteger`/
`kMinAdditiveSafeInteger`.

### TypedOptimization (`src/compiler/typed-optimization.cc`)

This phase uses type information to simplify the graph.

#### ReduceCheckBounds (line 206-220)
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

**Analysis**: This reduces bounds checks by removing unnecessary conversion
flags. The check itself isn't removed here - that happens in
SimplifiedLowering. But removing flags changes the semantics of the check.

#### ReduceCheckNotTaggedHole (line 223-231)
```cpp
if (!input_type.Maybe(Type::Hole())) {
  ReplaceWithValue(node, input);
  return Replace(input);
}
```

If the type says "not a hole", the hole check is eliminated. If the type
was wrong and a hole reaches this point, it will be treated as a valid value
→ potential info leak or type confusion.

### SimplifiedLowering (`src/compiler/simplified-lowering.cc`)

This is one of the most complex and security-critical phases. It runs a
three-phase fixpoint algorithm:

```
Phase 1: PROPAGATE (line 63-72)
  Traverse graph backwards, computing what representation each use needs.
  Iterates to fixpoint for phi nodes and loops.

Phase 2: RETYPE (line 74-75)
  Forward pass computing types based on actual representations chosen.

Phase 3: LOWER (line 77-84)
  Performs actual lowering: replaces Simplified ops with Machine ops,
  inserts representation changes, may remove checks.
```

#### CanOverflowSigned32 (line 216-242)
```cpp
bool CanOverflowSigned32(const Operator* op, Type left, Type right, ...) {
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
```

**Analysis**: This checks if speculative integer operations can overflow.
If it returns `false`, the overflow check is eliminated. The computation
uses double arithmetic on `Type::Min()` / `Type::Max()`, which could
lose precision for very large or very small int32 values when combined.
However, the Signed32 intersection bounds these to [-2^31, 2^31-1],
within safe double range.

---

## 2. Turboshaft Type System

### Architecture

Turboshaft is V8's newer compiler IR, gradually replacing TurboFan's Sea of
Nodes with a more traditional CFG-based IR. Its type system is defined in
a single large header file.

Defined in:
- `src/compiler/turboshaft/typer.h` (1,625 lines)
- `src/compiler/turboshaft/type-inference-reducer.h`

### WordOperationTyper (typer.h:50-403)

Handles integer types with fixed-width word semantics (Word32, Word64).
Uses `uint64_t` ranges internally.

### FloatOperationTyper (typer.h:405-1140)

Handles floating-point types with special cases for NaN and -0.

#### CONFIRMED BUG: Multiply (line 613-642)

```cpp
static Type Multiply(type_t l, type_t r, Zone* zone) {
  if (l.is_only_nan() || r.is_only_nan()) return type_t::NaN();
  bool maybe_nan = l.has_nan() || r.has_nan() ||
                   (IsZeroish(l) && (r.min() == -inf || r.max() == inf)) ||
                   (IsZeroish(r) && (l.min() == -inf || r.max() == inf));
  //                                              BUG: ^^^^^^^^^ should be l.max()
```

**The Bug**: On line 620, the condition for `IsZeroish(r)` (right operand
is zero-ish) checks `r.max() == inf` instead of `l.max() == inf`.

**Expected behavior**: When `r` can be zero and `l` can be infinity, the
result can be NaN (since 0 * inf = NaN).

**Actual behavior**: The condition checks if `r` can be infinity (which is
already covered by the first condition `IsZeroish(l) && r.max() == inf`).
This means when `r` is zero-ish and `l.max() == inf` (but `l.min() != -inf`),
NaN is incorrectly excluded from the result type.

**Proof by comparison**: TurboFan's correct version at
`operation-typer.cc:770-774`:
```cpp
(rhs.Maybe(cache_->kZeroish) &&
 (lhs.Min() == -V8_INFINITY || lhs.Max() == V8_INFINITY));
//                              ^^^ correctly checks lhs
```

**Impact**: If the result type of a multiplication doesn't include NaN when
it should, downstream optimizations may:
1. Eliminate NaN checks (`Number.isNaN()` branches)
2. Use the result in comparisons that assume non-NaN
3. Eliminate bounds checks if the non-NaN value is used as an index
4. Generate code that treats NaN as a valid number

See [06-VULNERABILITIES.md](06-VULNERABILITIES.md) for exploitation strategy.

#### allow_invalid_inputs() (line 1617-1619)

```cpp
// For now we allow invalid inputs (which will then just lead to very generic
// typing). Once all operations are implemented, we are going to disable this.
static bool allow_invalid_inputs() { return true; }
```

**Impact**: All operations with unrecognized or malformed input types silently
return `Type::Any()` instead of crashing. While this is "safe" in the sense
that `Type::Any()` is the widest type, it means:
1. Bugs where incorrect types flow through the graph go undetected
2. The type system doesn't enforce that all operations are properly typed
3. Precision loss cascades through the graph silently

---

## 3. Maglev Type System

### Architecture

Maglev uses `int64_t`-based ranges, which are inherently safer than
TurboFan's double-based ranges (no precision loss).

Defined in:
- `src/maglev/maglev-range.h` - Range class
- `src/maglev/maglev-range-analysis.h` - Range analysis framework
- `src/maglev/maglev-graph-builder.cc` - Graph construction with types
- `src/maglev/maglev-compiler.cc` - Compiler entry point

### NodeRanges (maglev-range-analysis.h)

```
NodeRanges tracks int64_t [min, max] ranges for each node.
Uses a 3-phase approach:
  1. Forward pass computing ranges
  2. Narrowing based on branch conditions
  3. Widening for loop phis (with configurable widening limits)
```

**Security-relevant**: The widening for loop phis must be carefully bounded.
Too much widening = safe but imprecise. Too little = might not converge
(infinite loop in compiler, or incorrect narrow type).

### IC Feedback Poisoning

Maglev relies heavily on inline cache (IC) feedback from Ignition. An
attacker can potentially poison IC feedback by:
1. Running a function many times with one type to establish feedback
2. Triggering Maglev compilation based on this feedback
3. Calling the function with a different type after compilation
4. If the deopt check is missing or bypassable, type confusion occurs

This is a general attack vector for all speculative JIT compilers.

---

## 4. Deoptimization

### Overview

When speculative optimizations fail at runtime, V8 must "deoptimize" -
transfer control from optimized code back to the interpreter. This involves:

1. Recording frame state information during compilation
2. At runtime, detecting type guard failures
3. Materializing interpreter frames from optimized state
4. Resuming in the interpreter

### Security Relevance

If deoptimization doesn't trigger when it should:
- Optimized code runs on unexpected types → type confusion
- Bounds checks may have been eliminated

If deoptimization materializes incorrect state:
- Variables may have wrong values → logic bugs
- Side effects may be replayed or lost

### Key Deoptimization Points

```
DeoptimizeIf / DeoptimizeUnless nodes:
  src/compiler/common-operator.cc

Deoptimization reasons:
  src/deoptimizer/deoptimize-reason.h

Deoptimizer implementation:
  src/deoptimizer/deoptimizer.cc

Frame state:
  src/compiler/frame-states.cc
```

### Attack Vector: Turboshaft Type Bug → Missing Deopt

If the Turboshaft typer computes an incorrect narrow type for a value,
the compiler may decide that a deoptimization guard is unnecessary:
- "The type says this is always a number, so no need to check"
- At runtime, the value is NaN (which wasn't included in the type)
- The deopt doesn't fire because the guard was removed
- Code continues with an unexpected NaN value
- NaN propagates through arithmetic, potentially affecting array indices

---

## 5. Exploitation Strategies

### Strategy 1: Typer Bug → OOB Array Access

```javascript
// Pseudo-exploit for Turboshaft Multiply NaN bug
function trigger(a) {
  // Force Turboshaft to type 'a' as Float64 range [1, Infinity]
  let x = a * 0;  // Typer thinks: result is in [0, 0] (not NaN)
                    // Reality: if a == Infinity, result is NaN

  // Use x in a way that assumes it's not NaN
  // NaN converts to 0 in integer context in some paths,
  // or propagates incorrectly through comparisons
  let idx = x < 0 ? 0 : x;  // Typer: idx is always 0 (since x is [0,0])
                               // Reality: NaN < 0 is false, idx = NaN

  // If further optimization uses idx as a known-constant 0, bounds check
  // may be eliminated. Then at runtime, actual behavior depends on how
  // NaN is handled in the generated code path.
}

// Warm up with normal values, then trigger with Infinity
for (let i = 0; i < 10000; i++) trigger(i + 1);
trigger(Infinity);
```

### Strategy 2: Element Kind Confusion → addrof/fakeobj

```javascript
// Classic V8 exploit primitive (general pattern, not specific bug):
// 1. Get a type confusion between PACKED_DOUBLE and PACKED elements
// 2. Use it to implement addrof(obj) and fakeobj(addr)

// addrof: Store an object, read it back as a double → get address
// fakeobj: Write a double (address), read it back as an object → fake object

// With addrof + fakeobj, construct arbitrary R/W primitives:
// 1. Create a fake ArrayBuffer with controlled backing store pointer
// 2. Read/write through the fake ArrayBuffer → arbitrary memory access
```

### Strategy 3: Sandbox Escape (post-initial-bug)

After achieving arbitrary R/W inside the sandbox:
```
1. Locate pointer tables in sandbox memory
2. Corrupt an External Pointer Table entry to point outside sandbox
3. Or corrupt a Code Pointer Table entry to redirect code execution
4. Or corrupt a JS Dispatch Table entry to hijack function calls
5. Use redirected pointer/code for out-of-sandbox access
```

See [05-SANDBOX-ANALYSIS.md](05-SANDBOX-ANALYSIS.md) for details.
