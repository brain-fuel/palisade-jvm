# Codebase Concerns

**Analysis Date:** 2026-04-21

## Tech Debt

### Core Unification & Type Checking Complexity

**Issue:** Core unification engine (`src/Core/Unify.idr`) is 1664 lines of interdependent constraint solving logic with insufficient refactoring markers.

- Files: `src/Core/Unify.idr`, `src/Core/UnifyState.idr`
- Impact: Bug fixes in unification are high-risk and require extensive test coverage. Adding new type system features is non-trivial.
- Fix approach: Extract pure constraint algorithms into separate modules. Add more granular unit tests for each unification case.

### Large Parser Module

**Issue:** Parser logic concentrated in single 2755-line file (`src/Idris/Parser.idr`) mixing concerns across operator parsing, syntax rules, and error recovery.

- Files: `src/Idris/Parser.idr`
- Impact: Difficult to maintain and extend parser features; parser bugs are hard to isolate.
- Fix approach: Split parser into modules by concern (expressions, declarations, operators, error messages). Standardize error recovery patterns.

### Desugaring Logic Spread Across Files

**Issue:** Core desugaring (~1446 lines) mixed with elaborate resugar/desugar patterns with insufficient separation of concerns.

- Files: `src/Idris/Desugar.idr` (1446 lines), `src/Idris/Resugar.idr`, `src/TTImp/Unelab.idr`
- Impact: Changes to syntax desugaring risk breaking undesugaring; bidirectional invariants hard to maintain.
- Fix approach: Create formalized tests proving desugaring ↔ resugar bidirectionality. Document invariants for each rule.

### Case Tree Builder Complexity

**Issue:** Case builder logic (`src/Core/Case/CaseBuilder.idr`, 1272 lines) handles pattern matching compilation with extensive pattern coverage analysis.

- Files: `src/Core/Case/CaseBuilder.idr`, `src/Core/Case/CaseTree.idr`
- Impact: Coverage checking bugs can silently allow incomplete patterns. Changing matching semantics is high-risk.
- Fix approach: Add property-based tests verifying coverage guarantees. Extract invariants as separate validation module.

### TTC (TT Intermediate Code) Binary Format

**Issue:** Binary serialization/deserialization (`src/Core/TTC.idr`, 1210 lines) handles compiler cache format with minimal version checking.

- Files: `src/Core/TTC.idr`, `src/Core/Binary.idr`
- Impact: Cache invalidation issues can cause stale data. Format changes require careful migration.
- Fix approach: Add strict version tagging. Implement cache validation with clear error messages for format mismatches.

## Known Limitations (Documented in README)

### Cumulativity Not Implemented

**Issue:** Type universe cumulativity is not supported; currently `Type : Type` holds.

- Problem: Can create inconsistent type hierarchies. Users must manually work around this.
- Impact: Type-theoretically unsound in principle, though pragmatically works for most programs.
- Workaround: Users avoid constructing types that require cumulativity.

### Rewrite on Dependent Types

**Issue:** The `rewrite` tactic does not yet work correctly on dependent types.

- Problem: Type-level rewriting with dependent equality is incomplete.
- Blocks: Advanced dependently-typed proofs requiring rewrite at type level.

## Unimplemented Features

### %start Pragma

**Issue:** `%start` directive for program entry point is declared but unimplemented.

- Files: `src/Idris/Desugar.idr:1406`
- Status: Throws `InternalError "%start not implemented"`
- Impact: Users cannot specify custom entry points; must use default `main` or script entry.

### Missing FFI Primitives

**Issue:** Several FFI primitives are referenced but not implemented in specific backends.

- Files:
  - `src/Compiler/ES/Codegen.idr:573` - `prim__os` not implemented in ES backend
  - `src/Compiler/RefC/RefC.idr:561-562` - RefC requires explicit primitive handling
  - `src/Compiler/RefC/RefC.idr:763` - Struct field access not implemented: `CFStruct` handling missing
- Impact: Programs using unimplemented primitives will crash at compile time with `InternalError`.
- Fix approach: Complete primitive coverage for each backend. Add compile-time checking for primitive availability.

### Superclass Default Implementations

**Issue:** Default superclass implementations in interfaces are not fully supported.

- Files: `src/Idris/Elab/Interface.idr:72`
- Status: Marked `TODO: Deal with default superclass implementations`
- Impact: Interface design patterns using defaults are incomplete.

## Performance Bottlenecks

### Quadratic Runtime in Compilation

**Issue:** Compiler optimization phase has quadratic runtime for certain patterns.

- Files: `src/Compiler/CompileExpr.idr:375`
- Problem: Unknown transformations are O(n²) on code size
- Impact: Compiling large programs shows performance degradation
- Improvement path: Profile and optimize the specific transformation; consider memoization or early termination.

### List-Based Data Structures

**Issue:** Multiple modules use `List` where `Set` would be more efficient:

