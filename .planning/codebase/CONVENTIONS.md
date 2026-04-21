# Coding Conventions

**Analysis Date:** 2026-04-21

## Naming Patterns

**Files:**
- CamelCase for Idris module files: `Core.idr`, `CompileExpr.idr`
- Module names match hierarchy: File at `src/Core/Context.idr` has module `Core.Context`
- Files in subdirectories use dot-separated module names: `src/Compiler/Opts/CSE.idr` becomes module `Compiler.Opts.CSE`
- Lowercase for configuration and shell scripts: `lint.py`, `testutils.sh`

**Functions:**
- camelCase for function names: `getPosition`, `newEntry`, `addCtxt`, `initCtxt`
- Underscore separator for helper/private functions: `_lint_file`, `_awk_clean_name`
- All-uppercase for constants: `DATACON`, `TYCON`, `ENUM`, `NIL`
- Private helpers use leading underscore convention

**Variables:**
- camelCase for local variables: `arr`, `idx`, `ctxt`
- Single letter for type parameters: `a`, `b`, `x`, `y`
- Underscore for bindings that won't be used: `_ => ...`
- Named pattern matches: `(n, i, gdef)` for tuple destructuring

**Types:**
- PascalCase for type constructors: `CExp`, `CConAlt`, `ConInfo`, `InlineOk`
- Record constructor prefixed with `Mk`: `MkConAlt`, `MkConstAlt`, `MkContext`
- Data constructors follow type style: `DATACON`, `TYCON`, `CLam`, `CLet`, `CApp`
- Phantom/marker types: `SortTag` for reference tag

## Code Style

**Formatting:**
- 2-space indentation for `.idr` files (set in `.editorconfig`)
- 4-space indentation for C files and Python files
- No trailing whitespace (enforced by `lint/lint.py`)
- Newline at end of file (enforced by linter)
- UTF-8 encoding with LF line endings

**Linting:**
- Custom Python linter: `lint/lint.py` checks for trailing whitespace and final newlines
- Runs in CI on all `.idr` files before accepting PRs
- No external formatter tool referenced, formatting is manual/by convention
- EditorConfig file (`.editorconfig`) specifies indent settings for all file types

## Import Organization

**Order:**
1. Core language imports: `import Core.Context`, `import Core.TT`
2. Standard library imports: `import Data.List`, `import Data.Vect`
3. Internal library imports: `import Libraries.Data.IntMap`, `import Libraries.Text.Parser`
4. Syntax/parser imports: `import Idris.Syntax`, `import Parser.Source`
5. Public re-exports: `import public Core.Context.Context`

**Path Aliases:**
- Idris uses hierarchy-based module paths (no alias syntax)
- Standard prelude re-exports use `public`: `import public Builtin`, `import public Prelude.Basics as Prelude`
- Qualified imports use full module path when needed

**Example from `src/Core/Context.idr`:**
```idris
import        Core.Case.CaseTree
import        Core.CompileExpr
import public Core.Context.Context
import public Core.Core
import        Core.Env
import        Core.Hash
import public Core.Name
import        Core.Options
import public Core.Options.Log
import public Core.TT

import Libraries.Utils.Binary
import Libraries.Utils.Path
import Libraries.Utils.Scheme
import Libraries.Text.PrettyPrint.Prettyprinter

import Idris.Syntax.Pragmas

import Data.Either
import Data.IOArray
import Data.List1
```

## Error Handling

**Patterns:**
- Use `throw` for exceptions: `throw (InternalError "Can't happen caseLam 1")`
- Pattern match with `Nothing => throw ...` for missing cases
- Data type constructors for errors: `MkNmError`, `MkAError`, `MkLError`
- `Core` monad for error propagation: `Core a` type signature
- Use `do` notation with `<-` for sequential operations

