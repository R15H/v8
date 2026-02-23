# Snapshots and Serialization in V8

## Overview

V8's snapshot system is a critical performance optimization that eliminates the
need to parse and compile built-in JavaScript code on every startup. Instead,
the heap state -- including compiled builtins, built-in objects, and the
read-only heap -- is serialized to a binary blob at build time using the
`mksnapshot` tool, then deserialized into memory at runtime in milliseconds.

The snapshot subsystem lives in `src/snapshot/` and comprises serializers,
deserializers, the `mksnapshot` build tool, and supporting infrastructure for
code caching.

## Architecture

```
Build Time:                              Runtime:

mksnapshot tool                          V8 Startup
  |                                        |
  v                                        v
Isolate + builtins                       Snapshot::Initialize()
  |                                        |
  v                                        v
Serializers:                             Deserializers:
  ReadOnlySerializer                       ReadOnlyDeserializer
  SharedHeapSerializer                     SharedHeapDeserializer
  StartupSerializer                        StartupDeserializer
  ContextSerializer                        ContextDeserializer
  |                                        |
  v                                        v
snapshot_blob.bin                        Live Isolate ready
(or embedded C++ array)
```

## The `mksnapshot` Tool

The entry point is `src/snapshot/mksnapshot.cc`, a standalone binary run during
V8's build process. Its `main()` function:

1. Initializes V8 with `--predictable` flag for reproducible output
2. Disables ICs globally (`--use-ic=false`) to avoid Code handler issues
3. Creates an `Isolate` and a `SnapshotCreator`
4. Optionally loads an "embedding" script and a "warmup" script
5. Calls `CreateSnapshotDataBlobInternal()` to serialize the heap
6. Writes the embedded builtins via `EmbeddedFileWriter`
7. Optionally generates static roots tables
8. Outputs the snapshot blob as either a binary file or a C++ source array

```cpp
// From mksnapshot.cc main():
v8::SnapshotCreator creator(isolate, create_params);
blob = CreateSnapshotDataBlob(creator, embed_script.get());
WriteEmbeddedFile(&embedded_writer);
snapshot_writer.WriteSnapshot(blob);
```

The snapshot blob can be output in two forms:
- **Binary blob** (`--startup-blob`): a raw binary file loaded at runtime
- **C++ source** (`--startup-src`): an auto-generated `.cc` file with the blob
  as a `static const uint8_t[]` array, linked directly into the binary

### Warm-Up Snapshots

If a warmup script is provided, `mksnapshot` creates a "cold" snapshot first,
then deserializes it into a fresh isolate, runs the warmup script, and
re-serializes -- producing a snapshot with pre-initialized state.

## Snapshot Data Format

### `SerializedData` and `SnapshotData`

The base `SerializedData` class (`src/snapshot/snapshot-data.h`) provides:
- A magic number (at offset 0) derived from `0xC0DE0000 ^ ExternalReferenceTable::kSize`
- Little-endian header value accessors
- Memory ownership tracking

`SnapshotData` extends this with a payload:
```
[0] Magic number (uint32)
[4] Payload length (uint32)
[8...] Serialized payload bytes
```

### Blob Structure

The overall snapshot blob contains multiple sub-snapshots:
1. **Read-only snapshot** -- immutable objects shared across isolates
2. **Shared heap snapshot** -- objects in the shared heap
3. **Startup snapshot** -- the main isolate heap (builtins, roots, etc.)
4. **Context snapshot(s)** -- one or more serialized contexts

Each sub-snapshot has its own checksum for integrity verification.

## The Serializer Hierarchy

All serializers inherit from a common base:

```
SerializerDeserializer (common bytecodes and constants)
    |
    Serializer (base serializer with reference tracking)
        |
        +-- RootsSerializer (handles root table serialization)
        |       |
        |       +-- ReadOnlySerializer
        |       +-- SharedHeapSerializer
        |       +-- StartupSerializer
        |       +-- ContextSerializer
        |
        +-- CodeSerializer (for cached compiled code)
```

### `Serializer` Base Class

The `Serializer` class (`src/serializer.h`) provides the core serialization
machinery:

- **`SnapshotByteSink sink_`** -- output buffer for serialized bytes
- **`SerializerReferenceMap reference_map_`** -- maps heap objects to back-references
- **`ExternalReferenceEncoder`** -- encodes external C++ addresses as indices
- **`RootIndexMap`** -- maps root objects to their root table indices
- **`HotObjectsList`** -- circular queue of recently serialized objects for
  compact encoding (8 entries, checked before emitting a full back-reference)

