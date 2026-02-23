# V8 JavaScript Engine — Study Index

This directory contains a systematic study of the [V8 JavaScript Engine](https://v8.dev/) codebase, generated following the methodology in `CLAUDE.md` and `prompts.md` at the repository root.

---

## Architecture Documents

| Document | Description |
|----------|-------------|
| [architecture/overview.md](architecture/overview.md) | High-level system architecture, compilation tiers, key subsystems |
| [architecture/module-map.md](architecture/module-map.md) | Complete directory-to-responsibility mapping for all of `src/`, `include/`, `test/`, `tools/` |
| [architecture/data-flow.md](architecture/data-flow.md) | How data flows through V8: parsing → compilation → execution, property access, GC, snapshots |
| [architecture/threading-model.md](architecture/threading-model.md) | Concurrency design: main thread, background compilation, concurrent GC, synchronization |
| [architecture/memory-model.md](architecture/memory-model.md) | Memory management: pointer compression, handles, zones, heap spaces, write barriers |

## Deep Dives

### Compilation Pipeline

| Document | Description |
|----------|-------------|
| [deep-dives/parsing.md](deep-dives/parsing.md) | Scanner, recursive-descent parser, lazy parsing, scope analysis, AST, source positions |
| [deep-dives/ignition-bytecode.md](deep-dives/ignition-bytecode.md) | Bytecode instruction set (244 opcodes), accumulator model, dispatch, feedback collection |
| [deep-dives/baseline-compiler.md](deep-dives/baseline-compiler.md) | Tier-1 non-optimizing JIT: direct bytecode-to-code, batch compilation, tier-up |
| [deep-dives/maglev-compiler.md](deep-dives/maglev-compiler.md) | Tier-2 mid-tier optimizer: graph IR, type specialization, concurrent compilation |
| [deep-dives/turbofan-turboshaft.md](deep-dives/turbofan-turboshaft.md) | Tier-3 full optimizer: sea-of-nodes, optimization passes, Turboshaft migration |

### Runtime Systems

| Document | Description |
|----------|-------------|
| [deep-dives/object-model.md](deep-dives/object-model.md) | Tagged pointers, Maps (hidden classes), transitions, property storage, strings, elements kinds |
| [deep-dives/heap-and-gc.md](deep-dives/heap-and-gc.md) | Generational GC: Scavenger, Mark-Compact, concurrent marking, write barriers, cppgc |
| [deep-dives/inline-caches.md](deep-dives/inline-caches.md) | IC architecture, monomorphic/polymorphic states, FeedbackVector, handler compilation |
| [deep-dives/runtime-builtins.md](deep-dives/runtime-builtins.md) | Torque DSL, Array/String/Promise builtins, runtime functions, bootstrapping |

### Specialized Subsystems

| Document | Description |
|----------|-------------|
| [deep-dives/webassembly.md](deep-dives/webassembly.md) | WASM module decoding, Liftoff baseline, TurboFan WASM, JS interop, SIMD |
| [deep-dives/regexp.md](deep-dives/regexp.md) | Irregexp engine: parsing, bytecode, JIT, Boyer-Moore, backtracking, linear-time fallback |
| [deep-dives/security-sandbox.md](deep-dives/security-sandbox.md) | Sandbox architecture, pointer compression, external pointer table, Spectre mitigations |
| [deep-dives/debugging-devtools.md](deep-dives/debugging-devtools.md) | Chrome DevTools Protocol, breakpoints, profiling, code coverage, d8 shell |
| [deep-dives/snapshots-serialization.md](deep-dives/snapshots-serialization.md) | mksnapshot, heap serialization, startup snapshots, read-only space |
| [deep-dives/platform-concurrency.md](deep-dives/platform-concurrency.md) | Platform API, thread pools, atomics, task scheduling, isolate groups |

## Reference Documents

| Document | Description |
|----------|-------------|
| [glossary.md](glossary.md) | V8-specific terminology: Isolate, Map, Smi, Handle, Zone, IC, Torque, etc. |
| [build-and-test.md](build-and-test.md) | How to build V8, run tests, use d8, configure build flags |
| [master-topic-list.md](master-topic-list.md) | 15 major topic areas × 10 sub-items = 150 investigation items |

---

## How to Use This Study

1. **Start with** `architecture/overview.md` for a high-level understanding
2. **Reference** `glossary.md` when encountering unfamiliar terms
3. **Navigate** `architecture/module-map.md` to find which directory owns which functionality
4. **Deep dive** into specific subsystems using the deep-dive documents
5. **Follow data flow** through `architecture/data-flow.md` to understand how subsystems connect

## Methodology

This study was generated following the CLAUDE.md workflow:
1. **Initial Repository Survey** — top-level structure, build system, entry points
2. **Module Mapping** — every directory mapped to its responsibility
3. **Deep Dives** — systematic investigation of each major subsystem
4. **Cross-referencing** — all reports reference actual file paths and code

All artifacts reference actual V8 source files. File paths are relative to the repository root.
