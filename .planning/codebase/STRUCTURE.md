# Codebase Structure

**Analysis Date:** 2026-04-21

## Directory Layout

```
palisade-jvm/
├── src/                           # Idris 2 compiler source code (written in Idris)
│   ├── Algebra.idr               # Algebraic structures (Monoid, Group, etc.)
│   ├── Idris/                    # High-level Idris language features
│   ├── TTImp/                    # Typed Template Implicit (intermediate representation)
│   ├── Parser/                   # Low-level parsing primitives
│   ├── Core/                     # Core type theory and context management
│   ├── Compiler/                 # Code generators and IR transformations
│   ├── Libraries/                # Standard library implementations (written in Idris)
│   ├── Protocol/                 # IDE protocol (S-expressions)
│   └── Yaffle/                   # Proof mode / tactic framework
├── libs/                          # Idris standard libraries
│   ├── prelude/                  # Core prelude (loaded by default)
│   ├── base/                     # Base library (system, data structures)
│   ├── contrib/                  # Community contributions
│   ├── linear/                   # Linear types library
│   ├── network/                  # Network I/O library
│   └── test/                     # Testing framework library
├── support/                       # C/Scheme runtime support code
│   ├── c/                        # C runtime for RefC backend
│   ├── chez/                     # Chez Scheme includes
│   ├── racket/                   # Racket runtime
│   ├── gambit/                   # Gambit runtime
│   ├── js/                       # JavaScript runtime
│   └── refc/                     # RefC reference counting runtime
├── bootstrap/                     # Bootstrap compiler (pre-built Idris)
├── docs/                          # Documentation source (Sphinx)
├── samples/                       # Example Idris programs
├── tests/                         # Test suite
├── Makefile                       # Build system
├── config.mk                      # Build configuration
├── idris2.ipkg                    # Compiler package definition
└── idris2api.ipkg                 # API package for developing tools
```

## Directory Purposes

**`src/Idris/`:**
- Purpose: High-level Idris-specific language features
- Contains: Parser for Idris syntax (higher level than TTImp), desugaring logic, REPL, IDE protocol, command-line interface
- Key files: 
  - `src/Idris/Main.idr`: Entry point
  - `src/Idris/Parser.idr`: Top-level Idris parser (desugars to TTImp)
  - `src/Idris/Desugar.idr`: High-level syntax transformations (infix, do-notation, etc.)
  - `src/Idris/ProcessIdr.idr`: File processing and module loading
  - `src/Idris/REPL.idr`: Interactive command loop
  - `src/Idris/Error.idr`: Error types and formatting

**`src/TTImp/`:**
- Purpose: Typed Template Implicit — intermediate representation for type checking
- Contains: Elaborator, declaration processor, implicit binding, partial evaluation
- Key files:
  - `src/TTImp/TTImp/TTImp.idr`: TTImp term definition
  - `src/TTImp/Elab/Check.idr`: Type checking and elaboration
  - `src/TTImp/ProcessDecls/`: Declaration processing (data, records, functions, etc.)
  - `src/TTImp/Elab/Term.idr`: Core elaboration logic
  - `src/TTImp/ProcessRunElab.idr`: Elaborator reflection support

**`src/Parser/`:**
- Purpose: Low-level lexing and parser combinator primitives
- Contains: Tokenizer rules, character-level parser combinators, source location tracking
- Key files:
  - `src/Parser/Lexer/`: Tokenization rules for symbols, keywords, identifiers
  - `src/Parser/Rule/`: Core parser combinators (sequential, choice, lookahead)
  - `src/Parser/Support/`: Parser utilities (escaping, locations)

**`src/Core/`:**
- Purpose: Core type theory implementation and definition context
- Contains: Term types, type checking primitives, context management, case compilation, binary serialization
- Key files:
  - `src/Core/TT.idr`: Term type definition (Ref, Bind, App, Case, etc.)
  - `src/Core/TT/Term.idr`: Core term constructors
  - `src/Core/Context.idr`: Global definition context and metadata
  - `src/Core/Unify.idr`: Unification algorithm
  - `src/Core/Case/CaseBuilder.idr`: Pattern matching to case tree compiler
  - `src/Core/CompileExpr.idr`: Term → CExp compilation
  - `src/Core/Binary.idr`: TTC (binary cache) serialization
  - `src/Core/Env.idr`: Variable environment management

