# Deep Dive: V8 Regular Expression Engine (Irregexp)

## Overview

V8's RegExp engine, "Irregexp," is a multi-stage compiler that transforms regex patterns into either native machine code or bytecode. It employs Boyer-Moore string searching, quick check optimizations, bytecode peephole optimization, and backtracking with safety limits. An experimental linear-time engine provides a fallback for pathological patterns.

Key files: `src/regexp/regexp-parser.h`, `src/regexp/regexp-compiler.h`, `src/regexp/regexp-bytecodes.h`, `src/regexp/regexp-interpreter.cc`

---

## 1. RegExp Parsing and AST Construction

### AST Node Types

**Source:** `src/regexp/regexp-ast.h`

```cpp
#define FOR_EACH_REG_EXP_TREE_TYPE(VISIT)
  VISIT(Disjunction)         // a|b|c alternatives
  VISIT(Alternative)         // Sequence of nodes
  VISIT(Assertion)           // ^, $, \b, \B
  VISIT(ClassRanges)         // Character classes [a-z]
  VISIT(Atom)                // Literal strings
  VISIT(Quantifier)          // *, +, ?, {n,m}
  VISIT(Capture)             // Capture groups ()
  VISIT(Group)               // Non-capturing groups (?:)
  VISIT(Lookaround)          // (?=), (?!), (?<=), (?<!)
  VISIT(BackReference)       // \1, \2, named backrefs
```

### Key Data Structures

- **CharacterRange**: Unicode code point ranges (0 to 0x10FFFF) with canonicalization, negation, and intersection
- **Interval**: Closed interval [from, to] for tracking capture register ranges
- **RegExpTextBuilder**: Handles UTF-16 surrogate pairs, coalesces adjacent atoms

### Parser Architecture

Recursive descent with:
- Stack limit tracking (`StackLimiter`) to prevent DoS
- Unicode mode handling for `/u` and `/v` flags
- Named capture tracking during parsing

---

## 2. Bytecode Compilation

**Source:** `src/regexp/regexp-bytecode-generator.h`, `src/regexp/regexp-bytecodes.h`

### Bytecode System

- Fixed initial buffer: `kInitialBufferSizeInBytes = 1 * KB`
- Dynamic growth up to `kMaxBufferGrowthInBytes = 1 * MB`
- Operand types: `Int16`, `Int32`, `Uint32`, `Char`, `JumpTarget`, `BitTable`

### Register System

- **Current Position Register**: Tracks position in input string
- **Capture Registers**: Pairs (start, end) for each group — `StartRegister(i) = i * 2`
- **Stack Pointer Register**: For backtrack stack

### Peephole Optimization

**Source:** `src/regexp/regexp-bytecode-peephole.cc`

Post-generation optimizer:
- Combines multiple character checks into single operations
- Eliminates redundant position updates
- Merges consecutive jumps
- Simplifies stack operations

---

## 3. JIT Compilation (Irregexp Core)

**Source:** `src/regexp/regexp-compiler.h/cc`

### Node Types for Compilation

```cpp
#define FOR_EACH_NODE_TYPE(VISIT)
  VISIT(End)       VISIT(Action)     VISIT(Choice)
  VISIT(LoopChoice) VISIT(NegativeLookaroundChoice)
  VISIT(BackReference) VISIT(Assertion) VISIT(Text)
```

### Trace-Based Code Generation

The **Trace** class encapsulates virtual execution state:
- `cp_offset_` — current position offset
- `flush_budget_` — deferred operations before forced flush
- `at_start_` — TriBool for string start position
- `backtrack_` — label for failure backtracking
- `quick_check_performed_` — details from previous node

### Quick Check Optimization

```cpp
class QuickCheckDetails {
  static constexpr int kMaxPositions = 4;
  Position positions_[kMaxPositions];
  uint32_t mask_;   // Combined AND mask
  uint32_t value_;  // Expected value after AND
};
```

Generates `(next_chars AND mask) == value` to reject impossible matches in a single comparison.

### Boyer-Moore Lookahead

```cpp
class BoyerMooreLookahead {
  static constexpr int kMaxLookaheadForBoyerMoore = 8;
  ZoneList<BoyerMoorePositionInfo*>* bitmaps_;
};
```

Analyzes up to 8 lookahead positions, creates skip tables for non-matching character sequences.

---

## 4. Backtracking Implementation

### Backtrack Stack

```cpp
class BacktrackStack {
  bool push(int v);    // Returns false if limit exceeded
  int peek() const;
  int pop();
  int sp() const;
  void set_sp(uint32_t new_sp);
};
```

