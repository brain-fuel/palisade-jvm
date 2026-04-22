# Requirements: Palisade

**Defined:** 2026-04-21
**Core Value:** How you smuggle dependent and linear types into a Spring Boot codebase without the JVM noticing you cheated.

**Stance:** Requirements at this level are intentionally category-grain. Phase-level requirements decompose further during plan-phase. Each requirement traces to one or more PROJECT.md Key Decisions (referenced as "D{n}") or research-derived items.

## v1 Requirements

### Foundation (FOUND)

Project-setup invariants that must hold from the first commit; retrofitting any of these is high-cost.

- [ ] **FOUND-01**: Repository vendors a snapshot of Idris 2 `NamedCExp`/`NamedDef` IR (D38)
- [ ] **FOUND-02**: CI runs an automated monthly upstream-sync job that opens a PR rebasing the vendored snapshot against the latest Idris 2 release/commit (D12, D38, Pitfall 11)
- [ ] **FOUND-03**: CI gates every commit with a reproducible-build check (`diffoscope` byte-for-byte hash comparison across two independent build hosts) (D36, Pitfall 12)
- [ ] **FOUND-04**: All Idris-side IR types preserve `FC` (source location) end-to-end through every codegen lowering (Anti-Pattern 3)
- [ ] **FOUND-05**: Repository builds, tests, and ships under EPL 2.0 with proper file-level license headers (D33)
- [ ] **FOUND-06**: CI infrastructure includes a dual-verifier gate (ASM + JEP 484 ClassFile API) producing the same observable bytecode for every emitted class (D27, Reconsider #6)

### Codegen Core (CODE)

The pipeline from upstream IR to verifier-clean JVM bytecode and module-info-bearing JARs.

- [ ] **CODE-01**: User can compile a "hello world" Idris program (`idris2 --cg jvm hello.idr && java -jar hello.jar`) to a runnable JAR (D1)
- [ ] **CODE-02**: User can compile recursive, pattern-matching Idris programs that pass `java -Xverify:all` and execute correctly (D27)
- [ ] **CODE-03**: Compiler emits self-recursive functions as JVM loops; mutual-recursive functions as trampolines (D2)
- [ ] **CODE-04**: Compiler specializes monomorphic `Int`/`Double` to JVM primitive `long`/`double`; `Integer` (unbounded) backs to `BigInteger` (D3)
- [ ] **CODE-05**: Pattern matches compile to `tableswitch`/`lookupswitch` with documented complexity bounds (D2 + Pitfall 1)
- [ ] **CODE-06**: Every emitted class bears `LineNumberTable` and `LocalVariableTable` populated with Idris source positions (D21 "easy half")
- [ ] **CODE-07**: Each Idris package compiles to a JPMS module with auto-emitted `module-info.class` derived from `.ipkg` dependencies; modulepath-clean by default (D40)
- [ ] **CODE-08**: ASM bridge subclasses `ClassWriter` and overrides `getCommonSuperClass` to handle Idris-specific class hierarchies (Architecture: Anti-Pattern, day-one work)
- [ ] **CODE-09**: Compiler emits GraalVM `META-INF/native-image/<group>/<artifact>/reachability-metadata.json` from codegen (not via agent tracing) for every produced JAR (D7, Pitfall 4)
- [ ] **CODE-10**: Codegen surface is reachable through a `Compiler.JVM.Emit.Backend` SPI; `Compiler.JVM.Emit.ASM` is the v1 implementation (D39, Reconsider #2 — enables future ClassFile API swap)
- [ ] **CODE-11**: Generator emits one JVM class per Idris function (Clojure-style "one-class-per-function") to avoid 64KB constant-pool / method-size limits; `@JvmStatic` produces a stable Facade class with delegation methods (gap 4 + user's architectural detail)

### FFI & Interop (FFI)

Bidirectional Idris ↔ Java/Kotlin interop. Adoption depends on this being Clojure-grade.

- [ ] **FFI-01**: Idris user can call any Java method on any Java object with one-line syntax, no `%foreign` declaration per method (D4)
- [ ] **FFI-02**: An importer driven by elaborator reflection generates Idris bindings from Java classpath entries (D4)
- [ ] **FFI-03**: Java generic types project as fully-typed parameterized Idris types via the importer; raw types/wildcards/F-bounded handled (D6, gap)
- [ ] **FFI-04**: Every Java call returning `T` projects as `Either JException T` (or `IO (Either JException T)`) preserving Idris totality across the boundary (D8)
- [ ] **FFI-05**: Idris user can declare `implements JavaInterface` for arbitrary Java interfaces (`Runnable`, `Comparator<T>`, `Filter`, `Function<T,R>`, `Callable<T>`); codegen emits proper interface-implementing class (gap 2)
- [ ] **FFI-06**: Idris user can define exception types extending `RuntimeException`/`Exception`; codegen handles constructors and super calls (gap 3 — exception types only in v1)
- [ ] **FFI-07**: Idris ADTs project as JEP 409 sealed-class hierarchies with a `Record` per constructor; Java callers get exhaustive-match enforcement (gap 5)
- [ ] **FFI-08**: `Maybe a` projects as `java.util.Optional<T>` at the FFI boundary; no implicit conversion (gap 5)
- [ ] **FFI-09**: `Either L R` projects as a Palisade `Result<L, R>` sealed type at the FFI boundary (gap 5)
- [ ] **FFI-10**: Idris definitions can be annotated with `@JvmName` and `@JvmStatic` to control Java-side names and static exposure; `@JvmStatic` generates delegation methods on a per-package Facade class (gap 1, gap 4)
- [ ] **FFI-11**: FFI importer annotates each imported method with blocking-ness (`@Blocking`/`@NonBlocking`/`@MayPin`) — feeds bounded-blocking-pool dispatch in Loom integration (Pitfall 3)

### Linearity & Concurrency (LIN)

The Linear Futures thesis + structured concurrency over Loom. Decision 18/19/20/24 are a coordinated track.

- [ ] **LIN-01**: `LinearFuture` over `CompletableFuture` ships in the runtime support library; compiler enforces use-exactly-once via QTT (Active req — central thesis)
- [ ] **LIN-02**: Linearity is tracked through codegen lowering (`JVMExpr.linearity` annotation lattice) and consumed by downstream LIMM and Loom passes (Architecture)
- [ ] **LIN-03**: Linear values that cross the FFI boundary into Java are wrapped in Guard Proxies with atomic consumed-bit; second consumption throws `PalisadeLinearityViolation` (D20 — Defensive Membrane)
- [ ] **LIN-04**: Internal Idris-to-Idris paths bypass Guard Proxy checks via signature-based dispatch (zero overhead) (D20)
- [ ] **LIN-05**: Compiler emits `VarHandle` acquire/release/opaque ordering for shared (q=n) state; bare unwrapped mutation of shared references is a compile-time error (D17 — LIMM)
- [ ] **LIN-06**: Compiler omits memory barriers for linear (q=1) resources — QTT proves no concurrent access possible (D17)
- [ ] **LIN-07**: Palisade ships an `OrderedContainer` stdlib type backed by `VarHandle`; documented mappings to Java `AtomicReference`/`AtomicLong` and Clojure `atom` (D9, D17)
- [ ] **LIN-08**: `LinearScope` Idris effect wraps `StructuredTaskScope` (preview through JDK 27+); user-facing Idris signatures reference `LinearScope`, not the preview type (D18, Reconsider #1)
- [ ] **LIN-09**: Compiler statically proves every task spawned in a `LinearScope` is joined or cancelled before scope closes — "thread leak" is a compile-time error (D18)
- [ ] **LIN-10**: Stdlib ships `LinearChannel` (zero-copy message passing) and Type-Bound `ScopedValue` helpers (security principals, transaction IDs cannot outlive scope) (D19)
- [ ] **LIN-11**: Compiler-enforced no-escape on `ScopedValue` closures (Pitfall 14); direct calls to `Thread.startVirtualThread` are compile-time errors
- [ ] **LIN-12**: Specialized JUC primitives (`Phaser`, `Exchanger`) are reachable only via FFI — not in core stdlib (D19)

### Performance (PERF)

Trading-grade dispatch + verified zero-overhead claims.

- [ ] **PERF-01**: Internal Idris↔Idris calls aggressively eliminate erased proof terms / unused arguments (Hybrid Metadata Erasure — internal tier) (D15)
- [ ] **PERF-02**: FFI-exported functions retain "Ghost Annotations" capturing erased argument metadata for external static analysis (D15)
- [ ] **PERF-03**: Linearity markers persist as standard JVM annotations on emitted bytecode; SonarQube + Error Prone rules detect Java-side linear-contract violations (D15, Pitfall 10)
- [ ] **PERF-04**: `@hot` / `@specialize` pragma forces monomorphization of polymorphic call sites; codegen emits direct (non-virtual) JVM calls (D16)
- [ ] **PERF-05**: Warm-path dispatch uses `invokedynamic` with Guard-With-Test chain (top 3–4 frequent types) keeping HotSpot inline-cache effective (D16)
- [ ] **PERF-06**: Closed-world fat-JAR builds run link-time analysis; interfaces with 1–2 implementations auto-monomorphize without user pragma (D16)
- [ ] **PERF-07**: Best-effort zero-allocation hot paths: profile + linter + JFR `jdk.ObjectAllocationOutsideTLAB` monitoring; first-class compile-time guarantee deferred until Valhalla value types stabilize (D14)
- [ ] **PERF-08**: Precision Stack Map Frame generator handles trampoline-induced control flow + dependent-type erasure shapes that ASM's `COMPUTE_FRAMES` algorithm cannot reliably infer (D27 precision tier, Pitfall 1)
- [ ] **PERF-09**: JMH-based continuous benchmark suite measures Linear Future vs Kotlin Coroutines vs Java 25 virtual threads; ADT pattern match vs Java sealed-class switching; runs in CI as a blocking gate (D28)
- [ ] **PERF-10**: Benchmark harness instruments JFR for JIT deoptimizations, megamorphic call-site detection, virtual-thread pinning detection (`jdk.VirtualThreadPinned`) (D28, Pitfall 3)
- [ ] **PERF-11**: Reproducible-build CI gate is promoted to cross-host hash-verification (independent build hosts; SLSA L3+ attestation) (D36)

### Spring & Observability (OPS)

The "transparent box" — operational tools see Idris code as Idris code, and Spring sees Idris classes as first-class beans.

- [ ] **OPS-01**: Idris definitions can bear Spring annotations (`@Service`, `@RestController`, `@Autowired`, `@RequestMapping`, etc.); codegen emits annotation entries on the resulting class — no Java/Kotlin shim layer (D5)
- [ ] **OPS-02**: Annotations derived from Idris totality emit `@org.jspecify.annotations.NonNull` / `@Nullable` (Spring 7 standard), not JetBrains annotations (D5, Reconsider #5)
- [ ] **OPS-03**: Palisade build plugins emit `META-INF/spring.components`, `META-INF/spring.factories`, `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` — Spring Context Indexer sees Idris beans (Pitfall 5)
- [ ] **OPS-04**: A `BeanPostProcessor` copies Ghost Annotations onto Spring AOP/CGLIB proxies so AOP doesn't strip linearity metadata (Pitfall 10)
- [ ] **OPS-05**: Every emitted class has `SourceDebugExtension` (JSR-45) populated; stack traces, profilers, heap dumps, JFR events show Idris file:line natively (D21 full)
- [ ] **OPS-06**: Palisade ships a `palisade trace` CLI that decodes generated stack traces back to Idris source positions (Pitfall 7 safety net)
- [ ] **OPS-07**: Generated class/method/field names follow a documented deterministic mangling scheme that round-trips to Idris source identifiers (D23)
- [ ] **OPS-08**: ADT codegen emits metadata so MAT/VisualVM resolve Idris constructors natively without plugins (D23)
- [ ] **OPS-09**: First-class JFR event vocabulary at Idris semantic level: `LinearFutureCompleted`, `EffectPerformed`, `LinearityViolationDetected`, `OrderedContainerAccess`, etc. (D22)
- [ ] **OPS-10**: JFR events are tiered with `@Throttle` rates (always-on vs on-demand); event-frequency taxonomy designed before any event emission to keep overhead within 1–2% budget (D22, Pitfall 8)
- [ ] **OPS-11**: JFR event classes are pre-generated at compile time (not via reflection) for GraalVM native-image compatibility (Pitfall 4 + Pitfall 8)
- [ ] **OPS-12**: OpenTelemetry integrated into the Idris effect system; compiler enforces span open/close across `LinearFuture` lifecycle (D24)
- [ ] **OPS-13**: OTel context propagation across virtual threads uses Palisade-shipped `ScopedValue`-backed bridge (not OpenTelemetry-Java's stock `ThreadLocal`-based bridge, which doesn't inherit) (D24, Pitfall 6)
- [ ] **OPS-14**: Spring Actuator + Micrometer natively export Idris-specific invariants (`StructuredTaskScope` backpressure, `OrderedContainer` contention, Linear Future success/failure rates) (D24)

### Build & Packaging (BUILD)

Both modes: integrate cleanly into Maven/Gradle for hybrid projects, AND `idris2`-owned standalone build that emits a consumable JAR.

- [ ] **BUILD-01**: A `palisade-maven-plugin` integrates with the standard Maven lifecycle; Idris sources coexist with Java/Kotlin in the same module; one JAR output (D10, D30)
- [ ] **BUILD-02**: A `palisade-gradle-plugin` provides equivalent integration for Gradle (Kotlin DSL primary) (D10, D30)
- [ ] **BUILD-03**: Both plugins emit `META-INF/spring.components` and Spring Boot 4 AOT metadata correctly (Pitfall 5)
- [ ] **BUILD-04**: `idris2 build` produces a JPMS-clean, modulepath-deployable JAR independently of Maven/Gradle (D10)
- [ ] **BUILD-05**: A `palisade spring my-service` template generates a runnable Spring Boot service skeleton (Spring Initializr-style): one Idris controller, working tests, JFR config, Spring DevTools wired (D31)
- [ ] **BUILD-06**: The template's CI workflow runs `nativeCompile`, OTel context-continuity tests, and Spring DI autowire tests from day one (D31, Pitfalls 4/5/6)
- [ ] **BUILD-07**: Two officially-supported deployment profiles: `dev/staging` (HotSpot, full observability, hot reload, REPL) and `production-native` (GraalVM native-image, restricted observability) — both tested in CI (D7, gap 8)
- [ ] **BUILD-08**: `production-native` profile supports an opt-in REPL-based dev-tools escape hatch for production troubleshooting (gap 8 — user requirement)
- [ ] **BUILD-09**: JPMS modules default to `open` for Spring reflection compatibility; Palisade detects and refuses split-package configurations (Pitfall 13)

### Standard Library — Alpha Core Baseline (STDLIB-A)

Curated v1 stdlib bindings — first-class Idris-side APIs, not just FFI patterns.

- [ ] **STDLIB-A-01**: Comprehensive survey of Idris 2 stdlib state-management abstractions (`IORef`, `State`, linear state, `Control.App`, STRef-style); each gets a documented JVM AND Clojure idiomatic mapping with rationale (D9)
- [ ] **STDLIB-A-02**: High-Assurance JFR DSL — Idris `Record` definitions generate JFR event classes via ASM at compile time; every IO-bound function emits JFR events with correlation IDs derived from the Idris effect system (user req)
- [ ] **STDLIB-A-03**: JDK-Native HttpClient bindings — Idris bindings for `java.net.http.HttpClient`, zero-dep, async; QTT manages response-body lifecycle and connection-pool release (user req)
- [ ] **STDLIB-A-04**: Jackson JSON serialization — Idris Records/ADTs ↔ Jackson `ObjectMapper`; annotation-based introspection lets Idris-generated classes participate in Spring Boot serialization pipelines (user req)
- [ ] **STDLIB-A-05**: Type-safe JDBC interface — dependent types verify SQL parameter counts and result-set mapping at compile time; HikariCP wraps as the default connection pool (user req)

### Standard Library — High-Assurance Extensions (STDLIB-B)

The Palisade-original stdlib pieces that justify the "high-assurance" framing.

- [ ] **STDLIB-B-01**: `Loom.Structured` module wraps `StructuredTaskScope` via QTT proving compile-time scope-closure; JVM `ScopedValue` maps to Idris implicit parameters for type-safe context propagation (user req)
- [ ] **STDLIB-B-02**: Verified Resilience Patterns — Circuit Breaker with formally-proven state-machine transitions; declarative retry/bulkhead with compiler-verified termination + resource bounds (user req)
- [ ] **STDLIB-B-03**: JEP 510 Security Bridge — wrap JDK 25 KDF API in linear types; map JWT/OAuth2 claims to validated Idris Records; fail-fast on schema/signature failure (user req)
- [ ] **STDLIB-B-04**: PDDT-Enabled Validation — PDDT integrated into validation library; refined types encode invariants checked at boundary, propagated as proofs through business logic (user req — methodology alignment)
- [ ] **STDLIB-B-05**: Modular JPMS Configuration — type-safe loader maps env vars + Vault secrets to Idris ADTs at startup; modulepath-clean (JPMS exports/opens); diagnostic report on startup failure (user req)
- [ ] **STDLIB-B-06**: Virtual-Thread Aware Scheduling — task scheduler optimized for millions of virtual threads; effect-system QoS guarantees; deterministic execution order where required (user req)

### Test & Coverage Infrastructure (TEST)

Mixed-language CI; Idris formal properties run in the same harness as Java services.

- [ ] **TEST-01**: Native JUnit 5 `TestEngine` (engine ID `palisade-idris`) discovers Idris test functions; integrates with Maven Surefire / Gradle Test (D25)
- [ ] **TEST-02**: Idris PBT (Hedgehog/QC-style) and PDDT tests are first-class within the JUnit 5 engine (D25, methodology)
- [ ] **TEST-03**: Idris tests participate in Spring TestContext lifecycle (DI, transactions, mocks) — same harness as Java service tests (D25)
- [ ] **TEST-04**: JaCoCo agent + JSR-45 SourceDebugExtension lift coverage results to Idris source level; per-Idris-function thresholds are enforceable in SonarQube/Codecov gates (D26)
- [ ] **TEST-05**: Mutation testing pipeline (PIT or equivalent) runs against Palisade-emitted bytecode; baseline mutation-killing thresholds documented per category (methodology)
- [ ] **TEST-06**: Membrane adversarial-test corpus — explicit test cases attempt linearity violation via reflection, serialization, lambda capture, Spring AOP/CGLIB proxy, Mockito mock, AspectJ weave, GraalVM native-image proxy. Every case must be either closed by Defensive Membrane or documented as out-of-Membrane (D20, Pitfall 2)

### Developer Experience (DX)

The inner-loop story; matches Clojure's reputation.

- [ ] **DX-01**: Idris LSP is extended to know about JVM types: completion shows imported Java methods/fields with full signatures; hover renders Idris type alongside source Java type (D11, Reconsider #4)
- [ ] **DX-02**: Idris LSP is extended to know about Spring annotations: completion suggests Spring annotations from classpath; hover renders annotation parameters (D11, Reconsider #4)
- [ ] **DX-03**: Sub-second incremental compile for typical Idris module changes (D32)
- [ ] **DX-04**: JRebel-grade hot reload: method bodies, signatures, and new classes apply without service restart via Palisade classloader-restart agent (D32, gap 7 — JRebel-grade chosen)
- [ ] **DX-05**: Hot-reload agent detects LIMM-affecting changes and forces full restart for those (Pitfall 15)
- [ ] **DX-06**: Idris REPL attaches to a running JVM service: introspect Idris types, query definitions, evaluate expressions in the live process — matches Clojure REPL ergonomics (D32)

### Documentation (DOC)

The Diataxis full set — the adoption surface.

- [ ] **DOC-01**: At least 5 tutorials of increasing complexity ("hello world" → multi-module Spring Boot service)
- [ ] **DOC-02**: At least 6 skills/how-to guides covering: implementing Java interfaces from Idris, Spring `@RestController` with Linear Futures, JFR event design, mutation testing setup, hot-reload usage, native-image deployment
- [ ] **DOC-03**: An IAM framework worked example built from scratch using Palisade primitives (Linear Futures, JEP 510 KDF Security Bridge, OTel-as-Proof, sealed-class ADT projection)
- [ ] **DOC-04**: At least 3 "choose-your-adventure" enterprise-ready software examples building on the IAM framework
- [ ] **DOC-05**: Reference manual covering all 40+ Key Decisions with implementation notes, rationale, and tradeoffs
- [ ] **DOC-06**: Operations runbook (the "transparent box" guide) for SREs: how to read Palisade stack traces, debug LinearityViolations in production, interpret Idris-vocabulary JFR events, profile with async-profiler, query a Palisade service via REPL-attach
- [ ] **DOC-07**: Compatibility matrix doc — Palisade version ↔ Idris release ↔ JDK version ↔ Spring Boot version ↔ feature-by-feature deployment-mode applicability (D34, gap 8)
- [ ] **DOC-08**: Migration guide — file-by-file Java → Idris in a Spring Boot module; documents hybrid-module patterns (D30)

### Release & Governance (REL)

Personal-project trajectory toward production-ready.

- [ ] **REL-01**: Strict semver applied to Palisade releases; documented compatibility matrix per release (D34)
- [ ] **REL-02**: Vulnerability disclosure policy (`security@palisade.…` + GitHub security advisories); CVE issuance workflow for confirmed vulnerabilities (D35)
- [ ] **REL-03**: Every release ships with a CycloneDX SBOM and Sigstore signatures; SLSA L3 provenance attached (D35)
- [ ] **REL-04**: Dependency scanning (Dependabot or equivalent) gates merges (D35)
- [ ] **REL-05**: Reproducible-build cross-host verification is part of every release pipeline (D36)
- [ ] **REL-06**: Repository hosted under user's personal or neutral GitHub org; documented contribution guide; CLA optional, governance independent of any single user (D37)

### Reference Service (REF)

The proof artifact: a Spring Boot service in this repo that exercises the full v1 surface.

- [ ] **REF-01**: A working Spring Boot 4 service lives in this repo, built and tested in CI on every commit
- [ ] **REF-02**: At least one HTTP endpoint backed by an Idris `@RestController` returning a sealed-class ADT serialized via Jackson JSON
- [ ] **REF-03**: At least one endpoint fans out async work using `LinearFuture` + `LinearScope` (StructuredTaskScope-based); compiler refuses build if a future is dropped or a task escapes its scope
- [ ] **REF-04**: At least one path emits the full JFR vocabulary (`LinearFutureCompleted`, `EffectPerformed`, `LinearityViolationDetected`); JMC dashboards visualize them
- [ ] **REF-05**: Service deploys cleanly under both deployment profiles (HotSpot dev/staging + GraalVM production-native); both verified in CI
- [ ] **REF-06**: A documented load test demonstrates trading-grade pause profiles under generational ZGC

## v2 Requirements

Acknowledged but deferred past v1.

### Performance

- **PERF-V2-01**: Strict zero-allocation compile-time guarantee (D14) — gated on Valhalla value types reaching GA
- **PERF-V2-02**: Reproducible GraalVM native-image builds (currently a research-level open problem)

### FFI

- **FFI-V2-01**: General Java class extension (`extends JavaClass` for non-exception classes) — `WebMvcConfigurer`, abstract-class extension (gap 3)
- **FFI-V2-02**: `@JvmField` and `@JvmOverloads` annotations (gap 1 — minimal v1 chose @JvmName + @JvmStatic only)

### Security

- **SEC-V2-01**: Third-party security audit before subsequent release milestones (D35 — deferred past v1)

### Stdlib

- **STDLIB-V2-01**: Additional HTTP client choices (OkHttp, Reactor Netty) beyond JDK-Native HttpClient
- **STDLIB-V2-02**: Additional JSON serializers (Jakarta JSON-B, kotlinx.serialization-compatible) beyond Jackson
- **STDLIB-V2-03**: Reactive streams interop (Reactor `Mono`/`Flux` projection)
- **STDLIB-V2-04**: gRPC bindings
- **STDLIB-V2-05**: Kafka client bindings

### Tooling

- **DX-V2-01**: First-party IntelliJ IDEA plugin (v1 ships LSP-only)
- **DX-V2-02**: Cross-platform support beyond JVM (Idris Multiplatform-style targets)

## Out of Scope

Explicitly excluded; documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Building on idris-jvm | Replaces it (D1); consumes vanilla `NamedCExp` directly |
| JVM < 25 support | Loom, generational ZGC, JFR, JPMS, JEP 446 ScopedValues, JEP 484 ClassFile API are load-bearing assets |
| "Idris-flavored JVM language" idioms | Palisade ships verified Idris into JVM environments; does not adapt Idris semantics to JVM convenience |
| Runtime adaptation of Idris syntax for JVM users | The user is the Idris programmer; Palisade does not subset Idris |
| `Serializable` / `Cloneable` on Palisade-emitted classes | Defensive Membrane holes; banned by codegen (Pitfall 2) |
| `Phaser` / `Exchanger` / other rare JUC primitives in core stdlib | Curated subset (D19) — these are FFI-only |
| Idris language-level changes | Upstream contributions only; Palisade is a backend, not a fork of the language |
| Adoption commitments | Personal project (D37); zero adopters acceptable |
| Fixed external deadline (NFM 2026 binding) | Aspirational publication target only |
| JPMC sponsorship | JPMC is a representative target environment, not a sponsor |
| Third-party security audit before v1 | Personal project doesn't fund external audit yet (D35); rest of enterprise security in place |

## Traceability

Each requirement maps to exactly one phase. Populated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-01 | Phase 0 | Pending |
| FOUND-02 | Phase 0 | Pending |
| FOUND-03 | Phase 0 | Pending |
| FOUND-04 | Phase 0 | Pending |
| FOUND-05 | Phase 0 | Pending |
| FOUND-06 | Phase 0 | Pending |
| CODE-01 | Phase 1 | Pending |
| CODE-02 | Phase 1 | Pending |
| CODE-03 | Phase 1 | Pending |
| CODE-04 | Phase 1 | Pending |
| CODE-05 | Phase 1 | Pending |
| CODE-06 | Phase 1 | Pending |
| CODE-07 | Phase 1 | Pending |
| CODE-08 | Phase 1 | Pending |
| CODE-09 | Phase 1 | Pending |
| CODE-10 | Phase 1 | Pending |
| CODE-11 | Phase 1 | Pending |
| FFI-01 | Phase 2 | Pending |
| FFI-02 | Phase 2 | Pending |
| FFI-03 | Phase 2 | Pending |
| FFI-04 | Phase 2 | Pending |
| FFI-05 | Phase 2 | Pending |
| FFI-06 | Phase 2 | Pending |
| FFI-07 | Phase 2 | Pending |
| FFI-08 | Phase 2 | Pending |
| FFI-09 | Phase 2 | Pending |
| FFI-10 | Phase 2 | Pending |
| FFI-11 | Phase 2 | Pending |
| LIN-01 | Phase 3 | Pending |
| LIN-02 | Phase 3 | Pending |
| LIN-03 | Phase 2 | Pending |
| LIN-04 | Phase 2 | Pending |
| LIN-05 | Phase 3 | Pending |
| LIN-06 | Phase 3 | Pending |
| LIN-07 | Phase 3 | Pending |
| LIN-08 | Phase 3 | Pending |
| LIN-09 | Phase 3 | Pending |
| LIN-10 | Phase 3 | Pending |
| LIN-11 | Phase 3 | Pending |
| LIN-12 | Phase 3 | Pending |
| PERF-01 | Phase 2 | Pending |
| PERF-02 | Phase 2 | Pending |
| PERF-03 | Phase 2 | Pending |
| PERF-04 | Phase 6 | Pending |
| PERF-05 | Phase 6 | Pending |
| PERF-06 | Phase 6 | Pending |
| PERF-07 | Phase 6 | Pending |
| PERF-08 | Phase 6 | Pending |
| PERF-09 | Phase 6 | Pending |
| PERF-10 | Phase 6 | Pending |
| PERF-11 | Phase 6 | Pending |
| OPS-01 | Phase 4 | Pending |
| OPS-02 | Phase 4 | Pending |
| OPS-03 | Phase 4 | Pending |
| OPS-04 | Phase 4 | Pending |
| OPS-05 | Phase 5 | Pending |
| OPS-06 | Phase 5 | Pending |
| OPS-07 | Phase 5 | Pending |
| OPS-08 | Phase 5 | Pending |
| OPS-09 | Phase 5 | Pending |
| OPS-10 | Phase 5 | Pending |
| OPS-11 | Phase 5 | Pending |
| OPS-12 | Phase 5 | Pending |
| OPS-13 | Phase 5 | Pending |
| OPS-14 | Phase 5 | Pending |
| BUILD-01 | Phase 7 | Pending |
| BUILD-02 | Phase 7 | Pending |
| BUILD-03 | Phase 7 | Pending |
| BUILD-04 | Phase 7 | Pending |
| BUILD-05 | Phase 7 | Pending |
| BUILD-06 | Phase 7 | Pending |
| BUILD-07 | Phase 7 | Pending |
| BUILD-08 | Phase 7 | Pending |
| BUILD-09 | Phase 7 | Pending |
| STDLIB-A-01 | Phase 3 | Pending |
| STDLIB-A-02 | Phase 5 | Pending |
| STDLIB-A-03 | Phase 3 | Pending |
| STDLIB-A-04 | Phase 4 | Pending |
| STDLIB-A-05 | Phase 4 | Pending |
| STDLIB-B-01 | Phase 3 | Pending |
| STDLIB-B-02 | Phase 4 | Pending |
| STDLIB-B-03 | Phase 4 | Pending |
| STDLIB-B-04 | Phase 6 | Pending |
| STDLIB-B-05 | Phase 4 | Pending |
| STDLIB-B-06 | Phase 3 | Pending |
| TEST-01 | Phase 7 | Pending |
| TEST-02 | Phase 7 | Pending |
| TEST-03 | Phase 7 | Pending |
| TEST-04 | Phase 7 | Pending |
| TEST-05 | Phase 7 | Pending |
| TEST-06 | Phase 7 | Pending |
| DX-01 | Phase 7 | Pending |
| DX-02 | Phase 7 | Pending |
| DX-03 | Phase 7 | Pending |
| DX-04 | Phase 7 | Pending |
| DX-05 | Phase 7 | Pending |
| DX-06 | Phase 7 | Pending |
| DOC-01 | Phase 8 | Pending |
| DOC-02 | Phase 8 | Pending |
| DOC-03 | Phase 8 | Pending |
| DOC-04 | Phase 8 | Pending |
| DOC-05 | Phase 8 | Pending |
| DOC-06 | Phase 8 | Pending |
| DOC-07 | Phase 8 | Pending |
| DOC-08 | Phase 8 | Pending |
| REL-01 | Phase 8 | Pending |
| REL-02 | Phase 8 | Pending |
| REL-03 | Phase 8 | Pending |
| REL-04 | Phase 8 | Pending |
| REL-05 | Phase 8 | Pending |
| REL-06 | Phase 8 | Pending |
| REF-01 | Phase 8 | Pending |
| REF-02 | Phase 8 | Pending |
| REF-03 | Phase 8 | Pending |
| REF-04 | Phase 8 | Pending |
| REF-05 | Phase 8 | Pending |
| REF-06 | Phase 8 | Pending |

**Coverage:**
- v1 requirements: 117 total (FOUND: 6, CODE: 11, FFI: 11, LIN: 12, PERF: 11, OPS: 14, BUILD: 9, STDLIB-A: 5, STDLIB-B: 6, TEST: 6, DX: 6, DOC: 8, REL: 6, REF: 6)
- Mapped to phases: 117 ✓
- Unmapped: 0 ✓

**Per-phase counts:**
- Phase 0 (Project Setup): 6 reqs (all FOUND)
- Phase 1 (Codegen Foundation): 11 reqs (all CODE)
- Phase 2 (FFI + Defensive Membrane): 16 reqs (11 FFI + LIN-03/04 + PERF-01/02/03)
- Phase 3 (Linearity + Loom): 14 reqs (10 LIN core + STDLIB-A-01/03 + STDLIB-B-01/06)
- Phase 4 (Spring Integration): 9 reqs (OPS-01/02/03/04 + STDLIB-A-04/05 + STDLIB-B-02/03/05)
- Phase 5 (Observability): 11 reqs (OPS-05–14 + STDLIB-A-02)
- Phase 6 (Performance): 9 reqs (PERF-04–11 + STDLIB-B-04)
- Phase 7 (Developer Experience + Test): 21 reqs (9 BUILD + 6 DX + 6 TEST)
- Phase 8 (Reference Service + Docs + Release): 20 reqs (6 REF + 8 DOC + 6 REL)

Total: 6+11+16+14+9+11+9+21+20 = **117** ✓

---
*Requirements defined: 2026-04-21*
*Last updated: 2026-04-21 — traceability populated by roadmapper (117/117 mapped)*
