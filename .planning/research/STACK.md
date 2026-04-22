# Stack Research

**Domain:** JVM compiler backend for a verified functional language (Idris 2 → JVM bytecode), targeting Spring Boot deployments on JVM 25+
**Researched:** 2026-04-21
**Confidence:** HIGH for JDK/Spring/ASM core; MEDIUM for prior-art deep-dive (mostly secondary sources); LOW where flagged inline.

## Executive Summary

A Palisade-class JVM backend in 2026 sits at the intersection of three stacks that have all stabilized recently:

1. **JVM 25 LTS** (released 2025-09-16) finalized every concurrency primitive PROJECT.md depends on: ScopedValues (JEP 506), StructuredTaskScope (JEP 505 — still **preview** in 25, finalized projection in 26+), and unpinned virtual-thread `synchronized` (JEP 491, JDK 24). Generational ZGC is the only ZGC and is production-grade for trading-class latency.
2. **Spring Boot 4.0** (released 2025-11-20, current 4.0.5 as of 2026-03-26) is the deployment substrate. Built on Spring Framework 7 with native JSpecify, OTel starter, and improved AOT/native-image support. Crucially, Spring Boot 4.0 keeps **Java 17 as the minimum baseline** while offering "first-class Java 25 support" — Palisade's JVM 25+ floor is an opinionated choice, not an inherited constraint.
3. **JVM bytecode tooling** is in transition: JEP 484 finalized the JDK `java.lang.classfile` API in JDK 24, with the explicit long-term goal of replacing ASM. ASM 9.8 (current) supports JDK 25 class files. The prior art (Kotlin, Scala 3, Clojure) all use ASM today; none have migrated to ClassFile API yet. PROJECT.md Decision 39 (ASM) is the right pick for v1, but the Class-File API is a real future migration target worth tracking.

The single biggest "you should reconsider" is **Decision 39 (ASM)** — not to overturn it, but to layer in an abstraction boundary so swapping to JEP 484 ClassFile API later is a single-module change. Details in "Reconsider" section below.

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **JDK** | OpenJDK 25 (LTS, GA 2025-09-16) | Compile + runtime baseline | Loom finalized (JEP 491 unpinning, JDK 24), ScopedValues finalized (JEP 506), Generational ZGC default, JFR matures, 8-year LTS. PROJECT.md Constraint already pins this — verified correct as of 2026-04-21. |
| **ASM** | 9.8 (latest as of 2026; supports JDK 25 class files) | Bytecode emission | What Kotlin, Scala 3 (`scala-asm` fork), Clojure, Groovy, idris-jvm all use. Battle-tested across every JVM verifier edge case. PROJECT.md Decision 39 — confirmed standard. |
| **Idris 2** | Tagged release (current 0.8.0+) with `feat/palisade-jvm` rebase cadence | Source language + compiler frontend | PROJECT.md Constraint. Vendoring `NamedCExp` snapshot per Decision 38. |
| **Spring Boot** | 4.0.x (current 4.0.5; Java 17 baseline, first-class Java 25) | Deployment substrate | Aligns with PROJECT.md Decision 5 (Spring DI) and Constraint. Spring Framework 7 + JSpecify + native OTel starter + improved AOT. |
| **Spring Framework** | 7.x (transitively via Boot 4) | Core DI / web / AOT | Required by Spring Boot 4.0. JSpecify-annotated for null-safety — relevant for Idris↔Spring interop projection. |
| **GraalVM Native Image** | Oracle GraalVM for JDK 25 | AOT deployment target | PROJECT.md Decision 7. Spring Boot 4.0 has GA native support; reachability metadata under `META-INF/native-image/<group>/<artifact>/reachability-metadata.json` (single-file format current as of 2026, replacing the older split `reflect-config.json` / `resource-config.json` / `proxy-config.json` files). |
| **Maven** | 3.9.x (latest stable line) | Primary build tool — first-class plugin | PROJECT.md Decision 10. Industry default for Spring Boot. Mature CycloneDX/JaCoCo/JMH plugin ecosystem. |
| **Gradle** | 8.x (Kotlin DSL) | Secondary build tool — first-class plugin | PROJECT.md Decision 10. Mandatory for Spring Boot/Kotlin shops. Better incremental story for compiler-heavy builds. |