- Files:
  - `src/Core/Context/Context.idr:108` - `mutwith : List Name` (should be `Set Name`)
  - `src/TTImp/Utils.idr:290` - association list should be `Map`
  - `src/TTImp/Utils.idr:503` - string association list should be `Set String`
  - `src/Compiler/Inline.idr:145` - `List Name` should be `Set`
  - `src/Core/Env.idr:291` - list of used names should be proper `Set`

- Impact: Lookups and membership checks are linear instead of logarithmic; significant for large programs.
- Improvement path: Replace with appropriate data structures; verify performance improvement in benchmarks.

### Lazy List Handling

**Issue:** List reversal in `Libraries/Data/SnocList/Extra.idr:7` marked TODO for left-to-right optimization.

- Files: `src/Libraries/Data/SnocList/Extra.idr:7`
- Problem: Current implementation inefficient for large stream processing
- Impact: Code using snoced lists on large data may be slow.

## Code Quality & Maintenance Debt

### Scattered TODO Comments (90+ occurrences)

**Critical TODOs:**

- `src/Core/Transform.idr:59` - Cannot weaken type variables yet (blocks advanced transformations)
- `src/Idris/REPL.idr:414-415` - FFI list syntax hack with TODO for cleanup
- `src/Idris/Desugar.idr:680` - FC (source location) merging incomplete in literal handling
- `src/TTImp/TTImp.idr:958` - Duplicate definitions need merge (744-747 overlap)
- `src/Core/Termination/SizeChange.idr:232` - Incomplete size change analysis
- `src/Idris/Elab/Implementation.idr:131,141` - Refactoring needed for method elaboration
- `src/TTImp/Elab/App.idr:384` - Conservative constraint handling in implicit resolution
- `src/Compiler/Scheme/Chez.idr:25,31` - Platform detection for Gambit scheme incomplete

**Low Priority TODOs:**

- 60+ comments about minor refactoring or edge cases
- Scattered across parser, elaboration, and library code
- Generally marked for future optimization or cleanup

**Fix approach:** Categorize TODOs by urgency. Schedule high-impact ones for next sprint. Document low-priority ones in separate tracking list.

### Incomplete Elaboration Refactoring

**Issue:** Implementation elaboration marked for refactoring (`src/Idris/Elab/Implementation.idr`).

- Files: `src/Idris/Elab/Implementation.idr:131`
- Problem: Monolithic function combining ~100 lines of tightly-coupled steps
- Impact: Hard to add new features; difficult to debug errors
- Safe modification: Extract each step into separate function. Verify test output unchanged.

### Interface Elaboration Incomplete

**Issue:** Interface checking has multiple TODOs:

- Files: `src/Idris/Elab/Interface.idr:71-72`
- Problems:
  - Not checking all parts of interface body
  - Default superclass implementations not handled
- Impact: Invalid interfaces may be accepted; inheritance is incomplete.
- Test coverage gaps: Add tests for deeply nested interface hierarchies.

## Fragile Areas

### With-Clause Pattern Matching

**Issue:** With-clause handling has incomplete pattern support.

- Files: `src/TTImp/WithClause.idr:63,256,258`
- Why fragile: Patterns in `Local`, `Update`, and `Lam` cases marked TODO; may not handle all valid patterns.
- Safe modification: Add test for each pattern type in with-clause context before making changes.
- Test coverage: Only basic patterns likely tested; complex nested patterns untested.

### Case Tree Construction

**Issue:** Case tree builder has "impossible" clauses and incomplete invariants.

- Files: `src/Core/Case/CaseTree.idr:46,109`
- Why fragile: 
  - Lambda types not handled (`TODO: Matching on lazy types`)
  - PI type ordering assumptions may break
- Safe modification: Run full test suite after any changes; verify coverage checker still works.
- Test coverage gaps: Lazy types in pattern matching; dependent PI matching.

### Hole Matching & Delayed Elaboration

**Issue:** Hole substitution and delayed elaboration have complex state management.

- Files: `src/Core/Unify.idr:815` - "Can't happen: Lost hole" error
- Why fragile: Assumes hole consistency throughout elaboration pipeline
- Safe modification: Add assertions and logging for hole state. Never remove holes without tracing.
- Test coverage: Add tests forcing multiple delays and hole resolution in complex types.

### Binary Serialization Format

**Issue:** TTC format lacks version checking for forward/backward compatibility.

- Files: `src/Core/Binary.idr:334,477`, `src/Core/Metadata.idr:434`
- Why fragile: Invalid format data can silently produce wrong behavior instead of clear error
- Safe modification: Always add version check before format changes. Document format changes.
- Test coverage: Add tests for cache invalidation and format mismatch detection.

## Scaling Limits

### Bootstrap Complexity

**Issue:** Multi-stage bootstrap process with manual version management.

- Current approach: Chez/Racket + separate bootstrap stages
- Limit: Adding new language features requires rebootstrapping entire compiler
- Scaling path: Consider incremental bootstrap or self-hosting approach; document bootstrap invariants.

### Mutual Recursion in Core Types

**Issue:** Type checking mutually recursive definitions has complexity.

