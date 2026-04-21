# Testing Patterns

**Analysis Date:** 2026-04-21

## Test Framework

**Runner:**
- Custom golden test framework built in Idris2 itself: `Test.Golden`
- Test harness: `tests/Main.idr` compiles to executable that runs all test suites
- Compiled with: `make testenv` then `make test`

**Assertion Library:**
- Golden file comparison: expected output vs actual output
- No dedicated assertion library; uses string comparison of outputs

**Run Commands:**
```bash
make test                              # Run all tests
make test only=chez                    # Run tests matching pattern (e.g., Chez backend)
make test only=idris2/basic001         # Run specific test
make test only=ttimp/basic             # Run tests in category
make retest                            # Re-run failed tests from failures file
make test INTERACTIVE=''               # Non-interactive mode
make test except="idris2/repl005"      # Exclude specific tests
make test THREADS=4                    # Run with 4 parallel threads
```

## Test File Organization

**Location:**
- `tests/` directory at repository root
- Organized by feature/backend: `tests/idris2/`, `tests/chez/`, `tests/ttimp/`, `tests/refc/`, etc.
- Each test suite is a separate Idris module in `tests/Main.idr`

**Naming:**
- Test directories: `{category}{nnn}` pattern (e.g., `basic001`, `basic002`, `envflags001`)
- Not strictly numeric but conventionally three-digit suffix after descriptive name
- Examples: `tests/idris2/basic/basic001/`, `tests/chez/chez001/`, `tests/ttimp/basic/ttimp001/`

**Structure:**
```
tests/
├── Main.idr              # Test harness: defines all test suites
├── testutils.sh          # Shell utilities for test scripts
├── tests.ipkg            # Package metadata for test suite
├── Makefile              # Test execution targets
├── README.md             # Testing documentation
├── idris2/               # Idris language tests
│   ├── basic/            # Fundamental language features
│   │   ├── basic001/
│   │   ├── basic002/
│   │   └── ...
│   ├── error/            # Error messages
│   ├── coverage/         # Coverage checking
│   ├── termination/      # Termination checking
│   ├── linear/           # Quantities and linearity
│   └── ...
├── chez/                 # Chez Scheme backend tests
├── refc/                 # Reference counting C backend tests
├── node/                 # Node.js backend tests
├── ttimp/                # TTImp (internal representation) tests
├── prelude/              # Prelude library tests
└── allschemes/           # Tests across all Scheme backends
```

## Test Structure

**Suite Organization in `tests/Main.idr`:**
```idris
ttimpTests : IO TestPool
ttimpTests = testsInDir "ttimp" "TTImp"

idrisTestsBasic : IO TestPool
idrisTestsBasic = testsInDir "idris2/basic" "Fundamental language features"

chezTests : IO TestPool
chezTests = testsInDir "chez" "Chez backend" {codegen = Just Chez}

main : IO ()
main = (runner =<<) $ sequence $
  [ ttimpTests
  , idrisTestsBasic
  , idrisTestsError
  , ...
  ]
  ++ map idrisTestsAllSchemes [Chez, Racket]
  ++ map idrisTestsAllBackends [Chez, Node, Racket, C]
```

**Individual Test Case Structure:**

Each test is a directory with:
1. **Idris source file(s)** — the code being tested
2. **`run` script** — shell script that invokes idris2 with the test
3. **`expected` file** — the expected output (golden file)
4. **`input` file** (optional) — standard input for the test

**Example: `tests/idris2/basic/basic001/`**
```
basic001/
├── Vect.idr      # Idris code defining data types and functions
├── run           # Shell script: . ../../../testutils.sh; idris2 --no-prelude Vect.idr < input
├── input         # Input to send to idris2 REPL
└── expected      # Expected output after running run script
```

**Example run script from `tests/idris2/basic/basic001/run`:**
```bash
. ../../../testutils.sh

idris2 --no-prelude Vect.idr < input
```

**Example run script from `tests/chez/chez001/run`:**
```bash
. ../../testutils.sh

run Total.idr
```

## Test Data Structure

**Idris Code Example (`tests/idris2/basic/basic001/Vect.idr`):**
```idris
data Nat = Z | S Nat

plus : Nat -> Nat -> Nat
plus Z     y = y
plus (S k) y = S (plus k y)

data Vect : Nat -> Type -> Type where
     Nil  : Vect Z a
     Cons : a -> Vect k a -> Vect (S k) a

reverse : Vect n a -> Vect n a
reverse
    = foldl (\m => Vect m a)
            (\rev => \x => Cons x rev) Nil
```