### Bytecode-Library Conventions (How ASM Is Used in Compiler Backends)

These are the load-bearing patterns Palisade's codegen layer should adopt. Each is documented in the prior art's source:

| Pattern | Recommendation | Why |
|---------|----------------|-----|
| **Frame computation** | `ClassWriter(ClassWriter.COMPUTE_FRAMES)` for v1; precompute manually only for hot codegen paths once profiled | `COMPUTE_FRAMES` implies `COMPUTE_MAXS`. Slower at compile time but correct against verifier. Scala/Kotlin both default to this then optimize specific cases. PROJECT.md Decision 27 (zero-tolerance verification) demands the safe default. |
| **Two-queue pipeline** | NamedCExp → ASM `ClassNode` (intermediate) → bytecode `byte[]` | Mirrors Scala 3 GenBCode (`CodeGen.scala` → `PostProcessor.scala`). The `ClassNode` intermediate enables parallel post-processing (write-to-disk, JSR-45 SMAP injection, JFR metadata). |
| **Lambdas / closures** | `invokedynamic` with `LambdaMetafactory.metafactory` bootstrap | Kotlin (since 1.5, default since 2.0 with `-Xlambdas=indy`), Scala 3, javac all do this. Avoids per-lambda anonymous classes; enables JVM hot-path inlining. Aligns with PROJECT.md Decision 16 (JIT shape — `invokedynamic` with Guard-With-Test). |
| **Module emission** | `ClassWriter.visitModule()` → `ModuleVisitor` (`visitRequire`, `visitExports`, `visitOpens`, `visitProvide`, `visitUse`) | Native ASM API; ASM itself uses BND tooling internally, but for a compiler emitting from `.ipkg` metadata you walk the `ModuleVisitor` directly. Aligns with PROJECT.md Decision 40. |
| **Source maps** | Inject SMAP via `ClassWriter.visitSource(name, debug)` where `debug` is the JSR-45 SMAP string; supplement with `LineNumberTable` and `LocalVariableTable` per method | Standard JSR-45 / Jakarta Debugging Support for Other Languages 2.0 path. Kotlin (since 1.5, JSR-45-compliant), Scala (in progress) use this. JaCoCo, IntelliJ, JDB, async-profiler all read it. PROJECT.md Decision 21. |
| **TCO trampolining** | Direct self-recursion → labeled loop with `GOTO`; mutual recursion → `invokedynamic` trampoline (idris-jvm style) | idris-jvm's documented approach. Aligns with PROJECT.md Decision 2. Verifier-friendly when paired with `COMPUTE_FRAMES`. |
| **Constant pool reuse** | When transforming existing classes, use `ClassWriter(reader, flags)` constructor | Copies constant pool + untouched methods bytecode-verbatim — large speedup. Less relevant for fresh emission, very relevant for bytecode rewriting tools (e.g., agent for hot reload, Decision 32). |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| **JUnit 5 (Jupiter + Platform)** | 5.12+ (latest 5.14.x as of 2026) | Test harness; custom `TestEngine` SPI for Idris PBT/PDDT discovery | PROJECT.md Decision 25. JUnit Platform's `TestEngine` interface is the documented way to expose non-Jupiter test sources to Maven Surefire / Gradle test runners. Engine ID convention: `palisade-idris`. Note: `junit-platform-jfr` module was removed; JFR for test events is now in `junit-platform-launcher`. |
| **JaCoCo** | 0.8.13+ (current line) | Coverage instrumentation | PROJECT.md Decision 26. JaCoCo reads JSR-45 SMAP and lifts coverage to source-language line numbers — Kotlin's compiler emits SMAPs JaCoCo understands; Palisade emits SMAPs the same way. |
| **JMH** | 1.37 | Microbenchmarks | PROJECT.md Decision 28. Industry standard. Use `me.champeau.jmh` Gradle plugin (latest 0.7.3) or `metlos/jmh-maven-plugin`. |
| **PIT (pitest)** | 1.19.1+ | Mutation testing | PDDT/PBT methodology in PROJECT.md Context. Use `pitest-maven` (1.15.5+) with `pitest-junit5-plugin` (1.2.1+). LOW confidence on whether PIT's mutators target Idris-emitted bytecode shape correctly without configuration — flag for Phase research. |
| **CycloneDX Maven/Gradle plugin** | `cyclonedx-maven-plugin` (current 2.x line); `cyclonedx-gradle-plugin` (current 2.x line) | SBOM generation | PROJECT.md Decision 35. CycloneDX is now de facto for Java SBOMs. |
| **OpenTelemetry Java** | OTel 1.x SDK + `spring-boot-starter-opentelemetry` (Spring Boot 4 ships this) | Tracing/metrics/logs | PROJECT.md Decision 24. Spring Boot 4.0's first-party OTel starter bridges Micrometer to OTLP automatically. For native-image deployments use the starter (not the agent) — agent doesn't generally work with GraalVM native. |
| **Micrometer** | 1.14+ (Spring Boot 4 line) | Metrics facade | Required transitively by Spring Boot Actuator. OTel bridge is automatic. |
| **JFR** | `jdk.jfr` (in `java.base`) | Custom production telemetry | PROJECT.md Decision 22. Native Java API; no dependency. Subclass `jdk.jfr.Event`, annotate with `@Label`, `@Description`, `@Category`, `@StackTrace`. Use `Event.shouldCommit()` guard for hot paths. JEP 509 (JDK 25) adds experimental CPU-time profiling on Linux. |
| **Spring Boot Actuator** | Spring Boot 4 line | Health/metrics endpoints | Standard for Decision 24. Pairs with Micrometer + OTel automatically. |