- Files: `src/TTImp/ProcessData.idr:425,514,538` - Placeholder types during mutual checking
- Current capacity: Handles typical mutual blocks efficiently
- Limit: Very large mutual blocks (20+ definitions) may show performance degradation
- Scaling path: Optimize placeholder lookup; batch process mutual groups.

## Dependencies at Risk

### Scheme Backend (Chez/Racket)

**Risk:** Multi-scheme backend support adds complexity.

- Impact: Bug fixes must work across multiple Scheme implementations
- Migration plan: Consider standardizing on single Scheme target (Chez); deprecate Racket backend.

### Compiler Backends Fragmentation

**Risk:** Three active backends (Scheme, RefC, ES) with different maturity levels.

- Files:
  - `src/Compiler/Scheme/*` - Most mature
  - `src/Compiler/RefC/*` - Growing (C target)
  - `src/Compiler/ES/*` - JavaScript target (newer)

- Impact: New features must be implemented 3 times; bug fixes are tripled work
- Migration plan: Consolidate on single backend (RefC recommended); archive others.

## Test Coverage Gaps

### Coverage Checker

**Untested areas:**
- Dependent pattern matching edge cases
- Deep mutual recursion
- Pattern exhaustiveness with lazy types

Files: `src/Core/Coverage.idr`, `src/Core/Case/*`
Risk: Silent acceptance of incomplete patterns
Priority: High

### Parser Error Recovery

**Untested areas:**
- Complex nested syntax errors
- Error recovery in deeply nested expressions
- Operator precedence edge cases

Files: `src/Idris/Parser.idr`
Risk: Confusing error messages on malformed code
Priority: Medium

### Elaboration Ambiguity Resolution

**Untested areas:**
- Very large ambiguous sets
- Circular type inference scenarios
- Overload resolution edge cases

Files: `src/TTImp/Elab/Ambiguity.idr`
Risk: Non-deterministic compiler behavior
Priority: High

### RefC Backend Primitives

**Untested areas:**
- All RefC-specific primitives
- Struct field access (`CFStruct`)
- Platform-specific integer operations

Files: `src/Compiler/RefC/RefC.idr`
Risk: Silent failures on platforms not tested
Priority: High

### ES/JavaScript Backend

**Untested areas:**
- Complex tail call optimization scenarios
- JavaScript keyword collision handling
- Unicode escaping in generated code

Files: `src/Compiler/ES/*`
Risk: Generated code malfunction on edge cases
Priority: Medium

## Security Considerations

### FFI Safety

**Risk:** Foreign function interface allows arbitrary C code execution.

- Files: `src/Compiler/RefC/RefC.idr:902-905`, `libs/base/Data/Buffer.idr`
- Current mitigation: Type checking on entry/exit
- Recommendations:
  - Add capability-based restrictions on FFI calls
  - Document safe FFI patterns
  - Provide isolation layer for untrusted code

### String Handling

**Risk:** String escape sequences in code generation could be exploited.

- Files: `src/Compiler/ES/Codegen.idr:42-55` (jsString), `src/Compiler/RefC/RefC.idr:97-112` (cStringQuoted)
- Current mitigation: Character-by-character escaping
- Recommendations:
  - Add fuzzing tests for string edge cases
  - Verify Unicode handling matches language spec
  - Test with strings containing control characters

## Half-Finished Backends / WIP Areas

### No JVM Backend

**Status:** Despite branch name `feat/palisade-jvm`, no JVM backend code is present. This is the main Idris2 repo.

- Branch state: `feat/palisade-jvm` points to same commit as `main` (214eb4547)
- Impact: JVM work (if any) is on different branch; this checkout is incomplete for JVM development
- Next step: Confirm intended branch or fetch proper JVM implementation

### Lazy Lists (WIP)

**Status:** Data structure for lazy list quantifiers marked WIP.

- Files: `libs/base/Data/List/Lazy/Quantifiers.idr:1` - "WIP: same as Data.List.Quantifiers but for lazy lists"
- Impact: Lazy list proofs are incomplete
- Completion path: Mirror all quantifier operations to lazy variant.

## Known Bugs & Regressions

### Integer Literal Edge Case

**Issue:** Int64 minimum value cannot be represented as literal.

- Files: `tests/refc/basicpatternmatch/Main.idr:89-90`
- Symptoms: Pattern matching on `Int64.min` shows FIXME comment
- Status: Known limitation; workaround is to use variable binding
- Trigger: Code trying to match literal `-9223372036854775808`

### Deprecated Buffer Operations

**Issue:** `setByte` and `getByte` deprecated in Buffer API.

- Files: `tests/refc/buffer/TestBuffer.idr:14,32`
- Status: Functions marked for removal in future release
- Migration: Tests document TODO to update once removed from libs
- Impact: Code using old API will break in next major version

### Record Type Annotation (from interactive test)

**Issue:** Null list used as placeholder in interactive test.

- Files: `tests/idris2/interactive/interactive031/Signatures.idr:14`
- Status: "TODO: Fix not entirely appropriate definition (nulls list of Node2)"
- Impact: Test may not properly exercise signature generation with complex types

---

*Concerns audit: 2026-04-21*
