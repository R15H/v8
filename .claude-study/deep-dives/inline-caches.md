# Inline Caches and Type Feedback

## Overview

Inline Caches (ICs) are V8's mechanism for accelerating property accesses and other polymorphic operations. Instead of performing a full property lookup on every access, ICs cache the result of previous lookups and reuse them when the same object shape (map) is encountered. ICs also serve as the primary feedback collection mechanism, recording type information that drives optimization decisions in Maglev and TurboFan.

Key source directory: `src/ic/`

## IC State Machine

Every IC goes through a progression of states based on the number of distinct object shapes it has observed:

```
UNINITIALIZED -> MONOMORPHIC -> POLYMORPHIC -> MEGAMORPHIC -> GENERIC
```

The states are defined as `InlineCacheState`:

1. **UNINITIALIZED (0)**: No access has been observed yet. The IC stub points to a generic handler that performs a full lookup and records the first observed shape.

2. **MONOMORPHIC (1)**: Exactly one object shape (map) has been seen. The IC directly checks for that map and, on match, executes a fast handler. This is the common case for most property accesses.

3. **POLYMORPHIC (2-4)**: A small number of different maps have been observed. The IC checks each map in sequence (or uses a dispatch mechanism) before falling back.

4. **MEGAMORPHIC**: Too many maps have been observed for efficient polymorphic handling. The IC uses the `StubCache` -- a global hash table -- for lookups.

5. **GENERIC**: The IC has given up on caching and uses a fully generic slow path.

6. **RECOMPUTE_HANDLER**: A transient state indicating the handler needs to be recomputed (e.g., due to a map migration).

The `IC` base class (`src/ic/ic.h`) manages this state machine:

```cpp
class IC {
  using State = InlineCacheState;
  State state() const { return state_; }
  void UpdateState(DirectHandle<Object> lookup_start_object,
                   DirectHandle<Object> name);
  void ConfigureVectorState(DirectHandle<Name> name,
                            DirectHandle<Map> map,
                            DirectHandle<Object> handler);
};
```

## IC Class Hierarchy

The IC system has specialized subclasses for different access patterns:

### LoadIC
Handles named property loads (`obj.prop`). Defined in `src/ic/ic.h`:
```cpp
class LoadIC : public IC {
  MaybeDirectHandle<Object> Load(Handle<JSAny> object, Handle<Name> name, ...);
  void UpdateCaches(LookupIterator* lookup);
  MaybeObjectHandle ComputeHandler(LookupIterator* lookup);
};
```

### LoadGlobalIC
Specialization of LoadIC for global variable loads. Handles both `typeof` and non-`typeof` contexts.

### KeyedLoadIC
Handles keyed property loads (`obj[key]`). Must handle integer indices (element access), string keys, and symbol keys:
```cpp
class KeyedLoadIC : public LoadIC {
  MaybeDirectHandle<Object> Load(Handle<JSAny> object, Handle<Object> key);
  void UpdateLoadElement(DirectHandle<HeapObject> receiver,
                         KeyedAccessLoadMode new_load_mode);
};
```

### StoreIC
Handles named property stores (`obj.prop = value`):
```cpp
class StoreIC : public IC {
  MaybeDirectHandle<Object> Store(Handle<JSAny> object, Handle<Name> name,
                                   DirectHandle<Object> value, ...);
  bool LookupForWrite(LookupIterator* it, DirectHandle<Object> value, ...);
};
```

### KeyedStoreIC
Handles keyed stores (`obj[key] = value`). Tracks elements kind transitions.

### DefineNamedOwnIC / DefineKeyedOwnIC
Handle property definition semantics (class fields, object literals).

## Handler Compilation

IC handlers are small code objects or Smi-encoded descriptors that perform the actual property access. The `LoadHandler` and `StoreHandler` classes (`src/ic/handler-configuration.h`) define the handler encoding.

### LoadHandler Kinds

