# V8 Security Vulnerability Analysis

## Target

- **Engine**: V8 JavaScript Engine
- **Version**: 14.7.0 (candidate release)
- **Repository**: chromium/v8
- **Analysis Date**: 2026-02-23
- **Codebase Size**: ~3,000+ source files across `src/`

## Executive Summary

This analysis covers the major attack surfaces of V8 14.7.0, focusing on areas
historically responsible for the majority of V8 CVEs: the JIT compiler type
system, bounds check elimination, garbage collector write barriers, and the
sandbox architecture. The analysis identified **1 confirmed bug** (potential
CVE) in the Turboshaft compiler and **6+ vulnerability patterns** warranting
further investigation.

## Findings Summary (Revised After Deep Dive)

| # | Finding | Severity | File | Status |
|---|---------|----------|------|--------|
| 1 | Turboshaft Multiply NaN Type Confusion | **MEDIUM** (latent **CRITICAL**) | `src/compiler/turboshaft/typer.h:620` | Confirmed bug |
| 2 | Turboshaft `allow_invalid_inputs()` always true | **HIGH** | `src/compiler/turboshaft/typer.h:1619` | Design weakness |
| 3 | TurboFan CheckBounds precision boundary | **LOW** | `src/compiler/operation-typer.cc:1347` | Mitigated by type cache |
| 4 | Sandbox partial reservation fallback | **MEDIUM** | `src/sandbox/sandbox.h:72,113` | Design weakness |
| 5 | `FatalNoSecurityImpact` crash suppression | **MEDIUM** | `src/base/logging.cc:101` | Design concern |
| 6 | Runtime function hardening gaps | **MEDIUM** | `src/runtime/` (1/671 SBXCHECK) | Pattern-based |
| 7 | RegExp visitor stack overflow pattern | **LOW** | `src/regexp/` | Partially fixed |
| 8 | StoreNoWriteBarrier audit targets | **HIGH** (if wrong) | `src/codegen/code-stub-assembler.cc` | 45+ occurrences |

## Attack Surface Priority Matrix

```
CRITICAL ──────────────────────────────────────────────────────────
  JIT Type System (TurboFan + Turboshaft typers)
  → Incorrect type inference → BCE → OOB read/write
  → Historical #1 source of V8 CVEs

HIGH ──────────────────────────────────────────────────────────────
  Bounds Check Elimination (simplified-lowering, typed-optimization)
  → Type-based removal of bounds checks
  → Direct path to arbitrary read/write

  GC Write Barriers (heap-write-barrier)
  → Missing barrier → dangling pointer → UAF
  → Especially during compaction / concurrent marking

  Element Kind Transitions (elements.cc, elements-kind.h)
  → Incorrect transitions → type confusion
  → 30+ element kinds with complex transition rules

MEDIUM ─────────────────────────────────────────────────────────────
  Sandbox Architecture (src/sandbox/)
  → Pointer table corruption, partial reservation bypass
  → Post-exploitation relevance (after initial OOB)

  WebAssembly Boundaries (src/wasm/)
  → JS↔WASM type confusion, memory bounds
  → Growing attack surface with WASM GC proposal

  Runtime Functions (src/runtime/)
  → Corrupted inputs, missing validation
  → Recent hardening suggests historical gaps

LOW ────────────────────────────────────────────────────────────────
  RegExp / Parser (src/regexp/, src/parsing/)
  → Stack overflow, malformed input handling
  → Generally well-fuzzed
```

## Document Index

| Document | Description |
|----------|-------------|
| [01-ARCHITECTURE.md](01-ARCHITECTURE.md) | V8 architecture, compilation pipeline, data flows |
| [02-ATTACK-SURFACES.md](02-ATTACK-SURFACES.md) | Categorized attack surfaces with severity ratings |
| [03-JIT-COMPILER-ANALYSIS.md](03-JIT-COMPILER-ANALYSIS.md) | Deep dive into JIT type system and bounds check elimination |
| [04-MEMORY-AND-GC.md](04-MEMORY-AND-GC.md) | Garbage collector, write barriers, object model |
| [05-SANDBOX-ANALYSIS.md](05-SANDBOX-ANALYSIS.md) | Sandbox architecture, pointer tables, bypass vectors |
| [06-VULNERABILITIES.md](06-VULNERABILITIES.md) | Specific bugs found with exploitation strategies |
| [07-FILE-REFERENCE.md](07-FILE-REFERENCE.md) | Quick-reference index of security-critical files |
| [poc-vuln001-analysis.md](poc-vuln001-analysis.md) | VULN-001 complete call chain, PoC, and exploitation analysis |
| [deep-dive-vuln002-allow-invalid-inputs.md](deep-dive-vuln002-allow-invalid-inputs.md) | VULN-002: 40+ InputIs() call sites, 30+ unimplemented ops |
| [deep-dive-vuln003-checkbounds-precision.md](deep-dive-vuln003-checkbounds-precision.md) | VULN-003: Bounds check elimination chain analysis |
| [deep-dive-vuln004-sandbox-bypass.md](deep-dive-vuln004-sandbox-bypass.md) | VULN-004: 8 sandbox bypass vectors with pointer table analysis |
| [deep-dive-vuln005-007-additional-patterns.md](deep-dive-vuln005-007-additional-patterns.md) | VULN-005/007: FatalNoSecurityImpact + write barriers + RegExp |
| [deep-dive-vuln006-runtime-hardening.md](deep-dive-vuln006-runtime-hardening.md) | VULN-006: Runtime SBXCHECK coverage gap (1/671 functions) |

## Methodology

1. **Static analysis** of V8 source code focusing on type system correctness,
   memory safety, and trust boundaries
2. **Differential analysis** between TurboFan and Turboshaft implementations
   of the same operations (leading to the Multiply bug discovery)
3. **Pattern matching** for common vulnerability classes: type confusion,
   integer overflow, use-after-free, bounds check elimination
4. **Architecture review** of the sandbox, GC, and pointer table systems
5. **Comment mining** for TODO/FIXME/HACK/BUG/SECURITY markers indicating
   known issues
