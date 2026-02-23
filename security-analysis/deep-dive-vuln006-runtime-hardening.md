# VULN-006: Runtime Function Hardening Gaps Deep Dive

## Summary

V8's runtime functions (`src/runtime/*.cc`) are C++ functions callable from
generated code and the interpreter. They perform complex operations like
property lookups, array manipulations, and type conversions. With 671+
RUNTIME_FUNCTION definitions across 32 files, but only **1 SBXCHECK** in
the entire `src/runtime/` directory, there is a significant gap in
sandbox-hardening of runtime functions.

## Severity: MEDIUM (Post-Initial-Bug Exploitation Surface)

Runtime functions are typically not directly exploitable from JavaScript
alone. However, after achieving memory corruption inside the sandbox (via
a JIT bug like VULN-001), runtime functions become a key exploitation
target because they:
1. Accept HeapObject parameters from potentially corrupted memory
2. Perform pointer-following operations on those objects
3. May write to memory based on corrupted object fields
4. Often don't validate that their inputs are consistent with expected types

## Quantitative Analysis

### SBXCHECK Distribution

```
SBXCHECK usage across V8 src/:

  src/objects/elements.cc:                  9  (array element access)
  src/wasm/module-instantiate.cc:           9  (WASM module setup)
  src/codegen/code-stub-assembler.cc:       8  (code generation)
  src/objects/code-inl.h:                   6  (code objects)
  src/ast/scopes.cc:                        6  (scope management)
  src/objects/intl-objects.cc:              6  (internationalization)
  src/wasm/wasm-objects.cc:                15  (WASM objects)
  src/wasm/wasm-objects-inl.h:              6  (WASM inline)
  src/execution/isolate.cc:                 3  (isolate operations)
  src/objects/casting.h:                    3  (type casting)

  src/runtime/ (all 32 files combined):     1  (runtime-regexp.cc)

  Total across src/:                      ~165 occurrences
```

### Runtime Functions Without SBXCHECK

**671 RUNTIME_FUNCTION definitions** across 32 files, with only **1 SBXCHECK**.
The vast majority of runtime functions have no sandbox-aware input validation.

Top files by RUNTIME_FUNCTION count (0 SBXCHECK each):
```
src/runtime/runtime-test.cc:         147 functions (test only, less critical)
src/runtime/runtime-object.cc:        74 functions
src/runtime/runtime-wasm.cc:          70 functions
src/runtime/runtime-internal.cc:      64 functions
src/runtime/runtime-scopes.cc:        30 functions
src/runtime/runtime-debug.cc:         26 functions
src/runtime/runtime-strings.cc:       23 functions
src/runtime/runtime-atomics.cc:       22 functions
src/runtime/runtime-compiler.cc:      21 functions
src/runtime/runtime-promise.cc:       15 functions
```

### High-Priority Audit Targets

Runtime functions that handle untrusted data and perform memory operations:

#### 1. `runtime-array.cc` (10 functions, 0 SBXCHECK)
Array operations that modify backing stores. If called with corrupted array
objects, they may write to arbitrary memory.

#### 2. `runtime-typedarray.cc` (8 functions, 0 SBXCHECK)
TypedArray operations that interact with ArrayBuffer backing stores. A
corrupted TypedArray with wrong byte_length or backing_store pointer could
lead to OOB access.

#### 3. `runtime-object.cc` (74 functions, 0 SBXCHECK)
Property operations on HeapObjects. A corrupted Map pointer could cause the
runtime to misinterpret object layout.

#### 4. `runtime-strings.cc` (23 functions, 0 SBXCHECK)
String operations with length fields. Corrupted string lengths could cause
buffer overflows during concatenation or slicing.

#### 5. `runtime-regexp.cc` (12 functions, 1 SBXCHECK)
The only runtime file with any SBXCHECK. The single check is at line 360,
with a comment: "trusted space, this is not a SBXCHECK". This is actually
a comment about a CHECK being sufficient rather than an SBXCHECK, since the
data is in trusted space.

