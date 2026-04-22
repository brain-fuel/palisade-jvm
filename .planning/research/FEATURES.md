# Feature Research

**Domain:** JVM compiler backend for a verified, dependently-typed, linear functional language (Idris 2 → JVM bytecode), targeting Spring Boot deployment on JVM 25+
**Researched:** 2026-04-21
**Confidence:** MEDIUM-HIGH (Kotlin/Clojure/Scala/idris-jvm features verified against official docs; "what formal-methods crowd wants" is partly inferred from CompCert/Lean/Coq/Dafny patterns)

---

## Domain Framing

Palisade is **not a general-purpose JVM language**. It is a **deployment vector** that delivers verified Idris 2 code into JVM environments where Idris's compile-time guarantees (totality, linearity, dependent types) must survive contact with the bytecode and the surrounding Java/Kotlin/Spring ecosystem.

This shapes the feature landscape uniquely:

- **Comparison against Kotlin/Clojure/Scala 3** sets the **adoption floor**: anything missing from these languages that JVM engineers expect = adoption blocker.
- **Comparison against idris-jvm** sets the **predecessor floor**: anything Palisade fails to ship that idris-jvm shipped is a regression.
- **Comparison against ScalaJS/Eta/Frege** surfaces the **niche-language failure modes**: what kills "language-X-on-platform-Y" projects.
- **The formal-methods axis** sets the **differentiator ceiling**: features unique to verified languages (proof preservation, telemetry-as-proof, semantic source maps) that would let Palisade win mindshare *as* a verified-systems story, not just "another JVM language."

The four user personas this research considers:

| Persona | What they need | What disqualifies a JVM language |
|---|---|---|
| **Java/Spring engineer** | One-line Java FFI, Spring annotations, Maven/Gradle plugin, IntelliJ/LSP, stack traces that point at source | Boilerplate FFI, can't be a `@Bean`, "you must drop into a Scheme REPL to debug" |
| **Kotlin engineer** | Coroutine-equivalent ergonomics (Loom), `@Jvm*`-style annotations on emitted classes, idiomatic property/method names from Java | Awkward generics, megamorphic dispatch, Idris-flavored exceptions |
| **Formal-methods researcher** | Guarantees survive lowering (linearity at runtime, totality at FFI), reproducible builds, proof artifacts visible in deployment, no "trust me" between proof and bytecode | Compile to bytecode and shrug at the FFI boundary; lose linearity in `CompletableFuture` |
| **SRE / production operator** | JFR events, source-mapped stack traces, profiler shows Idris names, hot reload, GraalVM native image, structured concurrency, predictable GC | "Heap dumps are gibberish," "stack traces stop at the FFI," "no observability story" |

---

## Feature Landscape

### Table Stakes (Users Expect These — Missing = Adoption Blocker)

These are non-negotiable. None of Kotlin, Clojure, or Scala 3 ship without them, and they define the floor any "serious JVM language" must clear. **Most are already covered by PROJECT.md decisions** — annotated with the relevant Decision number.

#### Compiler & Build

| Feature | Why Expected | Complexity | PROJECT.md Coverage |
|---------|--------------|------------|---------------------|
| Maven plugin (mixed-language module) | Spring Boot world is overwhelmingly Maven; Kotlin/Scala both ship one | HIGH | Decision 10 (both Maven + Gradle plugins) |
| Gradle plugin | Kotlin's primary build, Android-style projects, Spring Boot Initializr default | HIGH | Decision 10 |
| Standalone `idris2` build that emits a runnable JAR | Smallest possible adoption surface; "compile and run, no plugin needed" | MEDIUM | Decision 10 |
| Incremental compilation | Sub-second iteration; Kotlin K2 made this a competitive issue | MEDIUM | Inherits Idris 2 TTC cache (STACK.md, ARCHITECTURE.md) + Decision 32 (sub-second incremental) |
| Reproducible byte-identical builds | Modern supply-chain hygiene; SLSA L3+ is now table stakes for enterprise CI | MEDIUM | Decision 36 (by contract, hash-verified) |
| JPMS module emission (`module-info.class`) | JDK 25 deployment at scale; Spring Boot 3.x supports modular runtimes | MEDIUM | Decision 40 (auto-emit per `.ipkg`) |
| Bytecode that passes `-Xverify:all` | Non-negotiable; verifier failure = JVM refuses to load | HIGH | Decision 27 (zero-tolerance, CI build-breaker) |
| ASM as bytecode emission library | Battle-tested; what every other JVM language uses | LOW (choice), HIGH (use) | Decision 39 |
| Stack Map Frame generation for emitted methods | Required since Java 7; trampolines + dependent-type erasure make this hard | HIGH | Decision 27 (precision Stack Map Frame generator) |

#### Java/JVM Interop

