# Roadmap: Palisade

## Overview

Palisade ships verified (linear, dependent, total) Idris 2 code as first-class JVM bytecode runnable inside Spring Boot services on JVM 25+. The roadmap follows the **six-Floor build-order** derived from Architecture research — hard technical dependencies, not arbitrary phasing. Phase 0 establishes supply-chain and verification invariants that retrofitting would be ruinous (vendored `NamedCExp`, dual-verifier CI, reproducible builds, `FC` discipline). Phases 1–6 climb the Floors: Codegen Foundation → FFI + Defensive Membrane → Linearity + Loom (the central thesis) → Spring Integration → Observability ("transparent box") → Performance. Phase 7 lands developer experience (build plugins, LSP, hot reload, REPL-attach, test infrastructure). Phase 8 closes with the reference Spring Boot service, full Diataxis documentation, and the release/governance discipline that converts "verified Idris on JVM" into a personal-confidence-bar v1. The journey is 117 v1 requirements across 14 categories; every requirement maps to exactly one phase, ordered by what must be true before the next phase can start.

## Phases

**Phase Numbering:**
- Integer phases (0, 1, 2, …): Planned milestone work
- Decimal phases (e.g., 2.1): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 0: Project Setup** - Vendored IR snapshot, dual-verifier CI, reproducible builds, FC discipline — invariants from first commit
- [ ] **Phase 1: Codegen Foundation** - `hello world` compiles to verifier-clean modular JAR; recursive pattern-matching Idris runs with Idris file:line in stack traces
- [ ] **Phase 2: FFI + Defensive Membrane** - Clojure-grade Java FFI; `Either JException T` boundary; linear values across FFI refuse double-consumption
- [ ] **Phase 3: Linearity + Loom (the thesis)** - `LinearFuture` + `LinearScope` ship; compiler refuses dropped futures or escaped tasks; LIMM elides barriers on q=1
- [ ] **Phase 4: Spring Integration** - Idris classes are first-class Spring beans (JSpecify-typed); Spring discovers Idris-emitted components without shim layer
- [ ] **Phase 5: Observability (transparent box)** - JFR/profilers/heap dumps/OTel resolve to Idris source; Idris-vocabulary JFR events; OTel context survives virtual-thread boundaries
- [ ] **Phase 6: Performance** - `@hot`/`@specialize`, precision Stack Map Frames, JMH benchmarks gate CI, cross-host reproducibility
- [ ] **Phase 7: Developer Experience + Test Infrastructure** - Maven/Gradle plugins, LSP JVM/Spring extension, JRebel-grade hot reload, REPL-attach, JUnit 5 TestEngine, JaCoCo lift
- [ ] **Phase 8: Reference Service + Documentation + Release** - Working Spring Boot reference service, Diataxis full set (5 tutorials + 6 skills + IAM example + 3 enterprise-ready), signed/attested releases

## Phase Details