### Build/Tooling — Reproducibility & Supply Chain

| Tool | Version / Approach | Purpose |
|------|-------|---------|
| **Maven Reproducible Builds** | `project.build.outputTimestamp` + `--reproducible` (Maven 3.9+) | Decision 36. Reproducible by contract; pin source date. |
| **Gradle dependency locking** | `dependencies --write-locks` | Decision 36. Lockfile per configuration. |
| **SLSA provenance** | GitHub Actions `slsa-framework/slsa-github-generator` | Decision 36. SLSA L3 generator action publishes signed provenance artifacts. Cross-host hash compare in CI separately. |
| **Sigstore / cosign** | Latest | Signed releases (Decision 35). Keyless signing via OIDC. |
| **GitHub Security Advisories + CVE issuance** | GH-native | Decision 35. Standard OSS workflow. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| **Idris LSP** | IDE integration (Decision 11) | Extend with JVM-type / Spring-annotation knowledge. Already standard; works in VSCode, Emacs, Vim, IntelliJ via LSP plugin. |
| **`javap`** | Disassembly verification in CI | Standard verification pair with `java -Xverify:all` (Decision 27). Use `-v -p -c` for full bytecode + verifier metadata. |
| **JDK Mission Control (JMC)** | JFR consumer (Decision 22) | Bundled separately from JDK 11+. Custom event templates ship as `.jfc` XML; Palisade should publish a `palisade.jfc` template alongside releases. |
| **async-profiler** | Sampling profiler with JSR-45 awareness | Reads SMAP — Idris source lines visible in flame graphs without plugins. |
| **VisualVM / MAT** | Heap analysis (Decision 23) | Both honor JSR-45 source mapping when present. |

## Installation

```bash
# Idris 2 — already in tree; no change

# JVM toolchain (developer)
sdk install java 25-tem               # Eclipse Temurin 25
sdk install maven 3.9.9
sdk install gradle 8.10
sdk install springboot 4.0.5

# GraalVM (native-image deployment target)
sdk install java 25-graal             # Oracle GraalVM for JDK 25
gu install native-image               # if not bundled
```

```xml
<!-- Palisade compiler module — Maven dependencies for the codegen library itself -->
<dependencies>
  <dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm</artifactId>
    <version>9.8</version>
  </dependency>
  <dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-tree</artifactId>
    <version>9.8</version>
  </dependency>
  <dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-commons</artifactId>      <!-- GeneratorAdapter, etc. -->
    <version>9.8</version>
  </dependency>
  <dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-util</artifactId>         <!-- CheckClassAdapter, Textifier — CI verification -->
    <version>9.8</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

```xml
<!-- Reference Spring Boot service module — runtime dependencies on the test-bed app -->
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>4.0.5</version>
</parent>
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
  </dependency>