### Deferred Actions

The Trace defers actions until actually needed:
- Some actions become unnecessary if the match fails
- Saves work that never happens
- Enables specialized code generation

### Fixed-Length Loop Optimization

For greedy loops like `a*b`:
- Stores only initial position (not each step)
- Backtracks by decrementing counter
- O(1) space instead of O(n)

---

## 5. Unicode Property Support

### Flag System

```cpp
#define REGEXP_FLAG_LIST(V)
  V(global, 'g')    V(ignore_case, 'i')  V(multiline, 'm')
  V(dot_all, 's')   V(unicode, 'u')      V(unicode_sets, 'v')
  V(sticky, 'y')    V(has_indices, 'd')   V(linear, 'l')
```

### Unicode Mode Behavior

- Works with code points (not code units)
- Surrogate pairs counted as single characters
- `\p{PropertyName=Value}` property escapes via ICU integration
- Case equivalence differs between ASCII and Unicode modes

---

## 6. Named Capture Groups

```cpp
class RegExpCapture final : public RegExpTree {
  const ZoneVector<base::uc16>* name_;  // Capture name
  int index_;                           // Numeric index
};
```

Named captures (`(?<name>...)`) are:
- Validated for uniqueness during parsing
- Mapped to numeric indices
- Supported in backreferences (`\k<name>`)

---

## 7. Optimization Techniques Summary

| Technique | Location | Effect |
|-----------|----------|--------|
| Quick Check | `regexp-compiler.h` | Single mask/compare rejects impossible matches |
| Boyer-Moore | `regexp-compiler.h` | Skip table for 8-char lookahead |
| Peephole | `regexp-bytecode-peephole.cc` | Post-generation bytecode cleanup |
| Fixed-Length Loops | `regexp-compiler.cc` | O(1) space for greedy quantifiers |
| Frequency Analysis | `FrequencyCollator` | Assess constraint quality for BM |
| Anchoring Analysis | `RegExpTree` | Skip impossible anchor positions |
| Range Canonicalization | `CharacterRange` | Compress character classes |

---

## 8. Experimental Linear-Time Engine

**Source:** `src/regexp/experimental/`

Fallback engine for patterns causing exponential backtracking:

```cpp
class ExperimentalRegExp {
  static bool CanBeHandled(RegExpTree* tree, ...);
  static bool Compile(Isolate* isolate, ...);
  static int32_t MatchForCallFromJs(...);
  static constexpr bool kSupportsUnicode = false;
};
```

Activated by `/l` flag or automatic fallback via `FALLBACK_TO_EXPERIMENTAL` result code.

---

## 9. String Search Algorithms

**Source:** `src/strings/string-search.h`

| Pattern Length | Algorithm | Complexity |
|---------------|-----------|------------|
| 1 character | SingleCharSearch | O(n) |
| 2-6 characters | LinearSearch | O(n*m) |
| 7+ characters | Boyer-Moore-Horspool | O(n/m) average |
| Very long | Full Boyer-Moore | O(n) worst-case |

Template specializations for Latin1/UTF-16 combinations.

---

## 10. Test Suite and Conformance

### Test Directories

- `test/mjsunit/regexp/` — 30+ JavaScript test files
- `test/unittests/regexp/` — C++ unit tests
- `test/fuzzer/regexp/` — Fuzzing harnesses
- `test/test262/` — ECMAScript conformance (RegExp sections)

### Coverage Areas

- Capture groups (named, numbered), backreferences
- Unicode property escapes, surrogate pairs
- Boyer-Moore optimization correctness
- Backtrack limit exhaustion, stack overflow
- Regression tests by bug number

---

## Architecture

```
RegExp String → Parser → AST (RegExpTree nodes)
                           ↓
                    RegExpCompiler (AST → Node graph)
                           ↓
              ┌────────────┼────────────┐
              ↓            ↓            ↓
         Bytecode     NativeCode   Experimental
         Generator    (per-arch)   (linear-time)
              ↓            ↓            ↓
         Peephole      JIT Code    Linear Engine
         Optimizer
              ↓
         Interpreter ← BacktrackStack
              ↓
         Match Results
```

---

## Security

- **ReDoS Prevention**: Parse-time stack limits, backtrack limits, experimental engine fallback
- **Memory Safety**: Zone allocation freed on error, bounds checking
- **Recursion Limits**: `kMaxRecursion = 100` (50 on macOS)
