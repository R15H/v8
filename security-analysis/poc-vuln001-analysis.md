# VULN-001: Turboshaft Multiply NaN Type Confusion - Deep Analysis & PoC

## Bug Confirmation

### Location
- **File**: `src/compiler/turboshaft/typer.h:620`
- **Function**: `FloatOperationTyper<Bits>::Multiply()`
- **Bug Type**: Copy-paste error (wrong variable name)

### Buggy Code
```cpp
// Line 618-620:
bool maybe_nan = l.has_nan() || r.has_nan() ||
                 (IsZeroish(l) && (r.min() == -inf || r.max() == inf)) ||
                 (IsZeroish(r) && (l.min() == -inf || r.max() == inf));
//                                                     ^^^^^^^^^
//                                         BUG: should be l.max() == inf
```

### Correct Code (TurboFan, operation-typer.cc:770-774)
```cpp
bool maybe_nan = lhs.Maybe(Type::NaN()) || rhs.Maybe(Type::NaN()) ||
                 (lhs.Maybe(cache_->kZeroish) &&
                  (rhs.Min() == -V8_INFINITY || rhs.Max() == V8_INFINITY)) ||
                 (rhs.Maybe(cache_->kZeroish) &&
                  (lhs.Min() == -V8_INFINITY || lhs.Max() == V8_INFINITY));
//                                              ^^^ CORRECT: checks lhs
```

## Complete Call Chain

```
JavaScript: a * b
    ↓
Ignition bytecode: Mul
    ↓ (hot function, tier up to TurboFan/Turboshaft)
Turboshaft IR: FloatBinopOp { kind: kMul, rep: Float64 }
    ↓
type-inference-reducer.h:362-371:
  REDUCE(FloatBinop)(left, right, kind, rep)
    → Typer::TypeFloatBinop(GetType(left), GetType(right), kind, rep, zone)
    ↓
typer.h:1272-1309:
  TypeFloatBinop() dispatches based on kind and rep:
    case FloatBinopOp::Kind::kMul (Float64):
      → TypeFloat64Mul(left_type, right_type, zone)
    ↓
typer.h:1312-1323 (FLOAT_BINOP macro):
  TypeFloat64Mul() calls:
    → FloatOperationTyper<64>::Multiply(lhs.AsFloat64(), rhs.AsFloat64(), zone)
    ↓
typer.h:613-667:
  Multiply() - THE BUGGY FUNCTION
    → Computes result type with incorrect maybe_nan flag
    ↓
Type stored in graph, consumed by (IF --turboshaft-typed-optimizations):
  - typed-optimizations-reducer.h (constant folding, branch elimination)
  - Other downstream Turboshaft optimization passes
  - LessThan/LessThanOrEqual comparisons (typer.h:945-1011)

Pipeline phase ordering (pipelines.h:201-255):
  Phase 4: OptimizePhase (always) ← MachineOptimizationReducer
  Phase 5: TypedOptimizationsPhase (EXPERIMENTAL, default OFF)
    ↑ This is the only phase that uses the buggy type result
  Phase 6: TypeAssertionsPhase (debug only)

Note: The Maglev→Turboshaft transition path is:
  turbolev-graph-builder.cc:4389  PROCESS_FLOAT64_BINOP(Multiply, Mul)
    → assembler.h:1756  Float64Mul() emits FloatBinopOp(kMul, Float64)
```

## Exploitability Analysis (Revised)

### Trigger Condition
The bug manifests when:
- `r` (right operand) is "zeroish" (contains 0, -0, or NaN)
- `l` (left operand) has `l.max() == +Infinity` but `l.min() != -Infinity`
- Example: `l = Range[1, +Infinity]`, `r = Range[-1, 1]`

### Redundant NaN Detection (Mitigating Factor)

The Multiply function has **three NaN detection mechanisms**:

1. **`maybe_nan` flag (line 618-620)** ← BUGGY
2. **Set path (line 637-639 → ProductSet line 492-493)**: If both operands are
   sets, ProductSet independently detects NaN in computed products
3. **Range path (line 652-656)**: If any endpoint product is NaN, returns `Any()`

**The bug only matters when**:
- The range path is taken (at least one operand is not a set)
- The 4 endpoint products (`l_min*r_min`, `l_min*r_max`, `l_max*r_min`,
  `l_max*r_max`) do NOT produce NaN
- But interior values CAN produce NaN (specifically `Infinity * 0`)

This happens when:
- `l = Range[a, +Infinity]` where `a > 0` (Infinity is at an endpoint)
- `r = Range[b, c]` where `b < 0 < c` (0 is in the range but NOT at an endpoint)
- Example: `l = Range[1, inf]`, `r = Range[-1, 1]`