</dependencies>
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| **ASM 9.8** | **JEP 484 ClassFile API** (`java.lang.classfile`, finalized JDK 24) | When Palisade can drop pre-25 JDK class-file emission entirely AND the API has matured through 1-2 LTS cycles of community use. Built into JDK, evolves with class-file format every 6 months, no third-party dependency, cleaner immutable API. As of 2026, no major JVM language compiler has migrated. Recommend: build behind a `BytecodeEmitter` interface so swap is one module. |
| **ASM 9.8** | **ByteBuddy** | Never for Palisade — ByteBuddy is a high-level fluent DSL for runtime code generation (proxies, agents, instrumentation), built *on top of* ASM. Wrong abstraction level for a language compiler that needs fine control over every instruction. The prior art (Kotlin, Scala 3, Clojure) all use ASM directly. |
| **ASM 9.8** | **Kotlin's `Koffee`-style DSL or in-house Idris-flavored DSL** | Could wrap ASM in an Idris-native DSL that uses linear types to enforce visitor-call ordering at the type level — interesting research thread, not v1. Phase-level decision. |
| **Maven + Gradle (both)** | Bazel | If JPMC-class shops require it; Bazel has good JVM support but smaller plugin ecosystem. Decision 10 is "both Maven and Gradle"; Bazel is out of scope per implicit prioritization. |
| **JUnit 5 custom TestEngine** | ScalaTest-style standalone runner | JUnit 5 Platform is the integration point Spring/Maven/Gradle/IntelliJ/IDEA all consume natively. A standalone runner re-introduces the integration tax PROJECT.md Decision 25 explicitly avoids. |
| **JaCoCo** | Kover (Kotlin) / IntelliJ coverage agent | JaCoCo is the only mature JVM agent that reads JSR-45 SMAP and lifts to source-language coverage. Kover is Kotlin-specific. |
| **Generational ZGC (Decision 13)** | Shenandoah / G1 | G1 is JDK 25 default — better throughput, higher tail latency. Shenandoah (Generational, JDK 25) competes with ZGC; ZGC has better sub-ms tail-latency story, Shenandoah has better steady-state throughput. For "trading-grade" pause profiles ZGC remains the pick. PROJECT.md Decision 13 confirmed. |
| **OpenTelemetry Java agent** | **OpenTelemetry Spring Boot starter** | The Spring Boot starter is correct for Palisade because the agent doesn't work with GraalVM native-image. PROJECT.md Decision 7 (native-image first-class) forces the starter. |
| **Spring Boot 4.0** | Quarkus / Micronaut | Both are GraalVM-native-first frameworks with smaller startup/memory than Spring Boot. PROJECT.md explicitly targets Spring Boot as the dominant enterprise substrate (JPMC archetype). Micronaut/Quarkus are smaller markets. Could be future targets. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **`javassist`** | High-level bytecode manipulation library; uses string-based source-like API; slower, less precise, weaker JDK-25-class-file support, near-stagnant maintenance | ASM 9.8 (Decision 39) |
| **`cglib`** | Effectively dead; superseded by ASM directly and ByteBuddy. Spring 6+ has been migrating off it | ASM 9.8 directly |
| **Older ASM (9.7 or earlier)** | Doesn't fully support JDK 25 class-file format (major version 69) | ASM 9.8 |
| **JUnit 4** | EOL'd for new code; JUnit 5 Platform is the discovery contract every modern build tool reads | JUnit 5 Platform + custom `TestEngine` |
| **Manual reflect-config.json / resource-config.json / proxy-config.json (split files)** | Older split-file native-image metadata format; superseded | Single `META-INF/native-image/<group>/<artifact>/reachability-metadata.json` file (current format) |
| **`-Xlambdas=class`** (i.e., emit anonymous classes for lambdas) | Bypasses `LambdaMetafactory` — extra .class files, no JVM lambda fast-path, larger fat-JARs, defeats Decision 16's JIT story | `invokedynamic` + `LambdaMetafactory.metafactory` bootstrap |
| **`org.jetbrains:annotations` (`@NotNull`/`@Nullable`)** for null-safety on emitted bytecode | JetBrains-specific; Spring 7 / Spring Boot 4 standardize on JSpecify | JSpecify (`org.jspecify:jspecify`) annotations on emitted Idris-bound classes for Spring 7 null-safety interop |
| **`junit-platform-jfr` module** | Removed; functionality folded into `junit-platform-launcher` | `junit-platform-launcher` directly |
| **Non-generational ZGC** | Removed in JDK 24 | Generational ZGC is now the only ZGC (matches Decision 13 automatically) |
| **`-Xlambdas=indy` for Kotlin-style as-default before testing on JDK 25** | Compiler-team gotcha: `LambdaMetafactory` calls `MethodHandles.Lookup.defineHiddenClass`; with module-strict builds (`module-info.class` per Decision 40) the lookup must be in the right module | Test with JPMS modules from the first emitted class; document the lookup-context invariant |

