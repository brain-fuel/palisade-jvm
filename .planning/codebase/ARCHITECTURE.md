# Architecture

**Analysis Date:** 2026-04-21

## Pattern Overview

**Overall:** Compiler pipeline with layered IR transformations

**Key Characteristics:**
- Multi-stage compilation: Source → Parsed AST → TTImp → Elaborated Core → Optimized IR → Backend-specific code
- Modular backend system with pluggable code generators (Scheme, RefC, JavaScript, Interpreter)
- Separation of concerns: elaborator, type checker, case compiler, optimizer, and backend each handle distinct transformations
- Incremental compilation support with `.ttc` (Idris TTC) binary cache files
- Context-based definition management with lazy loading of definitions

## Layers

**Parsing & Syntax (`Parser/*`, `Idris/Parser`, `Idris/Syntax`):**
- Purpose: Convert source code text into structured Abstract Syntax Tree (AST)
- Location: `src/Parser/` (low-level lexer/parser combinators), `src/Idris/Parser.idr` (high-level Idris parser)
- Contains: Lexer rules, parser combinators, syntax tree types (in `src/Idris/Syntax.idr`)
- Depends on: `Libraries/Text/Parser` (parser combinator library), raw source text
- Used by: Idris.ProcessIdr, which calls `parseModule` to read `.idr` files

**High-Level Syntax & Desugaring (`Idris/Desugar`):**
- Purpose: Transform high-level Idris syntax into lower-level TTImp intermediate representation
- Location: `src/Idris/Desugar.idr` and `src/Idris/Desugar/` subdirectories
- Contains: Transformation of infix operators, do-notation, string interpolation, pattern matching, list notation
- Depends on: Parsed Idris syntax (PDecl), TTImp types
- Used by: ProcessIdr converts PDecl → desugared TTImp declarations

**Type Checking & Elaboration (`TTImp/Elab`, `TTImp/ProcessDecls`):**
- Purpose: Elaborate TTImp into fully type-checked Core terms, resolving implicits and performing unification
- Location: `src/TTImp/Elab/` (elaboration logic), `src/TTImp/ProcessDecls/` (declaration processing)
- Contains: Elaborator rules, implicit binding, constraint solving, totality checking
- Depends on: TTImp terms, unification state, context definitions
- Used by: ProcessIdr calls elaboration before code generation

**Core Type System (`Core/TT`, `Core/Context`, `Core/Unify`):**
- Purpose: Represent fully elaborated, type-checked terms and maintain global definition context
- Location: `src/Core/TT.idr` (term representation), `src/Core/Context.idr` (definition management)
- Contains: Term type (with binders, Pi, Lambda, Case), GlobalDef records, type environment
- Depends on: Names, locations (FC), typing environment (Env)
- Used by: All downstream compilation stages read definitions from context

**Case Compilation (`Core/Case`):**
- Purpose: Transform pattern matching into decision trees (case trees) for efficient matching
- Location: `src/Core/Case/CaseBuilder.idr`, `src/Core/Case/CaseTree.idr`
- Contains: CaseTree data structure, pattern coverage checking, tree optimization
- Depends on: Core terms, pattern matching definitions
- Used by: CompileExpr to produce CExp terms

**Intermediate Representations:**

  **Compiled Expressions (CExp):** `src/Core/CompileExpr.idr`
  - Simplified terms closer to runtime: local variables, function applications, constructor application
  - No binders, all variables are de Bruijn indexed
  - Entry point for most backend compilation

  **Lambda-Lifted Form:** `src/Compiler/LambdaLift.idr`
  - Eliminates nested function definitions by hoisting them to top level
  - Converts free variables in closures to explicit parameters

  **ANF (Administrative Normal Form):** `src/Compiler/ANF.idr`
  - All arguments to functions are either variables or null (erased)
  - Enables simpler code generation for strict languages

  **VMCode:** `src/Compiler/VMCode.idr`
  - Simple virtual machine bytecode suitable for Scheme/C backends
  - Represents function calls, constructor application, case analysis

