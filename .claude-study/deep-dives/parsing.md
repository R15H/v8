# Deep Dive: V8 Parsing and Lexical Analysis

## Overview

V8's parsing system is a multi-layered architecture designed for high-performance JavaScript parsing. It employs a **lazy parsing strategy** through the PreParser to defer full parsing of function bodies, combined with **recursive-descent parsing** in the main Parser, comprehensive Unicode handling in the Scanner, and elaborate scope analysis.

Key files: `src/parsing/scanner.h`, `src/parsing/parser.cc`, `src/parsing/preparser.h`, `src/ast/ast.h`, `src/ast/scopes.h`

---

## 1. Scanner/Lexer Token Generation and Unicode Handling

### Token System

Tokens are defined via macro-based system in `src/parsing/token.h`:

```cpp
#define TOKEN_LIST(T, K)
  T(kTemplateSpan, nullptr, 0)
  T(kTemplateTail, nullptr, 0)
  T(kPeriod, ".", 0)
  T(kLeftBracket, "[", 0)
  T(kString, nullptr, 0)
  T(kIdentifier, nullptr, 0)
  K(kAsync, "async", 0)
  K(kAwait, "await", 0)
  K(kYield, "yield", 0)
```

### Character Stream Abstraction

The `Utf16CharacterStream` provides buffered Unicode-aware reading:

```cpp
class Utf16CharacterStream {
  inline base::uc32 Peek();
  inline base::uc32 Advance();
  template <typename FunctionType>
  V8_INLINE base::uc32 AdvanceUntil(FunctionType check);
  inline void Back();

  const uint16_t* buffer_start_;
  const uint16_t* buffer_cursor_;
  const uint16_t* buffer_end_;
};
```

### 4-Token Lookahead Buffer

The Scanner maintains a circular buffer of token descriptors:

```cpp
struct TokenDesc {
  Location location = {0, 0};
  LiteralBuffer literal_chars;       // Cooked literal (escapes resolved)
  LiteralBuffer raw_literal_chars;   // Raw literal (original escapes)
  Token::Value token = Token::kUninitialized;
  uint32_t smi_value = 0;
  bool after_line_terminator = false;
};

TokenDesc token_storage_[4];
TokenDesc* current_;
TokenDesc* next_;
TokenDesc* next_next_;
TokenDesc* next_next_next_;
```

### Unicode Escape Handling

```cpp
template <bool capture_raw>
base::uc32 Scanner::ScanUnicodeEscape();  // \uXXXX and \u{...}

bool Scanner::CombineSurrogatePair() {
  if (unibrow::Utf16::IsLeadSurrogate(c0_)) {
    base::uc32 c1 = source_->Advance();
    if (unibrow::Utf16::IsTrailSurrogate(c1)) {
      c0_ = unibrow::Utf16::CombineSurrogatePair(c0_, c1);
      return true;
    }
    source_->Back();
  }
  return false;
}
```

---

## 2. Recursive-Descent Parser Architecture

### CRTP Template Design

V8 uses the Curiously Recurring Template Pattern where `ParserBase<Impl>` provides core logic, specialized by `Parser` and `PreParser`:

```cpp
template <typename Impl>
class ParserBase {
  ExpressionT ParseAssignmentExpression();
  ExpressionT ParseConditionalExpression();
  ExpressionT ParseBinaryExpression(int prec);
  // ...
};

class Parser : public ParserBase<Parser> { /* full AST generation */ };
class PreParser : public ParserBase<PreParser> { /* lightweight scanning */ };
```

### Expression Hierarchy (by precedence)

```cpp
ParsePrimaryExpression()         // literals, identifiers, this
ParseMemberExpression()          // obj.prop, obj[prop]
ParseCallExpression()            // func(), super()
ParseUnaryExpression()           // ++, --, !, ~, typeof, void, await
ParseExponentiationExpression()  // **
ParseBinaryExpression(prec)      // *, /, %, +, -, <<, >>, <, >, ==, &, ^, |, &&, ||, ??
ParseConditionalExpression()     // ? :
ParseAssignmentExpression()      // =, +=, -=
```

### Operator Precedence Table

