# Specific Vulnerabilities Found

## VULN-001: Turboshaft Multiply NaN Type Confusion [CRITICAL]

### Location
- **File**: `src/compiler/turboshaft/typer.h`
- **Line**: 620
- **Function**: `FloatOperationTyper::Multiply()`
- **Component**: Turboshaft compiler type system

### Description

Copy-paste bug in the NaN-detection logic of the Multiply type computation.
The second condition for `maybe_nan` incorrectly checks `r.max()` instead
of `l.max()`.

### Buggy Code (typer.h:618-620)

```cpp
bool maybe_nan = l.has_nan() || r.has_nan() ||
                 (IsZeroish(l) && (r.min() == -inf || r.max() == inf)) ||
                 (IsZeroish(r) && (l.min() == -inf || r.max() == inf));
//                                                     ^^^^^^^^^
//                                         BUG: should be l.max() == inf
```

### Correct Code (operation-typer.cc:770-774, TurboFan version)

```cpp
bool maybe_nan = lhs.Maybe(Type::NaN()) || rhs.Maybe(Type::NaN()) ||
                 (lhs.Maybe(cache_->kZeroish) &&
                  (rhs.Min() == -V8_INFINITY || rhs.Max() == V8_INFINITY)) ||
                 (rhs.Maybe(cache_->kZeroish) &&
                  (lhs.Min() == -V8_INFINITY || lhs.Max() == V8_INFINITY));
//                                              ^^^^^^^^^^^^^^^^^^^
//                                   CORRECT: checks lhs (left operand)
```

### Root Cause

The four conditions in `maybe_nan` encode the mathematical rule:
- `0 * Infinity = NaN` (regardless of signs)

The conditions should be:
1. `l.has_nan()` - left already contains NaN
2. `r.has_nan()` - right already contains NaN
3. `IsZeroish(l) && (r can be +-inf)` - left is zero, right is infinity
4. `IsZeroish(r) && (l can be +-inf)` - right is zero, left is infinity

Condition 4 has a typo: `r.max() == inf` should be `l.max() == inf`.
The typo checks if the RIGHT operand can be positive infinity when it's
already established that the RIGHT operand is ZEROISH. The intended check
is whether the LEFT operand can be positive infinity.

### Trigger Condition

The bug manifests when:
- `r` (right operand) is zero-ish (can be 0, -0, or NaN)
- `l` (left operand) has range where `l.min() != -inf` AND `l.max() == inf`
- Example: `l` has type `Float64[1, +Infinity]`, `r` has type `Float64{0}`

In this case:
- Condition 3 is false (l is not zeroish)
- Condition 4 is false (r.max() is 0, not inf) ← BUG
- Result: `maybe_nan = false`
- But `Infinity * 0 = NaN`, so the result CAN be NaN

### Impact

The result type of the multiplication will NOT include NaN, even though
the actual runtime result can be NaN. Downstream optimizations that rely
on the type not containing NaN may:

1. **Eliminate NaN checks**: `if (Number.isNaN(x))` branch eliminated
2. **Incorrect comparison folding**: `x === x` optimized to `true`
   (NaN !== NaN, but typer says not NaN)
3. **Bounds check elimination**: If the non-NaN result is used to compute
   an array index, and the type says it's in a safe range, the bounds
   check may be removed. NaN in arithmetic produces unexpected values.
4. **Branch elimination**: If a branch condition involves the result, and
   the type says the branch always goes one way, the other path is removed

### Exploitation Strategy

```javascript
// Step 1: Create a function that Turboshaft will compile
function vuln(a) {
  // Force 'a' to be typed as Float64[1, +Infinity]
  // by providing type feedback with values in that range
  if (a < 1) return;
  if (!isFinite(a) && a < 0) return;  // a is in [1, +Inf]

  let x = a * 0;
  // Turboshaft types x as Float64[0, 0] (without NaN) ← WRONG
  // Actual value when a=Infinity: NaN

  // Exploit the incorrect type...
  // The specific exploitation depends on what optimizations
  // Turboshaft performs on the incorrectly-typed value.

  // Example: use in comparison
  if (x === x) {  // Turboshaft: always true (no NaN)
    // This branch is always taken by the optimizer
    // But at runtime with a=Infinity, x=NaN, x!==x
    // So this should NOT be taken
    return doSomethingDangerous();
  }
}

// Step 2: Train with safe values
for (let i = 0; i < 100000; i++) vuln(i + 1);

// Step 3: Trigger the bug
vuln(Infinity);
```