Key serialization strategies:
1. **Root references**: Objects in the root table are serialized as root indices
2. **Back-references**: Previously seen objects get compact back-ref encodings
3. **Hot objects**: The 8 most recently serialized objects get 3-bit indices
4. **Deferred objects**: To avoid deep recursion, some objects are queued and
   serialized later via `SerializeDeferredObjects()`
5. **Forward references**: Objects not yet allocated can be referenced via
   pending forward references, resolved when the target is serialized

```cpp
class Serializer : public SerializerDeserializer {
  HotObjectsList hot_objects_;
  SerializerReferenceMap reference_map_;
  ExternalReferenceEncoder external_reference_encoder_;
  RootIndexMap root_index_map_;
  GlobalHandleVector<HeapObject> deferred_objects_;
  // Forward reference tracking
  int next_forward_ref_id_ = 0;
  int unresolved_forward_refs_ = 0;
  IdentityMap<PendingObjectReferences, ...> forward_refs_per_pending_object_;
};
```

### `Serializer::ObjectSerializer`

The inner `ObjectSerializer` class handles serialization of individual heap
objects. It implements the `ObjectVisitor` interface to walk object fields:

- `VisitPointers` -- serializes tagged pointer fields
- `VisitInstructionStreamPointer` -- handles Code -> InstructionStream references
- `VisitEmbeddedPointer` -- embedded pointers in compiled code
- `VisitExternalReference` -- external C++ function/data pointers
- `VisitExternalPointer` -- sandboxed external pointers
- `VisitIndirectPointer` -- sandbox indirect pointer table entries

Each object is serialized with a prologue specifying its space allocation
and map, followed by raw data interleaved with encoded references.

### `StartupSerializer`

Serializes the main isolate heap in a specific order:

1. **Strong roots** (root table entries)
2. **Builtins and bytecode handlers**
3. **Startup object cache** (objects needed during startup)
4. **Weak references** (string table, etc.)

Uses `SharedHeapSerializer` to delegate shared-heap objects.

### `ReadOnlySerializer`

Serializes the entire `ReadOnlySpace` using a memcpy-style approach.
The read-only space contains immutable objects (maps, oddball values, empty
collections, etc.) that are identical across all isolates.

### `ContextSerializer`

Serializes individual JavaScript contexts, allowing custom context snapshots
(used by Chrome for extension-specific contexts and by Node.js for
`vm.createContext` snapshots). Supports embedder field serialization callbacks.

## The Deserializer

### `Deserializer<IsolateT>`

The `Deserializer` template class (`src/snapshot/deserializer.h`) reconstructs
the heap from serialized bytes. It is templated on `IsolateT` to support both
main-thread (`Isolate`) and background-thread (`LocalIsolate`) deserialization.

Key internal state:
```cpp
template <typename IsolateT>
class Deserializer : public SerializerDeserializer {
  SnapshotByteSource source_;            // Input byte stream
  HotObjectsList hot_objects_;           // Mirrors serializer's hot list
  std::vector<IndirectHandle<HeapObject>> back_refs_;  // Back-reference table
  DirectHandleVector<Map> new_maps_;
  DirectHandleVector<Script> new_scripts_;
  std::vector<std::shared_ptr<BackingStore>> backing_stores_;
  // Forward reference resolution
  std::vector<UnresolvedForwardRef> unresolved_forward_refs_;
};
```

### Deserialization Bytecodes

The deserializer reads a stream of bytecodes. Each bytecode specifies how to
fill the next slot(s) in the object being deserialized:

| Bytecode Category         | Purpose                                    |
|---------------------------|--------------------------------------------|
| `ReadNewObject`           | Allocate a new object in a given space      |
| `ReadBackref`             | Reference a previously deserialized object  |
| `ReadReadOnlyHeapRef`     | Reference into the read-only heap           |
| `ReadRootArray`           | Reference a root table entry               |
| `ReadStartupObjectCache`  | Reference the startup object cache          |
| `ReadHotObject`           | Reference one of 8 recently seen objects    |
| `ReadExternalReference`   | Decode an external reference by index       |
| `ReadFixedRawData`        | Copy raw bytes directly                    |
| `ReadVariableRepeatRoot`  | Fill N slots with a repeated root           |
| `ReadRegisterPendingForwardRef` | Mark a slot for later resolution      |
| `ReadResolvePendingForwardRef`  | Resolve a previously pending reference |

### Post-Processing

After deserialization, `PostProcessNewObject()` handles:
- Inserting internalized strings into the string table
- Rehashing hash tables if needed (due to random hash seed)
- Logging new code objects and scripts
- Weakening descriptor arrays (deserialized as strong, weakened after)
- Setting up `AccessorInfo`, `InterceptorInfo`, and `FunctionTemplateInfo`