**Backend Code Generation (`Compiler/Scheme`, `Compiler/RefC`, `Compiler/ES`, `Compiler/Interpreter`):**
- Purpose: Lower intermediate representations to target language
- Locations:
  - `src/Compiler/Scheme/Chez.idr`: Chez Scheme code generation
  - `src/Compiler/Scheme/Racket.idr`: Racket code generation
  - `src/Compiler/Scheme/ChezSep.idr`, `Gambit.idr`: Alternative Scheme variants
  - `src/Compiler/RefC/RefC.idr`: C code generation via RefC reference counting implementation
  - `src/Compiler/ES/`: JavaScript/Node.js backends
  - `src/Compiler/Interpreter/VMCode.idr`: Direct bytecode interpretation
- Contains: Code emitters, type-specific codegen, FFI integration
- Used by: Compiler.Common interface dispatches to appropriate backend

**Optimization Passes (`Compiler/Opts`, `Compiler/Inline`):**
- Purpose: Improve generated code performance
- Location: `src/Compiler/Opts/` (constructor optimization, CSE, constant folding), `src/Compiler/Inline.idr` (inline expansion)
- Used by: Applied during CompileExpr phase

## Data Flow

**Main Compilation Pipeline:**

1. **Source → Parsed AST**
   - `idris2` main entry at `src/Idris/Main.idr` → `Idris.Driver.mainWithCodegens`
   - Reads `.idr` files via `Idris.ProcessIdr.readModule`
   - Parser (`Idris.Parser.sourceFile`) produces `PModule` (parsed module with `PDecl` declarations)

2. **Parsed AST → TTImp**
   - `ProcessIdr.processDecl` calls `Idris.Desugar.desugarDecl`
   - Transforms `PDecl` → `ImpDecl` (TTImp declarations)
   - Records syntax info (fixities, operators) in `SyntaxInfo` ref

3. **TTImp → Elaborated Core**
   - `TTImp.ProcessDecls.processDecl` elaborates each `ImpDecl`
   - Calls `TTImp.Elab.Check.checkDecl` for type checking
   - Populates `Ctxt` (context) with fully elaborated `GlobalDef` entries
   - Updates unification state for constraints and holes

4. **Core → Optimized IR**
   - `Compiler.CompileExpr.compileExp` converts `Term` → `CExp` (compiled expression)
   - Case compilation via `Core.Case.CaseBuilder`
   - Optional lambda lifting: `Compiler.LambdaLift.lambdaLift` if needed
   - Optional ANF conversion: `Compiler.ANF.toANF` for Scheme/C backends
   - VMCode generation: `Compiler.VMCode.toVM` for bytecode-based backends

5. **IR → Backend Code**
   - Dispatcher in `Compiler.Common.compile` selects backend via `Codegen` record
   - Backend's `compileExpr` method generates target language code
   - Example: `Compiler.Scheme.Chez.compileToScheme` → Chez Scheme `.ss` file
   - Executable built by Scheme compiler or RefC C compiler, etc.

**Module Processing Loop:**

```
readModule (transitively loads imports)
  ↓
parseModule (file → PModule)
  ↓
for each PDecl in module:
  desugarDecl → [ImpDecl]
  ↓
  for each ImpDecl:
    checkDecl (elaboration + type checking)
    ↓
    add GlobalDef to Ctxt
  ↓
writeModuleCache (save .ttc file)
  ↓
if this is main module: compile definitions
```

**State Management:**

- **Ctxt ref:** Global context holding all elaborated definitions (cached definitions loaded from `.ttc` files)
- **UST ref:** Unification state tracking unsolved holes and constraints
- **Syn ref:** Syntax information (operators, fixities, documentation)
- **MD ref:** Metadata (definitions locations, type info for IDE)
- **ROpts ref:** REPL options (editor, colors, etc.)

## Key Abstractions

**Term (Core.TT.Term):**
- Purpose: Fully elaborated type-checked terms with dependent types
- Examples: `Ref` (variable/function reference), `Bind` (lambda/Pi/Let), `App` (application), `Case` (pattern matching)
- Parameterized by variable scope: `Term vars` where `vars : List Name`
- Enables type-safe variable capture via dependent types