### Suggested Fix

```diff
- (IsZeroish(r) && (l.min() == -inf || r.max() == inf));
+ (IsZeroish(r) && (l.min() == -inf || l.max() == inf));
```

### Similar Past CVEs
- CVE-2023-2033: V8 type confusion in speculative compilation
- CVE-2023-3079: V8 type confusion in optimizing compiler
- CVE-2024-0517: V8 OOB write via incorrect type inference

---

## VULN-002: Turboshaft allow_invalid_inputs() Always True [HIGH]

### Location
- **File**: `src/compiler/turboshaft/typer.h`
- **Lines**: 1617-1619
- **Function**: `Typer::allow_invalid_inputs()`
- **Component**: Turboshaft type system framework

### Description

```cpp
// For now we allow invalid inputs (which will then just lead to very generic
// typing). Once all operations are implemented, we are going to disable this.
static bool allow_invalid_inputs() { return true; }
```

### Impact

When an operation receives inputs with unexpected types, instead of crashing
(which would detect bugs during testing), the Turboshaft typer silently
returns `Type::Any()`. This means:

1. Type inference bugs go undetected in testing
2. The Turboshaft type system is less precise than it could be
3. Cascading imprecision: `Type::Any()` propagates through the graph,
   reducing the precision of all downstream types
4. Operations that should have specific type requirements accept anything

### Security Concern

This is a **defense-in-depth weakness** rather than a direct vulnerability.
It makes other type system bugs harder to find because they're masked by
the permissive fallback. A bug that produces an incorrect type for an
operation's input won't crash during fuzzing - it'll just degrade precision.

### Verification

1. Build V8 with `allow_invalid_inputs()` returning `false`
2. Run the test suite - any failures indicate operations receiving
   incorrectly-typed inputs
3. Each failure is a potential type system bug worth investigating

---

## VULN-003: TurboFan CheckBounds Type Precision Boundary [MEDIUM]

### Location
- **File**: `src/compiler/operation-typer.cc`
- **Line**: 1347
- **Function**: `OperationTyper::CheckBounds()`

### Description

```cpp
Type OperationTyper::CheckBounds(Type index, Type length) {
  DCHECK(length.Is(cache_->kPositiveSafeInteger));
  if (length.Is(cache_->kSingletonZero)) return Type::None();
  Type const upper_bound = Type::Range(0.0, length.Max() - 1, zone());
```

The computation `length.Max() - 1` uses IEEE 754 double arithmetic. For
values beyond the safe integer range (> 2^53), `x - 1 == x` due to
precision loss. This would make the upper bound equal to `length.Max()`,
allowing an OOB access.

### Mitigating Factors

The `DCHECK(length.Is(cache_->kPositiveSafeInteger))` ensures length is
within [0, 2^53] where `x - 1` is always precise. **However**:
- DCHECKs are disabled in release builds
- If any path feeds a non-safe-integer length to CheckBounds, the DCHECK
  won't fire in production

### Exploitation

Requires finding a path where `length` exceeds safe integer range and
reaches `CheckBounds`. In practice, JavaScript array lengths are capped
at 2^32-1, making this unlikely but not impossible for TypedArrays with
very large backing stores.

### Verification

1. Search for all callers of CheckBounds in the TurboFan pipeline
2. Check if any can produce length types outside kPositiveSafeInteger
3. Test with TypedArrays backed by very large ArrayBuffers

---

## VULN-004: Sandbox Partial Reservation Weakness [MEDIUM]

### Location
- **File**: `src/sandbox/sandbox.h`
- **Lines**: 72, 109-113

### Description

```cpp
static constexpr bool kFallbackToPartiallyReservedSandboxAllowed = true;

bool is_partially_reserved() const { return reservation_size_ < size_; }
```

And from `src/sandbox/sandbox.cc:126`:
```
// virtual memory, but also don't get the desired security properties as
```

### Impact

When the sandbox falls back to partial reservation:
1. No guard regions → OOB accesses within sandbox escape undetected
2. Unrelated memory can be mapped inside the sandbox address range
3. An attacker with OOB inside the sandbox could reach non-V8 memory

### Exploitation

