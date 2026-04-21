# External Integrations

**Analysis Date:** 2026-04-21

## Code Generation Backends & Runtime Integrations

The Idris 2 compiler integrates with multiple target runtime systems through pluggable code generators. These are the external tools and runtimes that compiled Idris code targets.

**Supported Backends:**

### Scheme Runtimes
- **Chez Scheme** - Primary backend for code generation and bootstrapping
  - Location: `src/Compiler/Scheme/Chez.idr`, `src/Compiler/Scheme/ChezSep.idr`
  - Support library: `support/chez/`
  - Backend configuration: `config.mk`
  - Required version: 10.0.0+ for Apple Silicon support
  - Used for: Default executable compilation, bootstrapping

- **Racket** - Alternative Scheme runtime
  - Location: `src/Compiler/Scheme/Racket.idr`
  - Support library: `support/racket/`
  - Alternative when Chez unavailable

- **Gambit** - Third Scheme alternative
  - Location: `src/Compiler/Scheme/Gambit.idr`
  - Support library: `support/gambit/`

### Native Code Generation
- **RefC (Reference C)** - C code generation backend
  - Location: `src/Compiler/RefC/RefC.idr`, `src/Compiler/RefC/CC.idr`
  - Support library: `support/refc/` (C headers and runtime)
  - Generates portable C code targeting any C compiler (gcc/clang)
  - Runtime components: `support/refc/memoryManagement.c`, `support/refc/prim.c`, `support/refc/runtime.c`

### JavaScript Runtimes
- **Node.js** - Server-side JavaScript execution
  - Backend identifier: `Node`
  - Support library: `support/js/`
  - Runtime support: `support/js/support.js`, `support/js/support_system_*.js`
  - File I/O support: `support/js/support_system_file.js`
  - System operations: `support/js/support_system_directory.js`, `support/js/support_system_clock.js`

- **JavaScript (Browser)** - Browser-based JavaScript execution
  - Backend identifier: `Javascript`
  - Generates standard JavaScript for use in browsers and other JS environments

### Interpreter
- **VMCodeInterp** - Built-in VM code interpreter
  - Location: `src/Compiler/Interpreter/VMCode.idr`
  - Used for REPL and interactive execution
  - No external runtime dependency

## FFI & Foreign Function Interface

**Supported Calling Conventions:**
- Located at: `src/Compiler/Common.idr`
- Backends declare supported FFI targets via convention names (e.g., "scheme,chez", "c", "node")
- FFI allows Idris code to call external functions in native languages

**C FFI Support:**
- C FFI infrastructure supports calling C functions from compiled code
- Support library components handle C interop: `support/c/idris_*.c`
- Used for system calls, file I/O, signals, networking

## System Integration Libraries

**File I/O & File System:**
- C support: `support/c/idris_file.c`, `support/c/idris_file.h`
- Provides file handle management and I/O operations
- Exposed through: `System.File.*` modules in base library

**System Operations:**
- Directory operations: `support/c/idris_directory.c`
- Signal handling: `support/c/idris_signal.c`, `support/c/idris_signal.h`
- System/process interface: `support/c/idris_system.c`
- Network operations: `support/c/idris_net.c`

**Memory Management:**
- C runtime memory: `support/c/idris_memory.c`, `support/c/idris_memory.h`
- Reference counting and garbage collection
- Used by all code generators that compile to native code

**Terminal/Console Support:**
- Terminal capabilities: `support/c/idris_term.h`
- System utilities: `support/c/idris_util.h`
- Getline support: `support/c/getline.c`, `support/c/getline.h`

## Documentation & Build Integration

**Read the Docs:**
- Configuration: `.readthedocs.yaml`
- Python 3.12 runtime for building documentation
- Sphinx documentation generation
- Hosted at: https://idris2.readthedocs.io

**Nix Package Management:**
- Flake definition: `flake.nix`
- Declares build dependencies via nixpkgs
- Provides reproducible development environments
- Pre-built packages available for multiple systems

## Language Server & Editor Integration

**Idris 2 LSP (external dependency):**
- Not bundled but installed via Pack package manager
- Provides IDE support via Language Server Protocol
- GitHub: https://github.com/idris-community/idris2-lsp

**Editor Support:**
- Emacs mode: `nix/init.el` (Nix integration)
- VS Code support (via LSP)
- Vim/Neovim support (via LSP)

## Standard Library Dependencies

**Standard Libraries:**
- Prelude: `libs/prelude/` - Core definitions (zero external dependencies)
- Base: `libs/base/` - Extended standard library
- Network: `libs/network/` - Network operations
- Linear: `libs/linear/` - Linear types support
- Contrib: `libs/contrib/` - Community contributions
- Test: `libs/test/` - Testing utilities
- Papers: `libs/papers/` - Research paper implementations

All libraries depend on system integration through FFI and C support library.

## Bootstrap & Build Tools

**Scheme Bootstrapping:**
- Uses pre-generated Scheme code from previous Idris version
- Located: `bootstrap/idris2_app/`
- Stage 1: `bootstrap-stage1-chez.sh`, `bootstrap-stage1-racket.sh` - Compile pre-generated Scheme
- Stage 2: `bootstrap-stage2.sh` - Self-host compilation

**Build Integration:**
- GNU Make
- Platform detection via `config.mk`
- GMP library (required for arithmetic)
- Hash function (sha256sum/gsha256sum/openssl - auto-detected)

---

*Integration audit: 2026-04-21*
