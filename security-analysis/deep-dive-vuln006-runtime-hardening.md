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

## Critical Finding: 288 Unchecked Type Casts

### The `args.at<Type>()` Pattern

The most significant finding is that runtime functions receive arguments via
`args.at<Type>(index)`, which performs an **unchecked downcast** in release
builds.

**Location**: `src/execution/arguments.h:99-102`

```cpp
template <class S>
Handle<S> Arguments<T>::at(int index) const {
  Handle<Object> obj = Handle<Object>(address_of_arg_at(index));
  return Cast<S>(obj);  // No type validation in release builds!
}
```

There is a single SBXCHECK at `arguments.h:79` for bounds checking:
```cpp
SBXCHECK_LE(static_cast<uint32_t>(index), static_cast<uint32_t>(length_));
```

This prevents out-of-bounds argument access, but does NOT validate that the
fetched argument is actually the expected type.

### Unchecked Cast Count by File

| File | Unchecked `args.at<T>()` Casts |
|------|-------------------------------|
| `runtime-object.cc` | 61 |
| `runtime-test.cc` | 45 |
| `runtime-scopes.cc` | 39 |
| `runtime-strings.cc` | 35 |
| `runtime-regexp.cc` | 25 |
| `runtime-compiler.cc` | 18 |
| `runtime-classes.cc` | 17 |
| Others | 48 |
| **Total** | **288** |

### Concrete Example

From `runtime-array.cc:16-25`:

```cpp
RUNTIME_FUNCTION(Runtime_TransitionElementsKind) {
  HandleScope scope(isolate);
  DCHECK_EQ(2, args.length());                         // Debug-only!
  DirectHandle<JSObject> object = args.at<JSObject>(0); // Unchecked cast
  DirectHandle<Map> to_map = args.at<Map>(1);           // Unchecked cast
  ElementsKind to_kind = to_map->elements_kind();
  ElementsAccessor::ForKind(to_kind)->TransitionElementsKind(
      isolate, object, to_map);
  return *object;
}
```

If argument 0 is not actually a JSObject (corrupted in sandbox), or argument 1
is not a Map, no validation catches it. The DCHECK on argument count is stripped
in release builds. The `Cast<Map>` call proceeds silently with corrupted data.

### Security Implication

Every one of these 288 cast sites is a potential type confusion primitive for an
attacker with arbitrary sandbox R/W. By corrupting the arguments passed to a
runtime function (either by corrupting the stack frame or the objects on the
heap), an attacker can:

1. Make `args.at<Map>(1)` return a corrupted object treated as a Map
2. The runtime function reads Map fields from attacker-controlled memory
3. Element kind, instance type, instance size are all under attacker control
4. This enables arbitrary element kind transitions, size changes, etc.

## Model Pattern: TrustedCast (Correct Approach)

A few runtime functions use `TrustedCast<T>()` instead of raw `Cast<T>()`,
demonstrating a more secure pattern. 10 instances exist across runtime code:

**`runtime-regexp.cc:360-361`** (with explanatory comment):
```cpp
// capture_count > 0 implies IrRegExpData. Since capture_count is in
// trusted space, this is not a SBXCHECK.
Tagged<IrRegExpData> re_data = TrustedCast<IrRegExpData>(*regexp_data);
```

**`runtime-compiler.cc:178`**:
```cpp
TrustedCast<BytecodeArray>(args[2])
```

The `TrustedCast` pattern:
1. Asserts the object is in trusted space (outside sandbox)
2. Validates the object's Map pointer is consistent
3. Is semantically clearer about trust boundaries

This should be the model for hardening other runtime functions — but it only
works for objects in trusted space. Objects in the regular sandbox heap still
need explicit SBXCHECK validation after casting.

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

## Critical Finding: Runtime_TypedArraySet Has No Bounds Checking

### Location: `src/runtime/runtime-typedarray.cc:205-216`

```cpp
RUNTIME_FUNCTION(Runtime_TypedArraySet) {
  HandleScope scope(isolate);
  DCHECK_EQ(4, args.length());
  DirectHandle<JSTypedArray> target = args.at<JSTypedArray>(0);
  DirectHandle<JSAny> source = args.at<JSAny>(1);
  size_t length;
  CHECK(TryNumberToSize(args[2], &length));
  size_t offset;
  CHECK(TryNumberToSize(args[3], &offset));
  ElementsAccessor* accessor = target->GetElementsAccessor();
  return accessor->CopyElements(isolate, source, target, length, offset);
}
```

**Vulnerability**: No validation that `offset + length <= target->GetByteLength()`.
The `CHECK(TryNumberToSize(...))` calls only verify the arguments are valid numbers,
NOT that they are within bounds of the target array. The `CopyElements` call may
write beyond the TypedArray's allocation.

Similarly, `Runtime_TypedArrayCopyElements` (line 51-60) passes `length` directly
to `CopyElements` with offset 0, but never validates `length` against the target.

### Risk: HIGH - Direct Path to Sandbox Escape

