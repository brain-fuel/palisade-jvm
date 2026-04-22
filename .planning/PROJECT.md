# Palisade

## What This Is

Palisade is a JVM backend for Idris 2 that ships verified (linear, dependent, total) Idris code as first-class JVM bytecode — runnable inside Spring Boot services without compromise. It replaces idris-jvm with vanilla Idris 2 plus a new codegen consuming Idris's NamedCExp IR directly, treating the JVM as deployment target rather than design target. Built for environments where the JVM is non-negotiable (JPMorgan Chase being the representative example) and where the type system's guarantees must survive contact with the bytecode.

## Core Value

How you smuggle dependent and linear types into a Spring Boot codebase without the JVM noticing you cheated.

## Requirements

### Validated

<!-- Inherited from upstream Idris 2 — these are foundations Palisade builds on, not Palisade work. -->

- ✓ Idris 2 source parsing — existing
- ✓ TTImp elaboration and type checking — existing
- ✓ Core type system with dependent types (`Core.TT`) — existing
- ✓ Case compilation and pattern coverage checking — existing
- ✓ Multi-stage IR pipeline (CExp → LiftedDef → ANFDef → VMCode) — existing
- ✓ Existing backends (Chez, Racket, Gambit, RefC, ES, Node, Interpreter) — existing; will be paralleled by JVM
- ✓ Incremental compilation via TTC binary cache — existing
- ✓ Linear types via QTT — existing
- ✓ Totality checking — existing
- ✓ Effect system (`Control.App`) — existing
- ✓ Idris LSP — existing; to be extended for JVM/Spring

### Active

<!-- Requirements emerge iteratively phase-by-phase rather than being fully scoped up front. The high-level v1 scope: -->

- [ ] **JVM codegen** under `src/Compiler/JVM/` consuming `NamedCExp` directly
- [ ] **Linear Futures over CompletableFuture** — the central thesis (use-exactly-once enforced at compile time)
- [ ] **Compiler-path ring**: `NamedCExp` → bytecode → modular JAR
- [ ] **Interop-contract ring**: projection rules for `Maybe`, ADTs, exceptions, generics, Spring annotations
- [ ] **Operational-surface ring**: JFR events, source maps, hot reload, Spring Boot integration, GC tuning, debuggability
- [ ] **Reference Spring Boot service** in this repo, fully wired (Linear Futures handling real async work, `Maybe`/ADT projection, JFR emission, runnable JAR)
- [ ] **All 40 Key Decisions** (below) implemented to the personal-confidence bar: "I'd be happy using this in any extreme production situation on JVM 25+"

### Out of Scope

- **Building on idris-jvm** — Palisade replaces it; consumes Idris 2's `NamedCExp` IR directly. — *idris-jvm's design constraints aren't ours; full control over codegen*
- **JVM < 25 support** — Loom, generational ZGC, JFR, JPMS, JEP 446 ScopedValues, JEP 484 ClassFile API are load-bearing assets, not optional. — *modern JVM features are part of the value prop*
- **"Idris-flavored JVM language"** — Palisade ships verified Idris into JVM environments; it does not adapt Idris semantics to JVM idioms. — *the guarantees are the product*
- **Adoption commitments** — no stakeholder is contractually using Palisade; even zero adopters is acceptable. — *personal project; quality bar is internal*
- **Fixed external deadline** — NFM 2026 talk is aspirational; can slip to NFM 2027 or another venue. — *quality before timing*
- **JPMC sponsorship or stewardship** — JPMC is a representative target environment, not the project owner. — *independent OSS*
- **Third-party security audit before v1** — deferred. — *personal project doesn't fund external audit yet; full enterprise security process otherwise in place*

## Context

**Codebase state.** Existing checkout (`/home/brainfuel/matt/palisade-jvm`) is upstream Idris 2 0.8.0+ on branch `feat/palisade-jvm`, currently identical to `main` (commit `214eb4547`). No Palisade code yet. Codebase map at `.planning/codebase/` describes the upstream Idris 2 architecture Palisade extends.

**Backend pattern.** Upstream Idris 2 organizes backends as `src/Compiler/{Scheme,RefC,ES,Interpreter}/`. Palisade adds `src/Compiler/JVM/` paralleling this layout.