## Stack Patterns by Variant

**If targeting GraalVM native-image (Decision 7 — should be default):**
- Emit `META-INF/native-image/<group>/<artifact>/reachability-metadata.json` per Idris package, derived from compile-time analysis of FFI sites, reflection use, JFR event classes, Spring beans
- Treat the OpenTelemetry Spring Boot starter (not agent) as the only OTel option
- Annotate Spring beans on the *emitted bytecode* (Decision 5) so Spring AOT processor sees them at build time
- All `invokedynamic` bootstrap methods must be reachable; declare them in metadata
- Avoid runtime `defineClass` (no dynamic class loading at runtime)

**If targeting JVM-mode (HotSpot, no native-image):**
- Same emitted bytecode; reachability metadata is harmless extra files
- OTel agent OR starter both work; starter still preferred for consistency
- Hot reload (Decision 32) only viable in JVM mode — native-image precludes it

**If hybrid Java + Kotlin + Idris module (Decision 30):**
- Palisade compiler integrates as a Maven/Gradle plugin invoked before/after `kotlinc` and `javac` in the lifecycle
- Single output `target/classes/` directory; all three compilers contribute `.class` files
- `module-info.class` (Decision 40) emitted by Palisade with `requires` / `exports` aggregating across language sources
- Coordinate: Idris classes must compile *first* if Java/Kotlin import them; Java/Kotlin must compile first if Idris FFI-imports them. Two-phase build is the practical convention (Kotlin does this with Java already).

**If using Spring TestContext (Decision 25):**
- Custom `TestEngine` reports test instances to Spring's `TestContextManager` for DI/transaction/mock injection
- Idris properties run inside the same Spring `ApplicationContext` as Java tests — no separate harness

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| ASM 9.8 | JDK 6 – JDK 25 class files | Class-file major version 69 (JDK 25). For experimental JDK 26+ class files use ASM 9.9 (when released) or `Opcodes.ASM10_EXPERIMENTAL`. |
| Spring Boot 4.0.x | Java 17 baseline; first-class Java 25; Spring Framework 7.x | Palisade can compile-target JDK 25 bytecode and still deploy under Spring Boot 4 because Spring runs on 17+ JVM. |
| GraalVM Native Image (for JDK 25) | JDK 25 baseline | Oracle GraalVM for JDK 25; Spring Boot 4 native-image plugin pinned to compatible matrix. |
| JUnit 5.12+ | JDK 17 baseline (minimum); compiles JDK 8 bytecode for legacy module compatibility | Custom `TestEngine` API stable since 5.0; deprecations rare. |
| JaCoCo 0.8.13+ | All JDK ≤ 25; reads Kotlin 1.5+ SMAPs | LOW confidence on whether 0.8.13 fully understands JDK 25 class-file edge cases — Phase-1 verification item. |
| JMH 1.37 | All JDK; uses annotation processor — no JDK-version coupling | Pin annotation-processor to same 1.37 version. |
| ScopedValue (JEP 506) | Finalized JDK 25 | Stable API; safe to depend on without `--enable-preview`. |
| StructuredTaskScope | **Preview** in JDK 25 (JEP 505); 6th preview JDK 26 (JEP 525); 7th preview JDK 27 (JEP 533) | **Critical caveat** — see "Reconsider" §1 below. |

## Prior-Art Cross-Reference

