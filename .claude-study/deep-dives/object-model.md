# Deep Dive: V8 Object Model & Hidden Classes

## Overview

V8's object model is the foundation that makes JavaScript fast. Every JavaScript value — numbers, strings, objects, arrays, functions — is represented internally as a **tagged pointer**. Objects use **hidden classes** (called **Maps**) to track their property layout, enabling inline caches and optimizing compilers to generate fast property access code.

Key files: `src/objects/heap-object.h`, `src/objects/map.h`, `src/objects/js-objects.h`, `src/objects/tagged.h`, `include/v8-internal.h`

---

## 1. Tagged Pointer Representation (Smi vs. HeapObject)

V8 uses a **two-tag system** encoded in the low bits of every pointer-sized value:

| Tag Value | Meaning | Tag Size |
|-----------|---------|----------|
| `kSmiTag = 0` | Small Integer (Smi) | 1 bit |
| `kHeapObjectTag = 1` | Strong heap object pointer | 2 bits |
| `kWeakHeapObjectTag = 3` | Weak heap object pointer | 2 bits |

**Source:** `include/v8-internal.h`

```cpp
const int kSmiTag = 0;
const int kHeapObjectTag = 1;
const int kWeakHeapObjectTag = 3;
const int kHeapObjectTagSize = 2;
const intptr_t kSmiTagMask = (1 << kSmiTagSize) - 1;
const intptr_t kHeapObjectTagMask = (1 << kHeapObjectTagSize) - 1;
```

**Smi Layout (64-bit):** `[32-bit signed integer][31 zero bits][0]`
- The integer value is shifted left by `kSmiTagSize + kSmiShiftSize`
- Range: approximately -2^30 to 2^30 - 1
- No heap allocation needed — Smis are "free"

**HeapObject Pointer:** `[aligned address][01]`
- Actual pointer value obtained by masking off low 2 bits
- All heap objects are at least word-aligned, so low bits are available for tagging

The `Tagged<T>` template (`src/objects/tagged.h`) provides type-safe access to tagged values:

```cpp
Tagged<Smi> smi_value;
Tagged<HeapObject> heap_obj;
Tagged<Object> any_value;  // Could be either
```

---

## 2. Map (Hidden Class) Structure and Layout

Every `HeapObject`'s first field is a pointer to its **Map**. The Map is V8's hidden class — it describes the shape (property layout) of the object.

**Source:** `src/objects/map.h`

### Map Memory Layout

```
Map Object Layout:
┌──────────────────────────────────────────────────────┐
│ [offset  0] map_ (pointer to MetaMap)                │  ← Every object starts with a map
├──────────────────────────────────────────────────────┤
│ [offset  8] instance_size (bytes)                    │  ← Total object size
│            inobject_properties_start_or_ctor_idx     │
│            used_or_unused_instance_size_in_words      │
│            visitor_id                                 │
├──────────────────────────────────────────────────────┤
│ [offset 12] instance_type (16-bit InstanceType enum) │  ← What kind of object
│            bit_field (8 bits)                         │  ← is_callable, is_constructor, etc.
│            bit_field2 (8 bits)                        │  ← elements_kind (6 bits)
├──────────────────────────────────────────────────────┤
│ [offset 16] bit_field3 (32 bits)                     │
│   - enum_length (10 bits)                            │
│   - number_of_own_descriptors (10 bits)              │
│   - is_prototype_map (1 bit)                         │
│   - is_dictionary_map (1 bit)                        │
│   - owns_descriptors (1 bit)                         │
│   - is_deprecated (1 bit)                            │
│   - is_extensible (1 bit)                            │
├──────────────────────────────────────────────────────┤
│ [offset 24] prototype (TaggedPointer)                │
│ [offset 32] constructor_or_back_pointer              │
│ [offset 40] instance_descriptors (DescriptorArray)   │
│ [offset 48] dependent_code                           │
│ [offset 56] prototype_validity_cell                  │
│ [offset 64] raw_transitions or prototype_info        │
└──────────────────────────────────────────────────────┘
```

### Key Map Fields

