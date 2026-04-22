# Architecture Research

**Domain:** JVM compiler backend for a verified functional language (Idris 2 → JVM bytecode)
**Researched:** 2026-04-21
**Confidence:** HIGH on existing-backend parallels and idris-jvm structure (Idris 2 source + idris-jvm public tree); MEDIUM on cross-cutting decision wiring (LIMM, Defensive Membrane, JFR/SDE, etc.) — these are Palisade-original patterns inferred from the PROJECT.md decisions plus the Kotlin/Scala 3/Clojure precedent for "lowering pipeline + emitter + runtime support" decomposition.

This document is a research artifact, not a binding spec. It maps how JVM-targeted compilers are organized (Idris-jvm, Kotlin K2, Scala 3, Clojure), reconciles that against the existing `src/Compiler/{RefC,Scheme,ES,Interpreter}/` shape, and lands on a recommended internal structure for `src/Compiler/JVM/` plus the build-order implications. The 11 cross-cutting Key Decisions are mapped to specific components so the roadmap can sequence work without losing where each decision plugs in.

---

## Standard Architecture

### Pattern Across JVM-Targeted Compilers

Every production JVM compiler we surveyed (idris-jvm, Kotlin K2, Scala 3 / Dotty, Clojure) decomposes its backend into the same conceptual stages:

```
                      ┌─────────────────────────────────────────┐
                      │   Upstream IR (consumer boundary)       │
                      │   Idris2: NamedCExp / NamedDef          │
                      │   Kotlin: IR (FIR-lowered)              │
                      │   Scala3: TASTy → erased trees          │
                      │   Clojure: tools.analyzer AST           │
                      └────────────────┬────────────────────────┘
                                       │
                      ┌────────────────▼────────────────────────┐
                      │ Backend IR / Lowering Pipeline          │
                      │ • per-language lowerings                │
                      │ • specialization / type inference       │
                      │ • tail-call rewriting                   │
                      │ • closure / lambda lifting refinements  │
                      └────────────────┬────────────────────────┘
                                       │
                      ┌────────────────▼────────────────────────┐
                      │ Bytecode Emitter (ASM / ClassFile API)  │
                      │ • frame computation                     │
                      │ • method/class/field visitors           │
                      │ • debug info / source maps              │
                      └────────────────┬────────────────────────┘
                                       │
                      ┌────────────────▼────────────────────────┐
                      │ Class Files / JAR / Module assembly     │
                      │ + module-info, manifest, resources      │
                      └────────────────┬────────────────────────┘
                                       │
                      ┌────────────────▼────────────────────────┐
                      │ Runtime Support Library (Java)          │
                      │ thunks, channels, futures, FFI shims,   │
                      │ JFR event types, linearity guards       │
                      └─────────────────────────────────────────┘
```

Differences across the four are mostly local:

- **Kotlin K2** routes everything through a single shared IR, then has thin per-target backends (JVM, JS, Native) that translate IR to platform code. The lowering layer is large and platform-aware ([Kotlin compiler/ir tree](https://github.com/JetBrains/kotlin/tree/master/compiler/ir), [Crash Course on Kotlin Compiler](https://medium.com/google-developer-experts/crash-course-on-the-kotlin-compiler-k1-k2-frontends-backends-fe2238790bd8)).
- **Scala 3 / Dotty** organizes the back half as a long sequence of fused phases (`FirstTransform → … → Erasure → ElimErasedValueType → … → CollectSuperCalls → genBCode`). The bytecode emitter is the final phase; everything before it is tree-to-tree rewriting toward a JVM-shaped tree ([Scala 3 Compiler Phases](https://nightly.scala-lang.org/docs/contributing/architecture/phases.html), [Dotty Overall Structure](https://dotty.epfl.ch/docs/internals/overall-structure.html)).
- **Clojure** is form-by-form: read → analyze (AST) → emit bytecode with ASM into an `IFn`-implementing class, optionally persisted. `tools.analyzer` + `tools.emitter.jvm` factor that pipeline cleanly ([tools.emitter.jvm](https://github.com/clojure/tools.emitter.jvm), [Decompiling Clojure II](http://blog.guillermowinkler.com/blog/2014/04/21/decompiling-clojure-ii/)).
- **idris-jvm** (the existing third-party effort Palisade replaces) splits into three top-level Maven modules: `idris-jvm-compiler` (Idris-side codegen), `idris-jvm-assembler` (thin Java wrapper around ASM exposing an FFI surface), and `idris-jvm-runtime` (Java support library — thunks, futures, IO, concurrency). Inside the Idris-side compiler the modules are: `Asm`, `Codegen`, `Optimizer`, `InferredType`, `Foreign`, `Export`, `ExtPrim`, `FunctionTree`, `Variable`, `Tree`, `Tuples`, `Math`, `Jname`, `ShowUtil` ([idris-jvm GitHub tree](https://github.com/mmhelloworld/idris-jvm)). This decomposition is direct evidence of what proves necessary in practice for an Idris-on-JVM backend.

**Convergent shape across all four:** a *compiler-side* pipeline that lowers and emits, a *bytecode library boundary* (ASM today, JEP 484 ClassFile API tomorrow), and a *runtime support library* that lives outside the codegen tree. Palisade's architecture should adopt this three-tier shape verbatim.

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| **Upstream IR adapter** | Read the consumer's IR, vendor-snapshot it for stability | Direct dependency on `Compiler.CompileExpr`'s `NamedCExp`/`NamedDef`; vendored copy per Decision 38 |
| **Backend IR + lowerings** | Translate the language's IR into a JVM-shaped IR; specialize, infer, rewrite tail calls | New backend-local IR type (`JVMExpr` or similar) with `NamedCExp → JVMExpr` lowering passes |
| **Type inference / erasure** | Decide unboxed vs boxed, monomorphic vs polymorphic, retained metadata for FFI | Idris-jvm's `InferredType` (`IBool, IInt, IRef, IArray, IFunction, TypeParam, IUnknown`) is the proven shape; Palisade extends with linearity bits |
| **Optimizer** | Codegen-level passes: tail call → goto, trampoline insertion, inlining decisions, specialization | Per-function pass over backend IR before emission |
| **Name mangler** | Deterministic Idris-name → JVM-safe identifier mapping; round-trippable for source maps and profiler readability | Pure function; round-trip table cached for SDE / heap-dump tooling (Decision 23) |
| **Foreign / FFI elaborator** | Read `%foreign` specifiers; resolve Java signatures (incl. generics); emit invocation sites | Decision 4 + Decision 6 ride here; depends on elaborator-reflection metadata |
| **Bytecode emitter** | Walk backend IR, drive ASM (or ClassFile API) visitors, manage method/class/field state | One module per major construct: methods, classes, switches, lambdas |
| **Frame computer** | Stack map frames for verifier-clean output; handles trampoline + linear-resource control flow | Use ASM's auto-compute initially; precision generator for Decision 27 ("Zero-Tolerance Verification Contract") |
| **Debug info writer** | `LineNumberTable`, `LocalVariableTable`, `SourceDebugExtension` (JSR-45 SMAP) | Per Decision 21; populated for every emitted class. ASM exposes `visitLineNumber` and `cv.visitSource(name, smap)` for SMAP ([JSR-45 / SourceDebugExtension](https://jcp.org/en/jsr/detail?id=45)) |
| **Module/JAR packager** | `module-info.class` per Idris package; manifest; resources; reproducible JAR layout | Decision 40 + Decision 36; runs after all classes emitted |
| **Defensive Membrane / Guard Proxy generator** | Emit boundary proxies enforcing linearity at FFI; signature-based dispatch for zero overhead on Idris-internal paths | Decision 20; lives between FFI elaborator and emitter |
| **Annotation projector** | Project Idris-side metadata to JVM annotations (Spring `@Component`, `@Autowired`; ghost annotations for erased args; SonarQube-readable linearity markers) | Decisions 5 + 15; runs late in the lowering pipeline |
| **JFR vocabulary emitter** | Generate JFR `Event` subclasses + emission sites for `LinearFutureCompleted`, `EffectPerformed`, `LinearityViolationDetected`, `OrderedContainerAccess` | Decision 22; partly compile-time (event classes), partly runtime-library (emission helpers) |
| **Runtime support library** | Java-side: linear futures, ordered containers, scoped values, structured task scopes, channels, thunks, FFI helpers | Java module (`palisade-runtime`); the *only* part of Palisade most user code links against |

---

## Recommended Project Structure

```
src/Compiler/JVM/                       # Idris-side codegen (lives in idris2 repo)
├── Codegen.idr                         # Top-level: implements Compiler.Common.Codegen interface
├── Driver.idr                          # Pipeline orchestration: NamedDef → JAR
├── IR/
│   ├── JVMExpr.idr                     # Backend-local IR (lowered NamedCExp)
│   ├── Lower.idr                       # NamedCExp → JVMExpr translation
│   ├── InferredType.idr                # Java-type inference (per idris-jvm precedent)
│   └── Linearity.idr                   # Linearity tracking carried through lowering (Dec 17, 20)
├── Opt/
│   ├── TailCall.idr                    # Self-rec → goto; mutual → trampoline (Dec 2)
│   ├── Specialize.idr                  # @hot / @specialize forced monomorphization (Dec 16)
│   ├── ClosedWorld.idr                 # Link-time mono of 1-2 impl interfaces (Dec 16)
│   └── Erase.idr                       # Hybrid Metadata Erasure (Dec 15)
├── Asm/
│   ├── Emit.idr                        # Top-level emitter: walks JVMExpr → bytecode
│   ├── Method.idr                      # Method-level emission, frame computation
│   ├── Class.idr                       # Class-level emission, fields, ctors
│   ├── Switch.idr                      # tableswitch / lookupswitch for case trees
│   ├── InvokeDynamic.idr               # Guard-With-Test dispatch chains (Dec 16)
│   └── Frame.idr                       # Precision StackMapFrame generator (Dec 27)
├── Name/
│   ├── Mangle.idr                      # Deterministic, JVM-safe, human-readable (Dec 23)
│   └── Demangle.idr                    # Round-trip for SDE / heap-dump tooling
├── FFI/
│   ├── Foreign.idr                     # %foreign parsing + dispatch
│   ├── Importer.idr                    # Elaborator-reflection-driven Java signature import (Dec 4, 6)
│   ├── Generics.idr                    # Java generic signatures → typed Idris (Dec 6)
│   ├── Membrane.idr                    # Defensive Membrane + Guard Proxy emission (Dec 20)
│   └── ExceptionBridge.idr             # Java exception → Either JException T (Dec 8)
├── Spring/
│   ├── Annotations.idr                 # Project Idris metadata → Spring annotations (Dec 5)
│   └── DI.idr                          # Bean definition emission, lifecycle hooks
├── Debug/
│   ├── LineTable.idr                   # LineNumberTable population
│   ├── LocalVars.idr                   # LocalVariableTable population
│   └── SDE.idr                         # SourceDebugExtension / JSR-45 SMAP (Dec 21)
├── JFR/
│   ├── Events.idr                      # Generate JFR Event subclasses (Dec 22)
│   └── Emit.idr                        # Insert event-emission calls at boundaries
├── Memory/
│   ├── LIMM.idr                        # Linearity → barrier elision (Dec 17)
│   └── OrderedContainer.idr            # VarHandle acquire/release/opaque emission (Dec 17)
├── Loom/
│   ├── StructuredScope.idr             # Statically-proved StructuredTaskScope use (Dec 18)
│   └── LinearFuture.idr                # Virtual-thread lifecycle bound to LinearFuture (Dec 18)
├── Module/
│   ├── ModuleInfo.idr                  # Auto-emit module-info.class from .ipkg (Dec 40)
│   ├── Jar.idr                         # JAR packaging, reproducible byte-identical (Dec 36)
│   └── Manifest.idr                    # Manifest, resources, multi-release layout
├── Telemetry/
│   ├── OpenTelemetry.idr               # Span open/close across LinearFuture lifecycle (Dec 24)
│   └── Micrometer.idr                  # Idris-specific gauges / counters
├── Verify/
│   └── XVerify.idr                     # CI hook: java -Xverify:all gate (Dec 27)
└── Common.idr                          # Shared utilities, Show instances, helpers

support/jvm/                            # JVM runtime + ASM bridge (companion to support/c, support/chez)
├── pom.xml                             # Maven build for runtime artifact
├── palisade-asm-bridge/
│   └── src/main/java/.../              # Thin Java wrapper around ASM exposed via Idris FFI
│       ├── Assembler.java              # Mirrors idris-jvm's idris-jvm-assembler module
│       ├── ClassWriter.java            # IdrisClassWriter equivalent (custom getCommonSuperClass)
│       └── ...
└── palisade-runtime/
    └── src/main/java/.../              # User-linked runtime
        ├── Thunk.java                  # Lazy evaluation
        ├── LinearFuture.java           # The thesis primitive (Dec, requirements list)
        ├── LinearChannel.java          # Zero-copy channel (Dec 19)
        ├── OrderedContainer.java       # VarHandle-backed shared state (Dec 17)
        ├── GuardProxy.java             # Defensive Membrane runtime (Dec 20)
        ├── PalisadeLinearityViolation.java
        ├── jfr/
        │   ├── LinearFutureCompletedEvent.java
        │   ├── EffectPerformedEvent.java
        │   ├── LinearityViolationDetectedEvent.java
        │   └── OrderedContainerAccessEvent.java   (Dec 22)
        ├── concurrent/                 # StructuredTaskScope helpers, ScopedValue (Dec 18, 19)
        ├── ffi/                        # Java exception bridge, generics helpers
        └── spring/                     # Spring integration helpers, Actuator endpoints (Dec 5, 24)

build-tooling/                          # Outside src/, Decision 10
├── palisade-maven-plugin/              # Maven Mojo
└── palisade-gradle-plugin/             # Gradle plugin
```

### Structure Rationale

- **`src/Compiler/JVM/Codegen.idr` as the single Codegen interface implementation** mirrors the `src/Compiler/RefC/RefC.idr` and `src/Compiler/Scheme/Chez.idr` pattern: one entry-point file per backend that constructs the `Codegen` record (`compileExpr`, `executeExpr`, `incCompileFile`, `incExt`) defined in `Compiler.Common`. Driver.idr separates pipeline orchestration from the dispatch shim so the Codegen record stays lean.
- **`IR/`, `Opt/`, `Asm/`, `Name/` follow idris-jvm's split** (it has `Asm`, `Codegen`, `Optimizer`, `InferredType`, `Variable`, `FunctionTree`, `Jname`). That split has been validated by an independent practitioner who built essentially this backend and shipped it; we adopt the same decomposition rather than re-discovering it. We extend it with subdirectories that didn't exist in idris-jvm because the corresponding *concerns* didn't exist (LIMM, Loom, Membrane, JFR, SDE, OpenTelemetry, Spring annotations, JPMS).
- **`FFI/`, `Spring/`, `Loom/`, `Memory/`, `JFR/`, `Debug/`, `Telemetry/` are sibling concern-packages, not nested under `Asm/`.** Each represents a cross-cutting Key Decision that produces both lowering-side rewrites and emitter-side bytecode. Keeping them as siblings makes the dependency graph flat and the build order explicit: each one can be empty stubs in early phases and fleshed out per phase without restructuring.
- **`support/jvm/` parallels `support/c/`, `support/chez/`, `support/refc/`**, matching the existing convention where backend runtime support lives outside `src/`. The split into `palisade-asm-bridge/` and `palisade-runtime/` follows idris-jvm's split into `idris-jvm-assembler` + `idris-jvm-runtime`: the first is consumed only at compile time via FFI, the second is linked into every emitted JAR. Different release cadences, different audiences, different stability contracts.
- **`build-tooling/` lives outside `src/`** because Maven/Gradle plugins aren't part of the compiler — they wrap it. Same reason `idris-jvm` has `idris-jvm-maven-plugin` as a separate Maven module.
- **No `JVM/Common.idr` analogous to `Scheme/Common.idr`** is justified initially: there's only one JVM backend variant. If future variants emerge (e.g., a CLR target sharing JVM-shaped IR), refactor then.

---

## Architectural Patterns

### Pattern 1: Single-Entry Codegen Record (Idris 2 backend convention)

**What:** Backend exposes one `Codegen` record (defined in `Compiler.Common`) with four fields: `compileExpr`, `executeExpr`, `incCompileFile`, `incExt`. The Idris driver dispatches solely through this record.

**When to use:** Always — it's how every Idris 2 backend integrates. There is no other supported integration shape.

**Trade-offs:** Cannot expose backend-specific compile flags through a typed interface; falls back to `getDirectives` for backend pragmas. This is fine for v1 — Chez, RefC, ES all live with it.

**Example (from `src/Compiler/RefC/RefC.idr` and `src/Compiler/Scheme/Chez.idr` shape):**

```idris
export
codegenJVM : Codegen
codegenJVM = MkCG compileToJVM executeJVM (Just incCompileJVM) (Just "class")
```

### Pattern 2: Lowered Backend IR (NamedCExp → JVMExpr)

**What:** Don't emit bytecode directly from `NamedCExp`. Lower into a JVM-shaped IR first that carries Java-type information (boxed/unboxed, generic params, linearity bits), then walk that IR to emit. This is exactly what idris-jvm does (`InferredType` carrying `IBool, IInt, IRef String JavaReferenceType (List InferredType), IArray, IFunction`...) and what Kotlin K2 does (lowering passes between FIR and bytecode).

**When to use:** Required when the source IR (`NamedCExp`) is more abstract than the target requires. Idris's `NamedCExp` is dynamically-typed in shape (`NmApp f xs` says nothing about argument types); JVM bytecode is rigidly typed. The lowered IR is where that information is *recovered* (via inference) or *injected* (via FFI signatures, Decision 6).

**Trade-offs:** Adds a transformation step and an IR type to maintain. Pays for itself the first time you need to emit `iadd` instead of `BigInteger.add` (Decision 3) — the lowered IR is where that decision lives.

**Example shape:**

```idris
data JVMExpr : Type where
  JVar    : InferredType -> Name -> JVMExpr
  JCall   : JavaSignature -> JVMExpr -> List JVMExpr -> JVMExpr
  JIBox   : InferredType -> JVMExpr -> JVMExpr        -- explicit boxing
  JIUnbox : InferredType -> JVMExpr -> JVMExpr
  JLambda : ...                                       -- with carried lambda metadata
  JLinearGuard : LinearityMark -> JVMExpr -> JVMExpr  -- Defensive Membrane (Dec 20)
  ...
```

### Pattern 3: Three-Tier Decomposition (Compiler / Bridge / Runtime)

**What:** Separate the codegen (Idris) from the bytecode-emission bridge (Java FFI wrapping ASM) from the user-linked runtime (Java). Three release artifacts, three audiences, three stability contracts.

**When to use:** Always for JVM backends. Validated by idris-jvm's three-module Maven structure; analogous to `support/c/idris_runtime.c` being separate from the C codegen.

**Trade-offs:** More moving parts than a single-binary backend. The benefit is that `palisade-runtime` can be shipped via Maven Central as a normal JVM library, consumed by Java/Kotlin projects directly, and version-pinned independently from the compiler. Critical for Decision 30 (hybrid-module first-class — Java + Kotlin + Idris in one Spring Boot module).

### Pattern 4: ASM ClassWriter Subclass for `getCommonSuperClass`

**What:** Default `ASM ClassWriter` resolves `getCommonSuperClass` by reflection on the *compiler's* classpath, not the target's. JVM-targeted compilers always subclass `ClassWriter` to override this — idris-jvm has `IdrisClassWriter`; Kotlin, Scala, Groovy all do the same.

**When to use:** Required from day one. The default behavior is a runtime ClassNotFoundException waiting to happen the first time you generate code referring to a class not on the compiler's classpath.

**Trade-offs:** None. It's a 30-line subclass.

### Pattern 5: Source Map Population at Every Emission Point

**What:** Every `visitLineNumber` call in the emitter is paired with a corresponding entry in the SourceDebugExtension SMAP table. Every local variable gets `visitLocalVariable`. Every emitted class gets `cv.visitSource(name, smap)`.

**When to use:** From the *first* emitted bytecode. Retrofitting source maps is painful — it requires re-walking the entire emitter pipeline to thread `FC` (file:line) through every call site. Threading it from the start is free.

**Trade-offs:** Slightly more verbose emitter code. Pays for itself the first time a stack trace from a Spring Boot service points back to an Idris file:line — which is Decision 21's entire reason for existing.

**Reference:** [JSR-45 SourceDebugExtension](https://jcp.org/en/jsr/detail?id=45); ASM's `cv.visitSource(name, smap)` API; Kotlin and Scala already use this pattern in production.

### Pattern 6: Annotation Projection as a Lowering Pass

**What:** Spring annotations (`@Component`, `@Autowired`, `@RestController`, etc.), ghost annotations for erased args (Decision 15), linearity-marker annotations (Decision 15), and JFR event-class generation (Decision 22) all happen as *lowering passes* over JVMExpr, *not* during emission. The pass adds annotation metadata to the JVMExpr nodes; the emitter just consumes it.

**When to use:** When multiple Key Decisions need to attach annotations to the same emitted classes/methods. Doing it during emission means each emission site has to know about every annotation source. Doing it as lowering keeps emitters simple.

**Trade-offs:** Adds passes to the pipeline. They can be cheap (single tree-walk each) and easily composable.

### Pattern 7: Runtime Library Owns Concurrency Primitives

**What:** Linear futures, structured task scopes, ordered containers, scoped values, channels — these are *Java classes* in `palisade-runtime`, and the codegen emits bytecode that calls into them. The codegen does *not* re-implement Loom; it emits invocations.

**When to use:** Always. Loom semantics are JVM features; replicating them at the codegen level is duplicate work and a maintenance burden. The codegen's job is to enforce the *static guarantees* (every task joined or cancelled, Decision 18) and emit calls to the runtime primitives that provide the dynamic semantics.

**Trade-offs:** None substantive. This is how every JVM language interoperates with `java.util.concurrent`.

### Pattern 8: ASM Today, ClassFile API Tomorrow (deferred migration)

**What:** Use ASM for v1. Plan a migration path to JEP 484 ClassFile API ([JEP 484](https://openjdk.org/jeps/484), finalized in JDK 24). Wrap ASM in a thin Idris-side abstraction (the FFI bridge in `support/jvm/palisade-asm-bridge`) so the migration becomes "swap the bridge implementation."

**When to use:** Decide once at v1; revisit once ClassFile API has covered every edge case ASM handles today (frame computation parity, custom attributes, JSR-45 SDE). ASM remains the proven choice as of JDK 25 — the ClassFile API explicitly does not aim to obsolete ASM ([JEP 484](https://openjdk.org/jeps/484)).

**Trade-offs:** Indirection through the bridge costs a tiny bit of FFI overhead and a maintenance surface. Avoids being trapped by ASM's eventual decline once the ClassFile API matures.

---

## Data Flow

### Compilation Flow (single-module compile)

```
.idr source
    │
    ▼
[Idris frontend: parse → desugar → elab → CompileExpr]   (existing, untouched)
    │
    ▼
NamedDef list ──── (consumer boundary; Palisade starts here)
    │
    ▼
JVM/Driver.idr
    │
    ├──► JVM/IR/Lower.idr ──────► JVMExpr (lowered IR with InferredType)
    │       │
    │       ├──► JVM/IR/InferredType.idr  (Java-type inference)
    │       └──► JVM/IR/Linearity.idr     (linearity tracking, Dec 17/20)
    │
    ├──► JVM/Opt/* (lowering passes — order matters)
    │       1. Erase.idr            (Hybrid Metadata Erasure, Dec 15)
    │       2. TailCall.idr         (self-rec → goto, mutual → trampoline, Dec 2)
    │       3. Specialize.idr       (@hot / @specialize, Dec 16)
    │       4. ClosedWorld.idr      (link-time mono, Dec 16)
    │       5. Memory/LIMM.idr      (barrier elision for q=1, Dec 17)
    │
    ├──► JVM/FFI/* (FFI elaboration)
    │       Foreign.idr → Importer.idr → Generics.idr → Membrane.idr → ExceptionBridge.idr
    │       (emits Guard Proxy nodes into JVMExpr, Dec 20)
    │
    ├──► JVM/Spring/Annotations.idr     (annotation projection, Dec 5)
    ├──► JVM/Loom/StructuredScope.idr   (static task-leak proof, Dec 18)
    ├──► JVM/JFR/Emit.idr               (insert JFR event-emission calls, Dec 22)
    ├──► JVM/Telemetry/OpenTelemetry.idr (span open/close insertion, Dec 24)
    │
    ▼
JVMExpr (fully lowered, fully annotated)
    │
    ▼
JVM/Asm/Emit.idr ───────► drives ASM via support/jvm/palisade-asm-bridge
    │       │
    │       ├──► Method.idr  (per-method emission)
    │       ├──► Class.idr   (per-class emission)
    │       ├──► Switch.idr  (case trees → tableswitch/lookupswitch)
    │       ├──► InvokeDynamic.idr (Guard-With-Test chains, Dec 16)
    │       ├──► Frame.idr   (precision StackMapFrame, Dec 27)
    │       │
    │       ├──► Debug/LineTable.idr   (visitLineNumber, Dec 21)
    │       ├──► Debug/LocalVars.idr   (visitLocalVariable, Dec 21)
    │       └──► Debug/SDE.idr         (SourceDebugExtension SMAP, Dec 21)
    │
    ▼
[ClassWriter byte arrays — one per emitted Idris class]
    │
    ▼
JVM/Module/ModuleInfo.idr ────► auto-emit module-info.class (Dec 40, from .ipkg)
JVM/Module/Manifest.idr   ────► MANIFEST.MF, multi-release layout
JVM/Module/Jar.idr        ────► reproducible JAR (Dec 36, byte-identical contract)
    │
    ▼
JVM/Verify/XVerify.idr    ────► java -Xverify:all gate (Dec 27, CI build-breaker)
    │
    ▼
output.jar
```

### Runtime-Emit Coupling (what the JAR depends on at runtime)

```
emitted .class files
    │  (compile-time references to)
    ▼
support/jvm/palisade-runtime/
    │
    ├── LinearFuture, LinearChannel       ← every async path
    ├── OrderedContainer (VarHandle)      ← every shared mutable
    ├── GuardProxy + LinearityViolation   ← every FFI boundary
    ├── StructuredTaskScope helpers       ← every concurrent block
    ├── JFR Event subclasses              ← every instrumented site
    ├── Thunk, FunctionN                  ← lazy values, currying
    └── ffi/* helpers                     ← Java exception bridge, etc.
        │
        ▼
    JDK 25 (Loom, ZGC, JFR, ScopedValue, ClassFile API, JPMS)
```

### Key Data Flows

1. **`FC` (file:line) threading.** Every `NamedCExp` constructor carries `FC`. Every JVMExpr node carries `FC`. Every emission site uses that `FC` to drive `visitLineNumber` *and* SDE table population. If a node loses its `FC` during lowering, the source map degrades — make `FC` non-droppable in the JVMExpr type.
2. **`InferredType` propagation.** Inferred types are computed during lowering, attached to JVMExpr nodes, and consumed by the emitter to choose between `iadd` vs `invokevirtual BigInteger.add`, between `ireturn` vs `areturn`, between unboxed locals vs boxed. Erasure (Dec 15) operates *on* the inferred-type graph: erasable args get dropped from method signatures, retained ones get ghost-annotated.
3. **Linearity propagation.** Linearity bits (q=1 vs q=n from QTT) ride along inferred types. The Memory/LIMM pass uses them to elide barriers; the FFI/Membrane pass uses them to decide whether to wrap a value in a Guard Proxy when crossing into Java.
4. **Annotation accumulation.** Each lowering pass (Spring, ghost args, linearity markers, JFR events) adds annotation metadata to JVMExpr nodes. The emitter walks the accumulated set per class/method and calls the appropriate ASM `visitAnnotation` calls.
5. **Module-info derivation.** `.ipkg` dependencies → JPMS `requires` clauses; emitted public Idris namespaces → `exports` clauses. Auto-derived per Decision 40; runs after all classes are emitted (needs the full picture of what got generated).

---

## Build Order Implications

The 11 cross-cutting Key Decisions can't all land in phase 1. They have a *technical* dependency order independent of priority. This subsection orders them.

### Ground Floor (must exist before anything else compiles)

1. **`Compiler.Common.Codegen` integration shim** — `JVM/Codegen.idr` returning a `Codegen` record. Even if the body is a stub `compileExpr = throw NotImplemented`, this unblocks wiring the backend into the driver and running `idris2 --cg jvm hello.idr` end-to-end. Mirrors how every other backend bootstraps.
2. **ASM bridge skeleton** (`support/jvm/palisade-asm-bridge`) — minimal Java FFI wrapper exposing `ClassWriter`, `MethodVisitor`, `visitInsn`, `visitMethodInsn`, etc., to Idris. Required before `JVM/Asm/Emit.idr` can do anything. Subclass `ClassWriter` immediately to override `getCommonSuperClass` (Pattern 4 above).
3. **Name mangler** (`JVM/Name/Mangle.idr`) — every emission needs JVM-safe identifiers. Trivial first cut; semantic-naming refinement (Dec 23) lands later.
4. **Backend IR** (`JVM/IR/JVMExpr.idr`) — even an extremely thin first version. Without it the emitter is forced to walk `NamedCExp` directly, which forces every emitter site to redo type inference inline.
5. **Trivial lowering** (`JVM/IR/Lower.idr`) — `NamedCExp → JVMExpr` for the few constructors needed for "hello world": `NmRef`, `NmApp`, `NmCon`, `NmPrimVal` (String, Int), `NmExtPrim` (`prim__putStr`).
6. **Trivial emitter** (`JVM/Asm/{Emit,Method,Class}.idr`) — enough to emit a `Main` class with a `static main(String[])` calling into `palisade-runtime`'s `print`-equivalent.

**Exit criterion for Ground Floor:** `idris2 --cg jvm hello.idr && java -jar hello.jar` prints "hello world".

### First Floor (basic correctness; nothing about Spring/Loom/JFR yet)

7. **Inferred type system** (`JVM/IR/InferredType.idr`) — the full type lattice idris-jvm uses, extended with linearity bits.
8. **Tail call optimization** (`JVM/Opt/TailCall.idr`) — Decision 2. Idris is recursion-heavy; without this, even small programs blow the stack.
9. **BigInteger / primitive specialization** (Decision 3) — touches both InferredType and emitter.
10. **Case → switch lowering** (`JVM/Asm/Switch.idr`) — needed for any non-trivial pattern matching.
11. **Frame computation** (`JVM/Asm/Frame.idr`) — initially via ASM's `COMPUTE_FRAMES` flag; precision generator (Decision 27) is a refinement.
12. **`-Xverify:all` CI gate** (`JVM/Verify/XVerify.idr`) — Decision 27. Land it the moment any non-trivial bytecode emits, so verifier regressions are caught immediately rather than accumulating.
13. **Debug info: line table + local var table** (`JVM/Debug/{LineTable,LocalVars}.idr`) — Decision 21, the "easy half". SourceDebugExtension is the harder half but can lag slightly.
14. **Module-info auto-emission** (`JVM/Module/ModuleInfo.idr`) — Decision 40. Required for any modulepath-clean usage.
15. **JAR packaging** (`JVM/Module/Jar.idr`) — minimum viable JAR; reproducibility contract (Decision 36) is a CI verification step that can be added once builds are stable.

**Exit criterion for First Floor:** A non-trivial Idris program (recursive, pattern-matching, using `Maybe`/`Either`) compiles, runs on the JVM, passes `-Xverify:all`, and produces stack traces with Idris file:line via `LineNumberTable`.

### Second Floor (FFI; the membrane goes up)

16. **FFI elaborator** (`JVM/FFI/Foreign.idr`) — basic `%foreign` parsing and dispatch. Without this, no Java interop.
17. **Java exception bridge** (`JVM/FFI/ExceptionBridge.idr`) — Decision 8. Land it *with* the FFI elaborator, not after — every FFI call must be wrapped from day one or you're rewriting later.
18. **Importer (elaborator-reflection-driven)** (`JVM/FFI/Importer.idr`) — Decision 4. Big lift; needs elaborator reflection on the Idris side and Java signature parsing on the JVM side.
19. **Generics projection** (`JVM/FFI/Generics.idr`) — Decision 6. Builds on Importer.
20. **Defensive Membrane / Guard Proxy** (`JVM/FFI/Membrane.idr` + `support/jvm/palisade-runtime/GuardProxy.java`) — Decision 20. Land *immediately after* the basic FFI elaborator but *before* the Importer ergonomics polish — otherwise you're papering over a leaking type system with sugar.
21. **Hybrid Metadata Erasure** (`JVM/Opt/Erase.idr`) — Decision 15. Needs InferredType + FFI in place; the "internal vs FFI" distinction only makes sense once both exist.

**Exit criterion for Second Floor:** Idris code can call arbitrary Java methods, exceptions are wrapped as `Either`, and a `LinearFuture` value passed across an FFI boundary refuses double-consumption with `PalisadeLinearityViolation`.

### Third Floor (the thesis: linearity + Loom)

22. **`palisade-runtime/LinearFuture.java`** — the central thesis primitive (per requirements list). Lives in the runtime; codegen emits calls.
23. **Linearity tracking through lowering** (`JVM/IR/Linearity.idr`) — fully integrated, not just a marker bit.
24. **LIMM** (`JVM/Memory/LIMM.idr`) + **OrderedContainer** (`JVM/Memory/OrderedContainer.idr` + runtime) — Decision 17. Needs Linearity in place. Compile-time error for bare shared-mutation requires the inferred-type/linearity graph to be reliable.
25. **Structured Linear Runtime** (`JVM/Loom/{StructuredScope,LinearFuture}.idr`) — Decision 18. Statically proving every task is joined/cancelled requires the linearity tracking from #23.
26. **Concurrency primitives subset** (`palisade-runtime/concurrent/`, `palisade-runtime/LinearChannel.java`, `ScopedValue` helpers) — Decision 19.

**Exit criterion for Third Floor:** A reference Spring Boot endpoint that fans out work over `LinearFuture` and `StructuredTaskScope`, with the compiler refusing builds that drop a future or leak a task.

### Fourth Floor (operability; the "transparent box")

27. **Source maps full** (`JVM/Debug/SDE.idr`) — Decision 21 finished. Heap dumps, JFR events, profiler frames all show Idris file:line.
28. **Semantic naming + ADT reconstruction** (`JVM/Name/Mangle.idr` refinement; ADT metadata in emitter) — Decision 23.
29. **JFR vocabulary** (`JVM/JFR/Events.idr` + `JVM/JFR/Emit.idr` + runtime event classes) — Decision 22.
30. **Spring annotations** (`JVM/Spring/Annotations.idr` + `JVM/Spring/DI.idr`) — Decision 5. Depends on Importer (#18) for typed Spring API signatures.
31. **OpenTelemetry integration** (`JVM/Telemetry/OpenTelemetry.idr`) — Decision 24. Needs LinearFuture lifecycle in place (#22) so spans can bind to it.

### Fifth Floor (performance + production polish)

32. **Tiered JIT shape** (`JVM/Opt/Specialize.idr`, `JVM/Opt/ClosedWorld.idr`, `JVM/Asm/InvokeDynamic.idr`) — Decision 16. Specialize first; closed-world analysis later when fat-JAR builds are routinely produced.
33. **Precision StackMapFrame generator** (`JVM/Asm/Frame.idr` refinement) — Decision 27 in full. Replaces ASM's auto-compute for trampoline-heavy code.
34. **JMH benchmarks** (`tests/benchmarks/`) — Decision 28. CI gate against Kotlin Coroutines, Java 25 virtual threads.
35. **Reproducible build verification** (`JVM/Module/Jar.idr` refinement) — Decision 36 turned from contract into CI verification across two build hosts.

### Sixth Floor (developer experience; non-blocking but adoption-critical)

36. **Maven + Gradle plugins** (`build-tooling/`) — Decision 10.
37. **`idris2`-owned build emitting JAR** — Decision 10's other half.
38. **LSP extension for JVM/Spring types** — Decision 11. Lives in the LSP, not in `src/Compiler/JVM/`.
39. **Hot reload + REPL attach** — Decision 32. Builds on incremental compile (Codegen's `incCompileFile`).
40. **`palisade spring my-service` template** — Decision 31.

### Hard Build-Order Constraints (cycles to avoid)

- **Linearity before LIMM, LIMM before Loom.** LIMM (Dec 17) compile-time errors require linearity tracking; Loom proofs (Dec 18) require both linearity and LIMM's notion of "shared state" being well-defined.
- **FFI elaborator before Importer before Generics before Spring annotations.** Decision 5 (Spring DI) requires Decision 6 (Generics) requires Decision 4 (Importer) requires basic FFI.
- **Defensive Membrane co-lands with FFI, not after.** Decision 20 has to be on the membrane from day one of FFI; otherwise the linearity guarantees are paper while real code accumulates.
- **`-Xverify:all` gate lands with the *first* non-trivial emitter.** Decision 27. Verifier-failing bytecode is exponentially harder to debug retroactively.
- **Source map plumbing (`FC` threading) lands with the *first* emitter.** Decision 21. Threading `FC` through later means rewriting the emitter.

---

## Dependencies on Existing Idris 2 Components

| Existing component | What Palisade needs from it | How Palisade uses it |
|---|---|---|
| `Compiler.Common` (`Codegen` record, `CompileData`) | The integration interface | `JVM/Codegen.idr` constructs `MkCG ...` |
| `Core.CompileExpr` (`NamedCExp`, `NamedDef`, `NamedConAlt`, `NamedConstAlt`, `CFType`, `CDef`, `PrimFn`, `Constant`, `LazyReason`, `ConInfo`) | The IR consumed; constants and primitive ops; FFI type descriptors | `JVM/IR/Lower.idr` pattern matches across all `Nm*` constructors. **Vendored snapshot per Decision 38.** |
| `Core.TT` (`Name`, `FC`, `UserName`) | Names used throughout, file:line for every node | Name mangler input; `FC` threading for source maps |
| `Core.Context` (`Defs`, `GlobalDef`, `Ctxt`) | Reading definition metadata during codegen | Lookup of definition info during lowering and FFI elaboration |
| `Core.Directory` (`getDirs`, build/output dirs) | File-system locations for output | `JVM/Module/Jar.idr` writes JAR there |
| `Core.Reflect` / elaborator reflection | Reading Java signatures via reflection-driven importer | Decision 4's importer |
| `Idris.Syntax` (`SyntaxInfo`, `Syn`) | Syntax info passed to Codegen interface | Standard plumbing per `Codegen` record signature |
| `Compiler.Inline`, `Compiler.Opts.{Constructor,CSE}` | Existing optimization passes that run *before* `NamedCExp` | Free; Palisade consumes the post-optimized IR |
| `Compiler.Generated` | Build-version metadata for output banners | `JVM/Codegen.idr` writes a `// generated by Idris2 X.Y.Z` line in JAR manifest |
| `Compiler.NoMangle` | Mangle exceptions for `%export` | Honored in `JVM/Name/Mangle.idr` |
| `Compiler.Separate` | Multi-package separate compile support | Future: enables per-package JAR emission |
| Logging (`Core.Context.Log`) | Topic-based logging during codegen | New topic `"jvm"` for Palisade traces |
| TTC binary cache (`Core.Binary`) | Incremental compilation | `JVM/Codegen.idr`'s `incCompileFile` consumes per-file TTC |

**One-way dependency.** Palisade imports from upstream; upstream never imports from Palisade. This is enforced by the directory layout (everything in `src/Compiler/JVM/` and `support/jvm/`) and by the `Codegen` interface boundary.

---

## Key-Decision → Component Map (cross-cutting concerns)

The 11 Key Decisions called out in the question, mapped to where they live architecturally. (Decisions not in the question — pinning, license, governance, etc. — are intentionally omitted.)

| Decision | Lives in (Idris codegen) | Lives in (Java runtime) | Build-order floor |
|---|---|---|---|
| **15** Hybrid Metadata Erasure | `JVM/Opt/Erase.idr` + emitter annotation hooks | n/a | Second |
| **16** Tiered JIT shape | `JVM/Opt/Specialize.idr`, `JVM/Opt/ClosedWorld.idr`, `JVM/Asm/InvokeDynamic.idr` | n/a | Fifth |
| **17** LIMM | `JVM/Memory/LIMM.idr` + `JVM/IR/Linearity.idr` | `palisade-runtime/OrderedContainer.java` | Third |
| **18** Structured Linear Runtime | `JVM/Loom/StructuredScope.idr`, `JVM/Loom/LinearFuture.idr` | `palisade-runtime/LinearFuture.java`, `palisade-runtime/concurrent/` | Third |
| **20** Defensive Membrane | `JVM/FFI/Membrane.idr` | `palisade-runtime/GuardProxy.java`, `PalisadeLinearityViolation.java` | Second |
| **21** Telemetry-Native Mapping (source maps) | `JVM/Debug/{LineTable,LocalVars,SDE}.idr` | n/a (read by JVM tooling) | First (lines) / Fourth (SDE) |
| **22** JFR vocabulary | `JVM/JFR/{Events,Emit}.idr` | `palisade-runtime/jfr/*.java` | Fourth |
| **23** Semantic Naming + ADT Reconstruction | `JVM/Name/{Mangle,Demangle}.idr` + ADT metadata in emitter | n/a (consumed by MAT/VisualVM) | Fourth |
| **27** Zero-Tolerance Verification Contract | `JVM/Verify/XVerify.idr` + `JVM/Asm/Frame.idr` precision generator | n/a (runs in CI) | First (CI gate) / Fifth (precision frames) |

The two recurring patterns: (1) most decisions split into a *codegen-side* concern and a *runtime-side* concern; (2) almost every decision has a "minimal viable" version that can land at the floor indicated and a "polished" version that follows. The roadmap should sequence the minimal versions and treat polish as separate phases.

---

## Scaling Considerations

This isn't a multi-tenant service; "scale" here means scale of code Palisade compiles, not user load.

| Scale | Architecture Adjustments |
|---|---|
| Single-file demos, hello-world | Default everything. ASM auto-compute frames. No incremental compile. Fine. |
| Small services (~50 Idris files, single Spring Boot module) | Incremental compile (`incCompileFile`) becomes essential — full rebuilds get annoying. Module-info auto-emission is non-negotiable here, not later. |
| Mid-size services (~500 Idris files, mixed Java/Kotlin/Idris module per Decision 30) | Closed-world specialization (Dec 16) starts mattering. Reproducible build verification (Dec 36) must be in CI. JAR size starts to matter; consider per-namespace JARs. |
| Multi-module systems (~5000+ Idris files, several services) | Separate compile per package (Compiler.Separate); per-package JARs; consider an Idris-side equivalent of `kotlinc -Xuse-fast-jar-fs`. |

### Scaling Priorities

1. **First bottleneck: full-rebuild compile time.** Incremental compile via `incCompileFile` per file. Mitigated by Decision 32's "sub-second incremental compile" goal.
2. **Second bottleneck: bytecode verifier complexity.** Auto-computed frames are slow on large methods. The precision frame generator (Decision 27) helps both correctness and compile time at scale.
3. **Third bottleneck: JAR fat-up.** ADT metadata (Dec 23), JFR event classes (Dec 22), Guard Proxies (Dec 20) all add classes per Idris definition. Closed-world analysis (Dec 16) reduces this when applicable.

---

## Anti-Patterns

### Anti-Pattern 1: Emit bytecode directly from `NamedCExp`

**What people do:** Skip the lowered IR and emit ASM calls inline during a single `NamedCExp` walk.

**Why it's wrong:** Loses every opportunity for cross-cutting passes (erasure, linearity tracking, annotation projection, JFR insertion, source-map tagging). Forces every emission site to re-derive Java types and re-make boxing decisions. Makes the emitter a 3000-line file no one wants to touch — which is exactly the trap idris-jvm partly fell into in its earlier iterations.

**Do this instead:** Lower into `JVMExpr` with `InferredType` first (Pattern 2 above). Each lowering pass is a small, testable function over `JVMExpr → JVMExpr`. The emitter becomes a thin walk that only does emission.

### Anti-Pattern 2: ASM `ClassWriter` without overriding `getCommonSuperClass`

**What people do:** `new ClassWriter(COMPUTE_FRAMES)` and call it a day.

**Why it's wrong:** Default `getCommonSuperClass` reflects on the *compiler's* classpath. For any class generated for the *target's* classpath, this throws `ClassNotFoundException` at compile time, often on the first non-trivial program with branching control flow.

**Do this instead:** Subclass `ClassWriter`; override `getCommonSuperClass` to consult the importer's class graph. idris-jvm ships `IdrisClassWriter`; Kotlin, Scala, Groovy all have equivalents. Land it in week 1.

### Anti-Pattern 3: Drop `FC` during lowering

**What people do:** Convert `NamedCExp` to `JVMExpr` and discard the `FC` because "we'll add it back at emission."

**Why it's wrong:** `FC` cannot be reconstructed. Every dropped `FC` is a stack-frame line number that says "(unknown)" forever. Decision 21's contract — Idris file:line visible in stack traces, profilers, heap dumps, JFR events — is broken silently.

**Do this instead:** Make `FC` a non-optional field on every `JVMExpr` constructor. Make the lowering pass thread it forward unconditionally. Add a CI test that grep's the source for `FC` being dropped — there's no good reason it ever should be.

### Anti-Pattern 4: Re-implement Loom in the codegen

**What people do:** Try to express structured concurrency via emitted bytecode primitives (lock acquire/release, manual park/unpark, custom worker pools).

**Why it's wrong:** Duplicates JDK functionality, maintenance burden, hides JVM-native instrumentation (JFR sees `LockSupport.park` natively; not your bespoke equivalent).

**Do this instead:** Emit calls into `palisade-runtime`, which calls into `java.util.concurrent` and `java.lang.StructuredTaskScope`. The codegen's job is *static enforcement* (Decision 18's "every task joined or cancelled before scope closes"); the runtime is `java.util.concurrent`'s job.

### Anti-Pattern 5: Defer the verifier gate

**What people do:** Skip `-Xverify:all` in CI until "the codegen is more mature."

**Why it's wrong:** Verifier-failing bytecode accumulates silently. By the time it's caught, multiple emitter sites are wrong in subtle ways and the bisection surface is enormous. Decision 27 calls this out explicitly: build-breaker from day one.

**Do this instead:** Land `-Xverify:all` the moment any non-trivial bytecode emits. Treat verifier failures as you would treat type-check failures — shippable code does not have them.

### Anti-Pattern 6: Treat the runtime library as an afterthought

**What people do:** Implement primitives (futures, channels, thunks) inline in the codegen, scattering Java-side helper code across one-off generated classes.

**Why it's wrong:** Versions and ships with the compiler instead of as a library. Java/Kotlin code can't consume Palisade primitives from the outside (Decision 30's hybrid module is broken). JFR event classes need to exist at runtime regardless of who emitted what.

**Do this instead:** `support/jvm/palisade-runtime` is a normal Java library, published to Maven Central, versioned independently, consumed by every emitted JAR via `requires palisade.runtime` in the auto-emitted `module-info.class`. Same shape as `support/c/idris_runtime.c` but a real Java library.

### Anti-Pattern 7: Skip the ASM bridge layer

**What people do:** Have Idris-side codegen call directly into ASM via long FFI signatures, with hundreds of `%foreign` declarations.

**Why it's wrong:** Locks Palisade to ASM forever (no migration to ClassFile API per Pattern 8). Pollutes the Idris-side codebase with low-level Java-API noise. Makes per-call FFI overhead scale linearly with bytecode size.

**Do this instead:** Wrap ASM in a small Java-side bridge (`palisade-asm-bridge`) that exposes a coarser-grained API ("emit a method", "emit a class", "emit a switch") to Idris. Mirrors idris-jvm's `idris-jvm-assembler` module.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---|---|---|
| **ASM** (bytecode lib, Dec 39) | Java FFI via `palisade-asm-bridge` | Subclass `ClassWriter`; use `COMPUTE_FRAMES` initially; replace with precision generator (Dec 27) for trampoline-heavy code |
| **JEP 484 ClassFile API** (future) | Eventual replacement of ASM behind the bridge | Wait until parity with ASM proven; ClassFile API explicitly does not aim to obsolete ASM ([JEP 484](https://openjdk.org/jeps/484)) |
| **JDK 25 `java.lang.classfile`** | Tooling reads emitted classes | No special integration needed; SDE + line tables make standard tools (jstack, JMC, MAT, VisualVM) work natively (Dec 23 promise) |
| **Spring Boot** | Annotation projection (Dec 5) + actuator endpoints (Dec 24) | Idris-emitted classes carry Spring annotations; Spring sees them as normal beans |
| **JFR** | Custom event classes in `palisade-runtime/jfr/` | Dec 22; emitted code calls `event.commit()`; JMC sees Palisade vocabulary |
| **Maven / Gradle** | Plugin per Dec 10 | `palisade-maven-plugin`, `palisade-gradle-plugin`; lives in `build-tooling/` |
| **Idris LSP** | Extension per Dec 11 | Lives outside `src/Compiler/JVM/`; consumes type info from elaborator + JVM type info from importer |
| **GraalVM native-image** | AOT-friendly metadata emission per Dec 7 | Codegen emits `META-INF/native-image/reflect-config.json` etc.; informed by importer's class graph |

### Internal Boundaries

| Boundary | Communication | Notes |
|---|---|---|
| `Compiler.Common.Codegen` ↔ `JVM/Codegen.idr` | The `Codegen` record interface | Existing Idris convention; identical to RefC, Chez, ES |
| `JVM/IR/*` ↔ `JVM/Asm/*` | `JVMExpr` data type | Lowering passes don't know about ASM; emitter doesn't know about `NamedCExp` |
| `JVM/Asm/*` ↔ `support/jvm/palisade-asm-bridge` | Idris FFI calls into Java ASM bridge | One-way; bridge does not call back |
| Emitted classes ↔ `support/jvm/palisade-runtime` | JVM bytecode `invokestatic`/`invokevirtual` into runtime classes | Standard Java linkage; runtime is just a JAR on the modulepath |
| `JVM/FFI/Importer.idr` ↔ JDK reflection | Java FFI for class/signature lookup | Cached; results feed into InferredType graph |
| `JVM/Module/ModuleInfo.idr` ↔ `.ipkg` parser | Reads `package` deps from `Idris.Package` data | Per Dec 40; runs after all classes emitted |

---

## Sources

- [Idris 2 source: `src/Core/CompileExpr.idr`](https://github.com/idris-lang/Idris2/blob/main/src/Core/CompileExpr.idr) — `NamedCExp`, `NamedDef`, `CFType`, `CDef` definitions; the consumer boundary for Palisade.
- [Idris 2 source: `src/Compiler/Common.idr`](https://github.com/idris-lang/Idris2/blob/main/src/Compiler/Common.idr) — `Codegen` record interface that every backend implements.
- [Idris 2 source: `src/Compiler/RefC/RefC.idr`, `src/Compiler/Scheme/Chez.idr`, `src/Compiler/ES/Codegen.idr`](https://github.com/idris-lang/Idris2/tree/main/src/Compiler) — existing backend patterns Palisade parallels.
- [idris-jvm source tree (mmhelloworld/idris-jvm)](https://github.com/mmhelloworld/idris-jvm) — direct evidence of the `Asm/Codegen/Optimizer/InferredType/Foreign/Export/...` decomposition that proves out for an Idris-on-JVM backend; three-Maven-module split (compiler/assembler/runtime).
- [Kotlin compiler/ir directory](https://github.com/JetBrains/kotlin/tree/master/compiler/ir) — IR + lowering pipeline pattern.
- [Crash Course on the Kotlin Compiler (K1+K2)](https://medium.com/google-developer-experts/crash-course-on-the-kotlin-compiler-k1-k2-frontends-backends-fe2238790bd8) — compilation phase walkthrough.
- [Scala 3 / Dotty Compiler Phases](https://nightly.scala-lang.org/docs/contributing/architecture/phases.html), [Dotty Overall Structure](https://dotty.epfl.ch/docs/internals/overall-structure.html) — phase-fusion model for tree-to-tree lowering before `genBCode`.
- [Clojure tools.emitter.jvm](https://github.com/clojure/tools.emitter.jvm) — analyzer-AST → ASM bytecode pipeline.
- [Decompiling Clojure II: The Compiler](http://blog.guillermowinkler.com/blog/2014/04/21/decompiling-clojure-ii/) — Clojure analyze + emit walkthrough.
- [JEP 484: Class-File API](https://openjdk.org/jeps/484) — JDK 24 finalized; eventual successor to ASM, explicit non-goal of obsoleting ASM in the near term.
- [ASM Developer Guide](https://asm.ow2.io/developer-guide.html) — Frame computation (`Frame`/`CurrentFrame`), `MethodWriter`, stack-map auto-compute.
- [Tail Call Recursion in Java with ASM (UnlinkedList)](https://unlinkedlist.org/2023/03/19/tail-call-recursion-in-java-with-asm/) — pattern for self-recursion → goto via labels.
- [JSR-45 SourceDebugExtension](https://jcp.org/en/jsr/detail?id=45) — SMAP format for non-Java source maps, used by Kotlin and Scala in production; ASM exposes `cv.visitSource(name, smap)` to populate it.
- [Always up to date — The Class-File API for programmatic handling of bytecode (INNOQ)](https://www.innoq.com/en/articles/2025/04/java-class-file-api/?mode=eco) — current-state analysis of ASM vs ClassFile API trade-offs.

---

*Architecture research for: JVM compiler backend for a verified functional language (Idris 2 → JVM bytecode)*
*Researched: 2026-04-21*