**`src/Compiler/`:**
- Purpose: Code generation and IR transformations
- Contains: Backend implementations, IR passes (lambda lifting, ANF, VMCode)
- Subdirectories:
  - `src/Compiler/Scheme/`: Scheme backends (Chez, Racket, Gambit, ChezSep)
  - `src/Compiler/RefC/`: C code generation via reference counting
  - `src/Compiler/ES/`: JavaScript/Node.js backends
  - `src/Compiler/Interpreter/`: Direct bytecode interpretation
  - `src/Compiler/Opts/`: Optimization passes
- Key files:
  - `src/Compiler/Common.idr`: Backend abstraction (Codegen interface)
  - `src/Compiler/CompileExpr.idr`: CExp codegen (function application, constructors)
  - `src/Compiler/LambdaLift.idr`: Closure conversion (nested functions → top-level)
  - `src/Compiler/ANF.idr`: Administrative Normal Form conversion
  - `src/Compiler/VMCode.idr`: Virtual machine bytecode representation
  - `src/Compiler/Inline.idr`: Function inlining pass

**`src/Libraries/`:**
- Purpose: Idris standard library implementations (written in Idris itself)
- Contains: Core data structures and utilities that don't depend on external code
- Key files:
  - `src/Libraries/Data/`: Lists, maps, sets, vectors, SnocLists
  - `src/Libraries/Text/Parser`: General parser combinator library
  - `src/Libraries/Text/Lexer`: Lexer combinator library
  - `src/Libraries/Text/PrettyPrint/`: Pretty-printing library
  - `src/Libraries/System/`: File I/O, directory operations

**`src/Protocol/`:**
- Purpose: IDE protocol support (LSP via S-expressions)
- Contains: S-expression parsing, IDE mode message handling
- Key files:
  - `src/Protocol/SExp/SExp.idr`: S-expression type
  - `src/Protocol/IDE/`: IDE protocol message types

**`src/Yaffle/`:**
- Purpose: Proof mode (tactic framework) and interactive proof development
- Contains: Tactic types, proof state management, elaborator reflection

**`libs/prelude/`:**
- Purpose: Idris prelude — automatically loaded by every module
- Contains: Primitive operations, fundamental types (Nat, List, Bool)
- Key file: `libs/prelude/Prelude.idr`

**`libs/base/`:**
- Purpose: Base library with essential data structures and functions
- Contains: List operations, control flow (Maybe, Either), monadic interfaces
- Depends on: prelude

**`support/c/`:**
- Purpose: C runtime support for RefC backend
- Contains: Memory management, garbage collection, string handling, FFI wrappers
- Key file: `support/c/idris_runtime.c`

**`support/chez/`, `support/racket/`, etc.:**
- Purpose: Scheme-specific runtime libraries and FFI definitions
- Used by: Scheme backends to provide built-in operations

**`bootstrap/`:**
- Purpose: Pre-compiled Idris compiler (used to bootstrap the build)
- Contains: Compiled `.sch` or `.ss` files from previous build

**`tests/`:**
- Purpose: Test suite for Idris compiler and libraries
- Structure: Mirrored after functionality being tested
- Examples: `tests/idris2/` (Idris behavior), `tests/refc/` (RefC backend)

## Key File Locations

**Entry Points:**
- `src/Idris/Main.idr`: Minimal entry point
- `src/Idris/Driver.idr`: Main driver logic (orchestrates compilation)

**Configuration:**
- `Makefile`: Build orchestration
- `config.mk`: Build configuration (OS, Scheme executable, prefix paths)
- `idris2.ipkg`: Compiler package definition (dependencies, entry point)
- `idris2api.ipkg`: API package for tool development

**Core Compilation:**
- `src/Compiler/Common.idr`: Backend interface
- `src/Compiler/CompileExpr.idr`: Main compilation pass
- `src/Compiler/Scheme/Chez.idr`: Chez Scheme backend (reference implementation)

**Type System:**
- `src/Core/TT.idr`: Term type definition
- `src/Core/Context.idr`: Context and definition storage
- `src/Core/Unify.idr`: Unification and constraint solving