| Project | Frontend → Backend Bridge | Bytecode Library | TCO Strategy | Lambda Strategy | JSR-45 SMAP |
|---------|---------------------------|------------------|--------------|-----------------|-------------|
| **idris-jvm** (mmhelloworld) | Idris IR → custom JVM IR (Haskell-side, pre-self-hosting era) | ASM (via Java/Idris-side) | Direct loop for self-recursion; `invokedynamic` trampoline for mutual | (predates `invokedynamic` lambdas in Idris context) | Yes — emits source file/line debug info |
| **Kotlin** | Kotlin source → FIR → IR → backend | ASM | JVM-native (no language-level TCO except `tailrec` keyword → loop) | `invokedynamic` + `LambdaMetafactory` (default since 2.0) | Yes — JSR-45-compliant since 1.5 |
| **Scala 3 (Dotty)** | Scala source → Tasty → backend (`GenBCode`) | `scala-asm` (fork of ASM, kept current) | No general TCO; `@tailrec` → loop | `invokedynamic` since Scala 2.12 (JDK 8 baseline shifted to 17) | In progress — Scala Center initiative |
| **Clojure** | Reader → analyzer → emit | ASM (`clojure.asm` — internal repackage) | None (Clojure uses `recur` → loop, `trampoline` library function) | Anonymous classes historically; `invokedynamic` available via `tools.emitter.jvm` | Limited |
| **Eta** (defunct as of ~2018-2019) | GHC frontend → STG → JVM bytecode | ASM (via Java side) | Lazy-eval thunks; trampolined evaluator | Functions = JVM classes | Yes |
| **Palisade (target)** | Idris 2 frontend → vendored `NamedCExp` → JVM codegen | ASM 9.8 | Self-recursion → loop; mutual → `invokedynamic` trampoline | `invokedynamic` + `LambdaMetafactory` | Yes — JSR-45 SMAP per class (Decision 21) |

**Convergent picks** (everyone does this — high confidence Palisade should too): ASM, `invokedynamic` lambdas, JSR-45 SMAPs, `COMPUTE_FRAMES` default + manual override for hot paths.

**Divergent pick** (Palisade is more aggressive than prior art): mutual-recursion `invokedynamic` trampoline. Only idris-jvm did this; Kotlin/Scala leave general TCO to the user. Justification in PROJECT.md Decision 2 — Idris idiom is heavy mutual recursion.

## What's Load-Bearing in JDK 25 (Palisade-Specific)

| JEP / Feature | Status in JDK 25 | Why It Matters for Palisade |
|---------------|------------------|------------------------------|
| **JEP 506: Scoped Values** | **Final** | Decision 9 (state mgmt: `ScopedValue` mapping), Decision 19 (Type-Bound `ScopedValues` as concurrency primitive). Stable API — safe to depend on. |
| **JEP 491: Synchronize Virtual Threads without Pinning** | Shipped JDK 24, in 25 | Decision 18 (Loom + Linear Futures). Removes the largest carrier-pinning gotcha. Idris-emitted code can use `synchronized` (rare in pure FP, but Java FFI calls into it) without breaking virtual-thread scaling. |
| **JEP 505: Structured Concurrency** | **Preview (5th)** in 25 | Decision 18 (`StructuredTaskScope` formalized as Idris effect). **Hard caveat**: still preview, API surface continues to refine (6th preview JDK 26, 7th preview JDK 27). Palisade must wrap `StructuredTaskScope` behind a Palisade-stable Idris effect API and absorb upstream churn internally. |
| **Generational ZGC (only ZGC)** | Final / non-generational removed | Decision 13. Override JDK 25 default (G1) via `-XX:+UseZGC` in Palisade-emitted launchers. |
| **JEP 484: Class-File API** | Final in JDK 24, present in 25 | Future ASM replacement. See "Reconsider" §2. |
| **JEP 509: JFR CPU-Time Profiling (Experimental, Linux)** | **Experimental** | Decision 22 (JFR vocabulary). Low-cost CPU-time sampling on Linux — enables Palisade-emitted services to ship CPU-profiles in JFR streams without external profiler. |
| **Compact Object Headers (Final)** | Final | Decision 14 (zero-alloc). Smaller object headers shave per-object memory — partial mitigation while waiting for Valhalla. |
| **JEP 401: Value Classes (Valhalla)** | **Preview, JDK 26** (NOT in 25) | Decision 14 (Valhalla as the long-game zero-alloc lever). Realistically lands GA in JDK 27 / 28. Plan accordingly. |

## Reconsider These PROJECT.md Decisions