1. Consume virtual address space to force partial reservation
2. `mmap` controlled data into the sandbox address range
3. Achieve sandbox-internal OOB (via any JIT bug)
4. Reach the controlled mapping → arbitrary R/W outside sandbox

### Verification

Check `Sandbox::is_partially_reserved()` in a running Chrome instance.
On memory-constrained systems (mobile, older hardware), this is more likely.

---

## VULN-005: FatalNoSecurityImpact Crash Classification [MEDIUM]

### Location
- **File**: `src/base/logging.cc`
- **Lines**: 94-114
- **File**: `src/base/abort-mode.h`
- **Lines**: 32-57

### Description

```cpp
void FatalNoSecurityImpact(const char* format, ...) {
  OS::PrintError("\n\n#\n# Fatal error with no security impact:\n# ");
  ...
  if (FatalErrorsWithNoSecurityImpactShouldExit()) {
    // Just exit cleanly instead of crashing
  }
}
```

Some fatal errors are classified as having "no security impact" and can
be configured to exit cleanly instead of crashing. If any of these
classifications are incorrect (the error actually does have security
impact), the process quietly exits instead of producing a crash report.

### Impact

- Potential security bugs silently suppressed in production
- Fuzzers configured with `kExitIfNoSecurityImpact` won't report these
  as crashes, missing real vulnerabilities

### Verification

Audit all call sites of `FatalNoSecurityImpact` and
`FATAL_NO_SECURITY_IMPACT` to verify the "no security impact"
classification is correct.

---

## VULN-006: Runtime Function Hardening Gaps [MEDIUM]

### Location
- **Files**: `src/runtime/*.cc` (35 files)
- **Evidence**: Recent commit `a03a1a3c` ("Harden some runtime functions
  against corrupted input")

### Description

Runtime functions are C++ functions callable from JIT-compiled code and
the interpreter. They receive V8 objects as arguments. After the sandbox
threat model was adopted (attacker has arbitrary R/W in sandbox), these
functions need to validate that their inputs haven't been corrupted.

The recent hardening commit suggests that some runtime functions were
**not** validating their inputs, meaning corrupted objects could cause:
- Out-of-bounds accesses in C++ code
- Type confusion in C++ (treating corrupted Map as valid)
- Controlled writes through corrupted object fields

### Exploitation

1. Achieve sandbox R/W (via any initial bug)
2. Corrupt an object that will be passed to a runtime function
3. The runtime function processes the corrupted object unsafely
4. Achieve out-of-sandbox memory corruption

### Verification

1. Identify all runtime functions: `grep "RUNTIME_FUNCTION" src/runtime/*.cc`
2. For each, check if it validates its arguments against corruption
3. Focus on functions that:
   - Access array elements by index (OOB)
   - Follow pointer chains (could lead outside sandbox)
   - Write to computed offsets (controlled writes)

---

## VULN-007: RegExp Visitor Stack Overflow Pattern [LOW]

### Location
- **Files**: `src/regexp/regexp-*.cc`
- **Evidence**: Recent commit `b07860d0` (regexp AST visitor stack overflow fix)

### Description

The regexp parser and AST visitors use recursive descent, which can overflow
the stack with deeply nested patterns. While individual fixes have been
applied, the pattern of recursive visitors in the regexp subsystem suggests
more stack overflow vulnerabilities may exist.

### Impact

Stack overflow → potential for stack-based buffer overflow or controlled
crash. Generally low severity as modern OSes have guard pages.

### Verification

1. Fuzz regexp patterns with deep nesting
2. Audit all visitor classes in `src/regexp/` for recursion depth checks
3. Check if the existing stack overflow checks are consistent

---

## Summary Table

| ID | Title | Severity | Exploitability | Requires |
|----|-------|----------|----------------|----------|
| VULN-001 | Turboshaft Multiply NaN | CRITICAL | High | Turboshaft compilation path |
| VULN-002 | allow_invalid_inputs | HIGH | Medium | Turboshaft + secondary bug |
| VULN-003 | CheckBounds precision | MEDIUM | Low | Non-safe-integer length |
| VULN-004 | Partial sandbox | MEDIUM | Medium | Memory pressure + sandbox OOB |
| VULN-005 | FatalNoSecurityImpact | MEDIUM | Low | Misclassified error |
| VULN-006 | Runtime hardening | MEDIUM | High | Sandbox R/W + unhardened function |
| VULN-007 | RegExp stack overflow | LOW | Low | Deep regexp nesting |