- **instance_type**: 16-bit enum identifying the object kind (e.g., `JS_OBJECT_TYPE`, `JS_ARRAY_TYPE`, `MAP_TYPE`)
- **instance_size**: Total size of objects with this Map, in bytes
- **bit_field**: Packed flags — `is_callable`, `is_constructor`, `has_named_interceptor`, etc.
- **bit_field2**: Contains `elements_kind` (6 bits) — how indexed elements are stored
- **bit_field3**: Contains `number_of_own_descriptors`, `is_dictionary_map`, `is_deprecated`
- **instance_descriptors**: Pointer to `DescriptorArray` with property metadata
- **raw_transitions**: Links to Maps created by adding properties

---

## 3. Map Transitions and Transition Trees

When a property is added to an object, its Map transitions to a new Map. These transitions form a **tree**:

```
Initial Map (no properties)
    │
    ├─ add "x" ──→ Map1 {x}
    │                 │
    │                 ├─ add "y" ──→ Map2 {x, y}
    │                 │                  │
    │                 │                  └─ add "z" ──→ Map3 {x, y, z}
    │                 │
    │                 └─ add "z" ──→ Map2' {x, z}
    │
    └─ add "y" ──→ Map1' {y}
```

**Source:** `src/objects/transitions.h`

Objects created the same way (same constructor, same property additions in same order) share Maps. This is critical for performance — inline caches and optimizing compilers use Map identity checks as a fast way to verify object layout.

**TransitionsAccessor** manages transitions through:
- **Single weak reference** for simple cases (one transition)
- **TransitionArray** for multiple transitions from one Map
- Property names as keys, target Maps as values
- **Weak references** allow GC to clear dead transitions

Special transitions exist for:
- ElementsKind changes (e.g., `PACKED_SMI_ELEMENTS` → `PACKED_DOUBLE_ELEMENTS`)
- Prototype changes
- Integrity level changes (`Object.freeze()`, `Object.seal()`)

---

## 4. Property Storage: In-Object vs. PropertyArray vs. Dictionary

V8 has three property storage strategies:

### Fast Mode (In-Object Properties)

For objects with a stable shape (most objects):
- First N properties stored **directly in the object body** (in-object slots)
- Additional properties overflow to a **PropertyArray** (out-of-object)
- Property metadata in the Map's DescriptorArray

```
JSObject Layout (fast mode):
┌──────────────────────────────┐
│ map_ (pointer to Map)        │  ← 8 bytes
│ properties_or_hash           │  ← PropertyArray or hash
│ elements                     │  ← FixedArray for indexed elements
│ [in-object property 0]       │  ← Direct field access!
│ [in-object property 1]       │
│ [in-object property 2]       │
│ ...                          │
└──────────────────────────────┘
```

### PropertyArray (Out-of-Object Overflow)

**Source:** `src/objects/property-array.h`

When in-object slots are exhausted:
```cpp
class PropertyArray : public HeapObject {
  // [length_and_hash]: 10-bit length + hash code
  // [0..length): Property values
  static constexpr int SizeFor(int length) {
    return kHeaderSize + length * kTaggedSize;
  }
  static const int kMaxLength = (1 << 10) - 1;  // 1023 properties max
};
```

### Dictionary Mode (Slow Properties)

For highly dynamic objects (many property additions/deletions, computed property names):
- Properties stored in a **NameDictionary** (hash table)
- `is_dictionary_map` flag set in Map's bit_field3
- No inline cache optimization possible — falls back to hash lookup
- Triggered by: too many properties, `delete` operator, many non-uniform shapes

---

## 5. Elements Kinds and Array Backing Stores

Arrays use **ElementsKind** to optimize storage based on content type.

**Source:** `src/objects/elements-kind.h`

### Fast Element Kinds

```
PACKED_SMI_ELEMENTS        → Dense array of Smis (most optimized)
HOLEY_SMI_ELEMENTS         → Sparse array of Smis
PACKED_DOUBLE_ELEMENTS     → Dense array of unboxed doubles
HOLEY_DOUBLE_ELEMENTS      → Sparse array of doubles
PACKED_ELEMENTS            → Dense array of any tagged values
HOLEY_ELEMENTS             → Sparse array of any tagged values
```