**GlobalDef (Core.Context):**
- Purpose: Represents a top-level definition in the global context
- Contains: Type, definition form (PMDef, DCon, ExternDef, Builtin, etc.), visibility, coverage status, metadata
- Lazily loaded/unloaded to manage memory; encoded in binary `.ttc` files when referenced

**CExp (Core.CompileExpr) and variants:**
- Purpose: Simpler term representation for code generation (no type info, simplified structure)
- Related: `NamedDef` (CExp with metadata), `LiftedDef` (lambda-lifted), `ANFDef` (ANF-converted), `VMDef` (VM bytecode)
- Pipeline: CExp → LiftedDef → ANFDef → backend-specific code

**CaseTree (Core.Case.CaseTree):**
- Purpose: Efficient pattern matching decision tree
- Contains: `Case` nodes (scrutinee + alternatives), `Leaf` (final value)
- Built by `CaseBuilder` from pattern matching definition

**Codegen (Compiler.Common):**
- Purpose: Backend abstraction interface
- Methods: `compileExpr` (compile Term to file), `executeExpr` (interpret Term), `incCompileFile` (incremental)
- Implementations: Chez, Racket, RefC, JavaScript, Interpreter backends

## Entry Points

**Primary Entry Point:**
- Location: `src/Idris/Main.idr`
- Triggers: `idris2` command invocation
- Responsibilities: Minimal entry; delegates to `Compiler.Common.mainWithCodegens`

**Driver Entry Point:**
- Location: `src/Idris/Driver.idr` (`mainWithCodegens` function)
- Triggers: Called from Main
- Responsibilities: 
  - Parse command-line options (CLOpt)
  - Initialize context and state refs
  - Load environment configuration (IDRIS2_* vars)
  - Route to appropriate sub-action: compile, REPL, IDE mode, Yaffle, etc.
  - Call `Idris.ProcessIdr` for compilation

**Processing Entry Point:**
- Location: `src/Idris/ProcessIdr.idr` (`processTTCFile` for incremental, `process` for main module)
- Responsibilities: Load module files, orchestrate parsing → desugaring → elaboration → compilation

**REPL Entry Point:**
- Location: `src/Idris/REPL.idr` (`readLine`, `interpret`)
- Triggers: Interactive mode without input files
- Responsibilities: Read commands, elaborate on-the-fly, display results

## Error Handling

**Strategy:** Multi-level error recovery with postponement of non-fatal errors

**Patterns:**
- `Core` monad stacks `Either Error` for error propagation
- `catch` combinator to recover from errors and log them
- Delayed constraint checking: unsolved constraints don't immediately fail; recorded in unification state
- Coverage checking deferred; reported separately from elaboration errors
- Compiler continues to next module on parsing errors (error accumulation)

**Error Types** (from `Core.Core` and `Idris.Error`):
- `Fatal` error: unrecoverable, stops compilation immediately
- Type errors: CantConvert, CantSolveEq, PatternVariableUnifies
- Elaboration errors: UndefinedName, NotFunctionType, ArgsNotLinear
- Coverage errors: NonCoveringFunction, PartialFunction
- User errors: WithDoc (custom messages with hints)

## Cross-Cutting Concerns

**Logging:** 
- `Core.Context.Log` provides logging mechanism
- Configured via `--log` CLI flag
- Topics by module (e.g., "elab", "parse", "refc")
- Logged to stderr during compilation

**Metadata:**
- `Core.Metadata` tracks definition locations (FC), types, documentation
- Used by IDE mode, hover tooltips, documentation generation
- Serialized in `.ttc` files for faster IDE queries

**Binary Format:**
- `Core.Binary` (TTC = Idris Term Code) encodes definitions efficiently
- Module cache at `build/ttc/` directory
- Prevents re-elaboration on subsequent compiles
- Includes checksums for dependency tracking

**Incremental Compilation:**
- `Compiler.Common.UsePhase` controls how far to compile (Cases, Lifted, ANF, VMCode)
- Backends can override `incCompileFile` to compile per-source-file instead of whole module
- Scheme backends use this for faster development iteration

---

*Architecture analysis: 2026-04-21*