**Example:**
```idris
getTriple : Ref SortTag SortST => Name -> Core (Maybe (Name,FC,NamedDef))
getTriple n = map (lookup n . map) (get SortTag)

checkCrash : Ref SortTag SortST => (Name, FC, NamedDef) -> Core ()
checkCrash (n, _, MkNmError _) = update SortTag $ { nonconst $= insert n }
checkCrash (n, _, MkNmFun args (NmCrash {})) = update SortTag $ { nonconst $= insert n }
```

## Logging

**Framework:** `console` for print operations, no dedicated logging library

**Patterns:**
- Use `log` function for debug output: `log "doc.implementation" 20 $ "Got name \{show @{Raw} kn}"`
- Log levels: 10 (info), 20 (debug), higher numbers for more verbose
- String interpolation with `\{...}` syntax
- Conditional logging based on log level

**Example from `src/Idris/Doc/Display.idr`:**
```idris
log "doc.implementation" 20 $ "Got name \{show @{Raw} kn}"
log "doc.implementation" 10 $ "Invalid name \{show @{Raw} kn}"
```

## Comments

**When to Comment:**
- Use block comments `{- ... -}` for multi-line documentation
- Use line comments `-- ...` for single-line explanations
- Comment non-obvious logic and complex pattern matches
- Don't comment obvious code like simple constructors

**JSDoc/TSDoc:**
- Use triple-pipe `|||` for documentation comments on exported items
- Place before function signature: `||| Check if the given name has been hidden by the %hide directive.`
- Include parameter docs with `@param_name`: `||| @fc definition site`
- Used for IDE tooltips and documentation generation

**Example from `src/Core/Context.idr`:**
```idris
||| Check if the given name has been hidden by the `%hide` directive.
export
isHidden : Name -> Context -> Bool

||| Produce a new global definition with a lot of default values
||| @fc   definition site
||| @n    name
||| @rig  quantity annotation
export
newDef : FC -> Name -> Rigidity -> GlobalDef
```

## Function Design

**Size:** 
- Keep functions concise; average 5-15 lines
- Complex logic split into helper functions
- Each function does one thing well

**Parameters:**
- Use pattern matching in parameters: `getPosition (Resolved idx) ctxt = pure (idx, ctxt)`
- Explicit type annotations for top-level functions
- Use named record fields for better readability

**Return Values:**
- Always explicitly type functions: `initCtxtS : Int -> Core Context`
- Use `Core` monad for operations with side effects or errors
- Use `Maybe a` for optional values
- Use `Either e a` rarely; prefer `throw` in Core monad

**Example from `src/Compiler/Opts/ToplevelConstants.idr`:**
```idris
callGraph : List (Name, FC, NamedDef) -> CallGraph
callGraph = fromList . map (\(n,_,d) => (n, defCalls d))

isRecursive : CallGraph -> List1 Name -> Bool
isRecursive g (x ::: Nil) = maybe False (contains x) $ lookup x g
isRecursive _ _           = True

recursiveFunctions : CallGraph -> SortedSet Name
recursiveFunctions graph =
  let groups := filter (isRecursive graph) $ tarjan graph
   in concatMap (SortedSet.fromList . forget) groups
```

## Module Design

**Exports:**
- Use `export` before public items: `export newEntry : Name -> Context -> Core (Int, Context)`
- Use `public export` for types that are part of the public API
- Omit `export` for internal/helper functions (makes them private)
- Functions marked with `|||` are automatically exported if marked

**Barrel Files:**
- Prelude re-exports commonly used items: `import public Prelude.Basics as Prelude`
- Each library has a main module that re-exports submodules
- Example: `src/Algebra.idr` is a barrel file

**Visibility with %hide:**
- Use `%hide Symbols.equals` to hide imported symbols from context
- Prevents name conflicts when importing modules with overlapping names
- Example from `src/Idris/Doc/Display.idr`: `%hide Symbols.equals`

**Default Coverage:**
- Many modules use `%default covering` to require total or covering definitions
- Marks functions as checked for pattern completeness
- Pattern: placed near top of module after imports

---

*Convention analysis: 2026-04-21*