From `token.h`:
```
??  → precedence 3
||  → precedence 4
&&  → precedence 5
|   → precedence 6
^   → precedence 7
&   → precedence 8
==  → precedence 9
<   → precedence 10
<<  → precedence 11
*   → precedence 13
**  → precedence 14 (right-associative)
```

---

## 3. Lazy Parsing and Pre-Parser

### PreParser Lightweight Types

The PreParser uses minimal-allocation types:

```cpp
class PreParserIdentifier {
  enum Type : uint8_t {
    kNullIdentifier, kUnknownIdentifier, kEvalIdentifier,
    kArgumentsIdentifier, kConstructorIdentifier, kAsyncIdentifier
  };
};

class PreParserExpression {
  uint32_t code_;  // All state in a single 32-bit field
};
```

### Lazy Parsing Flow

```cpp
bool Parser::SkipFunction(
    const AstRawString* function_name, FunctionKind kind,
    DeclarationScope* function_scope,
    ProducedPreparseData** produced_preparsed_scope_data) {

  PreParser* pre_parser = reusable_preparser();
  PreparseDataBuilder builder(zone(), function_scope, &preparse_data_buffer_);
  pre_parser->SkipFunctionLiteral(...);
  *produced_preparsed_scope_data = builder.Build(zone());
}
```

### Preparsed Scope Data

`PreparseData` (`src/parsing/preparse-data.h`) stores metadata about skipped functions: start/end positions, parameter count, inner function count, and flags.

---

## 4. Scope Analysis and Variable Resolution

### Scope Hierarchy

```cpp
class Scope : public ZoneObject {
  bool is_eval_scope() const;
  bool is_function_scope() const;
  bool is_module_scope() const;
  bool is_block_scope() const;
  bool is_class_scope() const;

  Scope* outer_scope() const;
  UnresolvedList unresolved_list_;
};

class DeclarationScope : public Scope {
  Variable* function_var() const;
  Variable* receiver() const;         // 'this'
  Variable* new_target_var() const;
};

class ClassScope : public Scope {
  Variable* DeclarePrivateName(const AstRawString* name, ...);
  bool ResolvePrivateNames(ParseInfo* info);
};
```

### Variable Resolution

```cpp
class Variable {
  VariableMode mode() const;       // kVar, kLet, kConst, kModule
  VariableLocation location() const; // UNALLOCATED, PARAMETER, LOCAL, CONTEXT
  bool is_used() const;
  bool has_forced_context_allocation() const;
};

// Two-phase resolution after parsing
template <ScopeLookupMode mode>
static Variable* Scope::Lookup(VariableProxy* proxy, Scope* scope, ...);
```

---

## 5. Arrow Function and Destructuring Parsing

### Arrow Function Ambiguity

Arrow functions require special handling because `(x, y)` could be a parenthesized expression or arrow parameters:

```cpp
ExpressionT ParseAssignmentExpression() {
  ExpressionT expression = ParseConditionalExpression();
  if (peek() == Token::kArrow) {
    return ParseArrowFunctionLiteral(parameters, ...);
  }
  return expression;
}
```

### ArrowHeadParsingScope

```cpp
template <typename Types>
class ArrowHeadParsingScope {
  // Accumulates potential arrow parameters
  // If => appears, validates; otherwise discards
  void ValidateAsArrowFunctionFormalParameters();
};
```

### Destructuring

Destructuring patterns are parsed as expressions then reinterpreted as patterns in assignment/declaration contexts via `ExpressionParsingScope`.

---

## 6. Template Literal Parsing

### Token Types