```cpp
enum class Kind {
  kElement,                  // Array element access
  kElementWithTransition,    // Element access with elements kind transition
  kIndexedString,            // String character access by index
  kNormal,                   // Dictionary-mode property
  kGlobal,                   // Global object property
  kField,                    // Fast in-object or out-of-object field
  kConstantFromPrototype,    // Constant value from prototype chain
  kAccessorFromPrototype,    // Getter/setter from prototype chain
  kNativeDataProperty,       // Native data property (C++ accessor)
  kApiGetter,                // V8 API getter callback
  kInterceptor,              // Named property interceptor
  kSlow,                     // Slow/generic path
  kProxy,                    // Proxy object
  kNonExistent,              // Property does not exist
  kModuleExport,             // Module namespace export
  kGeneric,                  // Generic keyed access
};
```

Handlers for simple field loads are encoded as **Smi handlers** to avoid heap allocation. The Smi packs:
- The handler kind
- Whether to do access checks
- Whether to check the prototype chain
- Field location (in-object vs. out-of-object)
- Field index and representation

More complex handlers (prototype chain walks, accessor calls) use `DataHandler` heap objects that store additional data (validity cells, holder objects).

### AccessorAssembler

The `AccessorAssembler` (`src/ic/accessor-assembler.h`) extends `CodeStubAssembler` to generate IC stub code. It provides generators for every IC variant:

- `GenerateLoadIC()`, `GenerateLoadIC_Megamorphic()`, `GenerateLoadIC_Noninlined()`
- `GenerateStoreIC()`, `GenerateStoreIC_Megamorphic()`
- `GenerateKeyedLoadIC()`, `GenerateKeyedLoadIC_Megamorphic()`
- `GenerateKeyedHasIC()` -- for `in` operator
- `GenerateCloneObjectIC()` -- for object spread/destructuring
- Baseline-specific variants (`GenerateLoadICBaseline()`, etc.)

Each IC entry point:
1. Loads the feedback from the FeedbackVector
2. Checks if the feedback matches the current object's map
3. If match: executes the cached handler (fast path)
4. If miss: falls through to the miss handler, which invokes the C++ IC runtime

## FeedbackVector

The `FeedbackVector` (`src/objects/feedback-vector.h`) is a per-function array of feedback slots. Each slot corresponds to a feedback-collecting operation in the bytecode.

### FeedbackSlotKind

```cpp
enum class FeedbackSlotKind : uint8_t {
  kCall,                         // Function call feedback
  kLoadProperty,                 // Named property load
  kLoadGlobalNotInsideTypeof,    // Global load (throws on undefined)
  kLoadGlobalInsideTypeof,       // Global load (returns undefined)
  kLoadKeyed,                    // Keyed property load
  kHasKeyed,                     // 'in' operator
  kSetNamedSloppy/Strict,        // Named property store
  kSetKeyedSloppy/Strict,        // Keyed property store
  kStoreGlobalSloppy/Strict,     // Global store
  kDefineNamedOwn,               // Class field / object literal define
  kDefineKeyedOwn,               // Computed class field define
  kStoreInArrayLiteral,          // Array literal element store
  kBinaryOp,                     // Binary operation type feedback
  kCompareOp,                    // Comparison type feedback
  kForIn,                        // For-in enumeration feedback
  kInstanceOf,                   // instanceof feedback
  kTypeOf,                       // typeof feedback
  kCloneObject,                  // Object clone feedback
  kJumpLoop,                     // Loop tier-up feedback
};
```

### Feedback Slot Contents

A feedback slot can contain:
- **Weak reference to a Map**: Monomorphic IC state
- **Weak reference to a handler + Map**: Monomorphic with handler
- **WeakFixedArray of (Map, handler) pairs**: Polymorphic IC state
- **Smi marker**: `megamorphic_sentinel`, `uninitialized_sentinel`
- **BinaryOperationFeedback/CompareOperationFeedback**: Smi encoding of observed types

