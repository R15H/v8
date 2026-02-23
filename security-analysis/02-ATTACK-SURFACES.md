# V8 Attack Surfaces

## 1. JIT Compiler Type System [CRITICAL]

### Description
The JIT compilers (TurboFan, Turboshaft, Maglev) use type inference to
speculate about runtime values. If the type system computes an incorrect type
for a value, downstream optimizations (especially bounds check elimination)
may generate unsafe code.

This is historically the **#1 source of V8 CVEs**.

### Attack Pattern
```
1. Craft JS that causes the typer to compute an overly narrow type
   (e.g., "this value is always in range [0, 10]" when it can be 1000)
2. The optimizer removes a bounds check based on this narrow type
3. At runtime, the value exceeds the expected range
4. Out-of-bounds memory access occurs
5. Use OOB read/write to corrupt adjacent objects
6. Leverage corrupted objects for arbitrary read/write
7. (With sandbox) Use arbitrary R/W inside sandbox to escape
```

### Key Files
| File | Lines | Role |
|------|-------|------|
| `src/compiler/turbofan-typer.cc` | 2,787 | TurboFan type inference engine |
| `src/compiler/operation-typer.cc` | ~1,500 | Semantic type operations (Add, Multiply, CheckBounds, etc.) |
| `src/compiler/turbofan-types.cc` | 1,310 | Type representation (Range, Bitset, Union) |
| `src/compiler/turbofan-types.h` | ~800 | Type class definitions |
| `src/compiler/turboshaft/typer.h` | 1,625 | Turboshaft type system (**contains confirmed bug**) |
| `src/compiler/typed-optimization.cc` | 1,006 | Bounds check elimination |
| `src/compiler/simplified-lowering.cc` | ~6,000 | Representation selection (3-phase algorithm) |
| `src/compiler/type-cache.h` | ~200 | Cached type constants |

### Specific Vulnerabilities
- **Turboshaft Multiply NaN bug** (`typer.h:620`): See [06-VULNERABILITIES.md](06-VULNERABILITIES.md)
- **`allow_invalid_inputs()` always true** (`typer.h:1619`): Accepts malformed inputs silently

### Verification Strategy
1. Write JS that triggers Turboshaft compilation of `a * b` where
   `a` has range `[1, Infinity]` and `b` is `0`
2. Check if the result type includes NaN using `--trace-turbo`
3. Chain with a comparison/branch that assumes non-NaN to prove
   incorrect branch elimination

---

## 2. Bounds Check Elimination [HIGH]

### Description
After type inference, V8 eliminates redundant bounds checks. If the type
system says an index is always in `[0, length-1]`, the bounds check is
removed. If the type was wrong, the access goes out of bounds.

### Attack Pattern
```
1. Trigger incorrect typing of an array index
2. TypedOptimization::ReduceCheckBounds (typed-optimization.cc:206)
   removes bounds check flags
3. SimplifiedLowering further optimizes the access
4. Runtime access with an index > length = OOB
```

### Key Files
| File | Lines | Function |
|------|-------|----------|
| `src/compiler/typed-optimization.cc` | 206-220 | `ReduceCheckBounds()` |
| `src/compiler/operation-typer.cc` | 1344-1353 | `CheckBounds()` type computation |
| `src/compiler/simplified-lowering.cc` | 63-84 | 3-phase PROPAGATE/RETYPE/LOWER |

### Key Code Path
```
typed-optimization.cc:206-220:
  ReduceCheckBounds() checks if kConvertStringAndMinusZero flag can be removed
  based on type analysis. If input type doesn't include String or MinusZero,
  the conversion flag is removed.

operation-typer.cc:1344-1353:
  CheckBounds(index, length) computes:
    upper_bound = Range(0.0, length.Max() - 1)
    result = Intersect(index, upper_bound)

  DCHECK: length.Is(kPositiveSafeInteger) - protects against precision loss
  but only in debug builds.
```

---

## 3. GC Write Barriers [HIGH]

### Description
When a pointer from old generation to young generation is written, a write
barrier must record this reference so the GC can find it during scavenging.
Missing write barriers lead to dangling pointers and use-after-free.

### Attack Pattern
```
1. Find a code path that writes a pointer without a proper write barrier
2. Trigger garbage collection while the dangling reference exists
3. The young-gen object is moved/freed
4. Access through the old reference → use-after-free
5. Fill freed memory with attacker-controlled data
6. Type confusion when old code accesses the corrupted object
```

### Key Files
| File | Role |
|------|------|
| `src/heap/heap-write-barrier.cc` | Main write barrier implementation |
| `src/heap/heap-write-barrier-inl.h` | Inline fast-path write barriers |
| `src/heap/WRITE_BARRIER.md` | Documentation of write barrier design |
| `src/heap/marking-barrier.cc` | Barriers for concurrent marking |
| `src/codegen/code-stub-assembler.cc` | CSA write barrier emission |

### Barrier Types
- **Generational barrier**: old→young references (Scavenger needs these)
- **Marking barrier**: for concurrent marking (Mark-Compact needs these)
- **Shared barrier**: cross-isolate shared heap references
- **Indirect pointer barrier**: sandbox pointer table updates

---

## 4. Element Kind Transitions [HIGH]