**Idris runtime model.** Idris's runtime model (thunks for laziness, totality erased, dependent types erased, linearity erased at runtime) requires translation to bytecode. Linearity and totality become compile-time guarantees with no runtime representation in Idris's own model — Palisade preserves them at the FFI boundary via Guard Proxies (Defensive Membrane pattern, Decision 20).

**Companion projects.** Arca / CALM are companion projects in the same orbit, exploring verified-systems architecture and IaC. Palisade is the JVM-runtime leg of a larger verified-systems story: verified architecture (IaC) + verified services (JVM runtime), same type system underneath. NFM 2026 (or later venue) is the aspirational publication target.

**Engineering methodology.** PDDT (Parameterized Data-Driven Testing), PBT (Property-Based Testing), and Mutation Testing are existing parts of the methodology and inform the Test Framework decision (Decision 25 — High-Assurance Test Bridge).

**State management mapping.** Idris 2 stdlib has multiple state-management abstractions (`IORef`, `State`, linear state, `Control.App`, possibly STRef-style). Each gets surveyed and mapped to its idiomatic Java AND Clojure counterpart (Decision 9). Don't ship one `IORef` binding — ship the whole story.

**Iterative requirements stance.** Requirements not yet scoped will emerge phase-by-phase. PROJECT.md captures vision, principles, and decisions; REQUIREMENTS.md and phase-level requirements grow as they crystallize.

**The "Jane Street bar".** Design decisions throughout were probed against the question: "What would make this the kind of thing a Jane-Street-equivalent shop would actually adopt for trading systems?" The bar drove decisions on GC, zero-alloc, JIT shape, memory model, JFR vocabulary, bytecode verification, benchmarks, and reproducible builds.

## Constraints

- **Tech stack**: Vanilla upstream Idris 2 + own JVM backend codegen consuming `NamedCExp` — replaces idris-jvm
- **Platform**: JVM 25+ — Loom, generational ZGC, JFR, JPMS, JEP 446 ScopedValues, JEP 484 ClassFile API are load-bearing
- **Deployment substrate**: Spring Boot — the dominant target service shape; Palisade must be first-class within it
- **License**: EPL 2.0 — same family as Clojure; weak-copyleft (per-file); Apache-compatible
- **Idris version pinning**: tagged release + periodic rebase; "unstable" builds track commits between release tags
- **NamedCExp dependency**: vendor snapshot in Palisade tree with controlled upstream merges (insulates codegen from upstream IR churn)
- **Bytecode-emission library**: ASM (battle-tested across Kotlin/Scala/Groovy)
- **Quality bar**: "I'd be happy using this in any extreme production situation on JVM 25+" — personal confidence bar, not external SLA

## Key Decisions

<!-- Each decision below has a concrete rationale tied to the v1 personal-confidence bar. Outcomes are "— Pending" at initialization; phases will revisit them. -->