**Elaboration:**
- `src/TTImp/Elab/Check.idr`: Type checking
- `src/TTImp/ProcessDecls/`: Declaration elaboration

## Naming Conventions

**Files:**
- `.idr` files: Idris source code (primary language for compiler implementation)
- `.ipkg` files: Idris package definitions (metadata)
- `.ttc` files: Binary cached modules (TTC = Idris Term Code format)
- `.sch` or `.ss` files: Generated Scheme code (compiled output)
- `.c` files in support/: C runtime code

**Directories:**
- Lowercase with slashes: `src/Compiler/`, `src/Core/Case/`
- CamelCase module names map to directory structure: module `Compiler.Scheme.Chez` → `src/Compiler/Scheme/Chez.idr`
- Underscores in directory names indicate special/tooling directories: `support/`, `bootstrap/`

**Module Names:**
- Qualified paths: `Compiler.Common`, `Core.Context`, `Idris.Parser`
- Corresponds directly to file paths (without `.idr`): `Compiler.Common` → `src/Compiler/Common.idr`

**Functions & Types:**
- `camelCase` for functions: `compileExpr`, `elabTerm`, `addGlobalDef`
- `PascalCase` for type constructors: `Term`, `GlobalDef`, `Codegen`, `CaseTree`
- `UPPERCASE_SNAKE` for constants and tags: `UsePhase`, type variants use PascalCase

## Where to Add New Code

**New Feature (Language Feature):**
- Primary code: `src/Idris/` (syntax handling) → `src/TTImp/ProcessDecls/` (elaboration)
- Core representation: `src/Core/TT.idr` (extend Term type if needed)
- Backend support: `src/Compiler/Scheme/Chez.idr` as reference, add to each backend
- Tests: `tests/idris2/` (matching feature subdirectory)

**New Backend/Code Generator:**
- Backend module: `src/Compiler/MyBackend/MyBackend.idr`
- Implement: `Codegen` record from `src/Compiler/Common.idr`
- Required methods: `compileExpr`, `executeExpr`, optional `incCompileFile`
- Integration: Add to `Compiler.Common.getCG` function
- Support files: Add runtime code to `support/mybackend/` if needed

**New Data Structure / Utility:**
- Library code: `src/Libraries/Data/` (for general data structures)
- Core utilities: `src/Core/` (if type-system critical)
- Module naming: Match purpose (e.g., `src/Libraries/Data/MyStructure.idr`)
- Import consideration: Core modules imported by many targets; Library modules have fewer dependents

**Compiler Pass / Optimization:**
- Location: `src/Compiler/Opts/` (for optimization) or `src/Compiler/YourPass.idr`
- Input/Output: Operate on `CExp`, `LiftedDef`, `ANFDef`, or `VMDef`
- Integration: Wire into `Compiler.CompileExpr` compilation pipeline
- Backend interaction: May need tuning per backend

**New Optimization / Transformation:**
- Location: `src/Compiler/Opts/YourOpt.idr`
- Applied: During `compileToANF` or specific backend phase
- Performance tracking: Use `logTime` logging for measurements

## Special Directories

**`build/`:**
- Purpose: Build artifacts
- Generated: Yes
- Committed: No (in .gitignore)
- Structure:
  - `build/exec/idris2-app/`: Compiled application with runtime
  - `build/ttc/`: Module caches (.ttc files)
  - `build/env/`: Test environment

**`bootstrap/`:**
- Purpose: Bootstrap compiler (pre-built executable or source)
- Generated: Yes (by build from previous version)
- Committed: Yes (checked in to facilitate bootstrapping)
- Used for: Initial compilation of new compiler version

**`.planning/`:**
- Purpose: GSD planning and analysis (this directory)
- Generated: By GSD mapping agent
- Committed: Yes
- Contents: Architecture docs, structure docs, concerns analysis, etc.

**`docs/`:**
- Purpose: Sphinx documentation (renders to HTML/PDF)
- Type: RST (reStructuredText) source files
- Build: `make docs`

**`libs/*/build/`:**
- Purpose: Per-library build artifacts
- Generated: Yes
- Contents: `.ttc` files for each library

---

*Structure analysis: 2026-04-21*