These are not "you got it wrong" — they're "here's nuance worth a second look before the phase commits."

### 1. Decision 18 (Loom + StructuredTaskScope) — **API still preview through at least JDK 27**

`StructuredTaskScope` was JEP 505 (5th preview, JDK 25), JEP 525 (6th preview, JDK 26), JEP 533 (7th preview, JDK 27). It will likely finalize in JDK 27 LTS or JDK 28. **Recommendation:** Wrap it behind a Palisade-internal `LinearScope` Idris effect now; keep the JVM-side implementation hidden so each preview's API churn is a one-file change. Don't expose `StructuredTaskScope` types in user-facing Idris signatures until the JDK API stabilizes. This already aligns with the "formalized as Idris effect" framing in Decision 19, but worth making the abstraction-boundary discipline explicit.

**Confidence:** HIGH — preview status is documented across JEPs 505, 525, 533.

### 2. Decision 39 (ASM) — **Add a `BytecodeEmitter` SPI from day one**

ASM is the right pick for v1, full stop. But:
- JEP 484 finalized the JDK ClassFile API in JDK 24 with the explicit goal of replacing third-party ASM. The JDK is migrating its own internal ASM usage.
- Class-file format evolves every 6 months. A first-party API maintained by the JVM team will track newer formats faster than ASM's release cadence.
- Migration is a 5-10 year story, but it *is* coming.

**Recommendation:** Define a thin `Compiler.JVM.Emit.Backend` SPI inside `src/Compiler/JVM/`, with `Compiler.JVM.Emit.ASM` as the v1 implementation. All codegen calls go through the SPI. When ClassFile API is mature enough (~JDK 27 LTS + 1 cycle) a `Compiler.JVM.Emit.ClassFile` implementation is a single-module addition. This also makes the codebase testable against two backends — strong correctness signal.

**Confidence:** HIGH on the migration trajectory; MEDIUM on the timing.

### 3. Decision 16 (JIT shape — `invokedynamic` + Guard-With-Test) — **Verify against JDK 25's JIT changes**

The "Guard-With-Test chain" pattern works well on C2. JDK 25 JIT compiler (JEP 514: Ahead-of-Time Command-Line Ergonomics, and ongoing Leyden work) is changing AOT/JIT interaction. **Recommendation:** Validate the megamorphic call-site profile shape on JDK 25 JFR + `-XX:+PrintInlining` early in the relevant phase — don't assume Java 21-era profiling assumptions transfer.

**Confidence:** MEDIUM — Leyden is moving fast; specifics deserve their own phase research.

### 4. Decision 11 (LSP-only IDE strategy) — **No reconsideration; just a note**

Idris LSP exists and is the right primary target. Worth flagging that for *Spring annotation* completion / hover (Decision 5), the LSP layer needs to call into the Idris elaborator's reflection capability to discover annotation classes from the JVM classpath. That's an LSP capability extension, not a separate IntelliJ plugin — but it's nontrivial work that's invisible from "extend Idris LSP" framing.

**Confidence:** HIGH on the LSP choice; flagging effort-estimate risk.

### 5. Decision 5 (Annotations on emitted bytecode) — **Spring Boot 4 / Spring Framework 7 = JSpecify, not JetBrains annotations**

Spring 7 uses JSpecify for null-safety annotations as a deliberate move away from JetBrains' `@NotNull`/`@Nullable`. **Recommendation:** Palisade-emitted bytecode for Spring beans should emit `@org.jspecify.annotations.NonNull` / `@Nullable` derived from Idris's type-level non-emptiness facts (e.g., `Maybe a` → `@Nullable T`, plain `a` → `@NonNull T`). This makes Idris's totality and partiality information consumable by Spring 7's framework-level null-safety verifier.

**Confidence:** HIGH — verified against Spring Boot 4 release notes.

## Sources