Endpoint products: `1*(-1)=-1, 1*1=1, inf*(-1)=-inf, inf*1=inf` → **no NaN**

### Impact Assessment

With the above trigger:
- Result type: `Range[-inf, inf, -0]` **without NaN** (incorrect)
- Actual runtime range: includes NaN when `a=Infinity, b≈0`

**Current Impact: LIMITED (Multiple Mitigating Factors)**

**Factor 1: TypedOptimizations phase is EXPERIMENTAL (disabled by default)**

The phase that consumes the buggy type inference results is gated behind:
```cpp
// flag-definitions.h:1747
DEFINE_EXPERIMENTAL_FEATURE(turboshaft_typed_optimizations, ...)
// DEFINE_EXPERIMENTAL_FEATURE sets default=false (flag-definitions.h:245)
```

This means `--turboshaft-typed-optimizations` is **false by default**. The
`TypedOptimizationsReducer` (which would use the incorrect NaN type to fold
branches or constants) does NOT run unless explicitly enabled.

**Factor 2: Wide result range**

The result range `[-inf, inf]` is extremely wide, so:
- **Constant folding** (`typed-optimizations-reducer.h:108-118`): Not triggered
  (range is not a constant)
- **Branch elimination** (`typed-optimizations-reducer.h:32-50`): Not triggered
  (comparison results still `{0, 1}`)
- **LessThan optimization** (`typer.h:945-975`): The `has_nan()` check on line 971
  doesn't change the outcome because `can_be_false` is already true for the
  wide range

**Factor 3: Float64Equal is not typed**

The `kEqual` comparison returns `{0, 1}` unconditionally (TODO at
`typer.h:1492-1494`), so `x === x` cannot be folded to `true`.

### Future Exploitability: HIGH

The bug becomes exploitable when ANY of these conditions change:

1. **`--turboshaft-typed-optimizations` becomes default-on**:
   The flag is marked as experimental and will likely be enabled by default
   as Turboshaft matures. When enabled, all typed optimization reducers
   will consume the incorrect type.

2. **Float64Equal gets typed** (currently TODO at `typer.h:1492-1494`):
   If `Float64Equal(x, x)` is typed to `Constant(1)` when `!x.has_nan()`,
   then `x !== x` branch would be incorrectly eliminated, allowing NaN to
   flow into code that assumes non-NaN.

3. **New optimization passes** check `has_nan()`:
   Any future Turboshaft reducer that uses `has_nan()` to make optimization
   decisions (e.g., eliminating NaN guards, optimizing conversions) would be
   affected.

4. **Cross-tier interaction**: If Turboshaft types feed into TurboFan or Maglev
   (through OSR or shared feedback), the incorrect type could affect
   optimizations in those tiers.

## Proof-of-Concept (Static Analysis)

### PoC 1: Demonstrating the Type Bug

```javascript
// poc-vuln001-type-demo.js
// Run with: d8 --turboshaft --trace-turbo-types --allow-natives-syntax

function multiply_trigger(a, b) {
  // Training: a in [1, Infinity], b in [-1, 1]
  // At runtime: a = Infinity, b = 0 → result = NaN
  return a * b;
}

// Train with values that establish the desired type ranges
for (let i = 0; i < 10000; i++) {
  // a values: 1, 2, 3, ..., large numbers, Infinity
  let a = (i % 100 === 0) ? Infinity : (i + 1);
  // b values: between -1 and 1 (includes 0)
  let b = Math.sin(i);  // sin() returns [-1, 1]
  multiply_trigger(a, b);
}

// Force Turboshaft compilation
%OptimizeFunctionOnNextCall(multiply_trigger);

// Trigger the bug: Infinity * 0 = NaN
// But Turboshaft types the result as Range[-inf, inf, -0] without NaN
let result = multiply_trigger(Infinity, 0);
console.log("Result:", result);            // Expected: NaN
console.log("Is NaN:", Number.isNaN(result)); // Expected: true
console.log("Result === Result:", result === result); // Expected: false (NaN !== NaN)

// If Turboshaft incorrectly eliminated a NaN check based on the type,
// these would print incorrect results.
```

### PoC 2: Exploitation Sketch (Requires Float64Equal Typing)