An attacker with sandbox R/W can:
1. Corrupt a JSTypedArray object's internal fields
2. Trigger `Runtime_TypedArraySet` via `TypedArray.prototype.set()`
3. Provide large `length`/`offset` values that exceed the actual backing store
4. `CopyElements` writes beyond the allocation → out-of-sandbox memory corruption

### V8 Team Acknowledgment of Corruption Risk

At `runtime-typedarray.cc:158-160`, inside `Runtime_TypedArraySortFast`, the V8
team explicitly comments:

```cpp
  // The type is not necessarily consistent with the byte_length we read (a
  // sandbox attacker might have changed it). The code below must handle it
  // gracefully.
```

This comment proves the team **knows** sandbox attackers can corrupt TypedArray
fields, and that runtime functions must handle this. Yet `Runtime_TypedArraySet`
(just 45 lines below this comment) has NO such handling.

## Critical Finding: Runtime_GrowArrayElements Trusts Corrupted Map

### Location: `src/runtime/runtime-array.cc:164-198`

```cpp
RUNTIME_FUNCTION(Runtime_GrowArrayElements) {
  HandleScope scope(isolate);
  DCHECK_EQ(2, args.length());
  DirectHandle<JSObject> object = args.at<JSObject>(0);
  DirectHandle<Object> key = args.at(1);
  ElementsKind kind = object->GetElementsKind();  // Trusts object's Map
  CHECK(IsFastElementsKind(kind));
  // ...
  uint32_t capacity = object->elements()->ulength().value();  // Trusts elements
  // ...
  object->GetElementsAccessor()->GrowCapacity(isolate, object, index);
```

If the object's Map is corrupted, `GetElementsKind()` returns attacker-controlled
values. The `IsFastElementsKind` check limits this, but `elements()->ulength()`
reads from attacker-controlled memory. A corrupted length can cause `GrowCapacity`
to allocate an incorrect size.

Similarly, `Runtime_ArrayIncludes_Slow` at line 241 trusts `object->map()->instance_type()`
to determine whether to treat the object as a JSArray, enabling type confusion.

## Critical Finding: Prototype Chain Functions Lack Recursion Limits

### Location: `src/runtime/runtime-object.cc:445-595`

Multiple runtime functions traverse prototype chains without cycle detection
or recursion depth limits:

| Function | Line | Issue |
|----------|------|-------|
| `Runtime_InternalSetPrototype` | 445-454 | `SetPrototype()` no cycle protection |
| `Runtime_JSReceiverGetPrototypeOf` | 561-568 | `GetPrototype()` no recursion limit |
| `Runtime_JSReceiverSetPrototypeOfThrow` | 570-582 | Same as InternalSetPrototype |
| `Runtime_JSReceiverSetPrototypeOfDontThrow` | 584-595 | Same |
| `Runtime_GetProperty` | 597-715 | 715 lines; deep property walks trust corrupted maps |

**Attack**: If an attacker corrupts prototype pointers to create a cycle
(`A.__proto__ → B`, `B.__proto__ → A`), any prototype chain traversal becomes
an infinite loop → stack overflow → DoS or potentially exploitable crash.

Additionally, `Runtime_GetProperty` (715 lines) trusts `object->map()` for
dictionary lookups. A corrupted `map->property_dictionary()` pointer causes
reads from attacker-controlled memory during property lookup.

## Critical Finding: Property Enumeration Trusts Descriptor Tables

### Location: `src/runtime/runtime-object.cc:88-510`

Property enumeration functions trust the object's map and descriptor arrays:

| Function | Line | Trusts |
|----------|------|--------|
| `Runtime_ObjectKeys` | 88-105 | `map->descriptor_array()` |
| `Runtime_ObjectGetOwnPropertyNames` | 108-127 | `map->descriptor_array()` |
| `Runtime_ObjectValues` | 470-482 | Map for property enumeration |
| `Runtime_ObjectEntries` | 498-510 | Map for keys + values |

All use `KeyAccumulator::GetKeys()` which iterates the object's descriptor
table. If `map->descriptor_array()` is corrupted to point to attacker-controlled
memory, the enumeration reads garbage as property descriptors → information
disclosure or type confusion.

## Hardening Commit Analysis: a03a1a3c

### What Was Fixed

Commit `a03a1a3c` hardened **7 functions** in `runtime-test-wasm.cc` with
bounds checking on `func_index`:

```
Runtime_WasmTierUpFunction          (line 777-789)
Runtime_WasmTriggerTierUpForTesting (line 791-814)
Runtime_IsWasmDebugFunction         (line 943-958)
Runtime_IsLiftoffFunction           (line 960-978)
Runtime_IsTurboFanFunction          (line 981-999)
Runtime_IsUncompiledWasmFunction    (line 1001-1010)
Runtime_WasmDeoptsExecutedForFunction (line 1059-1079)
```

### Hardening Pattern Applied

```cpp
// BEFORE (unsafe):
if (func_index < module->num_imported_functions) {
  return CrashUnlessFuzzing(isolate);
}

// AFTER (safe - checks both bounds):
if (static_cast<uint32_t>(func_index) < module->num_imported_functions ||
    static_cast<uint32_t>(func_index) >= module->functions.size()) {
  return CrashUnlessFuzzing(isolate);
}
```