### FeedbackNexus

The `FeedbackNexus` provides a high-level API for reading and writing feedback:
```cpp
class FeedbackNexus {
  InlineCacheState ic_state() const;
  void ExtractMaps(MapHandles* maps) const;
  MaybeObjectHandle FindHandlerForMap(Handle<Map> map) const;
  void ConfigureMonomorphic(Handle<Name> name, Handle<Map> map,
                            const MaybeObjectHandle& handler);
  void ConfigurePolymorphic(...);
  void ConfigureMegamorphic();
};
```

## StubCache

The `StubCache` (`src/ic/stub-cache.h`) is a fixed-size hash table used for megamorphic property lookups:

```cpp
class StubCache {
  struct Entry {
    StrongTaggedValue key;    // Name
    TaggedValue value;        // Handler (weak or strong)
    StrongTaggedValue map;    // Map
  };
  Entry primary_[kPrimaryTableSize];
  Entry secondary_[kSecondaryTableSize];
};
```

The cache uses two-level hashing (primary and secondary tables) keyed by `(name, map)`. It does not need explicit invalidation when prototype chains change because handlers themselves verify the chain via validity cells.

## IC Miss Handling

When an IC encounters an object shape it has not cached:

1. The IC miss handler (a builtin) is invoked
2. It calls into the C++ IC runtime (`IC::Load()`, `IC::Store()`, etc.)
3. A `LookupIterator` performs the full property lookup
4. `ComputeHandler()` creates a new handler for the observed map
5. `UpdateCaches()` installs the handler in the FeedbackVector
6. The IC transitions to its next state (uninitialized->monomorphic, monomorphic->polymorphic, etc.)

## Binary and Comparison Operation Feedback

For arithmetic and comparison operations, feedback is simpler -- it records the observed operand types:

- `BinaryOperationFeedback`: `kSignedSmall`, `kSignedSmallInputs`, `kNumber`, `kNumberOrOddball`, `kString`, `kBigInt`, `kAny`
- `CompareOperationFeedback`: `kSignedSmall`, `kNumber`, `kString`, `kReceiver`, `kSymbol`, `kBigInt`, `kAny`

This feedback is embedded directly in bytecodes (`kEmbeddedFeedback` operand type) for comparisons, or stored in feedback vector slots for arithmetic operations.

## Interaction with Compilers

IC feedback is the bridge between execution and optimization:

1. **Ignition/Sparkplug**: Collect feedback during execution
2. **Maglev**: Reads feedback via `JSHeapBroker` to specialize operations. A monomorphic LoadIC becomes `CheckMaps` + `LoadTaggedField`.
3. **TurboFan**: Uses feedback for call-target specialization, type narrowing, and inlining decisions. `ProcessedFeedback` structures provide a normalized view of the raw feedback data.

If the IC is in megamorphic state, optimizing compilers fall back to generic operations, as the access pattern is too diverse for profitable specialization.

## Key Files

| File | Purpose |
|------|---------|
| `src/ic/ic.h/.cc` | IC base class and LoadIC/StoreIC/KeyedLoadIC/KeyedStoreIC |
| `src/ic/accessor-assembler.h/.cc` | CSA-based IC stub generation |
| `src/ic/handler-configuration.h/.cc` | LoadHandler/StoreHandler encoding |
| `src/ic/stub-cache.h/.cc` | Megamorphic lookup cache |
| `src/ic/binary-op-assembler.h/.cc` | Binary operation IC stubs |
| `src/ic/unary-op-assembler.h/.cc` | Unary operation IC stubs |
| `src/ic/keyed-store-generic.h/.cc` | Generic keyed store implementation |
| `src/ic/call-optimization.h/.cc` | API call optimization |
| `src/objects/feedback-vector.h` | FeedbackVector and FeedbackSlotKind |
| `src/objects/feedback-cell.h` | FeedbackCell (closure-specific feedback) |