### Transition Lattice

Elements kinds transition in one direction (can only get "weaker"):

```
PACKED_SMI → HOLEY_SMI
    ↓             ↓
PACKED_DOUBLE → HOLEY_DOUBLE
    ↓             ↓
PACKED_ELEMENTS → HOLEY_ELEMENTS
```

**PACKED_SMI** is the best case: elements are 31-bit integers stored without boxing. **HOLEY_ELEMENTS** is the worst fast case: elements can be any value and may have holes.

### Special Element Kinds

- `DICTIONARY_ELEMENTS`: Hash table for very sparse arrays
- `UINT8_ELEMENTS`, `INT32_ELEMENTS`, etc.: TypedArray element kinds
- `FAST_SLOPPY_ARGUMENTS_ELEMENTS`: For `arguments` objects
- `FAST_STRING_WRAPPER_ELEMENTS`: For String wrapper objects

The elements kind is stored in `Map.bit_field2` (6 bits, allowing 64 different kinds).

---

## 6. String Representations

V8 uses multiple string representations optimized for different usage patterns.

**Source:** `src/objects/string.h`

### String Type Hierarchy

| Type | Storage | When Used |
|------|---------|-----------|
| **SeqOneByteString** | Inline Latin-1 characters | Most ASCII strings |
| **SeqTwoByteString** | Inline UTF-16 characters | Strings with non-Latin-1 chars |
| **ConsString** | Left + Right pointers (rope) | String concatenation (`a + b`) |
| **SlicedString** | Parent + offset + length | `string.substring()` |
| **ThinString** | Forwarding pointer | After internalization |
| **ExternalString** | External data pointer | Embedder-provided strings |

### StringShape Classification

```cpp
class StringShape {
  inline bool IsSequential() const;   // SeqOneByteString or SeqTwoByteString
  inline bool IsExternal() const;     // ExternalString
  inline bool IsCons() const;         // ConsString (rope)
  inline bool IsSliced() const;       // SlicedString
  inline bool IsThin() const;         // ThinString
  inline bool IsDirect() const;       // Seq or External (data accessible)
  inline bool IsIndirect() const;     // Cons, Sliced, or Thin (needs resolution)
  inline bool IsOneByte() const;      // 8-bit encoding
  inline bool IsInternalized() const; // Unique canonical copy
};
```

### String Encoding in InstanceType

The low bits of the instance type encode string properties:

```
Bits 0-2: Representation (Seq=0, Cons=1, External=2, Sliced=3, Thin=5)
Bit 3:    Encoding (TwoByte=0, OneByte=1)
Bit 5:    Internalization (Internalized=0, NotInternalized=1)
```

**Internalized strings** guarantee identity: two internalized strings with the same content are the same object. This makes property name comparison a pointer comparison.

---

## 7. DescriptorArray and Property Metadata

**Source:** `src/objects/descriptor-array.h`

The DescriptorArray stores metadata about an object's named properties:

```
DescriptorArray Layout:
┌──────────────────────────────────────────┐
│ EnumCache → ["x", "y", "z"]              │
├──────────────────────────────────────────┤
│ Descriptor 0:                            │
│   Key: "x" (InternalizedString)          │
│   Details: PropertyDetails (packed Smi)  │
│   Value: (unused for fields)             │
├──────────────────────────────────────────┤
│ Descriptor 1:                            │
│   Key: "y"                               │
│   Details: PropertyDetails               │
│   Value: (unused for fields)             │
├──────────────────────────────────────────┤
│ ...                                      │
└──────────────────────────────────────────┘
```

Each descriptor has 3 slots (`kEntrySize = 3`): key, details, value.

### PropertyDetails

**Source:** `src/objects/property-details.h`

All property metadata packed into a single Smi:

```cpp
class PropertyDetails {
  using KindField = base::BitField<PropertyKind, 0, 1>;          // Data vs Accessor
  using ConstnessField = KindField::Next<PropertyConstness, 1>;  // Mutable vs Const
  using AttributesField = ConstnessField::Next<PropertyAttributes, 3>; // ENUM, CONFIG, RO

  // For fast mode:
  using LocationField = AttributesField::Next<PropertyLocation, 1>;     // Field vs Descriptor
  using RepresentationField = LocationField::Next<uint32_t, 3>;         // Smi/Double/Object
  using FieldIndexField = ...;  // Offset within the object
};
```

**Representation** tracks the observed type of a property's value:
- `kSmi`: Always a small integer
- `kDouble`: Always a double (stored unboxed in a MutableHeapNumber)
- `kHeapObject`: Always a heap object
- `kTagged`: Any tagged value (most general)

This enables the compiler to generate type-specialized property access code.

---

## 8. Prototype Chain and Property Lookup

Property lookup follows the prototype chain:

```
obj.property
  → Check obj's Map descriptors for "property"
  → If not found, follow Map.prototype
  → Check prototype's Map descriptors
  → Continue up the chain until null prototype
```

**Fast path** (monomorphic IC): Check Map identity, load from known offset.
**Slow path**: Walk the prototype chain, checking each object.

The prototype chain is cached in inline caches. When the prototype changes (rare), all dependent code is deoptimized via the **dependent_code** field in Map.

---

## 9. Object Creation and Map Sharing

When `new Constructor()` is called:

1. V8 looks up the constructor's **initial Map** (cached on the JSFunction)
2. Allocates an object with size specified by the initial Map
3. Initializes in-object properties to `undefined`
4. Runs the constructor body, which adds properties → transitions the Map

Objects created by the same constructor with the same property additions share the same Map chain. This is why consistent object initialization patterns are fast:

```javascript
// Good: all objects share Maps
function Point(x, y) { this.x = x; this.y = y; }

// Bad: different property order → different Map chains
function make(flag) {
  let o = {};
  if (flag) { o.a = 1; o.b = 2; }
  else      { o.b = 2; o.a = 1; }  // Different Map!
  return o;
}
```

---

## 10. Map Deprecation and Migration

When a property's representation changes (e.g., a field transitions from Smi to Double), V8 **deprecates** the old Map and creates a new one:

1. Old Map marked with `is_deprecated` in bit_field3
2. New Map created with updated representation
3. Existing objects with the deprecated Map are **lazily migrated** when accessed
4. The transition tree is updated to point to the new Map

This avoids scanning all objects but ensures all future accesses use the correct layout.

**Source:** Map deprecation is checked in IC handlers. When an IC encounters a deprecated Map, it migrates the object and updates the IC.

---

## Memory Layout Example

For `const obj = { x: 1, y: "hello", z: 3.14 }`:

```
JSObject at 0x1000 (tagged: 0x1001):
┌────────────────────────────────────┐
│ [+0]  Map* ──────────────────────→ Map describing {x, y, z}
│ [+8]  properties_or_hash: empty    │
│ [+16] elements: empty FixedArray   │
│ [+24] x: Smi(1) = 0x200           │  ← in-object, Smi representation
│ [+32] y: String* ────────────────→ "hello" (SeqOneByteString)
│ [+40] z: MutableHeapNumber* ─────→ 3.14 (unboxed double)
└────────────────────────────────────┘
```

The Map's DescriptorArray records:
- `x` at field index 0, representation Smi
- `y` at field index 1, representation HeapObject
- `z` at field index 2, representation Double

An optimizing compiler seeing this Map can emit:
- `x`: load Smi directly from offset +24
- `y`: load tagged pointer from offset +32
- `z`: load unboxed double from the MutableHeapNumber at offset +40

---

## Key Performance Insights

1. **Map identity = shape identity**: A single pointer comparison tells you the entire property layout
2. **In-object properties**: No indirection for the first few properties
3. **Representation tracking**: Avoid boxing/unboxing overhead
4. **Transition trees**: Objects created the same way share Maps, enabling monomorphic ICs
5. **Internalized strings**: Property name comparison is pointer equality
6. **Elements kinds**: Arrays with uniform types get specialized fast paths
7. **ConsString**: String concatenation is O(1) (lazy evaluation)