## Evidence of Known Gaps

### Recent Hardening Commit

Commit `a03a1a3c` ("Harden some runtime functions against corrupted input")
explicitly acknowledges that runtime functions were vulnerable:

```
This commit adds validation to specific runtime functions that are known
to be callable with corrupted inputs from inside the sandbox.
```

The fact that this was a **recent fix** suggests:
1. The V8 team knows this is a problem
2. They are incrementally adding hardening
3. Many functions likely remain unhardened

### Pattern in Other V8 Code

Compare with `src/objects/elements.cc` (9 SBXCHECKs) - this file handles
element access which is a known exploitation target. The SBXCHECKs verify:
- Array length consistency
- Element kind consistency
- Backing store validity

Similar checks should exist in runtime functions that perform equivalent
operations.

## Exploitation Scenario

### Post-Corruption Runtime Abuse

After achieving arbitrary R/W inside the sandbox (via JIT bug):

```javascript
// Step 1: Corrupt an array's Map pointer to change its element kind
// (via OOB write from initial bug)
//
// Step 2: Call a runtime function that reads the corrupted array
%RuntimeCall(corrupted_array);
// The runtime function trusts the Map's element kind
// Reads doubles as tagged pointers (or vice versa)
// → type confusion → further exploitation

// Step 3: Or corrupt a string's length field
// Then call a runtime function that copies/slices the string
// → OOB read/write during string operation
```

### Specific Attack Chain

```
1. JIT bug (VULN-001) → OOB write in array backing store
2. Corrupt adjacent JSArray's length field (make it larger)
3. Call Array.prototype.slice() on the corrupted array
   → runtime-array.cc handles this
   → no SBXCHECK on length → copies beyond actual allocation
   → heap corruption
4. Use heap corruption to target more sensitive objects
```

## Recommended Mitigations

### Short-term
1. **Audit runtime-array.cc**: Add SBXCHECK for array length and element kind
   before accessing elements
2. **Audit runtime-typedarray.cc**: Verify byte_length and byte_offset against
   backing store size
3. **Audit runtime-object.cc**: Validate Map consistency before property access
4. **Audit runtime-strings.cc**: Verify string length before copy operations

### Medium-term
5. **Add SBXCHECK linting**: Automated check that runtime functions accessing
   sandbox memory include appropriate SBXCHECKs
6. **Create runtime hardening tracker**: Track which runtime functions have
   been audited and which still need hardening

### Long-term
7. **Type-safe runtime wrappers**: Create wrapper types that enforce validation
   on construction, so runtime functions can't accidentally skip checks

## Key File References

| File | RUNTIME_FUNCTIONs | SBXCHECKs | Priority |
|------|-------------------|-----------|----------|
| `runtime-object.cc` | 74 | 0 | HIGH |
| `runtime-wasm.cc` | 70 | 0 | MEDIUM |
| `runtime-internal.cc` | 64 | 0 | HIGH |
| `runtime-scopes.cc` | 30 | 0 | MEDIUM |
| `runtime-strings.cc` | 23 | 0 | HIGH |
| `runtime-atomics.cc` | 22 | 0 | MEDIUM |
| `runtime-compiler.cc` | 21 | 0 | MEDIUM |
| `runtime-promise.cc` | 15 | 0 | LOW |
| `runtime-regexp.cc` | 12 | 1 | LOW (partially done) |
| `runtime-array.cc` | 10 | 0 | HIGH |
| `runtime-typedarray.cc` | 8 | 0 | HIGH |
| Total | 671 | 1 | -- |

## Related CVEs

- **CVE-2024-0517**: V8 OOB write exploiting runtime function behavior
- **CVE-2023-4427**: V8 OOB access through runtime property operations
- Runtime functions are commonly part of exploitation chains rather than
  being the initial vulnerability
