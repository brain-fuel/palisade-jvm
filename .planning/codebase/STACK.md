# Technology Stack

**Analysis Date:** 2026-04-21

## Languages

**Primary:**
- Idris 2 - Language used to implement the compiler and standard libraries
- Scheme (Chez Scheme, Racket, Gambit) - Runtime for code generation and bootstrapping
- C - Support libraries for system operations and runtime primitives

**Secondary:**
- JavaScript - Runtime support and FFI interop for Node.js backend
- Bash - Build and bootstrap scripts

## Runtime

**Environment:**
- Chez Scheme (primary, version 10.0.0+)
- Racket (alternative Scheme runtime)
- Gambit Scheme
- Node.js (for javascript/node backends)
- C runtime (for RefC backend)

**Package Manager:**
- Nix Flakes (development environment via `flake.nix`)
- Pack (external ecosystem package manager for Idris packages)

## Frameworks

**Core Language:**
- Idris 2 0.8.0 (the language itself)

**Code Generation Backends:**
- Chez (Scheme-based backend - default)
- ChezSep (Separate code generation variant)
- Racket (Scheme runtime alternative)
- Gambit (Scheme alternative)
- Node (JavaScript runtime compilation)
- Javascript (Browser/JavaScript target)
- RefC (C code generation backend)
- VMCodeInterp (VM code interpreter)

**Build System:**
- GNU Make (primary build tool)
- Nix (package management and development environments)

**Testing:**
- Built-in test framework (no external dependency)
- Located at: `tests/` directory with driver in Makefile

**Documentation:**
- Sphinx (Python documentation generator)
- Read the Docs configuration file: `.readthedocs.yaml`
- Markdown support via MyST

## Key Dependencies

**Critical:**
- GMP (GNU Multiple Precision library) - Required for arithmetic
- Chez Scheme (default backend and bootstrap compiler)

**Build/Development:**
- clang/gcc - C compiler for support libraries
- Bash - Build scripts
- GNU Make

**Optional (Backend Support):**
- Racket - Alternative Scheme runtime
- Gambit - Alternative Scheme implementation
- Node.js - JavaScript backend runtime
- Nix - Reproducible builds and development shells

**Documentation:**
- myst-parser 4.0.1+ - Markdown parsing for Sphinx
- sphinx 8.1.3+ - Documentation generation
- sphinx-book-theme 1.1.4+ - Documentation theme
- sphinx-rtd-theme 3.0.2+ - ReadTheDocs theme
- sphinx-copybutton 0.5.2+ - Copy button for code blocks
- sphinxcontrib-bibtex 2.6.5+ - Bibliography support

## Configuration

**Environment:**
- `IDRIS2_CG` - Select code generator (default: chez)
- `IDRIS2_INC_CGS` - Enable incremental compilation for specific CGs
- `IDRIS2_PREFIX` - Installation prefix (default: `$HOME/.idris2`)
- `IDRIS2_PATH` - Library search paths
- `IDRIS2_DATA` - Data file directories
- `IDRIS2_LIBS` - Library directories
- `IDRIS2_PACKAGE_PATH` - Package search paths
- `SCHEME` - Scheme executable name (for bootstrap, default: chez)
- `EDITOR` - Editor for REPL
- `NO_COLOR` - Disable colored output

**Build Configuration:**
- `config.mk` - Platform-specific build settings (C flags, ranlib, ar, etc.)
- `Makefile` - Main build orchestration
- `.readthedocs.yaml` - Documentation build configuration
- `flake.nix` - Nix flake definition with templates

## Platform Requirements

**Development:**
- Linux, macOS (Apple Silicon requires Chez 10.0.0+), Windows (via MSYS2), BSD
- Bash, GNU Make (gmake on FreeBSD/OpenBSD)
- gcc or clang
- GMP development headers
- sha256sum or equivalent hash utility
- Chez Scheme with thread support (built with `--threads` flag)

**Bootstrapping:**
- Existing Idris 2 compiler (version 0.8.0+ recommended) OR
- Pre-built Scheme code with working Scheme runtime
- Two-stage bootstrap process: Stage 1 (Scheme) → Stage 2 (Idris 2 compiler)

**Production/Installation:**
- Target platform depends on selected code generator:
  - **Chez backend:** Chez Scheme runtime required
  - **RefC backend:** C compiler and standard C libraries
  - **Node/JavaScript backend:** Node.js runtime (or browser environment)

---

*Stack analysis: 2026-04-21*