### Phase 0: Project Setup
**Goal**: Establish project-setup invariants that must hold from the first commit; retrofitting any of them is high-cost (vendor drift, reproducibility breakage, FC loss, verifier regressions).
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04, FOUND-05, FOUND-06
**Success Criteria** (what must be TRUE):
  1. Repository contains a vendored snapshot of Idris 2 `NamedCExp`/`NamedDef` IR with a documented upstream commit hash, and `idris2 --check` against the vendored snapshot succeeds (D38)
  2. CI runs an automated monthly upstream-sync job that opens a PR rebasing the vendored snapshot against the latest Idris 2 release/commit, with diffoscope output attached (D12, D38, Pitfall 11)
  3. CI fails any commit whose two-host build does not produce byte-identical JAR output (`diffoscope` exit 0); SLSA L3+ provenance template wired (D36, Pitfall 12)
  4. Every IR type along the planned codegen pipeline carries a non-droppable `FC` field (compile-time enforced by absence of `FC`-stripping smart constructors); CI lints reject any code path that drops `FC` (Anti-Pattern 3)
  5. Every emitted source file bears the EPL 2.0 SPDX header; license CI gate fails on any file lacking the header (D33)
  6. Dual-verifier CI infrastructure exists and runs against an empty stub class: ASM emitter and JEP 484 ClassFile API emitter both pass `-Xverify:all`, JaCoCo round-trip, and ByteBuddy round-trip on the stub (D27 contract; Reconsider #6; Pitfall 1 prevention)
**Plans**: TBD

### Phase 1: Codegen Foundation (Ground + First Floor)
**Goal**: A "hello world" Idris program compiles via `idris2 --cg jvm` to a verifier-clean modular JAR that runs on `java`. Non-trivial recursive pattern-matching Idris programs compile and run correctly, with stack traces showing Idris file:line. GraalVM reachability metadata is emitted from codegen, not retrofitted.
**Depends on**: Phase 0
**Requirements**: CODE-01, CODE-02, CODE-03, CODE-04, CODE-05, CODE-06, CODE-07, CODE-08, CODE-09, CODE-10, CODE-11
**Success Criteria** (what must be TRUE):
  1. `idris2 --cg jvm hello.idr && java -jar hello.jar` prints "hello world" (D1, CODE-01)
  2. A recursive, pattern-matching Idris program (e.g. `fact`, `map`, `sum`, `Maybe`/`Either` cases) compiles to bytecode that passes `java -Xverify:all` AND survives a JaCoCo + ByteBuddy round-trip without `VerifyError` (D27 + Pitfall 1 + CODE-02/05)
  3. Self-recursive functions emit as JVM loops (zero JVM stack growth confirmed via JFR); mutual-recursive functions emit as trampolines (verified via decompiled bytecode inspection in CI fixture) (D2, CODE-03)
  4. `Int`/`Double` operations compile to JVM `long`/`double` primitives (no boxing in tight numeric loops, confirmed by `jdk.ObjectAllocationOutsideTLAB` JFR sample); `Integer` (unbounded) backs to `BigInteger` (D3, CODE-04)
  5. A stack trace from a thrown exception shows the Idris source file and line number for every Palisade-emitted frame (D21 "easy half", CODE-06)
  6. Each Idris package compiles to a JPMS module with auto-emitted `module-info.class` derived from `.ipkg` dependencies; `jdeps --check` reports modulepath-clean (D40, CODE-07)
  7. `Compiler.JVM.Emit.Backend` SPI exists with `Compiler.JVM.Emit.ASM` as v1 implementation; `IdrisClassWriter` overrides `getCommonSuperClass` from day one; codegen emits `META-INF/native-image/.../reachability-metadata.json` for every produced JAR (D7, D39, Reconsider #2, Pitfall 4, CODE-08/09/10)
  8. Each Idris function emits exactly one JVM class (Clojure-style); `@JvmStatic` produces a stable per-package Facade class with delegation methods, verified by a fixture program calling Idris from Java without `%foreign` (CODE-11)
**Plans**: TBD

### Phase 2: FFI + Defensive Membrane (Second Floor)
**Goal**: Clojure-grade FFI lands together with the Defensive Membrane. An Idris program calls arbitrary Java methods one-line; every Java call returns `Either JException T` preserving totality; linear values that cross the FFI boundary into Java are protected by Guard Proxies that reject double-consumption; internal Idris-to-Idris paths bypass Membrane checks at zero cost. Hybrid Metadata Erasure ships using standard JVM annotations (Pitfall 10).
**Depends on**: Phase 1
**Requirements**: FFI-01, FFI-02, FFI-03, FFI-04, FFI-05, FFI-06, FFI-07, FFI-08, FFI-09, FFI-10, FFI-11, LIN-03, LIN-04, PERF-01, PERF-02, PERF-03
**Success Criteria** (what must be TRUE):
  1. An Idris user calls any public Java method on any Java object with one line of Idris syntax, no per-method `%foreign` declaration; exhibited by calling `String.length`, `List.add`, `HashMap.put`, and `CompletableFuture.thenApply` from a fixture (D4, FFI-01)
  2. The elaborator-reflection-driven importer ingests `java.util.*`, `java.net.http.*`, and a representative Spring Boot 4 classpath; generates Idris bindings with fully-typed parameterized signatures (raw types, wildcards, F-bounded handled — `Mono<ResponseEntity<List<User>>>` projects correctly) (D4, D6, FFI-02/03)
  3. Every Java call returning `T` projects as `Either JException T` (or `IO (Either JException T)`); a fixture program proving this for `FileInputStream`, `URL.openStream`, and `Class.forName` (D8, FFI-04)
  4. Idris user can declare `implements Runnable`, `implements Comparator<T>`, `implements Function<T,R>`, `implements Callable<T>` and have the codegen emit a proper interface-implementing class consumable from Java; user can define an exception type extending `RuntimeException` with custom constructors (FFI-05/06)
  5. `Maybe a` projects as `java.util.Optional<T>` at the boundary (no implicit conversion); `Either L R` projects as Palisade `Result<L, R>` sealed type; Idris ADT with N constructors projects as a JEP 409 sealed-class hierarchy with N records, exhaustive-match enforced by `javac` (FFI-07/08/09)
  6. `@JvmName` and `@JvmStatic` annotations on Idris definitions control Java-side names; `@JvmStatic` generates delegation methods on a per-package Facade class (FFI-10)
  7. A `LinearFuture` returned to Java code is wrapped in a Guard Proxy with atomic consumed-bit; second consumption (via reflection or direct call from Java) throws `PalisadeLinearityViolation`; internal Idris-to-Idris consumption uses signature-based dispatch and incurs zero overhead (verified by JMH micro-benchmark) (D20, LIN-03/04)
  8. Erased proof terms / unused arguments are eliminated on internal Idris-to-Idris calls (verified by decompiling representative output); FFI-exported functions retain "Ghost Annotations" using **standard JVM `@Retention(RUNTIME)` annotation classes**, not custom attributes (so SonarQube + Error Prone can detect Java-side linear-contract violations); FFI importer annotates each imported method with `@Blocking`/`@NonBlocking`/`@MayPin` (D15, Pitfall 10, PERF-01/02/03, FFI-11)
**Plans**: TBD

### Phase 3: Linearity + Loom — The Thesis (Third Floor)
**Goal**: The central thesis ships. `LinearFuture` over `CompletableFuture` is the runtime primitive; QTT enforces use-exactly-once at compile time. LIMM elides memory barriers on linear (q=1) resources and forces `VarHandle` ordering on shared (q=n) state. `LinearScope` (a Palisade-stable Idris effect wrapping `StructuredTaskScope`) statically proves every spawned task is joined or cancelled. State-management stdlib survey lands so Idris devs aren't forced to FFI for what Clojure devs get natively.
**Depends on**: Phase 2
**Requirements**: LIN-01, LIN-02, LIN-05, LIN-06, LIN-07, LIN-08, LIN-09, LIN-10, LIN-11, LIN-12, STDLIB-A-01, STDLIB-A-03, STDLIB-B-01, STDLIB-B-06
**Success Criteria** (what must be TRUE):
  1. `LinearFuture` ships in the runtime support library; an Idris program that consumes a `LinearFuture` twice fails to compile with a QTT linearity error; one that drops a `LinearFuture` (never consumes it) fails to compile (LIN-01, Active req — central thesis)
  2. Compiler emits `VarHandle` acquire/release/opaque ordering for shared (q=n) state mutation; bare unwrapped mutation of shared references fails to compile with a documented LIMM error; linear (q=1) resources emit zero memory barriers (verified by decompiled output diff) (D17, LIN-02/05/06)
  3. Stdlib `OrderedContainer` type backed by `VarHandle` ships with documented mappings to Java `AtomicReference`/`AtomicLong` and Clojure `atom`; comprehensive survey doc maps every Idris stdlib state-management abstraction (`IORef`, `State`, linear state, `Control.App`, STRef-style) to its idiomatic Java AND Clojure counterpart (D9, D17, LIN-07, STDLIB-A-01)
  4. `LinearScope` Idris effect wraps `StructuredTaskScope` (preview-stable; user-facing Idris signatures never reference the preview type); a fixture spawning N tasks and failing to join one fails to compile with a "thread leak" error; direct calls to `Thread.startVirtualThread` fail to compile (D18, Reconsider #1, LIN-08/09/11)
  5. Stdlib ships `LinearChannel` (zero-copy message passing) and Type-Bound `ScopedValue` helpers; compiler-enforced no-escape on `ScopedValue` closures (a fixture leaking a `ScopedValue` reference fails to compile); `Phaser`/`Exchanger` are reachable only via FFI (D19, Pitfall 14, LIN-10/12)
  6. Stdlib ships `Loom.Structured` (QTT-proven scope closure), JDK-Native `HttpClient` bindings (QTT manages response-body lifecycle), and a virtual-thread-aware scheduler optimized for millions of vthreads with effect-system QoS guarantees (STDLIB-A-03, STDLIB-B-01, STDLIB-B-06)
**Plans**: TBD

### Phase 4: Spring Integration
**Goal**: Idris classes are first-class Spring Boot 4 beans. Spring's Context Indexer discovers Idris-emitted components without a Java/Kotlin shim layer. Annotations derived from Idris totality emit as **JSpecify** (`@org.jspecify.annotations.NonNull`/`@Nullable`), the Spring Framework 7 standard. Spring AOP/CGLIB proxies preserve linearity metadata. The full enterprise Spring stack — Jackson serialization, JDBC with HikariCP, JEP 510 KDF Security Bridge, declarative resilience patterns, type-safe JPMS configuration — is integrated.
**Depends on**: Phase 3 (linearity must be in place before Spring beans expose `LinearFuture`)
**Requirements**: OPS-01, OPS-02, OPS-03, OPS-04, STDLIB-A-04, STDLIB-A-05, STDLIB-B-02, STDLIB-B-03, STDLIB-B-05
**Success Criteria** (what must be TRUE):
  1. An Idris definition annotated `@RestController` + `@RequestMapping` is autowired by Spring on application startup, returns from an HTTP endpoint, and is discovered without classpath fallback; `@Service` + `@Autowired` work analogously; codegen emits annotation entries on the resulting class with no Java/Kotlin shim (D5, OPS-01)
  2. Annotations derived from Idris totality emit as `@org.jspecify.annotations.NonNull`/`@Nullable` (NOT JetBrains `@NotNull`/`@Nullable`); Spring 7's null-safety verifier consumes them correctly on a fixture endpoint (D5, Reconsider #5, OPS-02)
  3. Palisade build plugins emit `META-INF/spring.components`, `META-INF/spring.factories`, and `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`; CI cross-checks the index against a full classpath scan and fails on drift — Spring discovers all Idris beans on the index path (Pitfall 5, OPS-03)
  4. A `BeanPostProcessor` ships that copies Ghost Annotations onto Spring AOP/CGLIB proxies; a fixture wrapping a `@Service` Idris class with `@Transactional` proves the `@Linear` annotation survives proxying (Pitfall 10, OPS-04)
  5. Idris Records and ADTs serialize via Jackson `ObjectMapper` (annotation-based introspection) and participate in Spring Boot serialization pipelines; type-safe JDBC stdlib verifies SQL parameter counts and result-set mappings at compile time, backed by HikariCP (STDLIB-A-04/05)
  6. Verified Resilience Patterns (Circuit Breaker with formally-proven state-machine transitions; declarative retry/bulkhead with compiler-verified termination + resource bounds), JEP 510 Security Bridge (linear-typed KDF; JWT/OAuth2 claims to validated Idris Records), and type-safe modulepath-clean JPMS configuration loader (env vars + Vault secrets → Idris ADTs at startup; diagnostic report on failure) all integrate cleanly with the Spring fixture (STDLIB-B-02/03/05)
**Plans**: TBD

### Phase 5: Observability — Transparent Box
**Goal**: Operational tools see Idris code as Idris code. `SourceDebugExtension` (JSR-45) populated on every emitted class so stack traces, profilers, heap dumps, and JFR events show Idris file:line natively. ADT codegen emits MAT/VisualVM-resolvable metadata. First-class Idris-vocabulary JFR events with throttle tiers stay within a 1–2% overhead budget. OpenTelemetry integrates into the Idris effect system; context survives virtual-thread boundaries via a Palisade-shipped `ScopedValue` bridge (not OTel-Java's `ThreadLocal` bridge).
**Depends on**: Phase 4 (OTel-Spring integration depends on Spring being in place)
**Requirements**: OPS-05, OPS-06, OPS-07, OPS-08, OPS-09, OPS-10, OPS-11, OPS-12, OPS-13, OPS-14, STDLIB-A-02
**Success Criteria** (what must be TRUE):
  1. Every emitted class carries a populated `SourceDebugExtension` (JSR-45 SMAP); IntelliJ-debugger step-through, async-profiler flame graph, JFR event stack frames, and Eclipse MAT heap snapshots all show Idris file:line for Palisade-emitted code; a `palisade trace` CLI decodes any generated stack trace back to Idris source positions (D21 full, Pitfall 7 safety net, OPS-05/06)
  2. Generated class/method/field names follow a documented deterministic mangling scheme that round-trips to Idris source identifiers; ADT codegen emits metadata so MAT/VisualVM resolve Idris constructors natively without plugins (verified by inspecting a heap dump of a fixture program with mixed ADTs) (D23, OPS-07/08)
  3. JFR stream emits Idris-vocabulary events: `LinearFutureCompleted`, `EffectPerformed`, `LinearityViolationDetected`, `OrderedContainerAccess` (and others). Event vocabulary is taxonomy-designed before any event ships; `@Throttle` rates assigned per-event; an end-to-end load test on the fixture stays within 1–2% overhead with full vocabulary on (D22, Pitfall 8, OPS-09/10)
  4. JFR event classes are pre-generated at compile time (no reflection) and a fixture compiled with GraalVM `nativeCompile` emits the events correctly under native-image; the Idris JFR DSL (Records → JFR event classes via ASM at compile time) lets every IO-bound function emit events with effect-system-derived correlation IDs (Pitfall 4, Pitfall 8, OPS-11, STDLIB-A-02)
  5. OpenTelemetry integrates into the Idris effect system; compiler enforces span open/close across `LinearFuture` lifecycle (a fixture dropping a span fails to compile); OTel context propagation across virtual threads uses a Palisade-shipped `ScopedValue`-backed bridge (not the stock `ThreadLocal` bridge) — a fixture spawning vthread-fanout traces shows continuous spans in Jaeger (D24, Pitfall 6, OPS-12/13)
  6. Spring Actuator + Micrometer natively export Idris-specific invariants (`StructuredTaskScope` backpressure gauge, `OrderedContainer` contention counter, `LinearFuture` success/failure rate); `/actuator/metrics` shows them on the fixture (OPS-14)
**Plans**: TBD

### Phase 6: Performance (Fifth Floor)
**Goal**: Trading-grade dispatch and verified zero-overhead claims. `@hot`/`@specialize` pragmas force monomorphization; `invokedynamic` Guard-With-Test handles warm paths; closed-world fat-JAR builds auto-monomorphize. Precision Stack Map Frame generator handles trampoline-induced control flow + dependent-type erasure shapes ASM cannot reliably infer. JMH benchmark suite runs as a CI-blocking gate against Kotlin Coroutines, Java 25 virtual threads, and Java sealed-class switching. Reproducibility is promoted to cross-host hash verification with SLSA L3+ attestation. PDDT-enabled validation lands.
**Depends on**: Phase 5 (JMH + JFR overhead budget validation needs the JFR vocabulary in place)
**Requirements**: PERF-04, PERF-05, PERF-06, PERF-07, PERF-08, PERF-09, PERF-10, PERF-11, STDLIB-B-04
**Success Criteria** (what must be TRUE):
  1. A polymorphic call site annotated `@hot` or `@specialize` emits a direct (non-virtual) JVM call to a monomorphic instance; warm-path call sites use `invokedynamic` with Guard-With-Test chain (top 3–4 frequent types), keeping HotSpot inline-cache effective (verified via `-XX:+PrintInlining` JMH harness) (D16, PERF-04/05)
  2. Closed-world fat-JAR builds run link-time analysis; interfaces with 1–2 implementations auto-monomorphize without user pragma (verified by decompiled output diff vs. open-world build of same source) (D16, PERF-06)
  3. `@hot` implies no-trampoline (compiler error if `@hot` applies to a function in a mutual-recursion group requiring trampolining); precision Stack Map Frame generator handles trampoline + dependent-type-erasure shapes — a fixture corpus of 50+ frame-edge-case classes passes `-Xverify:all` AND JaCoCo + ByteBuddy round-trips (D27 precision tier, Pitfall 1/9, PERF-08)
  4. Best-effort zero-allocation hot paths: profile + linter detect boxing on annotated paths; JFR `jdk.ObjectAllocationOutsideTLAB` monitors enabled in CI on hot-path fixtures (PERF-07)
  5. JMH-based continuous benchmark suite measures `LinearFuture` vs Kotlin Coroutines vs Java 25 virtual threads, ADT pattern match vs Java sealed-class switching; runs in CI as a blocking gate (regression > documented threshold fails build); JFR monitors JIT deoptimizations, megamorphic call-site detection, and `jdk.VirtualThreadPinned` events (D28, Pitfall 3, PERF-09/10)
  6. Reproducible-build CI gate is promoted to **cross-host hash-verification** (independent build hosts produce byte-identical JARs); SLSA L3+ attestation attached to every release-candidate build (D36, PERF-11)
  7. PDDT-enabled validation library: refined types encode invariants checked at boundary, propagated as proofs through business logic; a fixture proves a refined `Email`/`Url` type cannot be constructed with invalid input and the proof survives through three function-call hops (STDLIB-B-04)
**Plans**: TBD

### Phase 7: Developer Experience + Test Infrastructure (Sixth Floor)
**Goal**: The inner-loop story matches Clojure's reputation. Maven and Gradle plugins integrate Idris sources cleanly into hybrid-module Spring Boot projects; standalone `idris2`-owned build emits a deployable JAR. LSP extends to know JVM types and Spring annotations. JRebel-grade hot reload swaps method bodies and signatures without restart, falling back to classloader restart for LIMM-affecting changes. Idris REPL attaches to a running JVM. JUnit 5 `TestEngine` discovers Idris PBT/PDDT tests; Spring TestContext lifecycle integrates; JaCoCo+SMAP lifts coverage to Idris source level; mutation testing runs against emitted bytecode; the Defensive Membrane adversarial-test corpus exercises every documented hole.
**Depends on**: Phase 6 (JaCoCo lift + mutation testing must observe verifier-clean precision-frame bytecode)
**Requirements**: BUILD-01, BUILD-02, BUILD-03, BUILD-04, BUILD-05, BUILD-06, BUILD-07, BUILD-08, BUILD-09, DX-01, DX-02, DX-03, DX-04, DX-05, DX-06, TEST-01, TEST-02, TEST-03, TEST-04, TEST-05, TEST-06
**Success Criteria** (what must be TRUE):
  1. `palisade-maven-plugin` and `palisade-gradle-plugin` (Kotlin DSL primary) integrate with standard Maven/Gradle lifecycles; Idris sources coexist with Java/Kotlin in one module producing one JAR; both plugins emit `META-INF/spring.components` + Spring Boot 4 AOT metadata correctly; `idris2 build` produces a JPMS-clean modulepath-deployable JAR independently (D10, D30, BUILD-01/02/03/04)
  2. `palisade spring my-service` template generates a runnable Spring Boot service skeleton (Spring Initializr-style) with one Idris controller, working tests, JFR config, Spring DevTools wired; the template's CI workflow runs `nativeCompile`, OTel context-continuity tests, and Spring DI autowire tests from day one (D31, Pitfalls 4/5/6, BUILD-05/06)
  3. Two officially-supported deployment profiles (`dev/staging`: HotSpot + full observability + hot reload + REPL; `production-native`: GraalVM native-image + restricted observability) are both built and tested in CI on every commit; `production-native` supports an opt-in REPL-based dev-tools escape hatch; JPMS modules default to `open` for Spring reflection compatibility, and Palisade detects + refuses split-package configurations (D7, Pitfall 13, BUILD-07/08/09)
  4. Idris LSP (extended) shows imported Java method/field completions with full signatures on hover; suggests Spring annotations from classpath with parameter rendering on hover; backed by elaborator-reflection-to-classpath capability extension (D11, Reconsider #4, DX-01/02)
  5. Sub-second incremental compile for typical Idris module changes; JRebel-grade hot reload applies method bodies, signatures, and new classes without service restart via Palisade classloader-restart agent; the agent detects LIMM-affecting changes and forces full restart for those; Idris REPL attaches to a running JVM service to introspect types, query definitions, evaluate expressions in the live process (D32, Pitfall 15, DX-03/04/05/06)
  6. JUnit 5 `TestEngine` (engine ID `palisade-idris`) discovers Idris test functions; Idris PBT (Hedgehog/QC-style) and PDDT tests are first-class within the engine; tests participate in Spring TestContext lifecycle (DI, transactions, mocks) — `mvn test` and `gradle test` run mixed Idris + Java + Kotlin test suites in one harness (D25, TEST-01/02/03)
  7. JaCoCo agent + JSR-45 SMAP lifts coverage to Idris source level; per-Idris-function thresholds enforceable in SonarQube/Codecov gates; mutation testing pipeline (PIT or equivalent) runs against Palisade-emitted bytecode with documented baseline thresholds per category (D26, TEST-04/05)
  8. Membrane adversarial-test corpus runs in CI: every documented attack surface (reflection, serialization, lambda capture, Spring AOP/CGLIB proxy, Mockito mock, AspectJ weave, GraalVM native-image proxy) is either CLOSED by the Defensive Membrane or DOCUMENTED as out-of-Membrane with a justifying comment (D20, Pitfall 2, TEST-06)
**Plans**: TBD

### Phase 8: Reference Service + Documentation + Release
**Goal**: The proof artifact and the adoption surface ship together. A working Spring Boot 4 reference service in this repo exercises the full v1 surface — Idris `@RestController` returning sealed-class ADT serialized via Jackson, `LinearFuture` + `LinearScope` async fan-out, full JFR vocabulary visualized in JMC, both deployment profiles, generational ZGC pause profiles under load. Diataxis full set: 5 tutorials of increasing complexity, 6 skills, IAM framework worked example, ≥3 enterprise-ready software examples, reference manual covering all 40+ Decisions, operations runbook ("transparent box" guide), compatibility matrix, migration guide. Release & governance discipline: strict semver, vulnerability disclosure, signed releases with CycloneDX SBOM and SLSA L3 provenance, Dependabot, reproducibility verification per release, neutral OSS governance.
**Depends on**: Phase 7 (build plugins + test harness + LSP all in place; reference service is the integration test)
**Requirements**: REF-01, REF-02, REF-03, REF-04, REF-05, REF-06, DOC-01, DOC-02, DOC-03, DOC-04, DOC-05, DOC-06, DOC-07, DOC-08, REL-01, REL-02, REL-03, REL-04, REL-05, REL-06
**Success Criteria** (what must be TRUE):
  1. A working Spring Boot 4 service lives in `reference-service/` in this repo, built and tested in CI on every commit; at least one HTTP endpoint backed by an Idris `@RestController` returns a sealed-class ADT serialized via Jackson; at least one endpoint fans out async work using `LinearFuture` + `LinearScope` (compiler refuses build if a future is dropped or a task escapes); at least one path emits the full JFR vocabulary with JMC dashboards visualizing it (REF-01/02/03/04)
  2. The reference service deploys cleanly under both `dev/staging` (HotSpot + observability + hot reload + REPL) AND `production-native` (GraalVM native-image) profiles; both verified in CI on every commit; a documented load test demonstrates trading-grade pause profiles under generational ZGC (REF-05/06)
  3. Documentation set is complete and published: ≥5 tutorials of increasing complexity (hello-world → multi-module Spring Boot service); ≥6 skills/how-to guides (Java interface impl, `@RestController` with Linear Futures, JFR event design, mutation testing setup, hot-reload usage, native-image deployment); IAM framework worked example built from scratch using Linear Futures + JEP 510 KDF + OTel-as-Proof + sealed-class ADT projection; ≥3 enterprise-ready software examples building on the IAM framework (DOC-01/02/03/04)
  4. Reference manual covers all 40+ Key Decisions with implementation notes, rationale, and tradeoffs; operations runbook (the "transparent box" guide) covers reading Palisade stack traces, debugging LinearityViolations in production, interpreting Idris-vocabulary JFR events, profiling with async-profiler, querying via REPL-attach; compatibility matrix doc lists Palisade ↔ Idris release ↔ JDK ↔ Spring Boot ↔ feature-by-feature deployment-mode applicability; migration guide documents file-by-file Java → Idris in a Spring Boot module (D29, D34, D30, DOC-05/06/07/08)
  5. Strict semver applied with documented per-release compatibility matrix; vulnerability disclosure policy (`security@palisade.…` + GitHub security advisories) and CVE issuance workflow live; every release ships with CycloneDX SBOM + Sigstore signatures + SLSA L3 provenance; Dependabot (or equivalent) gates merges; reproducible-build cross-host verification is part of every release pipeline (D34/35/36, REL-01/02/03/04/05)
  6. Repository is hosted under user's personal or neutral GitHub org with documented contribution guide and CLA-optional governance independent of any single user; `palisade --version` matches the published GitHub release tag; `palisade build` of the reference service produces a byte-identical JAR on two independent build hosts in CI (D37, REL-06)
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 0. Project Setup | 0/TBD | Not started | - |
| 1. Codegen Foundation | 0/TBD | Not started | - |
| 2. FFI + Defensive Membrane | 0/TBD | Not started | - |
| 3. Linearity + Loom | 0/TBD | Not started | - |
| 4. Spring Integration | 0/TBD | Not started | - |
| 5. Observability | 0/TBD | Not started | - |
| 6. Performance | 0/TBD | Not started | - |
| 7. Developer Experience + Test | 0/TBD | Not started | - |
| 8. Reference Service + Docs + Release | 0/TBD | Not started | - |

---

*Roadmap created: 2026-04-21*
*Granularity: fine (9 phases, 5-10 plans each TBD during plan-phase)*
*Coverage: 117/117 v1 requirements mapped*