```javascript
// poc-vuln001-exploit-sketch.js
// This PoC would work IF Float64Equal typing is implemented in Turboshaft
// Currently Float64Equal returns {0,1} (TODO at typer.h:1492)

function exploit(a, b) {
  let x = a * b;
  // Turboshaft types x as Range[-inf, inf, -0] WITHOUT NaN
  // If Float64Equal(x, x) was typed:
  //   - Without NaN: x === x → always true → Constant(1)
  //   - With NaN: x === x → sometimes false → Set({0, 1})

  if (x !== x) {
    // With Float64Equal typing, this branch would be eliminated
    // (typer says x === x is always true, so x !== x is dead code)
    // But at runtime with x = NaN, this IS reachable

    // We could put exploit code here that shouldn't be reachable:
    // e.g., corrupt array metadata, trigger type confusion
    return "BUG: reached dead code!";
  }

  return "normal path";
}

// Train
for (let i = 0; i < 10000; i++) {
  exploit((i % 50 === 0) ? Infinity : (i + 1), Math.sin(i));
}

// Trigger
let result = exploit(Infinity, 0);
// If Float64Equal were typed: "normal path" (dead code eliminated)
// Correct behavior: "BUG: reached dead code!"
```

### PoC 3: Full Exploitation Chain (Theoretical)

```javascript
// poc-vuln001-full-chain.js
// Full exploitation chain IF Float64Equal typing exists

function pwn(a, b, arr, oob_arr) {
  let x = a * b;
  // x typed as Range[-inf, inf] without NaN

  // Step 1: Exploit dead code elimination via x !== x
  // (Requires Float64Equal typing)
  if (x !== x) {
    // This code is "dead" according to the compiler
    // but reachable at runtime with x = NaN

    // Step 2: Access array with "impossible" code path
    // Since this is "dead code", no bounds checks are emitted
    // Write to oob_arr to corrupt adjacent memory
    oob_arr[0] = 1.1;  // This write might go to wrong memory
  }

  // Alternative: Use NaN in arithmetic to confuse index computation
  // NaN | 0 === 0 in JS, but if typer thinks x is a number,
  // it might optimize (x | 0) differently
  let idx = x | 0;  // NaN | 0 = 0, but typer thinks x is number
  return arr[idx];
}
```

## Recommended Fix

```diff
--- a/src/compiler/turboshaft/typer.h
+++ b/src/compiler/turboshaft/typer.h
@@ -617,7 +617,7 @@
     if (l.is_only_nan() || r.is_only_nan()) return type_t::NaN();
     bool maybe_nan = l.has_nan() || r.has_nan() ||
                      (IsZeroish(l) && (r.min() == -inf || r.max() == inf)) ||
-                     (IsZeroish(r) && (l.min() == -inf || r.max() == inf));
+                     (IsZeroish(r) && (l.min() == -inf || l.max() == inf));

     // Try to rule out -0.
     bool maybe_minuszero = l.has_minus_zero() || r.has_minus_zero() ||
```

## Severity Assessment (Updated with Pipeline Analysis)

| Aspect | Rating |
|--------|--------|
| Bug Confirmed | YES - copy-paste error verified against TurboFan equivalent |
| TypedOptimizations Phase | EXPERIMENTAL - `--turboshaft-typed-optimizations` defaults to `false` |
| Current Exploitability | **VERY LOW** - typed opts disabled by default + redundant NaN detection + wide ranges |
| Future Exploitability | **CRITICAL** - when typed opts default on + Float64Equal typed + new passes |
| Ease of Fix | TRIVIAL - one character change (`r` → `l`) |
| Similar Past CVEs | CVE-2023-2033, CVE-2023-3079, CVE-2024-0517 (type confusion in JIT) |
| Recommended Severity | **LOW** currently, **CRITICAL** once `--turboshaft-typed-optimizations` is enabled by default |

### Key Finding: Three Gates to Exploitability

The bug requires ALL THREE of these conditions to become exploitable:

1. `--turboshaft-typed-optimizations` must be enabled (currently experimental, default OFF)
2. Float64Equal typing must be implemented (currently TODO at typer.h:1492)
3. The specific input range pattern must be triggered (`l=[1,inf]`, `r` contains 0 interior)

When ALL three conditions are met, the exploitation chain is:
```
Multiply(inf, 0) → NaN (but typed as non-NaN)
  → Float64Equal(NaN, NaN) → typed as Constant(1) (should be 0)
  → Branch(NaN !== NaN) eliminated as dead code
  → NaN flows into "dead" code path → type confusion → OOB
```

## Verification Steps

1. Build V8 with `--turboshaft-typed-optimizations --turboshaft-trace-typing`
2. Run PoC 1 to observe the type of the multiply result
3. Check if the type includes NaN: it should but doesn't
4. Apply the one-character fix and verify the type now includes NaN
5. Run V8 type system unit tests to confirm no regression
6. Note: Without `--turboshaft-typed-optimizations`, the bug is present in the
   type computation but has no downstream effect