## Code Serialization (Code Caching)

### `CodeSerializer`

The `CodeSerializer` (`src/snapshot/code-serializer.h`) handles V8's code
caching feature, which serializes compiled `SharedFunctionInfo` objects for
later reuse. This is the mechanism behind `ScriptCompiler::CachedData` in
the V8 API.

```cpp
class CodeSerializer : public Serializer {
  static ScriptCompiler::CachedData* Serialize(
      Isolate* isolate, Handle<SharedFunctionInfo> info);
  static MaybeDirectHandle<SharedFunctionInfo> Deserialize(
      Isolate* isolate, AlignedCachedData* cached_data,
      DirectHandle<String> source, const ScriptDetails& script_details);
};
```

### `SerializedCodeData` Format

The cached code data has an extended header:

```
[0]  Magic number
[4]  Version hash
[8]  Source hash
[12] Flag hash
[16] Read-only snapshot checksum
[20] Payload length
[24] Checksum
[28] (padding to pointer alignment)
[...] Serialized payload
```

The multiple hash checks ensure that cached code is only used when:
- The V8 version matches
- The source code matches
- The compilation flags match
- The read-only snapshot is compatible

### Off-Thread Deserialization

Code deserialization supports off-thread execution for better performance:

```cpp
static OffThreadDeserializeData StartDeserializeOffThread(
    LocalIsolate* isolate, AlignedCachedData* cached_data);
static MaybeDirectHandle<SharedFunctionInfo> FinishOffThreadDeserialize(
    Isolate* isolate, OffThreadDeserializeData&& data, ...);
```

This allows the expensive deserialization work to happen on a background
thread, with only the final integration step on the main thread.

## Read-Only Space and Snapshot Sharing

### Shared Read-Only Heap

V8's read-only space contains objects that never change after initialization:
- Maps for built-in object types
- Oddball values (`undefined`, `null`, `true`, `false`)
- Empty arrays, empty strings, etc.
- Root constants

When `V8_SHARED_RO_HEAP` is enabled (the default for many configurations),
a single read-only snapshot is deserialized once and shared across all
isolates in the process. This saves significant memory in multi-isolate
scenarios (like Chrome's site isolation).

The `ReadOnlyPage` class supports sealing pages to enforce immutability:
```cpp
void ReadOnlyPage::MakeHeaderRelocatableAndMarkAsSealed();
```

### Static Roots

When `V8_STATIC_ROOTS_GENERATION_BOOL` is enabled, `mksnapshot` generates
a static roots table (`StaticRootsTableGen::write()`) that assigns fixed
addresses to read-only objects. This enables compile-time constants for
comparing against well-known objects like `undefined` or `null`, eliminating
root-table loads in generated code.

## Snapshot Compression

The `SnapshotCompression` class (`src/snapshot/snapshot-compression.h`)
handles optional compression of snapshot data to reduce binary size.
Compression is applied after serialization and decompression before
deserialization.

## Startup Performance Flow

The complete startup sequence:

1. **`Snapshot::Initialize(isolate)`** -- entry point
2. **Verify checksum** and version compatibility
3. **`ReadOnlyDeserializer`** -- restores the read-only space
   (or reuses shared read-only heap)
4. **`SharedHeapDeserializer`** -- restores shared heap objects
5. **`StartupDeserializer`** -- restores the main heap
   (builtins, string table, root objects)
6. **Post-processing** -- rehash tables, log scripts, weaken descriptors
7. **Isolate is ready** -- JavaScript can now execute

On a typical desktop, this entire process takes 1-5ms, compared to 50-200ms
for bootstrapping from source. The snapshot accounts for the vast majority
of V8's perceived startup speed.

## Key Design Decisions

1. **Deterministic serialization**: `mksnapshot` runs with `--predictable` to
   ensure reproducible builds. ICs are disabled to avoid non-deterministic
   Code objects.

2. **Recursion depth limiting**: The serializer limits recursion to 32 levels
   (`kMaxRecursionDepth`) and defers deeper objects to avoid stack overflow.

3. **Forward references**: Objects may reference each other cyclically.
   The forward-reference mechanism resolves these by emitting placeholders
   and patching them when the referenced object is finally serialized.

4. **GC-free zones**: Both serialization and deserialization run with
   `DISALLOW_GARBAGE_COLLECTION` guards to ensure the heap is stable.

5. **External reference encoding**: All C++ function pointers and external data
   addresses are encoded as indices into the `ExternalReferenceTable`, making
   snapshots relocatable across different process address spaces.