### Primary (HIGH confidence)
- [JDK 25 (openjdk.org)](https://openjdk.org/projects/jdk/25/) — release feature list, JEPs
- [JEP 484: Class-File API](https://openjdk.org/jeps/484) — finalized JDK 24, ASM replacement trajectory
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491) — JDK 24 ship, `synchronized` no longer pins virtual threads
- [JEP 505: Structured Concurrency (Fifth Preview)](https://openjdk.org/jeps/505) — preview status confirmation, JDK 25
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506) — finalized JDK 25
- [JEP 446: Scoped Values (Preview history)](https://openjdk.org/jeps/446) — pre-final lineage
- [ScopedValue (Java SE 25 & JDK 25 API)](https://docs.oracle.com/en/java/javase/25/docs//api/java.base/java/lang/ScopedValue.html) — official API
- [jdk.jfr (Java SE 25 & JDK 25 API)](https://docs.oracle.com/en/java/javase/25/docs/api/jdk.jfr/jdk/jfr/package-summary.html) — JFR custom event API
- [Spring Boot 4.0.0 release announcement](https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now/) — release date, feature set
- [Spring Boot 4.0.5 release announcement](https://spring.io/blog/2026/03/26/spring-boot-4-0-5-available-now/) — current as of 2026-04-21
- [OpenTelemetry with Spring Boot (Spring blog, Nov 2025)](https://spring.io/blog/2025/11/18/opentelemetry-with-spring-boot/) — first-party OTel starter in Spring Boot 4
- [Reachability Metadata (GraalVM)](https://www.graalvm.org/latest/reference-manual/native-image/metadata/) — single-file `reachability-metadata.json` format
- [ASM Developer Guide](https://asm.ow2.io/developer-guide.html) — ClassWriter / ModuleVisitor / COMPUTE_FRAMES patterns
- [ClassWriter (ASM Javadoc)](https://asm.ow2.io/javadoc/org/objectweb/asm/ClassWriter.html) — flag semantics
- [Backend Internals — Scala 3 (EPFL)](https://dotty.epfl.ch/docs/internals/backend.html) — Scala 3's GenBCode pipeline architecture
- [JEP 401: Value Classes (Valhalla, JDK 26 preview)](https://openjdk.org/projects/valhalla/value-objects) — Valhalla status

### Secondary (MEDIUM confidence — synthesized from coverage)
- [InfoQ: Java 25 Released](https://www.infoq.com/news/2025/09/java25-released/) — feature roundup
- [Inside Java: What's New in Java 25 in 2 Minutes](https://inside.java/2025/10/17/new-in-jdk-25-2-mins/) — official YouTube channel summary
- [JaCoCo Change History](https://www.jacoco.org/jacoco/trunk/doc/changes.html) — Kotlin SMAP fixes timeline
- [idris-jvm README & blog (mmhelloworld)](https://github.com/mmhelloworld/idris-jvm) — prior-art bytecode strategy
- [Pauseless GC in Java 25: ZGC Guide (Andrew Baker)](https://andrewbaker.ninja/2025/12/03/deep-dive-pauseless-garbage-collection-in-java-25/) — Generational ZGC production notes
- [Jakarta Debugging Support for Other Languages 2.0](https://jakarta.eu/specifications/debugging/2.0/jdsol-spec-2.0) — JSR-45 successor spec
- [Kotlin `-Xlambdas=indy` discussion (multiple sources)](https://www.baeldung.com/java-invoke-dynamic) — `invokedynamic` lambda rationale
- [JEP 525: Structured Concurrency (Sixth Preview)](https://openjdk.org/jeps/525) — confirms continued preview status into JDK 26

### Confidence Notes
- **Java/Spring API specifics** (JEPs, Spring Boot version, ASM version): HIGH — verified against multiple official sources.
- **Prior-art compiler internals** (Kotlin FIR/IR pipeline, Scala 3 GenBCode, Clojure ASM use, Eta GHC fork): MEDIUM — synthesized from blog posts and project docs; I did not pull each compiler's source. Phases that depend on prior-art specifics should re-verify in source.
- **idris-jvm specifics**: MEDIUM — relies on README + 2020 blog post; codebase not exhaustively read. PROJECT.md Decision 1 already commits to *replacing* it, so deep idris-jvm archaeology has limited downstream value.
- **2026-specific freshness**: HIGH for items dated 2025-09 onwards; LOW for unverifiable 2026-only blog claims (e.g., "ASM 9.9 for JDK 26" appearing in one source — not yet on Maven Central as of this write-up; treat as forward-looking).

---
*Stack research for: JVM compiler backend for Idris 2, Spring Boot 4 / JVM 25+ deployment*
*Researched: 2026-04-21*