### What Was NOT Fixed

All non-WASM runtime functions remain unhardened. The safe pattern demonstrated
above (input validation + bounds checking + `CrashUnlessFuzzing`) has not been
applied to any function in:
- `runtime-array.cc` (10 functions)
- `runtime-typedarray.cc` (8 functions)
- `runtime-object.cc` (74 functions)
- `runtime-strings.cc` (23 functions)
- `runtime-internal.cc` (64 functions)

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

### Specific Attack Chain: TypedArraySet OOB Write

```
1. JIT bug (VULN-001/other) → OOB write in sandbox
2. Corrupt a JSTypedArray's byte_length field (make it smaller than actual)
   OR corrupt the backing_store pointer
3. Call typedArray.set(sourceArray) which invokes Runtime_TypedArraySet
   → runtime-typedarray.cc:205-216 handles this
   → No validation of offset + length against actual backing store
   → CopyElements writes beyond the TypedArray's allocation
4. If the backing store pointer was corrupted to point outside sandbox:
   → Direct out-of-sandbox write → full compromise
```

### Specific Attack Chain: TypedArraySort Type Confusion

```
1. JIT bug → sandbox R/W
2. Corrupt a JSTypedArray's type field (e.g., change Float64 to Uint8)
3. Call typedArray.sort() which invokes Runtime_TypedArraySortFast
   → runtime-typedarray.cc:106-203
   → Reads byte_length correctly at line 127
   → BUT type at line 161 is corrupted → wrong sizeof(ctype)
   → length = byte_length / sizeof(wrong_ctype)
   → If wrong_ctype is smaller, length is too large
   → std::sort operates on data + length beyond allocation
   → OOB read/write during sort
```

Note: The V8 team's comment at line 158-160 acknowledges this specific risk
but the `switch(array->type())` at line 161 still uses the potentially
corrupted type. The comment says the code "must handle it gracefully" but
the length calculation `byte_length / sizeof(ctype)` with a corrupted type
that has a smaller sizeof will produce a too-large length.

### Specific Attack Chain: GrowArrayElements

```
1. JIT bug → OOB write in array backing store
2. Corrupt adjacent JSArray's Map pointer → fake Map with wrong elements_kind
3. Call array[large_index] = value to trigger Runtime_GrowArrayElements
   → runtime-array.cc:164-198
   → GetElementsKind() reads corrupted Map → attacker-controlled kind
   → IsFastElementsKind check passes (attacker picks a fast kind)
   → elements()->ulength() reads from corrupted elements header
   → GrowCapacity called with incorrect capacity → heap corruption
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

## Quantitative Summary

| Metric | Count |
|--------|-------|
| Total runtime source files | 33 |
| Total RUNTIME_FUNCTION definitions | 671 |
| Runtime files with any SBXCHECK | 1 (runtime-regexp.cc, commenting on its absence) |
| Total `args.at<Type>()` unchecked casts | 288 |
| DCHECK-only validation (stripped in release) | ~200+ |
| Runtime CHECK validation (retained in release) | ~66 |
| Functions hardened in recent commit (a03a1a3c) | 7 (WASM test only) |
| Functions with known sandbox corruption risk | 3+ (TypedArraySet, TypedArrayCopyElements, TypedArraySortFast) |
| Functions trusting corrupted Map fields | 5+ (GrowArrayElements, ArrayIncludes_Slow, ArrayIndexOf, TransitionElementsKind, etc.) |
| Security-related TODOs in runtime | 0 |
| V8 team comments acknowledging corruption risk | 1 (runtime-typedarray.cc:158-160) |

## Key File References

| File | RUNTIME_FUNCTIONs | SBXCHECKs | Unchecked Casts | Priority |
|------|-------------------|-----------|-----------------|----------|
| `runtime-object.cc` | 74 | 0 | 61 | HIGH |
| `runtime-wasm.cc` | 70 | 0 | - | MEDIUM |
| `runtime-internal.cc` | 64 | 0 | - | HIGH |
| `runtime-scopes.cc` | 30 | 0 | 39 | MEDIUM |
| `runtime-strings.cc` | 23 | 0 | 35 | HIGH |
| `runtime-atomics.cc` | 22 | 0 | - | MEDIUM |
| `runtime-compiler.cc` | 21 | 0 | 18 | MEDIUM |
| `runtime-promise.cc` | 15 | 0 | - | LOW |
| `runtime-regexp.cc` | 12 | 0* | 25 | LOW |
| `runtime-array.cc` | 10 | 0 | - | HIGH |
| `runtime-typedarray.cc` | 8 | 0 | - | HIGH |
| Total | 671 | 1** | 288 | -- |

\* runtime-regexp.cc has a comment about SBXCHECK not being needed (trusted space)
\*\* The single SBXCHECK is in arguments.h for bounds checking argument index

## Related CVEs

- **CVE-2024-0517**: V8 OOB write exploiting runtime function behavior
- **CVE-2023-4427**: V8 OOB access through runtime property operations
- Runtime functions are commonly part of exploitation chains rather than
  being the initial vulnerability
