# Project Research Summary

**Project:** Palisade — JVM backend for Idris 2
**Domain:** Compiler backend for a verified, dependently-typed, linear functional language, targeting Spring Boot on JVM 25+
**Researched:** 2026-04-21
**Confidence:** HIGH on stack / architecture / historical failure modes; MEDIUM on cross-cutting novel-pattern wiring (LIMM, Defensive Membrane, Telemetry-as-Proof — Palisade-original, inferred from decisions + precedents).

**Reader's guide.** The spine is the 40 Key Decisions from PROJECT.md. The Executive Summary is the only section you can read in isolation. The core payload lives in three places: (1) **Disagreements and Reconsiders** — where research pushes back on PROJECT.md; (2) **Cross-Cut by Key Decision** — every research file's contribution per Decision; (3) **Build-Order Floors** — Architecture's 6-floor technical dependency sequence (most actionable artifact for the roadmapper). Feature gaps and the "graveyard pattern" appear under Expected Features and Critical Pitfalls respectively.

## Executive Summary

Palisade sits at the intersection of three stacks that have all stabilized in the last 24 months (JVM 25 LTS finalized 2025-09, Spring Boot 4.0 shipped 2025-11, JEP 484 ClassFile API finalized in JDK 24 as ASM's eventual successor). The research validates **the vast majority of PROJECT.md's 40 Key Decisions**: the technology picks (ASM, Spring Boot 4, JUnit 5 TestEngine, JaCoCo, CycloneDX, ZGC, JFR, JPMS, JSR-45), the architectural shape (Idris-side codegen + Java bridge + user-linked runtime), and the cross-cutting engineering invariants (source maps threaded from day one, `-Xverify:all` as CI build-breaker, reproducible builds by contract) all match the convergent practice of Kotlin, Scala 3, Clojure, and idris-jvm.

The **disagreements are narrow but important**: (1) JEP 505 StructuredTaskScope is still *preview* through at least JDK 27, which argues for a Palisade-stable `LinearScope` abstraction that absorbs upstream API churn — Decision 18/19 already implies this, research makes it explicit; (2) ASM (Decision 39) is correct for v1, but a thin `BytecodeEmitter` SPI from day one makes the eventual ClassFile API migration a single-module change rather than a rewrite; (3) Spring Framework 7 standardized on **JSpecify** for null-safety, so Decision 5's annotation projection should emit `@org.jspecify.annotations.{NonNull,Nullable}` derived from `Maybe a`, not JetBrains annotations; (4) nine concrete feature gaps are missing from PROJECT.md altogether — `@Jvm*`-equivalent naming hints, explicit Java-interface implementation story, explicit sealed-class ADT projection, explicit static-method exposure, explicit HTTP/JSON/JDBC plan, explicit hot-reload change-set scope, explicit "deployment modes" doc, explicit Loom-Integration-Track grouping, explicit subclassing policy.

The **risk profile** has two layers. Near-surface: 15 concrete pitfalls (Stack Map Frame bugs, Defensive Membrane holes, virtual-thread pinning, GraalVM reachability gaps, Spring `META-INF/spring.components` index drift, OTel ThreadLocal context loss, JSR-45 tooling patchiness, JFR high-cardinality throttling, trampoline pollution, Ghost Annotation stripping, vendor snapshot drift, reproducibility breakage, JPMS-Spring conflicts, ScopedValue lifecycle, hot-reload correctness). Deeper: the **graveyard pattern** (idris-jvm, Eta, Frege) — they all got codegen mostly right and died on execution discipline around Spring integration, FFI ergonomics, and tooling investment. Codegen quality is necessary but not sufficient — the **adoption-surface decisions** (LSP, JaCoCo lift, Diataxis docs, `palisade spring` template, hybrid-module, REPL-attach) are what convert "verified Idris on JVM" into "deployable verified Idris on JVM." PROJECT.md addresses this correctly; research calls out that slippage on *any* of those features recreates the Frege outcome.

## Key Findings

### Recommended Stack

Every convergent pick aligns with PROJECT.md: ASM 9.8 with migration ramp to JEP 484; `COMPUTE_FRAMES` default with precision generator for trampoline-heavy code; `invokedynamic` + `LambdaMetafactory` for closures; JSR-45 SMAP + LineNumberTable + LocalVariableTable on every class; subclass `ClassWriter` to override `getCommonSuperClass` (mandatory day-one work). The **divergent pick** is Decision 2's mutual-recursion `invokedynamic` trampoline — only idris-jvm did this; Kotlin/Scala/Clojure leave general TCO to the user. Justified by Idris's mutual-recursion idiom but pitfall-dense (Pitfall 9).

**Core technologies:**
- **OpenJDK 25 (LTS)** — Loom + JEP 491 unpinning, JEP 506 ScopedValues *final*, generational ZGC as only ZGC, 8-year support.
- **ASM 9.8** — supports JDK 25 class files (major 69); convergent JVM-language standard.
- **Spring Boot 4.0.x** (current 4.0.5) — Spring Framework 7, native JSpecify, first-party OTel starter, improved AOT/native. Java 17 baseline, first-class Java 25.
- **GraalVM Native Image for JDK 25** — single-file `META-INF/native-image/<group>/<artifact>/reachability-metadata.json`.
- **Maven 3.9.x + Gradle 8.x (Kotlin DSL)** — both first-class per Decision 10.
- **JUnit 5.12+ custom `TestEngine`** — engine ID `palisade-idris`. `junit-platform-jfr` removed; folded into `junit-platform-launcher`.
- **JaCoCo 0.8.13+, JMH 1.37, PIT 1.19.1+, CycloneDX 2.x, OpenTelemetry Spring starter, Micrometer 1.14+, JFR (`jdk.jfr` in `java.base`)**.

Full detail: `.planning/research/STACK.md`.

### Expected Features

~28 of 40 PROJECT.md Decisions map onto table-stakes or differentiator features validated against Kotlin, Clojure, Scala 3, and idris-jvm. The coverage is deliberate, not accidental.

**Must have (table stakes already in PROJECT.md):** One-line Java FFI (Decision 4), Java generics projection (Decision 6), Spring annotation emission (Decision 5), `Either JException T` boundary (Decision 8), Maven + Gradle + standalone build (Decision 10), source maps (Decision 21), `-Xverify:all` gate (Decision 27), TCO (Decision 2), `BigInteger` + primitive specialization (Decision 3), JPMS per-package (Decision 40), Diataxis docs (Decision 29), semver + compat matrix (Decision 34), enterprise security (Decision 35), LSP (Decision 11), JUnit 5 + Spring TestContext (Decision 25), JaCoCo lift (Decision 26), reproducible builds (Decision 36), hybrid-module (Decision 30), `palisade spring` template (Decision 31), sub-second incremental + hot reload + REPL-attach (Decision 32).

**Should have (differentiators):** Linear Futures over CompletableFuture (Active + Decision 18, 20) — the central thesis; Defensive Membrane (Decision 20); LIMM (Decision 17); Telemetry-as-Proof (Decision 24); JFR Idris-vocabulary events (Decision 22); Semantic Naming + ADT Reconstruction (Decision 23); Hybrid Metadata Erasure + Ghost Annotations (Decision 15); link-time closed-world JIT specialization (Decision 16); reproducible builds with cross-host hash verification (Decision 36); empirical benchmark suite vs Kotlin Coroutines / Java virtual threads, CI-blocking (Decision 28). **None of Kotlin, Clojure, or Scala 3 ship any of the above.**

**Nine concrete gaps (features Palisade needs that PROJECT.md does not pin down):**

1. **`@Jvm*`-equivalent naming hints** (Kotlin's `@JvmStatic`, `@JvmName`, `@JvmField`, `@JvmOverloads`) — needed for clean Java-side APIs.
2. **Implementing Java interfaces from Idris** — `Runnable`, `Comparator`, `Filter`, `Function<T,R>`, `Callable`. Implied by Decision 4/5, not stated.
3. **Extending Java classes from Idris** — `RuntimeException` subclassing, `WebMvcConfigurer`. Needs explicit policy.
4. **Static method exposure** — Java callers need `IdrisClass.method(...)`.
5. **Curated `Maybe` / ADT / collection projection rules** — recommend: Idris ADTs → sealed-class hierarchy + record per constructor; `Maybe` → `Optional` at boundary, no implicit conversion.
6. **HTTP client / JSON SerDe / JDBC bindings** — defer to v1.x with Diataxis FFI walkthrough.
7. **Explicit "supported vs restart" hot-reload change set.**
8. **Explicit "deployment modes" doc** — Decision 7 (native-image) vs Decisions 22/24/32 (require dynamic JVM).
9. **Loom Integration Track grouping** — Decisions 18/19/20/24 collectively define Linear Futures + structured concurrency; treating as parallel risks loss of coherence.

**Defer (v2+):** Valhalla zero-allocation contract (Decision 14, gated on Valhalla GA, likely JDK 27/28); third-party security audit (Decision 35, funding); cross-platform / Idris Multiplatform; IntelliJ-native plugin.

Full detail: `.planning/research/FEATURES.md`.

### Architecture Approach

All four surveyed JVM compilers (idris-jvm, Kotlin K2, Scala 3/Dotty, Clojure) decompose into the same three-tier shape: **compiler-side pipeline (IR + lowerings + emitter) → bytecode library bridge (ASM today, ClassFile API tomorrow) → Java-side runtime support library** outside codegen. Palisade adopts this verbatim: `src/Compiler/JVM/` (paralleling `RefC/Scheme/ES/Interpreter`) + `support/jvm/{palisade-asm-bridge,palisade-runtime}/` + `build-tooling/{palisade-maven-plugin,palisade-gradle-plugin}/`.

**Major components:** (1) Upstream IR adapter consuming vendored `NamedCExp`/`NamedDef` (Decisions 1+38). (2) Backend IR (`JVMExpr`) + lowerings — extending idris-jvm's proven `InferredType`/`Asm`/`Codegen`/`Optimizer`/`Foreign` decomposition with sibling packages (`Memory/`, `Loom/`, `JFR/`, `FFI/Membrane/`, `Spring/`, `Debug/SDE/`, `Telemetry/OpenTelemetry/`). (3) ASM bridge mirroring idris-jvm's `idris-jvm-assembler`; subclass `ClassWriter` from day one. (4) Runtime support library (`LinearFuture`, `LinearChannel`, `OrderedContainer` VarHandle-backed, `GuardProxy`, JFR event classes, StructuredTaskScope helpers, ScopedValue bridges). (5) Build tooling.

**Anti-patterns flagged:** emit bytecode directly from `NamedCExp`; default `ClassWriter` without `getCommonSuperClass` override; drop `FC` during lowering; re-implement Loom in codegen; defer `-Xverify:all` gate; treat runtime library as afterthought; skip ASM bridge layer.

Full detail: `.planning/research/ARCHITECTURE.md`.

### Critical Pitfalls — Top 5 (of 15)

1. **Stack Map Frame bugs producing `VerifyError` in downstream agents (Pitfall 1).** ASM's `COMPUTE_FRAMES` has documented edges for trampolined control flow, exception-handler overlap, dependent-type-erasure shapes. Classes pass `-Xverify:all` in isolation and fail under JaCoCo / Spring AOT / OpenTelemetry agents (stricter type-inference verifier). Prevention: dual-verifier CI gate (ASM + ClassFile API + JaCoCo + ByteBuddy round-trips), precision frame generator, regression corpus. Decision 27 *must* include downstream-agent compat.

2. **Defensive Membrane holes (Pitfall 2).** QTT linearity is *static*; Membrane reifies as *dynamic* runtime check. JVM has no "use exactly once" — reflection, serialization, `Cloneable`, lambda-capture-and-replay, `invokedynamic` adaptation each are holes. Prevention: defense-in-depth not primary guarantee; disallow `Serializable`/`Cloneable`; ship `@Linear` JVM annotation + SonarQube + Error Prone; document FFI contract ("reflection erases linearity guarantee").

3. **Virtual-thread pinning in FFI and class-init (Pitfall 3).** JEP 491 fixed `synchronized` pinning; native-call (JNI/Panama) and class-init pinning remain. 257 concurrent pinning FFI calls deadlock the VM. Prevention: FFI importer annotates blocking-ness; bounded blocking-pool dispatcher; eager class-init via `module-info`; `jdk.VirtualThreadPinned` JFR event as benchmark CI gate; explicit `jdk.virtualThreadScheduler.maxPoolSize` in template.

4. **GraalVM reachability metadata gaps in production (Pitfall 4).** Code paths only hit under load throw `ClassNotFoundException`; `EventFactory` JFR reflection is GraalVM-hostile; `AtomicReferenceFieldUpdater` trips static analysis. Retrofitting GraalVM compat is the documented graveyard (Kotlin+Spring Boot 4+Java 24 still fighting it in 2026). Prevention: emit `reachability-metadata.json` from codegen, not via agent tracing; `palisade spring` template runs `./gradlew nativeCompile` in CI from day one; pre-generate JFR event classes at compile time.

5. **Spring bean discovery misses Idris beans — the Kotlin/KAPT repeat (Pitfall 5).** Spring Context Indexer reads `META-INF/spring.components`; if partial (Idris beans missing because Palisade's plugin didn't update it), Spring assumes it's authoritative and skips classpath scanning. Idris beans silently vanish. Prevention: Palisade build plugins emit `META-INF/spring.components`, `META-INF/spring.factories`, `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Cross-check index against classpath scan in CI.

**The graveyard pattern (load-bearing for the roadmapper).** idris-jvm (single-maintainer bus factor; Spring/enterprise never the design center), Eta (purity leakage at FFI; Hackage non-problem inverted for Idris but warns against bringing *too little* of Idris stdlib forward), Frege ("wrapping Java means everything's in IO" — monadification of the world). **Decision 8's `Either JException T` boundary *is* this trap unless Decision 4's importer + Decision 6's generics projection make the wrapping invisible at call sites.** Codegen quality is necessary but not sufficient — Clojure won via tooling investment, Scala 3 needed a working group in 2018 to close the IDE gap, Kotlin won on JetBrains+Google not language design. PROJECT.md addresses this correctly with Decisions 11/25/26/28/29/30/31/32 — the risk is *execution discipline*: any one slipping recreates the Frege outcome.

Full detail: `.planning/research/PITFALLS.md`.

## Disagreements and Reconsiders (Where Research Pushes Back on PROJECT.md)

| # | Decision | Nuance | Recommendation | Confidence |
|---|----------|--------|----------------|------------|
| 1 | **Decision 18** — Loom + `StructuredTaskScope` | *Preview* through JEP 505/525/533 (JDK 25/26/27); likely finalizes JDK 27 or 28 | Wrap behind a Palisade-stable `LinearScope` Idris effect *now*. Don't expose `StructuredTaskScope` types in user-facing Idris signatures. (Decision 19 already implies this — make explicit.) | HIGH |
| 2 | **Decision 39** — ASM | JEP 484 ClassFile API finalized JDK 24 with explicit intent to replace ASM. 5-10 year migration but inevitable | Define `Compiler.JVM.Emit.Backend` SPI from day one. `Compiler.JVM.Emit.ASM` is v1; `Compiler.JVM.Emit.ClassFile` is one added module when ready. Bonus: dual-backend testing is strong correctness signal (ties to Pitfall 1's dual-verifier gate). | HIGH trajectory / MEDIUM timing |
| 3 | **Decision 16** — JIT shape | JDK 25 JIT (JEP 514 AOT, Leyden) changes AOT/JIT interaction; Java 21-era profiling assumptions may not transfer | Validate megamorphic call-site profile shape on JDK 25 via JFR + `-XX:+PrintInlining` early in the relevant phase | MEDIUM — Leyden moving |
| 4 | **Decision 11** — LSP-only | Spring annotation completion/hover (Decision 5) requires LSP layer to call into elaborator reflection to discover annotation classes from JVM classpath — a capability extension, not small polish | Budget the capability-extension work explicitly | HIGH on choice; flagging effort |
| 5 | **Decision 5** — Annotations on emitted bytecode | Spring Framework 7 standardized on **JSpecify**, moving away from JetBrains' `@NotNull`/`@Nullable` | Emit `@org.jspecify.annotations.NonNull` / `@Nullable` derived from `Maybe a` etc. — makes Idris totality consumable by Spring 7's null-safety verifier | HIGH — verified against Spring Boot 4 |
| 6 | **Decision 27** — Zero-Tolerance Verification | "Passes `-Xverify:all`" insufficient — classes pass in isolation, fail under downstream agents (Pitfall 1) | Contract must include downstream-agent compat: dual-verifier CI gate (ASM + ClassFile API + JaCoCo + ByteBuddy round-trips). Sub-clause not addendum. | HIGH |

**Not reconsidered, affirmed:** Decisions 1, 2, 3, 7, 8, 10, 13, 14 (with Valhalla caveat), 29, 30, 33, 34, 36, 37, 38, 40.

## Cross-Cut by Key Decision

For each Decision, what every research file contributed. Use as lookup index when working on a specific Decision.

| # | Decision | STACK | FEATURES | ARCHITECTURE | PITFALLS |
|---|----------|-------|----------|--------------|----------|
| 1 | Vanilla Idris 2 + own JVM codegen on `NamedCExp` | Convergent prior-art shape | Anti-feature "build on idris-jvm" correctly rejected | Keystone; `src/Compiler/JVM/` paralleling existing | Pitfall 11 (vendor-snapshot drift) risk surface |
| 2 | Tail calls: self-rec → loop; mutual → trampoline | Matches idris-jvm; `invokedynamic` trampoline is idris-jvm-specific | Table stakes | `JVM/Opt/TailCall.idr`; First Floor | **Pitfall 9** — trampolines pollute stack traces, defeat JIT inlining, interact badly with try/catch |
| 3 | Primitives monomorphic; `BigInteger` for unbounded | Matches Idris `Integer` semantics | Table stakes | `JVM/IR/InferredType.idr`; First Floor | Performance trap: `BigInteger` boxing at polymorphic boundaries |
| 4 | Clojure-grade FFI via elaborator-reflection importer | Confirms adoption floor | P0 adoption blocker; absence = Frege death | `JVM/FFI/{Foreign,Importer}.idr`; Second Floor | Counter to Eta/Frege monadification trap |
| 5 | Spring annotations on emitted bytecode | **Reconsider:** use JSpecify | P0 adoption blocker | `JVM/Spring/{Annotations,DI}.idr`; Fourth Floor | **Pitfall 5** (`META-INF/spring.components`), **Pitfall 4** (native-image AOT), **Pitfall 10** (AOP strips Ghost Annotations) |
| 6 | Java generics: faithful projection | Required for Spring API signatures | Deceptively hard; F-bounded, wildcards, raw types in Spring | `JVM/FFI/Generics.idr`; Second Floor | Vendor "ugly real-world Java signatures" fixture set |
| 7 | GraalVM native-image first-class | Single-file `reachability-metadata.json` | Conflict with Decision 32 hot reload (document) | Codegen emits metadata; agent-tracing fallback only | **Pitfall 4** — retrofitting compat is graveyard; native-image CI from day one |
| 8 | Exception ↔ totality bridge: `Either JException T` | Standard Java FFI patterns | Differentiator vs Kotlin/Scala/Clojure | `JVM/FFI/ExceptionBridge.idr`; lands *with* FFI elaborator | **Graveyard warning** — *this is* Frege monadification trap unless Decision 4/6 make wrapping invisible |
| 9 | State management survey → JVM + Clojure mappings | `AtomicReference`/atom, `ScopedValue`, `ConcurrentHashMap`/PersistentMap | Table stakes; counters Eta "too little stdlib" trap | Across `Memory/`, `Loom/`, runtime | Pitfall 14 (ScopedValue lifecycle) |
| 10 | Maven + Gradle + standalone build | Mature plugin ecosystems | P0 adoption blocker | `build-tooling/`; Sixth Floor | Pitfall 5 (plugin must emit `META-INF/spring.components`) |
| 11 | LSP-based IDE | IntelliJ LSP API now public (2025) | Table stakes | Outside `src/Compiler/JVM/`; Sixth Floor | **Reconsider #4**: Spring completion needs elaborator reflection → JVM classpath — nontrivial |
| 12 | Idris pinning + periodic rebase | — | — | — | Pitfall 11 (automate monthly sync PR) |
| 13 | GC default: ZGC override JDK 25 G1 | Generational ZGC only ZGC in 25 | — | Runtime launcher config | Performance trap: ZGC 15-30% higher heap memory |
| 14 | Zero-alloc: best-effort today; Valhalla later | JEP 401 preview JDK 26, GA likely 27/28 | v2+ | — | — |
| 15 | Hybrid Metadata Erasure + Ghost Annotations | — | Differentiator | `JVM/Opt/Erase.idr` + emitter; Second Floor | **Pitfall 10** — use standard runtime annotations not custom attrs (ASM strips); `BeanPostProcessor` for Spring proxies |
| 16 | Tiered JIT: `@hot`/`@specialize` + indy + closed-world | `invokedynamic` + `LambdaMetafactory` standard | Differentiator | `JVM/Opt/{Specialize,ClosedWorld}.idr`, `JVM/Asm/InvokeDynamic.idr`; Fifth Floor | **Reconsider #3**; Pitfall 9 (`@hot` should imply no-trampoline) |
| 17 | LIMM (Linear-Inferred Memory Model) | `VarHandle` acquire/release/opaque standard JVM | Differentiator | `JVM/Memory/{LIMM,OrderedContainer}.idr`; Third Floor. **Requires linearity tracking.** | Pitfall 15 (hot reload must detect LIMM-affecting changes); Pitfall 14 |
| 18 | Structured Linear Runtime (Loom + LinearFuture) | **Reconsider #1**: preview through JDK 27+ | Central thesis | `JVM/Loom/{StructuredScope,LinearFuture}.idr`; Third Floor | **Pitfall 3** (pinning), **Pitfall 6** (OTel context), Pitfall 14 |
| 19 | Curated concurrency: Linear Channels, ScopedValues, StructuredTaskScope | ScopedValue *final* JDK 25 (JEP 506) | Differentiator | `palisade-runtime/concurrent/`; Third Floor | **Pitfall 14** — ScopedValue genuinely new; lifecycle subtle; compiler-enforced no-escape |
| 20 | Defensive Membrane (Guard Proxies at Java boundary) | — | Differentiator | `JVM/FFI/Membrane.idr` + runtime; Second Floor; **co-lands with FFI** | **Pitfall 2** — dynamic check for static property has holes; defense-in-depth |
| 21 | Telemetry-Native Mapping (SDE + LineNumberTable + LocalVariableTable) | JSR-45 SMAP what Kotlin/Scala use | Table stakes; enables 23+26 | `JVM/Debug/{LineTable,LocalVars,SDE}.idr`; First Floor lines / Fourth Floor SDE | **Pitfall 7** — JSR-45 tooling patchy; IntelliJ Ultimate-only; ship stack-trace decoder CLI |
| 22 | JFR Idris-vocabulary events | `jdk.jfr` native; JEP 509 CPU-time experimental | Differentiator | `JVM/JFR/{Events,Emit}.idr` + runtime; Fourth Floor | **Pitfall 8** — `EffectPerformed`/`OrderedContainerAccess` high-cardinality; `@Throttle` tiers; pre-generate event classes (Pitfall 4) |
| 23 | Semantic Naming + ADT Reconstruction | — | Differentiator | `JVM/Name/{Mangle,Demangle}.idr`; Fourth Floor | Redundancy safety-net for Pitfall 7 |
| 24 | Telemetry: Observability-as-Proof (OTel into effect system) | Spring Boot 4 first-party OTel starter (not agent for native) | Differentiator | `JVM/Telemetry/OpenTelemetry.idr`; Fourth Floor | **Pitfall 6** — OTel ThreadLocal doesn't inherit across virtual threads; ScopedValue bridge required |
| 25 | JUnit 5 TestEngine + Spring TestContext | Engine ID `palisade-idris`; `junit-platform-jfr` removed | Table stakes | Separate module | — |
| 26 | Coverage: JaCoCo + JSR-45 lift | Only mature JVM agent reading SMAP | Table stakes | — | Per-Idris-function SonarQube thresholds |
| 27 | Zero-Tolerance Verification Contract | `javap -v -p -c` + `java -Xverify:all` | Table stakes | `JVM/Verify/XVerify.idr` + `JVM/Asm/Frame.idr`; First Floor gate / Fifth Floor precision | **Reconsider #6**; **Pitfall 1** is core risk |
| 28 | Empirical Verification Suite (JMH, JFR-monitored, blocking CI) | JMH 1.37; `me.champeau.jmh` Gradle plugin | Differentiator | Separate benchmarks module | Include pinning (Pitfall 3) + JFR overhead (Pitfall 8) |
| 29 | Full Diataxis docs | — | Table stakes | — | Tutorials *first*, ramp complexity |
| 30 | Hybrid-module first-class | Two-phase build (Palisade before/after `kotlinc`+`javac`) | Differentiator (idris-jvm never offered) | Runtime as Maven Central artifact | Pitfall 5 (Spring DI all three), Pitfall 13 (JPMS split-package) |
| 31 | `palisade spring my-service` template | Spring Initializr-style | Table stakes | — | Native-image CI (Pitfall 4) + trace continuity (Pitfall 6) + working autowire (Pitfall 5) |
| 32 | Dev loop: incremental + hot reload + REPL-attach | Standard Spring DevTools | Table stakes (Clojure-grade DX) | Builds on `Codegen.incCompileFile`; Sixth Floor | **Pitfall 15** — classloader-restart default; detect LIMM-affecting changes |
| 33 | License: EPL 2.0 | — | — | — | — |
| 34 | Versioning: semver + compat matrix | — | Table stakes | — | JFR vocabulary becomes public API; Ghost Annotation schema too (Pitfall 10) |
| 35 | Security: full enterprise process | CycloneDX, SLSA L3, Sigstore | Table stakes | — | Pitfall 12 (reproducible builds → SLSA) |
| 36 | Reproducible builds; hash-verified | `project.build.outputTimestamp`; Gradle lockfiles | Table stakes | — | **Pitfall 12** — sorted collections; `SOURCE_DATE_EPOCH`; defer reproducible native-image |
| 37 | Governance: independent OSS | — | — | — | — |
| 38 | NamedCExp vendor snapshot | — | — | — | **Pitfall 11** — automate monthly sync CI; `idris2 --check` as oracle; "N versions behind" SLA |
| 39 | Bytecode library: ASM | ASM 9.8; JDK 25 class files | — | `support/jvm/palisade-asm-bridge/` | **Reconsider #2**: `BytecodeEmitter` SPI from day one |
| 40 | JPMS auto-emit `module-info.class` per `.ipkg` | `ClassWriter.visitModule()` + `ModuleVisitor` | Table stakes | `JVM/Module/ModuleInfo.idr`; First Floor | **Pitfall 13** — default `open` modules for Spring; detect split-package; Spring profile for `opens` |

## Implications for Roadmap

### Build-Order Floors (from Architecture — most actionable artifact)

Architecture research derived a six-floor technical dependency sequence. **This should be the skeleton of the roadmap.** Priority and Floor are independent — a P0 feature like Linear Futures (Decision 18) sits at Third Floor because of *technical dependencies*, not lower priority.

#### Ground Floor — Must exist before anything compiles
**Exit:** `idris2 --cg jvm hello.idr && java -jar hello.jar` prints "hello world".
- Codegen integration shim (`JVM/Codegen.idr` implementing `Compiler.Common.Codegen`)
- ASM bridge skeleton with `IdrisClassWriter`-style `getCommonSuperClass` override from day one
- Name mangler (basic; semantic refinement later for Decision 23)
- Backend IR (`JVM/IR/JVMExpr.idr`) — thin but present (avoid Anti-Pattern 1)
- Trivial `NamedCExp → JVMExpr` lowering for `NmRef`, `NmApp`, `NmCon`, `NmPrimVal`, `NmExtPrim`
- Trivial emitter — `Main` with `static main(String[])`

**Also at project-setup time** (even if Ground Floor incomplete):
- Vendored NamedCExp snapshot + **automated monthly sync CI** (Pitfall 11; Decision 38/12)
- **`diffoscope` reproducible-build CI gate** from first commit (Pitfall 12; Decision 36)
- **`FC` threading discipline** — make `FC` non-droppable on `JVMExpr` (Anti-Pattern 3)

#### First Floor — Basic correctness
**Exit:** Non-trivial Idris (recursive, pattern-matching, `Maybe`/`Either`) compiles, runs, passes `-Xverify:all`, stack traces show Idris file:line.
- Full `InferredType` lattice (idris-jvm's + linearity bits)
- TCO (Decision 2) — **prefer self-loop aggressively; trampoline only for genuine mutual** (Pitfall 9)
- `BigInteger`/primitive specialization (Decision 3)
- Case → tableswitch/lookupswitch
- Frame computation via `COMPUTE_FRAMES` (precision generator at Fifth Floor)
- **`-Xverify:all` CI gate** the moment non-trivial bytecode emits — **includes JaCoCo + ByteBuddy round-trips + dual-verifier (ASM + JEP 484 ClassFile API)** (Decision 27 + Reconsider #6 + Pitfall 1)
- Debug info: `LineNumberTable` + `LocalVariableTable` (Decision 21 "easy half")
- Module-info auto-emission (Decision 40)
- JAR packaging (minimum viable)
- **GraalVM `reachability-metadata.json` emission from codegen** starts here (Decision 7 + Pitfall 4) — retrofit is graveyard

#### Second Floor — FFI; the membrane goes up
**Exit:** Idris calls arbitrary Java; exceptions wrapped as `Either`; `LinearFuture` across FFI refuses double-consumption.
- Basic FFI elaborator (`JVM/FFI/Foreign.idr`)
- **Exception bridge lands *with* FFI** (Decision 8)
- Importer (elaborator-reflection) (Decision 4) — big lift
- Generics projection (Decision 6) — vendor "ugly real-world Java signatures" fixture corpus
- **Defensive Membrane co-lands with basic FFI, before Importer polish** (Decision 20 + Pitfall 2)
- Hybrid Metadata Erasure (Decision 15) — **standard runtime annotations, not custom attributes** (Pitfall 10)

#### Third Floor — The thesis: linearity + Loom
**Exit:** Reference Spring endpoint fans work over `LinearFuture` + `StructuredTaskScope`; compiler refuses builds dropping a future or leaking a task.
- `palisade-runtime/LinearFuture.java` + codegen call-site emission
- Linearity tracking integrated through lowering
- LIMM + `OrderedContainer` (Decision 17) — **requires linearity in place first**
- Structured Linear Runtime wrapped behind Palisade-stable `LinearScope` (Reconsider #1)
- Concurrency primitives subset — Linear Channels, ScopedValue helpers, `StructuredTaskScope` helpers (Decision 19)
- **Compiler-enforced no-escape on ScopedValue closures** (Pitfall 14); ban direct `Thread.startVirtualThread`
- **FFI call-site annotation taxonomy** (`@Blocking`, `@NonBlocking`, `@MayPin`) — enables bounded blocking-pool dispatch (Pitfall 3)

#### Fourth Floor — Operability: the "transparent box"
- Source maps completed — SDE / JSR-45 SMAP (Decision 21)
- Stack-trace decoder CLI (Pitfall 7 safety net)
- Semantic naming + ADT reconstruction (Decision 23)
- **JFR vocabulary with event taxonomy + `@Throttle` tiers** (Decision 22 + Pitfall 8); pre-generate event classes
- Spring annotations (Decision 5) — **JSpecify, not JetBrains** (Reconsider #5); emit `META-INF/spring.components` (Pitfall 5); `BeanPostProcessor` to copy Ghost Annotations onto Spring proxies (Pitfall 10)
- OpenTelemetry integration (Decision 24) — **ScopedValue-backed context bridge, not ThreadLocal** (Pitfall 6)

#### Fifth Floor — Performance + production polish
- Tiered JIT shape (Decision 16) — `@hot`/`@specialize` first, closed-world later; `@hot` implies no-trampoline
- Precision StackMapFrame generator (Decision 27 full) — replaces ASM auto-compute for trampoline-heavy code
- JMH benchmark suite (Decision 28) as CI gate — pinning detection (Pitfall 3) + JFR overhead budget (Pitfall 8)
- Reproducible build verification promoted to **cross-host hash verification** CI gate (Decision 36)

#### Sixth Floor — Developer experience
- Maven + Gradle plugins (Decision 10) — **plugins emit `META-INF/spring.components`** (Pitfall 5)
- `idris2`-owned build emitting JAR
- LSP JVM/Spring extension (Decision 11) — budget elaborator-reflection-to-classpath capability (Reconsider #4)
- Hot reload + REPL-attach (Decision 32) — **classloader-restart default; detect LIMM-affecting changes** (Pitfall 15)
- `palisade spring` template (Decision 31) — native-image + trace-continuity + autowire CI from day one

### Suggested Phase Structure

Eight phases track the Floors directly, plus Phase 0 for project setup and Phase 7 for documentation/release discipline.

**Phase 0: Project Setup.** Vendored NamedCExp + automated monthly sync CI; `diffoscope` reproducible-build CI gate; `FC`-threading discipline; empty skeletons; dual-verifier CI infrastructure; GraalVM native-image smoke-test CI from empty project. Avoids Pitfalls 11, 12, 4.

**Phase 1: Codegen Foundation (Ground + First Floor).** `hello world`; recursive pattern-matching Idris compiling to verifier-clean bytecode; stack traces with Idris file:line. Addresses Decisions 1, 2, 3, 21 (half), 27 (gate), 38, 39, 40. Avoids Pitfalls 1, 7 (lines), 9, 11, 12. **Research flag: NEEDED** — Stack Map Frame algorithm details, trampoline ASM patterns, `getCommonSuperClass` override specifics.

**Phase 2: FFI + Defensive Membrane (Second Floor).** Clojure-grade FFI; `Either JException T` at every Java boundary; `PalisadeLinearityViolation` on double-consumption. Addresses Decisions 4, 6, 8, 15 (partial), 20. Avoids Pitfalls 2, 10; Frege/Eta graveyard. **Research flag: NEEDED** — elaborator-reflection capabilities; Java-generics edge cases; Membrane adversarial test corpus.

**Phase 3: The Thesis — Linearity + Loom (Third Floor).** `LinearFuture` primitive; LIMM barrier-elision for q=1; `StructuredTaskScope` via Palisade-stable `LinearScope`; compile-time proof tasks joined or cancelled. Addresses Active Requirements + Decisions 17, 18, 19. Avoids Pitfalls 3, 14; partially 6. **Research flag: NEEDED** — `StructuredTaskScope` preview-churn absorption; `VarHandle` idioms; ScopedValue lifecycle.

**Phase 4: Operability — Transparent Box (Fourth Floor).** Stack traces / heap dumps / JFR / profilers / distributed traces resolving to Idris source; Spring beans autowiring Idris classes; connected traces across `LinearFuture` boundaries. Addresses Decisions 5, 21 (completion), 22, 23, 24. Avoids Pitfalls 5, 6, 7, 8, 10; partially 4. **Research flag: NEEDED** — JSR-45 tooling gaps; JFR taxonomy + throttle tiers; Spring Context Indexer; ScopedValue-OTel bridge.

**Phase 5: Performance (Fifth Floor).** `@hot`/`@specialize`; closed-world monomorphization on fat-JAR; precision StackMapFrame; benchmark suite as CI gate; reproducibility cross-host. Addresses Decisions 16, 27 (precision), 28, 36 (promotion). **Research flag: NEEDED** — JDK 25 JIT-specific inlining (Reconsider #3); Leyden interaction.

**Phase 6: Developer Experience (Sixth Floor).** Maven + Gradle plugins; `idris2`-owned build; LSP JVM/Spring extension; hot reload (classloader-restart default); REPL-attach; `palisade spring` template. Addresses Decisions 10, 11, 30, 31, 32. Avoids Pitfalls 4, 5, 13, 15. **Research flag: NEEDED** — LSP capability extension depth; JVMTI redefinition constraints; Spring DevTools on JDK 25.

**Phase 7: Documentation + Release Discipline (cross-cuts; packages at end).** Diataxis docs (5 tutorials + 6 skills + IAM example + ≥3 enterprise-ready); compatibility matrix; signed + attested releases. Addresses Decisions 29, 33, 34, 35, 37. Standard patterns; minimal research.

### Phase Ordering Rationale

- **Hard technical constraints** (from Architecture): linearity precedes LIMM precedes Loom proofs; FFI elaborator precedes Importer precedes Generics precedes Spring annotations; Defensive Membrane co-lands with FFI; `-Xverify:all` co-lands with first non-trivial emitter; `FC` threads from first emitter; SMAP (Fourth Floor) depends on line-number strategy from Phase 1.
- **Risk-driven sequencing**: GraalVM reachability-metadata starts Phase 1, not Phase 6 — retrofit is the graveyard. Reproducibility + vendor-sync CI are Phase 0.
- **Adoption gate**: Spring (Phase 4) gates `palisade spring` template (Phase 6); both gate the reference Spring Boot service.

### Research Flags

**Needs research before planning:** Phase 1 (Stack Map Frames, ASM `ClassWriter` override, trampoline+exception-handler interactions), Phase 2 (elaborator reflection on JVM classpath; Membrane adversarial corpus; generics edge cases), Phase 3 (`StructuredTaskScope` preview-churn; LIMM+ScopedValue under load), Phase 4 (JSR-45 tooling workarounds; JFR taxonomy; Spring Context Indexer; ScopedValue-OTel bridge), Phase 6 (LSP elaborator-reflection capability; JVMTI + Spring DevTools on JDK 25).

**Standard patterns (skip research):** Phase 0 (supply-chain hygiene), Phase 5 (JMH is mature), Phase 7 (Diataxis, CycloneDX, SLSA, Sigstore standard).

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | JDK 25 / Spring Boot 4 / ASM / JSR-45 / JUnit 5 / JaCoCo / JMH / CycloneDX / GraalVM verified against primary sources (JEPs, release announcements, vendor docs). Prior-art compiler internals (Kotlin FIR, Scala 3 GenBCode, Clojure ASM) MEDIUM — synthesized from project docs, not source. |
| Features | MEDIUM-HIGH | Kotlin/Clojure/Scala/idris-jvm features verified against official docs. Formal-methods researcher expectations partly inferred from CompCert/Lean/Coq/Dafny. The nine gaps are concrete and actionable. |
| Architecture | HIGH on existing-backend parallels + idris-jvm structure (Idris 2 source + idris-jvm public tree directly inspected); MEDIUM on novel-pattern wiring — LIMM, Defensive Membrane, Telemetry-as-Proof, JFR vocabulary are Palisade-original. |
| Pitfalls | HIGH | Each pitfall has a documented prior incident (JaCoCo #1009, ASM #317986, Spock #2080, Spring Boot #28046, opentelemetry-java #11950, graalvm-reachability-metadata #655, JDK-8257602, Dotty #14773). Graveyard lessons from first-hand retrospectives. |

**Overall confidence: HIGH** — with caveat that Palisade-original patterns (LIMM, Defensive Membrane) lack prior-art validation at this combination. Phase 2 + Phase 3 should expect higher-than-usual experimental scope.

### Gaps to Address

- **`StructuredTaskScope` preview-churn absorption** — concrete shape of the Palisade-stable `LinearScope` Idris effect. Phase 3 research.
- **`BytecodeEmitter` SPI design** — abstraction boundary between Palisade codegen and ASM/ClassFile API. Phase 1 research.
- **Defensive Membrane hole enumeration** — authoritative list of reflection/serialization/lambda-capture surfaces and which are closable. Phase 2 research.
- **Spring Context Indexer integration** — file format, ordering, toolchain hooks. Phase 4 (or Phase 6 with plugins).
- **JFR event taxonomy + throttle tiers** — always-on vs detail-only; `@Throttle` rates; category hierarchy. Phase 4 research.
- **JSR-45 SMAP vs `LineNumberTable` strategy** — which source wins; stack-trace decoder CLI scope. Phase 1 (lines) + Phase 4 (SMAP).
- **The nine feature gaps** — `@Jvm*` hints, Java interface impl, Java class extension policy, static method exposure, ADT projection rules, HTTP/JSON/JDBC, hot-reload change-set scope, deployment-modes doc, Loom-Integration-Track grouping. Some become Decisions 41+, some become roadmap items, some documentation. Resolve during roadmap creation.
- **Reference Spring Boot service scope** — concrete app isn't scoped. Resolve at Phase 3 planning.
- **Idris compiler Set-vs-List iteration order** (codebase CONCERNS.md) — affects reproducibility. Audit during Phase 1.

## Sources

### Primary (HIGH confidence)
- JVM platform: [JDK 25](https://openjdk.org/projects/jdk/25/), [JEP 484](https://openjdk.org/jeps/484), [JEP 491](https://openjdk.org/jeps/491), [JEP 505](https://openjdk.org/jeps/505), [JEP 506](https://openjdk.org/jeps/506), [JEP 525](https://openjdk.org/jeps/525), [JEP 509](https://openjdk.org/jeps/509), [JEP 439](https://openjdk.org/jeps/439), [JEP 401](https://openjdk.org/projects/valhalla/value-objects), [`jdk.jfr` API](https://docs.oracle.com/en/java/javase/25/docs/api/jdk.jfr/jdk/jfr/package-summary.html), [`ScopedValue` API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/ScopedValue.html).
- Spring Boot 4 / Framework 7: [Spring Boot 4.0.0 release](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now/), [Spring Boot 4.0.5](https://spring.io/blog/2026/03/26/spring-boot-4-0-5-available-now/), [OTel with Spring Boot](https://spring.io/blog/2025/11/18/opentelemetry-with-spring-boot/), [Spring Classpath Scanning](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html), [Spring Boot AOT](https://docs.spring.io/spring-boot/reference/packaging/aot.html).
- GraalVM: [Reachability Metadata](https://www.graalvm.org/latest/reference-manual/native-image/metadata/), [graalvm-reachability-metadata #655](https://github.com/oracle/graalvm-reachability-metadata/issues/655), [Spring Boot Native Image](https://docs.spring.io/spring-boot/reference/packaging/native-image/introducing-graalvm-native-images.html).
- Prior-art compiler sources (inspected): [Idris 2 source](https://github.com/idris-lang/Idris2/tree/main/src/Compiler), [idris-jvm](https://github.com/mmhelloworld/idris-jvm), [Kotlin compiler/ir](https://github.com/JetBrains/kotlin/tree/master/compiler/ir), [Scala 3 phases](https://nightly.scala-lang.org/docs/contributing/architecture/phases.html), [Clojure tools.emitter.jvm](https://github.com/clojure/tools.emitter.jvm).
- ASM/bytecode: [ASM Developer Guide](https://asm.ow2.io/developer-guide.html), [ClassWriter Javadoc](https://asm.ow2.io/javadoc/org/objectweb/asm/ClassWriter.html), [JaCoCo #1009](https://github.com/jacoco/jacoco/issues/1009), [ASM #317986](https://gitlab.ow2.org/asm/asm/-/issues/317986).
- Test/coverage/benchmark: [JUnit 5](https://junit.org/junit5/), [JaCoCo](https://www.jacoco.org/jacoco/trunk/doc/), [JMH](https://openjdk.org/projects/code-tools/jmh/), [PIT](https://pitest.org/), [CycloneDX Maven](https://github.com/CycloneDX/cyclonedx-maven-plugin).
- JFR + observability: [Custom JFR Events (Inside.java)](https://inside.java/2022/04/25/sip48/), [JDK-8257602 JFR Throttling](https://bugs.openjdk.org/browse/JDK-8257602), [JSR-45](https://jcp.org/en/jsr/detail?id=45).
- Loom pitfalls: [JEP 491](https://openjdk.org/jeps/491), [Carrier Pinning Trap](https://azguards.com/distributed-systems/the-carrier-pinning-trap-diagnosing-virtual-thread-starvation-in-spring-boot-3-migrations/), [OTel context with virtual threads](https://softwaremill.com/propagating-opentelemetry-context-when-using-virtual-threads-and-structured-concurrency/), [opentelemetry-java-instrumentation #11950](https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/11950).

### Secondary (MEDIUM confidence)
- Graveyard retrospectives: [The Story of Eta](https://medium.com/@rahulmuttineni/the-story-of-eta-pure-love-pure-functional-programming-2a690f3082b4), [Frege, a JVM Haskell](https://taylor.fausak.me/2015/06/25/frege-a-jvm-haskell/), [Idris JVM 0.7.0](https://mmhelloworld.github.io/blog/2024/07/15/idris-jvm-0-7-0-release/).
- Prior-art deep dives: [Scala 3 backend internals](https://dotty.epfl.ch/docs/internals/backend.html), [Crash Course on Kotlin Compiler](https://medium.com/google-developer-experts/crash-course-on-the-kotlin-compiler-k1-k2-frontends-backends-fe2238790bd8), [Decompiling Clojure II](http://blog.guillermowinkler.com/blog/2014/04/21/decompiling-clojure-ii/), [ClassFile API vs ASM (INNOQ)](https://www.innoq.com/en/articles/2025/04/java-class-file-api/).
- Niche-language failures: [Scala.js Semantics](https://www.scala-js.org/doc/semantics.html), [Why I Bet on Scala.js](http://www.lihaoyi.com/post/FromfirstprinciplesWhyIbetonScalajs.html).
- Reproducible builds: [Reproducible JVM builds](https://reproducible-builds.org/docs/jvm/), [Apache Maven Reproducible Builds](https://maven.apache.org/guides/mini/guide-reproducible-builds.html).
- Tail calls: [On Recursion, Continuations and Trampolines](https://eli.thegreenplace.net/2017/on-recursion-continuations-and-trampolines/), [Dotty #14773](https://github.com/lampepfl/dotty/issues/14773).
- Linear types: [Tweag — inline-java safe memory management](https://www.tweag.io/blog/2020-02-06-safe-inline-java/), [ACM — Safe-by-default Concurrency](https://dl.acm.org/doi/fullHtml/10.1145/3462206).

### Tertiary (LOW confidence — needs validation)
- PIT mutator behavior against Idris-emitted bytecode — phase-time verification.
- JaCoCo 0.8.13 JDK 25 class-file edges — Phase 1 verification.
- "ASM 9.9 for JDK 26" claims (one 2026 forward-looking source) — not yet on Maven Central.
- LSP API depth for elaborator-reflection-driven Spring annotation completion — Reconsider #4 flags this.

### PROJECT.md and codebase cross-references
- `.planning/PROJECT.md` — 40 Key Decisions, Active Requirements, Out of Scope, Constraints.
- `.planning/research/{STACK,FEATURES,ARCHITECTURE,PITFALLS}.md` — research documents synthesized.
- `.planning/codebase/{STACK,ARCHITECTURE,CONCERNS}.md` — upstream Idris 2 codebase shape.

---

*Research synthesized: 2026-04-21*
*Ready for roadmap: yes*