**Shell Test Wrapper (`tests/testutils.sh`):**
- Provides utility functions: `idris2()`, `check()`, `run()`, `clean_names()`
- Handles name normalization in output (replaces IDs with generic names for stability)
- Manages environment setup: `IDRIS2_PREFIX`, `IDRIS2_PACKAGE_PATH`, `LC_ALL` for reproducibility
- AWK script `_awk_clean_name` normalizes machine-specific output:
  - Replaces hex IDs with generic counters: `P:xyz:12345` → `P:xyz:1`
  - Replaces file cache references: `ttc/1234567890` → `ttc/1`
  - Replaces resolved variable IDs: `$resolved123` → `$resolved1`
  - Makes output stable across different runs and machines

**Location of shared utilities:**
- `tests/testutils.sh` — sourced by all test run scripts
- Used at top of run script: `. ../../../testutils.sh` (relative path depends on nesting depth)

## Mocking

**Framework:** Not applicable — golden file tests compare actual compiler/REPL output

**What to Test:**
- Type checking behavior
- Compilation to target backends (Chez, Node, RefC)
- REPL commands and output
- Error messages with proper formatting
- Library functionality across backends

**What NOT to Test:**
- External system dependencies in core tests
- Platform-specific behavior (with exceptions for known issues like `issue2362` for RefC)
- Non-deterministic output (tests must be reproducible)

## Test Fixtures and Factories

**Test Data:**
- No factory pattern; tests use inline Idris code in `.idr` files
- Shared test utilities defined in individual Idris files when needed
- Example: `tests/idris2/basic/basic001/Vect.idr` defines test data types inline

**Fixture Organization:**
- Test libraries available: `libs/test/` contains testing utilities
- Base library tests in `tests/base/` run against multiple codegens
- Contrib library tests in `tests/contrib/`
- Individual `.idr` files are self-contained

**Location:**
- Test source files live in test directories: `tests/idris2/basic/basic001/Vect.idr`
- Reusable test definitions in library Idris files: `libs/test/src/Test/Golden.idr`
- No separate fixture directories; data is embedded in test files

## Coverage

**Requirements:** No hard target enforced

**Implicit Coverage Goals:**
- Frontend tests: every language feature should have at least one test
- Compiler tests: critical transformations (ANF, LambdaLift, case optimization)
- Backend tests: each backend (Chez, Node, RefC) must successfully compile and run
- Error tests: important error messages and edge cases

**View Coverage:**
- No automated coverage tool configured
- Coverage assessed manually by reviewing test directory structure
- Gaps evident in absence of test directories for specific features

## Test Types

**Unit Tests (Compiler Components):**
- Scope: Type checking, elaboration, core transformations
- Approach: Input Idris program → expected compiler error or compiled output
- Location: `tests/idris2/` subdirectories for specific features
- Examples: `tests/idris2/error/` for error messages, `tests/idris2/coverage/` for coverage checking

**Integration Tests (Full Compilation):**
- Scope: From Idris source to executable/REPL interaction
- Approach: Compile with `-c` or run with `--exec`, capture output
- Location: `tests/chez/`, `tests/refc/`, `tests/node/`, `tests/racket/`
- Verify correct compilation to each backend

**End-to-End Tests (Backend Execution):**
- Scope: Compiled code actually runs and produces correct output
- Approach: Run compiled Idris, compare output to golden file
- Location: Test cases with `run Total.idr` in the run script
- Example: `tests/chez/chez001/run` executes compiled code

**Language Feature Tests:**
- Scope: Specific language constructs (pattern matching, recursion, generics, etc.)
- Approach: Type-check or compile Idris code with specific features
- Location: Organized by feature in `tests/idris2/`
  - `basic/` — fundamental features
  - `linear/` — quantity checking
  - `with/` — with abstraction
  - `operators/` — operator definitions
  - `reflection/` — quotation and reflection

## Common Patterns

**REPL Testing:**
```bash
# Test provides Idris code and expected REPL output
idris2 --no-prelude Vect.idr < input
# Input file contains REPL commands
# Output is compared to expected file (with name normalization applied)
```

**Compilation Testing:**
```bash
# From tests/chez/chez001/run
. ../../testutils.sh
run Total.idr
# run function compiles and executes with: idris2 --exec main
```

**Error Testing:**
```bash
# From tests/idris2/error/
# Run script invokes idris2 without --exec
# Captures compiler errors/warnings in expected file
idris2 ErrorFile.idr
```

**Output Normalization (Golden Files):**
- `testutils.sh` provides `clean_names()` function
- AWK script replaces all unique identifiers with generic counters
- Ensures tests pass across machines, builds, and over time
- Example normalizations:
  - `P:Map:12345` → `P:Map:1`
  - `{Foo:999` → `{Foo:1`
  - `$resolved567` → `$resolved1`

---

*Testing analysis: 2026-04-21*