| Feature | Why Expected | Complexity | PROJECT.md Coverage |
|---------|--------------|------------|---------------------|
| One-line Java method calls | Kotlin/Clojure baseline; "do I really have to write a `%foreign` per method?" kills adoption | HIGH | Decision 4 (Clojure-grade FFI via elaborator-reflection-driven importer) |
| Java generics projection (read Java signatures, generate parameterized Idris types) | Required for `Mono<ResponseEntity<List<User>>>`-shaped Spring APIs | HIGH | Decision 6 (faithful projection; erasure at codegen) |
| Java exception handling (catch Java exceptions in Idris) | Every Java method can throw; ignoring this means every FFI call is a footgun | MEDIUM | Decision 8 (always wrap in `Either JException T` at FFI boundary) |
| Annotation emission on Idris-emitted classes | Spring/JPA/JAX-RS all annotation-driven; without this, Idris classes can't be beans | HIGH | Decision 5 (Spring annotations on Idris bytecode; Idris classes are first-class beans) |
| `@Jvm*`-style hints (override emitted name, emit static, suppress getters) | Kotlin demonstrates these are needed for clean Java-side APIs; idiomatic name mismatch is a wart | MEDIUM | **Gap — needs roadmap addition** (Palisade equivalent of Kotlin's `@JvmStatic`, `@JvmName`, `@JvmField`, `@JvmOverloads` for Idris-side declarations) |
| Implementing Java interfaces from Idris | Spring `Filter`, `Runnable`, `Comparator`, `Callable`, `Function<T,R>` are pervasive | HIGH | Implied by Decision 4/5 but **needs explicit roadmap mention** |
| Extending Java classes (where unavoidable, e.g. `RuntimeException` subclassing for app errors) | Some frameworks demand subclasses, not interfaces | HIGH | **Gap — needs explicit decision** (and may be deliberately limited; many JVM languages restrict this) |
| Static method exposure (Java-callers can call Idris module functions as `IdrisClass.method(...)`) | Otherwise Java code cannot reach Idris code without instantiation gymnastics | MEDIUM | Implied by Decision 5; **needs explicit roadmap mention** |

#### Runtime & Type Mapping

| Feature | Why Expected | Complexity | PROJECT.md Coverage |
|---------|--------------|------------|---------------------|
| Primitive specialization (`Int`, `Double` not boxed) | Performance baseline; megamorphic boxing is the historical reason "language-X on JVM" is slow | HIGH | Decision 3 (codegen specialization for monomorphic `Int`/`Double`) |
| `Maybe`/`Optional` projection (Idris `Maybe a` ↔ Java `Optional<T>` or null at boundary) | Otherwise every Java return value forces a manual conversion | MEDIUM | Implied by Active Requirements ("Interop-contract ring: projection rules for `Maybe`, ADTs, exceptions, generics") |
| ADT projection (Idris ADTs visible to Java/Kotlin as sealed classes / records) | Kotlin sealed classes set the expectation; Scala 3 enums likewise | HIGH | Implied by Active Requirements; **roadmap should formalize** (probably emit sealed-class hierarchy + records per constructor) |
| `BigInteger` for unbounded `Integer` | Idris `Integer` is unbounded; truncating to `long` would silently break Idris semantics | LOW | Decision 3 (BigInteger for unbounded) |
| Tail call elimination (self-recursion → loop, mutual → trampoline) | Idris is heavily recursive; without TCE, even tutorials stack-overflow | MEDIUM | Decision 2 |
| Garbage collector configuration | Trading-grade pause profiles need ZGC; G1 default isn't acceptable for that audience | LOW (config), HIGH (validation) | Decision 13 (ZGC default override) |

#### Tooling & Developer Experience

| Feature | Why Expected | Complexity | PROJECT.md Coverage |
|---------|--------------|------------|---------------------|
| LSP-based IDE support | Kotlin, Scala 3, Java all ship LSP/LSP-equivalent; IntelliJ now exposes LSP API to all plugin developers | MEDIUM | Decision 11 (extend Idris LSP for JVM/Spring) |
| Source maps (Idris file:line in Java stack traces) | JSR-45 is what Kotlin/Scala use; without it, production debugging is impossible | HIGH | Decision 21 (Telemetry-Native Mapping: SourceDebugExtension + LineNumberTable + LocalVariableTable) |
| Debug info preserved (variable names visible in debugger) | `LocalVariableTable` table stakes; hjkl through anonymous variables in jdb is unacceptable | MEDIUM | Decision 21 |
| Project template generator (`palisade spring my-service`) | Spring Initializr set the bar; "show me a working starter" is the first thing engineers ask | MEDIUM | Decision 31 |
| Hot reload during development | Spring DevTools normalized this; Kotlin/Java engineers expect it | HIGH | Decision 32 (Spring DevTools-style hot reload of Idris classes) |
| REPL that attaches to running JVM | Clojure set this expectation; "REPL-driven dev" is the killer feature for that crowd | HIGH | Decision 32 (Idris REPL attaches to running JVM, eval in live process) |
| JUnit 5 `TestEngine` integration | Mixed-language CI is the only realistic enterprise reality | MEDIUM | Decision 25 (native JUnit 5 TestEngine; Idris PBT/PDDT discoverable in Maven/Gradle) |
| Spring TestContext support (DI in tests, `@Transactional` rollback) | Anyone writing Spring code expects this; absence makes Idris untestable in Spring projects | MEDIUM | Decision 25 |
| JaCoCo-compatible coverage | Enterprise CI gates measure this; if Palisade code is invisible to JaCoCo, it doesn't exist to compliance | HIGH | Decision 26 (Lifted Semantic Coverage: JaCoCo + JSR-45 source map) |

#### Standard-Library Coverage (Adoption Floor)

| Feature | Why Expected | Complexity | PROJECT.md Coverage |
|---------|--------------|------------|---------------------|
| State management story (`IORef` + `State` + `Control.App` + linear state — each mapped to JVM/Clojure idioms) | Idris 2 stdlib has multiple state abstractions; if engineers must FFI for things Clojure devs get for free, adoption stalls | HIGH | Decision 9 (comprehensive survey + idiomatic JVM AND Clojure mapping per abstraction) |
| File / directory / array / IORef / buffer FFI | idris-jvm shipped these; not having them is a regression | MEDIUM | **Implicit in inheriting Idris 2 stdlib + Decision 4 FFI**; roadmap should explicitly call out parity with idris-jvm primitives |
| String/regex/collection conversion at FFI | Java uses `String`, Idris uses `String`; List ↔ `java.util.List`; Map ↔ `java.util.Map`; details matter | MEDIUM | Implied by interop-contract ring; **needs explicit roadmap entry** |
| HTTP client / JSON SerDe / JDBC bindings | idris-jvm explicitly listed these as gaps; Spring engineers can't operate without them | MEDIUM | **Gap — not explicitly in PROJECT.md**; either ship as Palisade-curated bindings or document the FFI path |

#### Documentation & Adoption

| Feature | Why Expected | Complexity | PROJECT.md Coverage |
|---------|--------------|------------|---------------------|
| Diataxis-quadrant docs (tutorial/how-to/reference/explanation) | Modern docs bar; Rust/Django/Kotlin all do this | HIGH | Decision 29 (full Diataxis: 5 tutorials, 6 skills, IAM worked example, ≥3 enterprise-ready examples) |
| Strict semver + compatibility matrix | Enterprise procurement asks for this; Spring publishes one, Kotlin publishes one | LOW (process), MEDIUM (sustaining) | Decision 34 |
| Security disclosure policy + signed releases + SBOM | Enterprise security review demands; CVE issuance, SLSA provenance | MEDIUM | Decision 35 (full enterprise security process; third-party audit deferred past v1) |
| Choice of OSS license compatible with enterprise (not AGPL) | EPL 2.0 same family as Clojure; Apache-compatible | LOW | Decision 33 |

---

### Differentiators (Competitive Advantage — Reasons to Pick Palisade)

These are where Palisade wins. Most are already PROJECT.md decisions; the value is recognizing **which** are differentiators (vs which are table stakes), and how they map to "reasons to pick Palisade over Kotlin."

| Feature | Value Proposition | Complexity | PROJECT.md Coverage | Comparable Feature in Other Languages |
|---------|-------------------|------------|---------------------|---------------------------------------|
| **Linear Futures over `CompletableFuture`** (use-exactly-once enforced at compile time) | The central thesis: structured concurrency where async lifecycle is a *type-level* guarantee, not a convention. Kotlin coroutines provide ergonomics; Loom provides scheduling; Palisade provides *proof*. | VERY HIGH | Active Requirements + Decision 18 (Structured Linear Runtime) | Kotlin coroutines (ergonomics, no static proof); Java Loom (scheduling, no static proof); ZIO/Cats Effect (Scala — purity + effects, but JVM-heavy boxing) |
| **Defensive Membrane** (Guard Proxies for linear types at Java boundary) | Idris guarantees survive contact with the JVM heap. No other JVM language preserves linearity at FFI. | HIGH | Decision 20 | None. Kotlin has no linear types; Scala's `IO` doesn't enforce single-use. |
| **Telemetry-as-Proof** (OpenTelemetry integrated into the Idris effect system; spans enforced across `LinearFuture` lifecycle by the compiler) | Solves "lost trace in async" formally. Observability becomes type-safe, not best-effort. | HIGH | Decision 24 | None. Java's OpenTelemetry agents are best-effort; lost-context-on-async is the canonical observability bug. |
| **JFR with Idris-vocabulary events** (`LinearFutureCompleted`, `LinearityViolationDetected`, `OrderedContainerAccess`) | JMC functions as a production formal-verification monitor. "Transparent box" for ops. | MEDIUM | Decision 22 | None. Kotlin/Scala emit generic JFR events; no language ships verification-aware events. |
| **Semantic Naming + ADT Reconstruction in profilers/heap dumps** | MAT/VisualVM see Idris ADTs as ADTs; deterministic mangling = readable in production tools. | HIGH | Decision 23 | Kotlin/Scala both produce mangled-but-recognizable names; neither emits ADT metadata for native MAT understanding. |
| **Hybrid Metadata Erasure + Ghost Annotations** (linearity markers retained as JVM attributes for SonarQube / external static analyzers) | Internal speed + external verifiability. Enterprise linters can enforce Palisade invariants. | HIGH | Decision 15 | None. Kotlin retains some Kotlin-specific metadata (`@Metadata` annotation); none of it is verification-relevant. |
| **LIMM (Linear-Inferred Memory Model)** — QTT lets compiler omit barriers for linear resources; bare mutation of shared refs is a compile-time error | C-level perf on linear hot paths, Java-level safety for shared state, with the JMM proven at type-check. | VERY HIGH | Decision 17 | None. Java has `VarHandle` but no static enforcement; Kotlin/Scala defer to JMM unchanged. |
| **JIT shape engineering** (`@hot`/`@specialize` pragmas, invokedynamic + Guard-With-Test for warm paths, link-time monomorphization in fat-JAR builds) | Trading-grade dispatch without across-the-board code bloat. | HIGH | Decision 16 | Kotlin's inline functions + Scala's `@specialized` are the closest analogs but neither does link-time closed-world monomorphization. |
| **Exception ↔ totality bridge** (every Java call returns `Either JException T`) | Totality preserved across every Java boundary; no implicit "this might throw" hiding behind a friendly type | MEDIUM | Decision 8 | None of Kotlin/Scala/Clojure preserve totality; checked exceptions in Java are universally despised, but Palisade's bridge converts them into Idris's idiom. |
| **Reproducible builds by contract, hash-verified across two hosts** | Supply-chain integrity demonstrated, not promised. SLSA L3+. | MEDIUM | Decision 36 | Java reproducible builds are aspirational; Bazel achieves it but only with strict configuration. Palisade ships it as a CI gate. |
| **Spring Boot Initializr-style template (`palisade spring my-service`)** | Frictionless first 5 minutes — a runnable Idris-controller Spring service with tests, JFR config, the lot | MEDIUM | Decision 31 | Spring Initializr exists for Java/Kotlin/Groovy; no Idris equivalent. |
| **Hybrid-module first-class** (Java + Kotlin + Idris in one module, one JAR, integrated with `javac`/`kotlinc`) | File-by-file or method-by-method conversion in live services | HIGH | Decision 30 | Kotlin pioneered hybrid modules; Scala 3 supports it. Idris-on-JVM has never offered it. |
| **Empirical benchmark suite (JMH vs Kotlin Coroutines / Java 25 virtual threads / sealed-class switching), JFR-monitored, blocking CI gate** | Performance becomes a verifiable artifact, not a marketing claim | MEDIUM | Decision 28 | Kotlin and Scala publish JMH benchmarks; neither blocks CI on regressions across competitor baselines. |

---

### Anti-Features (Deliberately NOT Built)

These are surface-appealing requests that PROJECT.md already rules out, plus a few additional anti-features that the niche-language failure modes (Eta, Frege, ScalaJS in some respects) suggest avoiding.

| Anti-Feature | Why Tempting | Why Problematic | What to Do Instead | PROJECT.md Coverage |
|---|---|---|---|---|
| **"Idris-flavored JVM language"** (relax Idris semantics to fit JVM idioms — make `Maybe` autoconvert to null silently, allow non-total functions to ship without warning, etc.) | Smoother JVM-engineer onboarding; smaller cognitive jump | Defeats the entire value proposition. The whole point is that the guarantees survive. Frege's lesson: once you compromise on purity to fit the host platform, you've built a worse Scala. | Keep Idris semantics intact; project them at the boundary via Defensive Membrane (Decision 20) | Out of Scope #3 |
| **Building on idris-jvm** | "Don't reinvent the wheel" | idris-jvm's design constraints (built on Idris 1, custom IR, megamorphic dispatch) are precisely what we need to escape. Inheriting them recreates the issues that motivated Palisade in the first place. | Vanilla Idris 2 + new `src/Compiler/JVM/` consuming `NamedCExp` directly | Out of Scope #1, Decision 1 |
| **JVM < 25 support** | Broader deployment surface | JVM 25's Loom, generational ZGC, JFR, JPMS, JEP 446 ScopedValues, JEP 484 ClassFile API are *load-bearing assets*, not optional. Palisade's value proposition depends on them. | Pin to JVM 25+; document the runtime requirement clearly | Out of Scope #2 |
| **General TCO for all functions (not just self-recursion)** | Cleaner functional code; matches Scala's @tailrec ambitions | JVM lacks general TCO; trying to fake it via trampolines for *all* calls explodes the call graph and breaks profilers/stack traces. | Direct self-recursion → JVM loop; mutual → trampoline (the Clojure-style proven approach) | Decision 2 |
| **Native-image as the only deployment target** (skip dynamic JVM) | Faster startup, lower memory | Spring DevTools, REPL, hot reload, JFR streaming, jdb — all of it depends on the dynamic JVM. Native-image is a deployment option, not the only one. | First-class GraalVM native-image *and* first-class JVM dynamic execution | Decision 7 (native-image as first-class deployment, not exclusive) |
| **Reflection-heavy API (mirror `java.lang.reflect`)** | Familiarity for Java engineers | Reflection breaks GraalVM native-image, breaks erasure, breaks the type system's promises, and is the #1 source of JVM library brittleness | Elaborator reflection at compile time + Decision 7's AOT-friendly metadata generation; no runtime reflection in Palisade-emitted code | Implied by Decision 7 (AOT-friendly metadata) |
| **Parallel-everything stdlib** (`pmap`, parallel collections, parallel for-comprehension) | Trendy; Scala has it | Forces decisions about thread pools, work-stealing, cancellation, and fairness on every collection user. Most uses are wrong. | Curated concurrency primitives (Decision 19): Linear Channels, Type-Bound ScopedValues, StructuredTaskScope. Specialized JUC primitives at FFI boundary. | Decision 19 (curated subset) |
| **A custom build tool (à la `sbt`, `lein`)** | Total control over build lifecycle | The Java/Kotlin world standardized on Maven/Gradle. A custom tool is an immediate adoption blocker; Spring engineers will not learn `sbt` to use Palisade. | Maven plugin + Gradle plugin first-class; `idris2`-owned build is the alternate path, not the primary | Decision 10 |
| **An "Idris standard library reimagined for JVM"** (custom collections, custom string type, custom IO) | Performance opportunity; cleaner semantics | Creates an island. Idris developers have to learn two stdlibs; Java developers can't pass `java.util.List` in. The state-management story (Decision 9) is the *correct* level of mapping — abstraction-by-abstraction, not wholesale replacement. | Decision 9: comprehensive mapping of existing Idris stdlib state abstractions to JVM/Clojure idioms; preserve `java.util.*` interop | Decision 9 |
| **Aggressive runtime type erasure (drop *all* type info, like Scala 2's most aggressive erasure)** | Performance, smaller class files | Loses the Ghost Annotations needed for SonarQube/JPMC linters to verify Palisade invariants; loses generic projection (Decision 6) | Hybrid Metadata Erasure: aggressive on internal Idris↔Idris, preserve at FFI boundary | Decision 15 |
| **Adoption-driven feature requests** (build XYZ because Big Co X asked) | "Real customer demand" | PROJECT.md is explicit: zero adopters is acceptable. Personal-confidence bar drives features, not prospective customer asks. | Use the personal-confidence bar as the gate. Adoption requests are signal but not requirement. | Out of Scope #4, Decision 37 |

---

### Phase-Specific Warnings (Features That Look Easy But Are Hard)

Several features in the Table Stakes table are listed as MEDIUM complexity but are deceptively dangerous. Calling them out for the roadmap:

| Feature | Why It Looks Easy | Why It's Actually Hard | Mitigation |
|---|---|---|---|
| Stack Map Frame generation (Decision 27) | "ASM has `COMPUTE_FRAMES` flag" | Trampolining + dependent-type erasure produces control-flow ASM cannot infer correctly; common-superclass algorithm has known limitations; verifier failures only surface at JVM load time. JaCoCo and similar tools have hit this exact wall. | Custom Stack Map Frame generator; CI gate via `-Xverify:all` on every emitted class; differential testing against ASM's COMPUTE_FRAMES output |
| Java generics projection (Decision 6) | "Just read the signature attribute" | Higher-kinded Java types, F-bounded polymorphism (`Enum<E extends Enum<E>>`), wildcards, raw types — and Spring API signatures use *all* of these | Importer must handle the long tail; vendor a fixture set of "ugly real-world Java signatures" (Spring, JPA, Reactor) and treat as conformance tests |
| Annotation emission for Spring (Decision 5) | "Just write `@Service` to the class file" | Spring expects specific annotation parameters, sometimes with nested annotation arguments (`@RequestMapping(method = {RequestMethod.GET})`); Spring's classpath scanning is sensitive to retention policies and visibility | Build against actual Spring 6+ ApplicationContext; verify bean discovery via reflection-based test before declaring done |
| Hot reload of Idris classes (Decision 32) | "Spring DevTools handles this" | Spring DevTools assumes JVM class redefinition rules (HotSwap can't change method signatures; can't add fields); Idris incremental recompilation may regenerate signatures | May need JRebel-style class redefinition agent; document the supported-change set; failure mode is "engineer must restart" not "silent corruption" |
| JFR custom events (Decision 22) | "Just extend `jdk.jfr.Event`" | Event class registration, EventStreaming API integration, JMC dashboards understanding the events, custom periodic events vs request events — non-trivial | Allocate a dedicated phase for JFR; write a minimal JMC dashboard as conformance test |
| Linear Futures (Active Requirement, Decision 18, Decision 20) | "Wrap CompletableFuture, mark as linear" | This is the *thesis*. Use-exactly-once at compile time, lifecycle bound to virtual thread scope, Defensive Membrane at FFI, integrated with structured concurrency, observable via JFR/OTel — this is months of work and merits its own phase | Treat as a multi-phase concern; ship in stages (linear marking → guard proxy → scope binding → telemetry integration) |

---

## Feature Dependencies

```
NamedCExp consumption (codegen foundation, Decision 1)
    └──requires──> Vendored NamedCExp snapshot (Decision 38)
    └──requires──> ASM bytecode emission (Decision 39)
    └──requires──> Stack Map Frame generator (Decision 27)
        └──requires──> Tail call strategy (Decision 2)
        └──requires──> Verification CI gate (Decision 27)

Java FFI / one-line method calls (Decision 4)
    └──requires──> Java generics projection (Decision 6)
    └──requires──> Java exception bridge (Decision 8)
    └──requires──> Type mapping (Maybe/ADTs/primitives, Decisions 3 + interop-contract ring)
        └──enables──> Spring DI annotations (Decision 5)
            └──enables──> Spring TestContext (Decision 25)
            └──enables──> palisade spring template (Decision 31)
            └──enables──> Hybrid-module first-class (Decision 30)

Linear Futures (Active Requirement)
    └──requires──> Defensive Membrane / Guard Proxies (Decision 20)
    └──requires──> Loom / Structured Linear Runtime (Decision 18)
    └──requires──> Curated concurrency primitives (Decision 19)
    └──enhances──> Telemetry-as-Proof (Decision 24)
        └──requires──> OpenTelemetry effect integration
        └──enhanced-by──> JFR Idris-vocabulary events (Decision 22)

Source maps (Decision 21, JSR-45)
    └──enables──> Lifted Semantic Coverage (Decision 26, JaCoCo + JSR-45)
    └──enables──> Profiler/heap semantic naming (Decision 23)
    └──enables──> Production debugging (table stakes)

GraalVM native-image (Decision 7)
    └──requires──> No runtime reflection in Palisade-emitted code
    └──requires──> AOT-friendly metadata generation
    └──conflicts──> Hot reload (Decision 32) — different deployment paths
    └──conflicts──> Spring DevTools live mode — different deployment paths

JPMS modules (Decision 40)
    └──requires──> Module-info derivation from .ipkg
    └──requires──> JAR layout decisions (Decision 10)

Reproducible builds (Decision 36)
    └──requires──> Deterministic codegen output
    └──requires──> Dependency pinning
    └──requires──> CI cross-host hash verification

@Jvm*-style hint annotations (GAP — not in PROJECT.md)
    └──required-for──> Idiomatic Java-side names of Idris-emitted code
    └──required-for──> Static method exposure for Java callers
    └──enabled-by──> Annotation-emission infrastructure (already needed for Decision 5)
```

### Dependency Notes

- **NamedCExp consumption underpins everything.** Decision 1 is the keystone; nothing else can be implemented until codegen consumes the vendored IR.
- **Stack Map Frames are the single most likely place to introduce silent verifier failures** — every other feature ultimately depends on the verifier accepting emitted bytecode.
- **Linear Futures is a multi-phase concern**, not a single feature. The roadmap should treat the four sub-decisions (18/19/20/24) as a coordinated Loom integration *track*, not as parallel phases.
- **Source maps unlock three downstream features** (coverage, profiler naming, production debugging). Sequence early.
- **GraalVM native-image conflicts with hot reload and DevTools** — this is OK (they're different deployment paths) but the docs need to make this explicit so engineers don't expect both at once.
- **The "@Jvm*"-style hint annotations are a gap in PROJECT.md** — every other JVM language has them, Spring engineers will demand them, and the infrastructure for annotation emission already exists (Decision 5).

---

## MVP Definition

### Launch With (v1)

The personal-confidence bar from PROJECT.md ("I'd be happy using this in any extreme production situation on JVM 25+") is the v1 target. Everything in the Active Requirements list ships, plus all 40 Key Decisions implemented. Concretely, the **non-negotiable v1 features**:

- [ ] **Codegen producing verifier-clean bytecode** from `NamedCExp` — without this, nothing else matters
- [ ] **One-line Java FFI** with elaborator-reflection-driven importer (Decision 4) — adoption blocker if missing
- [ ] **Spring annotation emission** for `@Component`/`@Service`/`@Repository`/`@Controller`/`@RestController`/`@Bean`/`@Autowired` minimum set (Decision 5)
- [ ] **Java generics projection** sufficient to type the Spring 6+ public API surface (Decision 6)
- [ ] **`Either JException T` exception bridge** at every FFI boundary (Decision 8)
- [ ] **Maven plugin + Gradle plugin + standalone `idris2` build** (Decision 10) — engineers need all three flows
- [ ] **Source maps + LineNumberTable + LocalVariableTable** on every emitted class (Decision 21)
- [ ] **Reference Spring Boot service** in this repo, fully wired — proves the thesis end-to-end
- [ ] **Linear Futures over CompletableFuture** with use-exactly-once enforcement (Active Requirement, Decision 18, 20)
- [ ] **JFR Idris-vocabulary events** for the linearity/structured-concurrency invariants (Decision 22)
- [ ] **JUnit 5 TestEngine integration** + Spring TestContext support (Decision 25)
- [ ] **Bytecode verification CI gate** via `-Xverify:all` (Decision 27)
- [ ] **Diataxis docs** at the depth specified in Decision 29
- [ ] **Hybrid-module support** (Java + Kotlin + Idris in one Spring Boot module, one JAR) (Decision 30)
- [ ] **`palisade spring my-service` template** (Decision 31)
- [ ] **Sub-second incremental compile + REPL-attach + hot reload** (Decision 32)
- [ ] **Reproducible builds with cross-host hash verification CI gate** (Decision 36)
- [ ] **JPMS module emission per .ipkg** (Decision 40)

### Add After Validation (v1.x)

These are real improvements but not required for the personal-confidence bar; they emerge once the v1 surface is exercised:

- [ ] **`@Jvm*`-style hint annotations** (gap from PROJECT.md) — once engineers complain about Java-side naming
- [ ] **Curated HTTP/JSON/JDBC bindings** — alternative is to document the FFI path; native bindings ship if FFI proves verbose
- [ ] **Extended Spring annotation set** — JPA `@Entity`, JAX-RS, validation annotations as use cases emerge
- [ ] **JMC dashboard for Palisade events** — the JFR events are emitted at v1 but the JMC plugin/dashboard may follow
- [ ] **SonarQube plugin** consuming the Ghost Annotations metadata (Decision 15) — depends on whether enterprise demand materializes
- [ ] **Additional state-management mappings** beyond the v1 survey — long tail of Idris stdlib state abstractions
- [ ] **JRebel-style class redefinition agent** — if Spring DevTools' default HotSwap proves insufficient

### Future Consideration (v2+)

Things gated on external technology stabilizing or major scope expansion:

- [ ] **Valhalla value-types as zero-allocation contract** (Decision 14) — explicitly gated on Valhalla GA
- [ ] **Third-party security audit** (Decision 35) — gated on funding model
- [ ] **Cross-platform compilation** (Idris source compiled for both JVM and another backend in one project) — Scala.js demonstrates the model; not in scope until v2 at earliest
- [ ] **Kotlin Multiplatform-style "Idris Multiplatform"** — far future; the JVM target must mature first
- [ ] **IntelliJ-native plugin** (vs the LSP-only Decision 11 path) — only if enterprise IntelliJ users demand depth beyond LSP

---

## Feature Prioritization Matrix

Selected highest-leverage features. (Complete per-Decision prioritization belongs in the roadmap; this is the strategic shape.)

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Codegen producing verifier-clean bytecode (Decision 1, 27) | HIGH | VERY HIGH | P0 (foundation) |
| One-line Java FFI (Decision 4) | HIGH | HIGH | P0 (adoption blocker if missing) |
| Linear Futures + Defensive Membrane (Active, Decision 18, 20) | HIGH | VERY HIGH | P0 (thesis) |
| Spring annotation emission (Decision 5) | HIGH | HIGH | P0 (adoption blocker if missing) |
| Source maps + JSR-45 + LineNumberTable (Decision 21) | HIGH | HIGH | P0 (production debugging blocker if missing) |
| Java generics projection (Decision 6) | HIGH | HIGH | P0 (Spring API blocker if missing) |
| Reference Spring Boot service (Active) | HIGH | MEDIUM | P0 (proof of thesis) |
| Maven + Gradle plugins (Decision 10) | HIGH | HIGH | P0 (adoption blocker if missing) |
| `palisade spring my-service` template (Decision 31) | HIGH | MEDIUM | P1 (frictionless onboarding) |
| JFR Idris-vocabulary events (Decision 22) | MEDIUM | MEDIUM | P1 (differentiator, not blocker) |
| Telemetry-as-Proof / OTel effect integration (Decision 24) | MEDIUM | HIGH | P1 (differentiator) |
| LIMM memory model (Decision 17) | MEDIUM | VERY HIGH | P1 (differentiator; complexity may push to P2) |
| Hot reload + REPL-attach (Decision 32) | HIGH | HIGH | P1 (Clojure-grade DX expected) |
| JaCoCo coverage lifting (Decision 26) | MEDIUM | HIGH | P1 (enterprise CI gate) |
| GraalVM native-image (Decision 7) | MEDIUM | HIGH | P1 |
| Hybrid-module support (Decision 30) | HIGH | HIGH | P1 (migration story) |
| Reproducible builds (Decision 36) | MEDIUM | MEDIUM | P1 (supply chain) |
| `@Jvm*`-style hint annotations (GAP) | MEDIUM | LOW | P2 (add when engineers complain) |
| HTTP/JSON/JDBC bindings (GAP) | MEDIUM | MEDIUM-HIGH | P2 (FFI path documented at v1) |
| Valhalla value-types (Decision 14) | MEDIUM | MEDIUM | P3 (gated on Valhalla GA) |

**Priority key:**
- P0: Must ship for v1; absence breaks the personal-confidence bar
- P1: Should ship for v1; some flexibility in sequencing
- P2: Nice to have; ship in v1.x as demand materializes
- P3: Future consideration; gated on external dependencies

---

## Competitor Feature Analysis

| Feature | Kotlin (gold standard for adoption) | Clojure (gold standard for interop ergonomics) | Scala 3 (gold standard for type system) | idris-jvm (predecessor) | Palisade Approach |
|---------|-------------------------------------|------------------------------------------------|----------------------------------------|------------------------|-------------------|
| Java FFI ergonomics | One-line method calls; Java types feel native | Macros/`.` syntax; near-Java terseness; reflective fallback | Strong; some friction with Scala-specific types | Per-method `%foreign` declarations | Decision 4: Clojure-grade via elaborator reflection |
| Java generics projection | Full; Kotlin types project Java generics faithfully | Untyped (`Object` everywhere; runtime checks) | Full; HKT extensions where Java types lack expressivity | Limited; manual signature work | Decision 6: faithful projection from Java signatures |
| Spring DI integration | First-class; `@Service` on Kotlin class works | Via Java interop; no Kotlin-style annotations on Clojure code | First-class; `@Service` on Scala class works | Limited; Java shim layer needed | Decision 5: annotations on Idris-emitted bytecode, no shim |
| Build tools | Kotlin Maven plugin, Gradle Kotlin DSL | Leiningen, deps.edn, tools.build | sbt (custom), Mill, Maven plugin | Limited Maven/Gradle support (open issue) | Decision 10: Maven + Gradle + standalone, all first-class |
| Concurrency | Coroutines (state-machine, compiler-generated) | core.async (CSP), Java executors | ZIO/Cats Effect (purely functional effect runtimes), Akka | None Idris-specific | Active + Decision 18, 19, 20: Linear Futures over Loom; structured concurrency typed |
| Memory model awareness | Defers to JMM; no compile-time enforcement | Persistent data structures + STM (Refs) | Defers to JMM | None | Decision 17: LIMM enforced at type-check |
| Source maps / debugging | JSR-45 + IntelliJ debugger integration | JVM-native + nREPL debugger | JSR-45 (in progress) | Debug info present | Decision 21: full JSR-45 + LineNumberTable + LocalVariableTable + LineNumber on every emitted class |
| GraalVM native-image | Excellent; Spring AOT plugin handles hints | Workable but reflection-heavy makes it hard | Excellent; Scala Native is separate | Not addressed | Decision 7: first-class deployment target |
| REPL | Limited (Kotlin REPL exists but underused) | The killer feature; REPL-driven dev | Worksheets + Ammonite REPL | Idris REPL (offline) | Decision 32: REPL attaches to running JVM |
| Hot reload | Spring DevTools (mature) | First-class; redefine functions in live REPL | Spring DevTools | None | Decision 32: Spring DevTools-style hot reload of Idris classes |
| Test framework | JUnit 5 + Kotest | clojure.test, Midje | ScalaTest, MUnit, Specs2 | Limited (open issue) | Decision 25: native JUnit 5 TestEngine; PBT/PDDT integration |
| Coverage | JaCoCo via JSR-45 | cloverage, Cloverage | scoverage, scalac plugin | Not addressed | Decision 26: JaCoCo + JSR-45 lift to Idris source level |
| Linearity / use-once enforcement | None | None | Affine types in `scala-effekt` (research) | None | Active Requirement: Linear Futures with compile-time use-exactly-once |
| Totality enforcement | None | None | Partial (via `-Xfatal-warnings` and exhaustivity) | Inherited from Idris | Inherited from Idris + Decision 8 boundary preservation |
| Dependent types | None | None | Match types + dependent function types (limited) | Inherited from Idris | Inherited from Idris (compile-time only, erased at runtime per Decision 15) |
| Custom JFR events | None idiomatic | None idiomatic | None idiomatic | None | Decision 22: first-class Idris-vocabulary events |
| OpenTelemetry integration | Manual (Java OTel agent works) | Manual | Manual; some libraries | None | Decision 24: type-safe via Idris effect system |
| Reproducible builds | Possible with effort | Possible with effort | Difficult (sbt non-determinism) | Not addressed | Decision 36: hash-verified CI gate |

### Lessons from the niche-language failure modes

| Project | What it shows | Lesson for Palisade |
|---|---|---|
| **idris-jvm** | Built on Idris 1; large case trees exceeded JVM method size limit; Maven/Gradle/JDBC/REST all listed as gaps; project went mostly read-only | Don't inherit predecessor design constraints. Plan for case-tree size limits upfront. Ship Maven/Gradle from v1, not as TODO. |
| **Eta** (Haskell on JVM) | Strongly-typed FFI was elegant; "wrapping mutable Java stuff means everything ends up in IO/ST" defeated purity gains; project went dormant ~2020 | Idris's QTT + `Either JException T` boundary (Decision 8) preserves purity better than monadic-only FFI; Defensive Membrane (Decision 20) avoids the "everything is IO" infection. |
| **Frege** (Haskell on JVM) | Same purity-leakage issue as Eta; remained niche; "Scala is a better choice" was the recurring critique | Differentiation must be on something Scala/Kotlin *can't* provide (linear types, totality, dependent types) — not on "purity," which Scala provides via Cats Effect/ZIO. |
| **Scala.js** | Successful by *embracing* JS semantics at the boundary (instance tests by value, not type; reflection unsupported) — accepted platform-shaped compromises in clearly-documented places | Be explicit about Palisade's compromises (e.g. dependent types erased at runtime; runtime reflection not supported in emitted code). Document them prominently. |
| **Kotlin** | Won by being *Java-shaped* enough to read at first glance + adding 80%-of-Scala value | Lesson does *not* apply: Palisade is not trying to be the next Kotlin. The thesis is "verified Idris in JVM environments" — not "a more comfortable JVM language." |
| **Clojure** | Won the interop-ergonomics game by treating the JVM as a first-class peer, not a deployment target | Decision 4 (Clojure-grade FFI) and Decision 32 (REPL-attach to running JVM) are direct lessons from Clojure. |

---

## Gaps Identified in PROJECT.md

These are features the roadmap should consider explicitly; they are implied or absent in the 40 Key Decisions but matter for the personas above:

1. **`@Jvm*`-equivalent hint annotations** — Kotlin's `@JvmStatic`, `@JvmName`, `@JvmField`, `@JvmOverloads` are needed for clean Java-side APIs. Decision 5 covers Spring annotations *on* Idris-emitted classes; this gap is about Idris-side declarations *controlling* how the bytecode is named/exposed. **Recommend: add a Decision or roadmap item for "Java-facing naming hints."**

2. **Implementing Java interfaces from Idris** — Required for `Runnable`, `Comparator`, `Filter`, `Function<T,R>`, etc. Implied by Decision 4/5 but not stated. **Recommend: explicit roadmap entry; likely a Phase output.**

3. **Extending Java classes from Idris** — Many frameworks demand subclasses (custom `RuntimeException`, `WebMvcConfigurer`, etc.). Some JVM languages (e.g. Clojure) restrict this. **Recommend: explicit decision (probably "limited subclassing where unavoidable, prefer composition").**

4. **Static method exposure** — Java callers expect `IdrisClass.method(...)`; without it, every Idris module function requires instance gymnastics from Java. **Recommend: explicit roadmap entry; likely satisfied automatically by Decision 5 infrastructure.**

5. **Curated `Maybe`/ADT/collection projection rules** — Mentioned as part of the "interop-contract ring" (Active Requirements) but no Decision pins down the projection strategy. **Recommend: explicit decision (e.g. "Idris ADTs project as sealed-class hierarchy with a record per constructor; `Maybe` projects as `Optional` at boundary, neither converts implicitly").**

6. **HTTP client / JSON SerDe / JDBC bindings** — idris-jvm explicitly listed these as gaps. PROJECT.md doesn't address. Either ship Palisade-curated bindings or document the FFI path explicitly. **Recommend: defer to v1.x but document the FFI walk-through in the Diataxis tutorial set (Decision 29).**

7. **Class redefinition mechanism for hot reload** — Decision 32 says "Spring DevTools-style hot reload" but Spring DevTools relies on JVM HotSwap, which has constraints (no signature changes). **Recommend: explicit scope statement on what changes hot reload supports vs requires restart.**

8. **Documentation of GraalVM-native vs dynamic-JVM tradeoffs** — Decision 7 makes native-image first-class; Decisions 22/24/32 require dynamic JVM. Not a conflict, but engineers will be confused if not explicitly documented. **Recommend: explicit "deployment modes" doc.**

9. **Loom integration roadmap as a coordinated track** — Decisions 18, 19, 20, 24 collectively define the Linear Futures + structured concurrency story. Treating them as four parallel decisions risks loss of coherence. **Recommend: explicit roadmap-level "Loom Integration Track" grouping.**

---

## Sources

### Kotlin (gold standard for JVM-language adoption)
- [Kotlin vs Java 2026: 94% Faster K2 Builds](https://tech-insider.org/kotlin-vs-java-2026/) — adoption stats, K2 compiler
- [Kotlin 2.x vs Java 21+ — Java Code Geeks](https://www.javacodegeeks.com/2026/04/kotlin-2-x-vs-java-21the-language-choice-for-new-jvm-projects.html)
- [Spring Projects in Kotlin — Spring Framework docs](https://docs.spring.io/spring-framework/reference/languages/kotlin/spring-projects-in.html)
- [Spring Dependency Injection With Kotlin — Baeldung](https://www.baeldung.com/kotlin/spring-dependency-injection)
- [Guide to JVM Platform Annotations in Kotlin — Baeldung](https://www.baeldung.com/kotlin/jvm-annotations) — `@JvmStatic`, `@JvmName`, `@JvmField`, `@JvmOverloads`
- [Calling Kotlin from Java — Kotlin docs](https://kotlinlang.org/docs/java-to-kotlin-interop.html)
- [Kotlin LSP — GitHub](https://github.com/Kotlin/kotlin-lsp)
- [Inside Kotlin Coroutines: State Machines, Continuations, and Structured Concurrency — droidcon](https://www.droidcon.com/2025/11/24/inside-kotlin-coroutines-state-machines-continuations-and-structured-concurrency/)
- [Structured Concurrency: Will Java Loom Beat Kotlin's Coroutines? — Xebia](https://xebia.com/blog/structured-concurrency-will-java-loom-beat-kotlins-coroutines-2/)
- [Current State of Spring Boot Native with Kotlin GraalVM — Javarevisited](https://medium.com/javarevisited/current-state-of-spring-boot-native-with-kotlin-graalvm-699b1812cc65)

### Clojure (gold standard for interop ergonomics)
- [Clojure — Java Interop (official)](https://clojure.org/reference/java_interop)
- [Clojure Guides: Language: Java Interop](https://clojure-doc.org/articles/language/interop/)
- [REPL Reloaded — Practicalli Clojure](https://practical.li/clojure/clojure-cli/repl-reloaded/)
- [Why Clojure Developers Love the REPL So Much — Flexiana](https://flexiana.com/news/2025/04/why-clojure-developers-love-the-repl-so-much)
- [insn — Functional JVM bytecode generation for Clojure](https://github.com/jgpc42/insn) — ASM in the Clojure ecosystem

### Scala 3 (gold standard for type system)
- [The Scala Programming Language](https://www.scala-lang.org/)
- [Scala 3 Metaprogramming docs](https://docs.scala-lang.org/scala3/reference/metaprogramming/index.html)
- [Scala 3 ushers in 'complete overhaul' — InfoWorld](https://www.infoworld.com/article/2262802/scala-3-moves-to-release-candidate-stage.html)
- [Scala 3 Type-Level Programming — Rock the JVM](https://rockthejvm.com/articles/scala-3-type-level-programming)

### idris-jvm (predecessor)
- [idris-jvm — GitHub](https://github.com/mmhelloworld/idris-jvm) — features: trampoline, JVM GOTO for tail calls, file/directory/array/IORef/buffer primitives, debug info, dependency analysis
- [Idris 2 Bootstrap on JVM with JVM backend](http://mmhelloworld.github.io/blog/2020/12/30/idris-2-bootstrap-compiler-on-the-jvm-with-a-jvm-backend/)
- [Welcome to idris-jvm Discussions](https://github.com/mmhelloworld/idris-jvm/discussions/113) — explicit gaps: Maven/Gradle, JDBC FFI, REST client, unit testing, JSON SerDe, annotations
- [Use as part of a Maven build (open issue)](https://github.com/mmhelloworld/idris-jvm/issues/109)

### Niche-language backends (Scala.js, Eta, Frege)
- [Scala.js Semantics — what differs from JVM Scala](https://www.scala-js.org/doc/semantics.html)
- [10 years of Scala.js](https://www.scala-lang.org/blog-detail/2023/02/05/ten-years-of-scala-js.html)
- [From first principles: Why I bet on Scala.js — Li Haoyi](http://www.lihaoyi.com/post/FromfirstprinciplesWhyIbetonScalajs.html)
- [Eta Programming Language (Haskell on JVM)](https://eta-lang.org/) — STM, MVars, Fibers, FFI
- [The Story of Eta — Rahul Muttineni](https://medium.com/@rahulmuttineni/the-story-of-eta-pure-love-pure-functional-programming-2a690f3082b4)
- [Frege — GitHub](https://github.com/Frege/frege)
- [Frege, a JVM Haskell — taylor.fausak.me](https://taylor.fausak.me/2015/06/25/frege-a-jvm-haskell/) — purity-leakage critique, niche-stays-niche dynamics

### JVM platform infrastructure
- [JSR-45 — Java Debugging Support for Other Languages](https://jcp.org/en/jsr/detail?id=45)
- [Deciphering the Stacktrace — Inside.java](https://inside.java/2021/02/12/deciphering-the-stacktrace/)
- [Monitoring Java Applications with Flight Recorder — Baeldung](https://www.baeldung.com/java-flight-recorder-monitoring)
- [JFR Streaming + AI monitoring — Inside.java](https://inside.java/2026/03/01/jfr-ai-monitor/)
- [Inject custom JDK Flight Recorder events — Red Hat Developer](https://developers.redhat.com/articles/2022/03/08/inject-custom-jdk-flight-recorder-events-containerized-applications)
- [JDK Flight Recorder with Native Image — GraalVM](https://www.graalvm.org/latest/reference-manual/native-image/debugging-and-diagnostics/JFR/)
- [GraalVM Native Image Support — Spring Boot docs](https://docs.spring.io/spring-boot/docs/3.2.x/reference/html/native-image.html)
- [ASM Developer Guide — OW2](https://asm.ow2.io/developer-guide.html)
- [Stack Map Frames + ASM challenges — JaCoCo issue #1009](https://github.com/jacoco/jacoco/issues/1009)
- [LSP API now available to all IntelliJ IDEA users and plugin developers — JetBrains](https://blog.jetbrains.com/platform/2025/09/the-lsp-api-is-now-available-to-all-intellij-idea-users-and-plugin-developers/)

### Verified / formal-methods context
- [Formal Verification of a Realistic Compiler — CACM (CompCert)](https://cacm.acm.org/research/formal-verification-of-a-realistic-compiler/)
- [Formal Verification of a Java Compiler in Isabelle — ResearchGate](https://www.researchgate.net/publication/2930264_Formal_Verification_of_a_Java_Compiler_in_Isabelle)
- [Differential Testing of a Verification Framework (GraalVM IR + Isabelle/HOL)](https://arxiv.org/pdf/2212.01748)
- [Lean Into Verified Software Development — AWS Open Source Blog](https://aws.amazon.com/blogs/opensource/lean-into-verified-software-development/)
- [Practical Formal Methods in industry (curated list)](https://github.com/ligurio/practical-fm)
- [Revisiting Idris in 2026 — Dev.to](https://dev.to/ujja/revisiting-idris-in-2026-is-it-ready-for-real-work-1mco) — Idris 2 production-readiness commentary

### PROJECT.md cross-references
- `/home/brainfuel/matt/palisade-jvm/.planning/PROJECT.md` — all 40 Key Decisions, Active Requirements, Out of Scope, Constraints
- `/home/brainfuel/matt/palisade-jvm/.planning/codebase/STACK.md` — Idris 2 backend layout, existing capabilities
- `/home/brainfuel/matt/palisade-jvm/.planning/codebase/ARCHITECTURE.md` — IR pipeline, NamedCExp position in the pipeline

---

*Feature research for: JVM compiler backend for verified functional language (Idris 2 → Spring Boot on JVM 25+)*
*Researched: 2026-04-21*