| # | Decision | Rationale | Outcome |
|---|----------|-----------|---------|
| 1 | Vanilla Idris 2 + own JVM codegen consuming `NamedCExp` directly; new `src/Compiler/JVM/` paralleling existing backend layout | Replaces idris-jvm; full control over codegen; idiomatic to upstream backend pattern | — Pending |
| 2 | Tail calls: direct self-recursion → JVM loop; mutual → trampoline | JVM lacks general TCO; Clojure-style is proven; matches Idris's heavy recursion idiom | — Pending |
| 3 | Primitives: codegen specialization for monomorphic `Int`/`Double`; `BigInteger` for unbounded precision | Matches Idris `Integer` semantics; competitive numerics on JVM | — Pending |
| 4 | FFI ergonomics: Clojure-grade via elaborator-reflection-driven importer; one-line Java method calls without per-method `%foreign` | Adoption depends on it; Kotlin engineers expect this baseline ergonomics | — Pending |
| 5 | Spring DI: annotations on Idris-emitted bytecode (no Java/Kotlin shim layer); Idris classes are first-class Spring beans | Pairs with FFI choice; demands elaborator-reflection-driven codegen | — Pending |
| 6 | Java generics: faithful projection; importer reads Java signatures and generates fully-typed parameterized Idris types; erasure happens at codegen | Required for typed Spring API signatures (e.g. `Mono<ResponseEntity<List<User>>>`) | — Pending |
| 7 | GraalVM native-image: first-class deployment target; codegen emits AOT-friendly metadata | Modern JVM stacks demand it; constrains FFI metadata generation | — Pending |
| 8 | Exception ↔ totality bridge: always wrap Java calls in `Either JException T` (or `IO (Either JException T)`) at the FFI boundary | Preserves totality across every Java boundary | — Pending |
| 9 | State management: comprehensive survey of Idris stdlib state abstractions (`IORef`, `State`, linear state, `Control.App`, etc.); each gets a documented JVM AND Clojure idiomatic mapping (`AtomicReference`/atom, `ThreadLocal`/`binding`, `ScopedValue`, `ConcurrentHashMap`/PersistentMap, etc.) | Idris devs shouldn't be forced to FFI for things Clojure devs get natively | — Pending |
| 10 | Build: both modes — first-class Maven + Gradle plugins (mixed-language projects) AND `idris2`-owned build that emits a consumable JAR | Engineers need both flows | — Pending |
| 11 | IDE: LSP-based, IDE-agnostic; extend Idris LSP to know JVM types and Spring annotations | Broad reach; lower investment than IntelliJ plugin | — Pending |
| 12 | Idris pinning: tagged release with periodic rebase; "unstable" builds track commits between release tags | Stable releases for production; fast-track upstream features when needed | — Pending |
| 13 | GC default: ZGC (override JDK 25's G1 default for Palisade) | Low-pause, generational; trading-grade pause profiles | — Pending |
| 14 | Zero-allocation hot paths: best-effort with JFR/linting today; first-class compile-time guarantee once Valhalla value types stabilize | Polymorphism + erasure forces boxing today; Valhalla unblocks the strict contract | — Pending |
| 15 | Erasure: Hybrid Metadata Erasure — aggressive elimination on internal Idris↔Idris calls; "Ghost Annotations" preserving erased-arg metadata at FFI boundary; linearity markers retained as JVM attributes for external static analysis (SonarQube, custom JPMC linters) | Internal speed + external verifiability | — Pending |
| 16 | JIT shape: tiered — `@hot`/`@specialize` pragma forces monomorphization; `invokedynamic` with Guard-With-Test chain (top 3–4 frequent types) for warm paths; link-time closed-world analysis auto-monomorphizes interfaces with 1–2 implementations in fat-JAR builds | Trading-grade dispatch without across-the-board code bloat | — Pending |
| 17 | Memory model: LIMM (Linear-Inferred Memory Model) — QTT lets compiler omit barriers for linear (q=1) resources; shared (q=n) state must use "Ordered Containers" mapped to `VarHandle` acquire/release/opaque; bare mutation of shared references is a compile-time error | Type-checks JMM safety; C-level perf on linear hot paths, Java-level safety for shared state | — Pending |
| 18 | Loom integration: Structured Linear Runtime — virtual thread lifecycle bound to `LinearFuture`; compiler statically proves every `StructuredTaskScope` task is joined or cancelled before scope closes | Static thread-leak prevention; Loom becomes a formal-safety primitive, not just a throughput optimization | — Pending |
| 19 | Concurrency primitives: curated subset — Linear Channels (zero-copy), Type-Bound `ScopedValues`, `StructuredTaskScope` formalized as Idris effect; specialized JUC primitives (`Phaser`, `Exchanger`) relegated to FFI boundary | Lean stdlib aligned with JDK 25 concurrency model | — Pending |
| 20 | Linearity safety at FFI: Defensive Membrane — Guard Proxies for linear types at Java boundary, with atomic consumed-bit + `PalisadeLinearityViolation`; zero overhead on internal Idris-to-Idris paths via signature-based dispatch | Idris guarantees survive contact with the JVM heap | — Pending |
| 21 | Source maps: Telemetry-Native Mapping — `SourceDebugExtension` (JSR-45) + `LineNumberTable` + `LocalVariableTable` populated for every emitted class; Idris file:line visible in stack traces, profilers, heap dumps, JFR events | Production debugging precision; high-assurance systems demand absolute source ↔ bytecode link | — Pending |
| 22 | JFR: first-class Idris-vocabulary events emitted into the JFR stream — `LinearFutureCompleted`, `EffectPerformed`, `LinearityViolationDetected`, `OrderedContainerAccess`, etc. | JMC functions as a production formal-verification monitor; "transparent box" for ops teams | — Pending |
| 23 | Profiler/heap: Semantic Naming with ADT Reconstruction — deterministic mangling to human-readable JVM-safe identifiers; ADT codegen emits metadata so MAT/VisualVM resolve constructors natively without plugins | Operational tools see Idris code as Idris code | — Pending |
| 24 | Telemetry: Observability-as-Proof — OpenTelemetry integrated into the Idris effect system; compiler enforces span open/close across `LinearFuture` lifecycle (solves "lost trace" in async); Spring Actuator + Micrometer natively export Idris-specific invariants (backpressure, container contention) | Observability becomes a type-safe requirement, not a side effect | — Pending |
| 25 | Test framework: High-Assurance Test Bridge — native JUnit 5 `TestEngine`; Idris PBT + PDDT discoverable in Maven/Gradle; Spring TestContext integration (DI, transactions, mocks) | Mixed-language CI; Idris formal properties run in the same harness as Java services | — Pending |
| 26 | Coverage: Lifted Semantic Coverage — JaCoCo agent + JSR-45 source map lift coverage to Idris source level; SonarQube/Codecov gates enforce per-Idris-function thresholds | Coverage of proof-dependent paths visible in enterprise CI | — Pending |
| 27 | Bytecode verification: Zero-Tolerance Verification Contract — every emitted class passes `java -Xverify:all` as a CI build-breaker; precision Stack Map Frame generator handles tail-call trampolining + dependent-type erasure complexity | Binary-level reliability; trampoline-induced control flow is verifiable | — Pending |
| 28 | Benchmarks: Empirical Verification Suite — JMH-based continuous benchmarks vs Kotlin Coroutines, Java 25 virtual threads, sealed-class switching; JFR-monitored for JIT deopts and megamorphic dispatch; blocking CI gate | Trading-grade perf evidence; performance becomes a verifiable artifact | — Pending |
| 29 | Documentation: full Diataxis Framework — minimum 5 tutorials of increasing complexity, 6 skills, an IAM framework worked example, ≥3 "choose-your-adventure" enterprise-ready software examples covering all four Diataxis quadrants (tutorial, how-to, reference, explanation) | Adoption-grade depth | — Pending |
| 30 | Migration: hybrid-module first-class — Java + Kotlin + Idris source files coexist in a single Spring Boot module; Palisade compiler integrates with `javac`/`kotlinc` lifecycle; one JAR | File-by-file or method-by-method conversion in live services | — Pending |
| 31 | Onboarding: `palisade spring my-service` Spring-Initializr-style template — generates a runnable Spring Boot service with Idris controller, working tests, JFR config, the lot | Frictionless first 5 minutes | — Pending |
| 32 | Dev loop: best-in-class — sub-second incremental Idris compile; Spring DevTools-style hot reload of Idris classes; Idris REPL attaches to running JVM (introspect types, query definitions, eval in live process) | Match Clojure REPL ergonomics | — Pending |
| 33 | License: EPL 2.0 | Same family as Clojure; weak-copyleft per-file; Apache-compatible | — Pending |
| 34 | Versioning: strict semver + published compatibility matrix (Palisade ↔ Idris release ↔ JDK ↔ Spring Boot) | Enterprise predictability | — Pending |
| 35 | Security: full enterprise process — disclosure policy, GH security advisories, CVE issuance, signed releases, CycloneDX SBOM per release, SLSA provenance, dependency scanning; third-party audit deferred past v1 | Personal project doesn't fund external audit yet; everything else is in place | — Pending |
| 36 | Reproducible builds: by contract — same source + same compiler version → byte-identical JAR; CI verifies via hash comparison across two independent build hosts; SLSA L3+ attestation | Supply-chain integrity | — Pending |
| 37 | Governance: independent OSS, personal project; adoption-agnostic | Personal craftsmanship; zero adopters is acceptable | — Pending |
| 38 | NamedCExp dependency: vendor snapshot in Palisade tree with controlled upstream merges | Maximum codegen stability against upstream IR churn | — Pending |
| 39 | Bytecode-emission library: ASM | Battle-tested across Kotlin, Scala, Groovy; covers every JVM verifier edge case | — Pending |
| 40 | JPMS modules: auto-emit `module-info.class` per Idris package, derived from `.ipkg` dependencies; modulepath-clean by default | Required for modern JVM service deployment at scale | — Pending |

---
*Last updated: 2026-04-21 after initialization*