```
kTemplateSpan → `text${       (start/middle of template)
kTemplateTail → text`          (end of template)
```

### Dual Literal Buffers

Template literals maintain both raw and cooked forms:
- **Cooked**: Escape sequences processed (`\n` → newline)
- **Raw**: Escape sequences preserved (`\n` → literal `\n`)

For tagged templates, invalid escape sequences don't cause parse errors — cooked value becomes `undefined` but raw is preserved.

### AST Representation

```cpp
class TemplateLiteral : public Expression {
  const ZonePtrList<const AstRawString>* strings() const;      // Cooked
  const ZonePtrList<const AstRawString>* raw_strings() const;  // Raw
  const ZonePtrList<Expression>* expressions() const;           // ${...} parts
};
```

---

## 7. Module Parsing and Import/Export Resolution

### SourceTextModuleDescriptor

**Source:** `src/ast/modules.h`

```cpp
class SourceTextModuleDescriptor : public ZoneObject {
  void AddImport(const AstRawString* import_name,
                 const AstRawString* local_name,
                 const AstRawString* specifier, ...);
  void AddStarImport(...);
  void AddExport(const AstRawString* local_name,
                 const AstRawString* export_name, ...);
  void AddStarExport(...);

  bool Validate(ModuleScope* module_scope, ...);
};
```

### Import Attributes

```cpp
// import x from "y" with { key: "value" }
class ImportAttributes {
  struct Attribute {
    const AstRawString* key;
    const AstRawString* value;
  };
};
```

---

## 8. Error Recovery and Error Messages

### PendingCompilationErrorHandler

```cpp
class PendingCompilationErrorHandler {
  void ReportMessageAt(int start, int end, MessageTemplate message, ...);
  bool has_pending_error() const;
  void ReportErrors(Isolate* isolate, Handle<Script> script) const;
};
```

### Error Message Templates

V8 uses `MessageTemplate` enums for hundreds of error types:
```cpp
enum MessageTemplate {
  kUnexpectedToken, kUnexpectedTokenIdentifier,
  kInvalidHexEscapeSequence, kInvalidUnicodeEscapeSequence,
  kSloppyFunction, ...  // hundreds more
};
```

The parser continues after errors when possible to report multiple issues in one pass.

---

## 9. AST Node Types and Visitor Pattern

### Node Hierarchy

Defined via macros in `src/ast/ast.h`:

```cpp
#define EXPRESSION_NODE_LIST(V)
  V(Assignment) V(BinaryOperation) V(Call) V(CallNew)
  V(ClassLiteral) V(Conditional) V(FunctionLiteral)
  V(TemplateLiteral) V(UnaryOperation) V(VariableProxy) ...
```

### Key AST Nodes

| Node | Represents |
|------|-----------|
| `FunctionLiteral` | Function declarations/expressions (tracks async, generator, arrow flags) |
| `ClassLiteral` | Class definitions (properties, static elements, constructor) |
| `Call` | Function calls (tracks optional chaining, eval detection) |
| `Property` | Property access (computed, private, optional chain) |
| `VariableProxy` | Variable references (resolved to `Variable` during scope analysis) |
| `Assignment` | All assignment forms (`=`, `+=`, destructuring) |

### Visitor Pattern

```cpp
template <typename Subclass>
class AstVisitor {
  #define DECLARE_VISIT(type) virtual void Visit##type(type* node);
  AST_NODE_LIST(DECLARE_VISIT)
};
```

All AST nodes are zone-allocated (`ZoneObject`) for bulk deallocation after compilation.

---

## 10. Source Position Tracking

### Position Representation

```cpp
static constexpr int kNoSourcePosition = -1;

struct SourceRange {
  int32_t start;  // 0-based, inclusive
  int32_t end;    // 0-based, exclusive
};
```

### RAII Position Capture

```cpp
class SourceRangeScope {
  SourceRangeScope(const Scanner* scanner, SourceRange* range)
      : scanner_(scanner), range_(range) {
    range_->start = scanner->peek_location().beg_pos;
  }
  ~SourceRangeScope() {
    range_->end = scanner_->location().end_pos;
  }
};
```

### Source Range Kinds

Detailed range tracking for debugging and code coverage:
```cpp
enum class SourceRangeKind {
  kBody, kCatch, kContinuation, kElse, kFinally, kRight, kThen
};
```

Positions flow through: Scanner → Parser → AST → Bytecode Generator → Source Position Table → Stack Traces & Debugging.

---

## Performance Optimizations

1. **Lazy Parsing**: PreParser defers function bodies
2. **4-Token Lookahead**: Circular buffer avoids re-scanning
3. **Zone Allocation**: O(1) bulk deallocation of all parse-time data
4. **Inline Methods**: Critical scanner methods marked `V8_INLINE`
5. **Precomputed Tables**: Token precedence and flags
6. **Bit-packed Structures**: Flags stored as bit fields
7. **Reusable PreParser**: Single instance across all lazy-parsed functions