### Description
JavaScript arrays in V8 have an "elements kind" that determines their internal
storage format. Transitions between kinds must preserve type invariants. Bugs
in transitions can lead to type confusion where the engine accesses memory
with the wrong interpretation.

### Attack Pattern
```
1. Create array with PACKED_SMI_ELEMENTS
2. Trigger a transition to PACKED_DOUBLE_ELEMENTS
3. If transition is buggy, engine may still interpret doubles as SMIs
   (or vice versa)
4. A double like 1.1 interpreted as a SMI pointer = arbitrary address
5. Conversely, a pointer interpreted as a double = info leak (fakeobj/addrof)
```

### Key Files
| File | Role |
|------|------|
| `src/objects/elements-kind.h` | Element kind enum (30+ kinds) |
| `src/objects/elements.cc` | Element accessor implementations |
| `src/objects/js-array.h` | JSArray definition |
| `src/objects/js-array-buffer.h` | ArrayBuffer/SharedArrayBuffer |
| `src/objects/js-typed-array.h` | TypedArray definitions |

### RAB/GSAB Complexity
Resizable ArrayBuffers (RAB) and Growable SharedArrayBuffers (GSAB) add
significant complexity. The backing store can be resized from another thread
(for SAB) or the same thread, potentially invalidating cached lengths and
pointers. All RAB/GSAB element kinds have separate implementations:
`RAB_GSAB_UINT8_ELEMENTS`, `RAB_GSAB_FLOAT64_ELEMENTS`, etc.

---

## 5. Sandbox Architecture [MEDIUM]

### Description
The V8 Sandbox isolates V8's heap in a large virtual address region (ideally
1TB) and uses pointer indirection tables to prevent in-sandbox memory
corruption from escaping to out-of-sandbox memory.

Post-exploitation, an attacker with OOB read/write inside the sandbox needs
to escape the sandbox to achieve arbitrary code execution. The sandbox is
V8's second line of defense.

### Attack Pattern (Sandbox Escape)
```
1. Achieve arbitrary R/W inside sandbox (via JIT bug, etc.)
2. Corrupt entries in pointer tables:
   - External Pointer Table → redirect external pointers
   - Code Pointer Table → redirect code execution
   - JS Dispatch Table → hijack function dispatch
3. Or exploit partially-reserved sandbox (weaker isolation)
4. Or exploit trusted objects that live outside sandbox
```

### Key Files
| File | Role |
|------|------|
| `src/sandbox/sandbox.h` | Main sandbox class |
| `src/sandbox/check.h` | SBXCHECK macro |
| `src/sandbox/external-pointer-table.h` | External pointer indirection |
| `src/sandbox/code-pointer-table.h` | Code pointer indirection |
| `src/sandbox/trusted-pointer-table.h` | Trusted pointer indirection |
| `src/sandbox/js-dispatch-table.h` | JS dispatch indirection |
| `src/sandbox/testing.h` | Memory Corruption API |
| `src/sandbox/GLOSSARY.md` | Security properties documentation |

---

## 6. WebAssembly [MEDIUM]

### Description
WebAssembly has its own type system, validation, and memory model. The
boundaries between JS and WASM are security-critical, as are WASM memory
bounds checks and the trap handler mechanism.

### Attack Pattern
```
1. Exploit WASM validation bypass to load invalid module
2. Or exploit JS↔WASM boundary type confusion
3. Or trigger WASM memory access that bypasses bounds checks
4. Or exploit trap handler race condition
```

### Key Files
| File | Role |
|------|------|
| `src/wasm/module-decoder.cc` | Module structure validation |
| `src/wasm/function-body-decoder.cc` | Function body validation |
| `src/wasm/wasm-js.cc` | JS ↔ WASM interface |
| `src/wasm/wasm-engine.cc` | Engine management |
| `src/trap-handler/trap-handler.h` | Hardware trap handling |

---

## 7. Runtime Functions [MEDIUM]

### Description
Runtime functions (`src/runtime/*.cc`) are C++ functions callable from
generated code. They perform operations that are too complex for inline code
generation. If they don't validate inputs against corruption, they can be
exploited after achieving initial memory corruption.

### Key Files
| File | Role |
|------|------|
| `src/runtime/runtime-array.cc` | Array operations |
| `src/runtime/runtime-object.cc` | Object operations |
| `src/runtime/runtime-strings.cc` | String operations |
| `src/runtime/runtime-typedarray.cc` | TypedArray operations |
| `src/runtime/runtime-wasm.cc` | WASM support functions |

### Evidence of Gaps
Recent commit `a03a1a3c` ("Harden some runtime functions against corrupted
input") indicates that some runtime functions were vulnerable to corrupted
inputs, suggesting more may remain.

---

## 8. RegExp / Parser [LOW]

### Description
The regular expression engine and JavaScript parser process untrusted input.
While heavily fuzzed, stack overflow and integer overflow bugs still occur.

### Key Files
| File | Role |
|------|------|
| `src/regexp/regexp-compiler.cc` | RegExp compilation |
| `src/regexp/regexp-interpreter.cc` | RegExp interpretation |
| `src/regexp/regexp-parser.cc` | RegExp parsing |
| `src/parsing/parser.cc` | JS parser |
| `src/parsing/scanner.cc` | JS scanner/lexer |

### Evidence
Recent commit `b07860d0` fixed a stack overflow in the regexp AST visitor,
suggesting this class of bug recurs in the regexp subsystem.
