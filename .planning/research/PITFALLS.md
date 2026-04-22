# Pitfalls Research

**Domain:** JVM compiler backend for a verified functional language (Idris 2 → JVM bytecode), Spring Boot integration, Loom/ZGC/JFR/JPMS/GraalVM-native target
**Researched:** 2026-04-21
**Confidence:** HIGH (history of similar projects is well-documented; JVM/Loom/Graal/JFR pitfalls have substantial literature; PROJECT.md decisions cross-reference cleanly to known failure modes)

---

## Historical Failure Analysis: Why "Similar" Projects Fizzled

Before the pitfall catalog, this is the post-mortem-of-the-graveyard. Palisade is one node in a chain of attempts to put a verified or pure FP language on the JVM. Most of those attempts under-delivered. The pattern matters because PROJECT.md's bar ("Jane Street happy") is exactly the bar the predecessors missed.

### idris-jvm (mmhelloworld)

- **Status (2026-04-21):** Active but slow-moving. 0.7.0 released July 2024. Single-maintainer. Adoption-light.
- **What they got right:** First working Idris 2 → JVM. Has Java method export, annotations, basic FFI.
- **What killed momentum:** Single-maintainer bus factor; JVM-as-design-target compromises (tail-call/erasure choices baked into the IR they emit); not built around a Spring/enterprise integration story; FFI ergonomics never reached "Kotlin-grade." Adoption never crossed the chasm because the **deployment substrate story was never first-class** — Spring Boot, JFR, GraalVM, JPMS were not the design center.
- **Lesson for Palisade:** Decision 1 (vanilla Idris 2 + own codegen consuming `NamedCExp`) is the right structural call: it puts Palisade ahead of idris-jvm's design-debt accumulated over 9 years. But the bus-factor and Spring-substrate problems are the actual killers; codegen quality is necessary, not sufficient.

Source: [GitHub - mmhelloworld/idris-jvm](https://github.com/mmhelloworld/idris-jvm), [Idris JVM 0.7.0 Release](https://mmhelloworld.github.io/blog/2024/07/15/idris-jvm-0-7-0-release/)

### Eta (Haskell on JVM)

- **Status:** ~4 years inactive as of 2023; effectively abandoned.
- **What killed it:** Inability to leverage Hackage (the deciding question for a Haskell on JVM was: can I `cabal install` aeson? Answer: no, not really.) Loss of purity at the Java boundary (any `HashSet` use forced everything to `IO`). Funding model collapsed (TypeLead pivoted away). Tooling never reached production-grade.
- **Lesson for Palisade:** Idris's situation is structurally different — Idris doesn't have Hackage to lose, so the "ecosystem leverage" trap is inverted. **Palisade's risk is the opposite: bringing too little of the Idris stdlib forward, not too much.** Decision 9 (comprehensive state-management mapping) and Decision 4 (Clojure-grade FFI) explicitly counter this.

Source: [The Story of Eta](https://medium.com/@rahulmuttineni/the-story-of-eta-pure-love-pure-functional-programming-2a690f3082b4), [Frege vs. Eta](https://dev.to/awwsmm/haskell-on-the-jvm-frege-vs-eta-5238)

### Frege (Haskell on JVM)

- **Status:** Quasi-dormant.
- **What killed it:** "Wrapping Java in Frege means everything ends up in `IO`, so there goes purity." Missing Haskell extensions (no MultiParamTypeClasses, etc.) prevented Haskell library reuse. Library mismatch with Prelude. Niche positioning.
- **Lesson for Palisade:** **The "monadification of the world" trap is real and lethal.** PROJECT.md Decision 8 (always wrap Java calls in `Either JException T`) explicitly creates an FFI monad layer — this *is* the trap unless the projection (Decision 6: faithful generics) and the importer (Decision 4: elaborator-reflection) make the wrapping **invisible at call sites**. If every Spring API call requires manual `Either` plumbing, Palisade dies the Frege death.

Source: [Frege: a Haskell-like Language for the JVM (InfoQ)](https://www.infoq.com/news/2015/08/frege-haskell-for-jvm/), [Frege, a JVM Haskell (taylor.fausak.me)](https://taylor.fausak.me/2015/06/25/frege-a-jvm-haskell/)

### Cross-cutting historical lesson: tooling > language

Clojure succeeded on JVM by **investing in tooling** (clojure-lsp, clj-kondo, CIDER, babashka). Scala 3 needed a tooling working group founded in 2018 to close the IDE gap. Groovy struggles outside of Gradle/CI niches because of tooling weakness. **Kotlin's success was tooling (JetBrains) + Google blessing, not language design.**

PROJECT.md addresses this correctly with Decisions 11 (LSP-based, IDE-agnostic), 25 (JUnit `TestEngine`), 26 (JaCoCo lift), 28 (JMH benchmarks), 29 (Diataxis docs), 30 (hybrid-module first-class), 31 (`palisade spring`), 32 (REPL-attaches-to-running-JVM). **The risk is execution discipline: if any one of these slips to "later," Palisade looks like Frege.**

Source: [InfoQ: Java Champion James Ward on State of Java and JVM Languages](https://www.infoq.com/articles/james-ward-java-jvm-languages/)

---

## Critical Pitfalls

### Pitfall 1: Stack Map Frame Generation Bugs Producing `VerifyError` at Runtime

**What goes wrong:**
ASM's `COMPUTE_FRAMES` flag silently produces *invalid* stack map frames in non-trivial control-flow scenarios — particularly trampoline-induced jumps, exception handlers that overlap with branch targets, and code shapes generated for dependent-type erasure. Class loads but JVM verifier rejects with `java.lang.VerifyError: Inconsistent stackmap frames at branch target N` or `Expecting a stackmap frame at branch target N`. **The error often surfaces only when a downstream agent (JaCoCo, JMC Agent, Spring AOT, OpenTelemetry instrumentation) re-walks the bytecode** — meaning Palisade-emitted classes can pass `-Xverify:all` in isolation and *still* fail in production environments that use any standard observability agent.

**Why it happens:**
- ASM `COMPUTE_FRAMES` performs dataflow analysis that is correct for *Java-shaped* bytecode but has documented edge cases (issue ow2/asm #317986) where it produces semantically wrong frames for: (1) trampolining into exception-protected regions, (2) constructor-chain super-calls with intervening computations, (3) `aload_0`-after-`new`-before-`<init>` patterns common in functional codegen, (4) `dup_x2` / `swap` patterns used to thread environments through pattern-matching trees.
- `-Xverify:all` runs the *type-checker* verifier; downstream agents like JaCoCo invoke the *type-inference* verifier (split verification) which is stricter. Spock 2.4-M5 hit exactly this (Spock issue #2080).
- Idris codegen will exercise these edge cases more than Java compilers do, because dependent-type erasure leaves "dead" computations behind that ASM treats as live.

**How to avoid:**
1. Build a **secondary frame validator** that replays every emitted method with the JEP 484 ClassFile API as an independent oracle. Two-implementation cross-check — if ASM and the standard ClassFile API disagree, the class is rejected.
2. Run `-Xverify:all` AND a JaCoCo round-trip AND a Byte-Buddy round-trip in CI. Decision 27's "Zero-Tolerance Verification Contract" must include the *downstream-agent compatibility* dimension, not just `-Xverify:all`.
3. For trampolining (Decision 2), *never* let the trampoline target sit inside a `try`/`catch` region with cross-region jumps — emit trampoline dispatch *outside* exception scopes and use thunks for exception-bound paths.
4. Maintain a regression corpus: every `VerifyError` ever discovered is preserved as a minimal `.idr` reproducer + golden bytecode.

**Warning signs:**
- Any `VerifyError` from CI on emitted output (treat as P0 build break).
- Compiled classes pass `-Xverify:all` but fail when loaded via `java -javaagent:jacoco.jar`.
- Frame size oscillates between builds (sign of nondeterministic dataflow analysis).
- The phrase "works without `-javaagent`" in any bug report.

**Phase to address:**
Phase 1 (codegen foundation) — establish the dual-verifier CI gate before any complex emission. Re-validated in every phase that touches codegen (LIMM emission, trampolining, exception-totality bridge, ScopedValue emission).

**PROJECT.md Decisions affected:** **Decision 27** (Zero-Tolerance Verification Contract — must explicitly include downstream-agent compatibility), **Decision 2** (tail-call trampolining), **Decision 39** (ASM choice — implies maintaining compensating validation infrastructure), **Decision 8** (FFI exception wrapping creates exception-handler density that exercises ASM edge cases).

Sources: [JaCoCo issue #1009](https://github.com/jacoco/jacoco/issues/1009), [ASM issue #317986](https://gitlab.ow2.org/asm/asm/-/issues/317986), [Spock #2080](https://github.com/spockframework/spock/issues/2080), [Eclipse 545567](https://bugs.eclipse.org/bugs/show_bug.cgi?id=545567), [JEP 484 design philosophy on detail-hiding](https://openjdk.org/jeps/484)

---

### Pitfall 2: The Defensive Membrane Has Holes — Linear Types Aren't Actually Linear at the FFI Boundary

**What goes wrong:**
Decision 20 promises "Guard Proxies for linear types at Java boundary, with atomic consumed-bit + `PalisadeLinearityViolation`." The naive failure mode: Java code receives a `LinearFuture<T>`, the proxy enforces single-consumption *for that reference*, but Java code can:
1. Capture the proxy in a lambda that is then called twice (proxy sees one access on first call, throws on second — *but the second caller is now in a Java stack frame, so the diagnostic is useless*).
2. Pass the proxy through reflection (`MethodHandle.invokeWithArguments`) bypassing the dispatch path.
3. Serialize/deserialize the proxy, producing two "originals" with independent consumed-bits.
4. Hold the proxy across a JVM-level `clone()`.
5. Pass the proxy across an `invokedynamic` callsite where the bootstrap method materializes a fresh handle.

The bigger structural problem: **linearity in QTT is a *static* property at type-checking time, but the Defensive Membrane re-implements it as a *dynamic* property at the JVM boundary.** A static proof + a dynamic check is not the same guarantee — it's a runtime assertion that *something Idris already proved* is being preserved. If the dynamic check has any blind spot (and the list above is non-exhaustive), the static guarantee is silently invalidated.

**Why it happens:**
JVM has no notion of "use exactly once." Reference identity is preserved across casts, generics erasure, reflection, serialization, lambda capture, and `MethodHandle` adaptation. Every one of those mechanisms is a hole in the membrane. Worse, **the membrane is invisible to Java developers** — nothing in the Java type system warns them they're holding a linear resource.

**How to avoid:**
1. Treat the membrane as **defense in depth, not a primary guarantee.** Static QTT enforcement at the Idris-side call site is the actual proof; the membrane is a runtime tripwire.
2. Make linearity violation **fail loudly with full Idris source context** via Decision 21's source maps (so the post-mortem points back at Idris code, even if the violator is a Java caller).
3. Encode a `@Linear` JVM annotation (Decision 15: linearity markers retained as JVM attributes) and ship a SonarQube rule + Error Prone check that warns Java callers of linear methods.
4. Disallow `Serializable`/`Cloneable` on `LinearFuture` and any guard-proxy class. Override `writeReplace`/`readResolve` to throw.
5. Document explicitly in the FFI contract: "passing a linear value into reflection erases its linearity guarantee." Make this a documented capability boundary, not a silent failure.
6. Specifically test the membrane against the failure modes above (lambda capture replay, reflection invocation, serialization round-trip, `invokedynamic` adaptation).

**Warning signs:**
- Any `PalisadeLinearityViolation` thrown in production from a Java stack frame (means a Java caller has a linear reference; investigate the FFI contract).
- "It compiled in Idris but the runtime threw `PalisadeLinearityViolation`" — means the membrane caught a real bug *or* the static analysis missed something.
- Reflection-using libraries (Spring AOP, Hibernate proxies, Mockito) interacting with linear types in tests.
- Any need to document a "don't do this" workaround for Java callers.

**Phase to address:**
Phase 2 or 3 (after basic codegen, before Spring integration) — design the membrane with adversarial tests *first*, then integrate with FFI. Re-validate in the Spring/AOP integration phase (Spring proxies are the most likely violator).

**PROJECT.md Decisions affected:** **Decision 20** (Defensive Membrane is the central thesis here), **Decision 15** (Hybrid Metadata Erasure — linearity-as-JVM-attribute is the lever), **Decision 17** (LIMM relies on linearity proofs being trustworthy), **Decision 18** (Structured Linear Runtime — virtual thread lifecycle is bound to linear futures, so a membrane hole is a thread-leak), **Decision 5** (Spring DI on Idris classes — Spring's reflection/proxy machinery is the membrane's stress test).

Sources: [Tweag — Safe memory management in inline-java using linear types](https://www.tweag.io/blog/2020-02-06-safe-inline-java/) (precedent for linear types at JVM FFI), [ACM — Safe-by-default Concurrency](https://dl.acm.org/doi/fullHtml/10.1145/3462206)

---

### Pitfall 3: Virtual Thread Pinning in Native Calls and Class Initialization (Loom Footgun)

**What goes wrong:**
JEP 491 in JDK 24/25 fixed the synchronized-monitor pinning footgun, but **two classes of pinning remain**:

1. **Native code** holding stack-allocated pointers (any FFI to JNI/Panama). Decision 4 (Clojure-grade FFI) plus Decision 18 (Loom-bound LinearFutures) means *every* FFI call risks pinning the carrier thread holding a virtual thread. With max 256 carriers and many Idris programs being FFI-dense, **257 concurrent pinning calls deadlock the entire VM scheduler.**
2. **Class loading and initialization** happens via native code, so virtual threads are pinned during `<clinit>`. Idris codegen produces *many* small classes (one per ADT, one per closure, etc.). First-load surge during cold start can pin the carrier pool.

**Why it happens:**
Loom's design: virtual threads must be unmounted to free carriers. JNI stack frames pin because their pointers can't survive a stack copy. `<clinit>` runs during native class-resolution code paths.

**How to avoid:**
1. **Map all FFI calls to a bounded blocking-pool dispatcher** — Idris-side `LinearFuture` materializes work through `BlockingQueue`-fed platform threads when the call site is known-blocking. The compiler can mark FFI signatures as blocking/non-blocking via Decision 4's importer.
2. Aggressive class **eager-loading at startup** — emit `module-info.class` that pre-resolves Idris-internal classes (Decision 40); use `-XX:+EagerInitialize` or generate a startup `Class.forName` warm-up sequence emitted into `main`.
3. Make **`jdk.VirtualThreadPinned` JFR events a CI gate** in the benchmark suite (Decision 28) — any pinning over a threshold fails the build.
4. Set `jdk.virtualThreadScheduler.maxPoolSize` explicitly in the `palisade spring` template (Decision 31) — defaults are insufficient for FFI-dense workloads.
5. Document FFI call-site annotations: `@Blocking`, `@NonBlocking`, `@MayPin` as part of Decision 4's importer output.

**Warning signs:**
- `jdk.VirtualThreadPinned` events in JFR exceed ~100/min under load.
- Latency P99 collapses while throughput plateaus (carrier-pool exhaustion signature).
- Cold-start P99 latency spike that disappears after warm-up (class-init pinning).
- Tests pass, production deadlocks under concurrency.

**Phase to address:**
Phase 4-5 (Loom/concurrency integration). The FFI importer (Decision 4) phase needs to surface blocking/non-blocking signatures *first*; the LinearFuture/Loom integration phase consumes that metadata.

**PROJECT.md Decisions affected:** **Decision 18** (Structured Linear Runtime), **Decision 4** (FFI importer must annotate blocking-ness), **Decision 22** (JFR events — `LinearFutureCompleted` should *exclude* time spent pinned, otherwise metrics lie), **Decision 28** (benchmarks must include pinning detection), **Decision 31** (template config), **Decision 40** (JPMS pre-resolution).

Sources: [How to solve the pinning problem in Java virtual threads (TheServerSide)](https://www.theserverside.com/tip/How-to-solve-the-pinning-problem-in-Java-virtual-threads), [The Carrier Pinning Trap (Azguards)](https://azguards.com/distributed-systems/the-carrier-pinning-trap-diagnosing-virtual-thread-starvation-in-spring-boot-3-migrations/), [JEP 491](https://openjdk.org/jeps/491), [Java 24 Stops Pinning Virtual Threads (nipafx)](https://nipafx.dev/inside-java-newscast-80/)

---

### Pitfall 4: GraalVM Native-Image Reachability Metadata Gaps Discovered Only in Production

**What goes wrong:**
The Spring Boot + GraalVM workflow is: AOT-process at build, generate reachability hints, build native image. Failure modes:
1. **Code path executed only under load** uses reflection → no hint generated → image builds, runtime throws `ClassNotFoundException` weeks after deployment.
2. **Idris-emitted classes** don't carry the same annotations Spring AOT scans for, so Spring's hint generator misses them entirely. The Spring Context Indexer issue with Kotlin/KAPT (Spring Boot #28046) is the exact pattern: tooling that works for Java doesn't transparently work for "another JVM language."
3. **Linearity guards** use atomic operations that GraalVM's static analysis flags as runtime-reflective (because `AtomicReferenceFieldUpdater` is involved).
4. **JFR events** (Decision 22) use `EventFactory` reflection at runtime — this is a known GraalVM hostile pattern and requires explicit metadata.
5. Spring 4.0.0-M1 + Java 24 + GraalVM has logged an active issue (`graalvm-reachability-metadata` #655) showing this is an *ongoing* moving target.

**Why it happens:**
Native-image static analysis is a *whole-program* analysis from a closed-world assumption. Anything dynamic (reflection, dynamic proxies, serialization, MethodHandle, JNI registration) requires explicit metadata. Idris codegen will produce code shapes the Spring AOT engine has never seen.

**How to avoid:**
1. Emit GraalVM `reachability-metadata.json` directly from Palisade codegen — every emitted class is registered with its reflection/proxy/serialization needs. Don't rely on the agent-tracing approach (which is incomplete by definition).
2. Build the **`palisade spring` template (Decision 31) with native-image as a first-class CI target**, not an afterthought. If `./gradlew nativeCompile && ./build/native/nativeCompile/app` doesn't run, the template is broken.
3. Decision 7 says "first-class deployment target; codegen emits AOT-friendly metadata" — *this must be design-from-day-one, not retrofitted*. Retrofitting GraalVM compatibility is a documented graveyard for JVM languages and frameworks.
4. Have a "reachability completeness" test corpus: every Idris construct (ADTs, type classes, traits, dependent erasure, linear values) has a native-image smoke test.
5. Disable `EventFactory` runtime reflection — pre-generate JFR event classes at compile time.

**Warning signs:**
- `ClassNotFoundException` / `NoSuchFieldException` / `NoSuchMethodException` appearing in native-image runs.
- "Works on JVM, fails on native" bug reports.
- Native-image build size growing without obvious cause (over-broad reachability config).
- Slow cold-start on native (slower than JVM warm) — usually means reflection metadata is forcing fallback paths.

**Phase to address:**
Phase 1 codegen foundation must include reachability-metadata emission. Phase for Spring integration must include native-image CI from day one. **Do not defer this** — every JVM language that deferred GraalVM compat paid for it (e.g. Kotlin Spring Boot 4.0.0-M1 is actively fighting this in 2026).

**PROJECT.md Decisions affected:** **Decision 7** (GraalVM first-class), **Decision 5** (Spring DI on Idris bytecode — Spring's AOT engine must see Idris beans), **Decision 22** (JFR — runtime reflection is GraalVM-hostile), **Decision 15** (Ghost Annotations need GraalVM hints), **Decision 31** (template), **Decision 6** (Java generics — type metadata for native is more complex than JVM).

Sources: [Reachability Metadata (graalvm.org)](https://www.graalvm.org/latest/reference-manual/native-image/metadata/), [graalvm-reachability-metadata #655](https://github.com/oracle/graalvm-reachability-metadata/issues/655), [Spring Boot AOT docs](https://docs.spring.io/spring-boot/reference/packaging/aot.html), [Spring Boot #28046](https://github.com/spring-projects/spring-boot/issues/28046), [Spring native intro (Baeldung)](https://www.baeldung.com/spring-native-intro)

---

### Pitfall 5: Spring Component Discovery Misses Idris Beans (the Kotlin/KAPT Repeat)

**What goes wrong:**
Spring's component scan looks for `@Component` (and stereotype variants) at *bytecode* level via classpath scanning. If Idris-emitted classes don't carry the annotation in the *exactly correct* bytecode format Spring expects (including parameter annotations on constructors, retention policies, and `@AliasFor` resolution), Spring silently doesn't see the bean and the application starts up missing wiring. Specifically: **Spring Context Indexer reads `META-INF/spring.components` for fast startup**, and if Idris emits beans but doesn't update this file (because Maven/Gradle's `spring-context-indexer` annotation processor only runs against `javac` source), you get a *partial index* — Spring sees the index file, assumes it's authoritative, *skips classpath scanning entirely*, and Idris beans vanish from DI.

This is the exact failure mode Kotlin had with KAPT and `spring-context-indexer` (Spring Boot #28046) — and the diagnostic looks like "the bean is on the classpath but Spring doesn't autowire it." A maddening multi-hour debug.

**Why it happens:**
Spring's tooling ecosystem assumes Java source. Annotation processors (`javac`'s `Processor` SPI) run during `javac`. Anything not going through `javac` is invisible to them. KSP (Kotlin Symbol Processing) was Kotlin's solution; Palisade has no equivalent.

**How to avoid:**
1. Palisade's build plugins (Decision 10) **must directly emit the entries `spring-context-indexer` would have emitted** — `META-INF/spring.components`, `META-INF/spring.factories`, `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`, etc.
2. **Emit annotations into bytecode in the byte-exact format Spring expects** — including `RuntimeVisibleAnnotations`, `RuntimeInvisibleAnnotations`, `RuntimeVisibleParameterAnnotations`, `AnnotationDefault` attributes. Test with Spring's actual annotation reader (`AnnotationMetadataReadingVisitor`).
3. **Hybrid-module test (Decision 30): an Idris bean autowired into a Java service and a Java bean autowired into Idris code, with `@Profile`, `@Conditional`, `@Lazy`, and `@Primary` all exercised.** This is the membrane between Decision 5 and reality.
4. Smoke-test Spring AOT processing on a hybrid module — the AOT engine must traverse Idris-emitted classes correctly.
5. Cross-check `spring.components` index file against actual classpath scan results in CI; mismatches fail the build.

**Warning signs:**
- "Spring isn't injecting my Idris bean" tickets.
- `NoSuchBeanDefinitionException` for a class that exists on the classpath.
- Beans visible in `application.run()` debug output but missing from autowiring.
- Spring AOT-generated source code has fewer classes than expected.
- `spring.components` file size does not include Idris beans.

**Phase to address:**
Phase for Spring integration (whichever phase introduces Decision 5/30). Cannot be retrofitted cleanly — annotation emission must be designed into codegen from the start of bytecode generation.

**PROJECT.md Decisions affected:** **Decision 5** (Spring DI), **Decision 10** (build plugins), **Decision 30** (hybrid-module), **Decision 31** (template — first thing the template proves is that an Idris bean autowires).

Sources: [Spring Boot #28046 (KAPT issue)](https://github.com/spring-projects/spring-boot/issues/28046), [Spring Classpath Scanning docs](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html), [Annotation Processing best practices (kt.academy)](https://kt.academy/article/ak-annotation-processing)

---

### Pitfall 6: OpenTelemetry Context Lost Across LinearFuture / Virtual Thread Boundaries

**What goes wrong:**
Decision 24 promises "compiler enforces span open/close across `LinearFuture` lifecycle (solves 'lost trace' in async)." OpenTelemetry's Java SDK uses `ThreadLocal` for context storage. **`ThreadLocal` is not inherited across virtual thread creation** (open-telemetry/opentelemetry-java-instrumentation #11950). Net result: spawn a virtual thread inside a `LinearFuture`, the child sees a fresh empty context, the trace splits into two unrelated traces. The compiler-enforced span lifecycle is then enforcing an invariant on a context that is silently being lost.

**Why it happens:**
OTel-Java's design pre-dates Loom. ScopedValues (Decision 19) would solve this but OTel-Java doesn't yet integrate with ScopedValues. The `Thread.startVirtualThread` API does not propagate ThreadLocal because doing so by default would defeat the purpose of virtual threads being cheap.

**How to avoid:**
1. **Wire OTel context into Decision 19's Type-Bound `ScopedValues`** — Idris's effect-system span tracking uses ScopedValue, not ThreadLocal. Bridge to OTel's API by writing the active context into both ScopedValue *and* OTel's storage at span open, and clearing both at span close.
2. **Compiler-enforced context capture at LinearFuture creation** — when codegen emits a virtual-thread spawn, it also emits the context-capture call. Make this part of the `StructuredTaskScope` formalization (Decision 19).
3. **Test:** every `palisade spring` template trace must show a single connected trace across (controller → linear future → DB call → response), zero orphaned spans. CI gate on this.
4. Provide a `Palisade-OpenTelemetry` integration module that overrides `ContextStorage` to use ScopedValue rather than ThreadLocal.

**Warning signs:**
- Traces in Jaeger/Tempo showing "child" spans appearing as roots.
- Span counts higher than expected per request.
- `LinearFutureCompleted` JFR events without correlated OTel span IDs.
- Distributed tracing breaking specifically at async boundaries.

**Phase to address:**
Phase for Loom/concurrency integration (after Decisions 17/18/19 are landed). The OTel bridge is enabled by ScopedValues, so Decision 19's formalization gates this.

**PROJECT.md Decisions affected:** **Decision 24** (Observability-as-Proof — directly threatened), **Decision 19** (ScopedValues — the structural fix lives here), **Decision 18** (Structured Linear Runtime — context propagation must follow lifecycle).

Sources: [Propagating OpenTelemetry context with Virtual Threads (Softwaremill)](https://softwaremill.com/propagating-opentelemetry-context-when-using-virtual-threads-and-structured-concurrency/), [opentelemetry-java-instrumentation #11950](https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/11950), [OTel Java context propagation discussion](https://github.com/open-telemetry/opentelemetry-java/discussions/2884)

---

### Pitfall 7: JSR-45 Source Maps Look Right But Tooling Ignores Them

**What goes wrong:**
Decision 21 promises Idris file:line in stack traces, profilers, heap dumps, JFR events. JSR-45's `SourceDebugExtension` attribute is a *standard*, but tooling support is patchy:
- IntelliJ requires the **Ultimate edition** JSR-45 plugin for full debugger support; community edition cannot follow Idris source through SMAP-mapped bytecode.
- The base `LineNumberTable` (which IntelliJ Community uses) maps to *one* source file — if you populate it with the Idris file, decompiled-Java view in IntelliJ Community shows wrong line numbers; if you populate it with synthetic Java line numbers, the profiler shows wrong lines.
- Async profilers (async-profiler, JFR method profiling) read `LineNumberTable` and ignore SMAP entirely.
- Heap dumps via VisualVM/MAT show only `LocalVariableTable` and class names; SMAP isn't read.
- Even when SMAP is correctly emitted, IDE documentation is, per JetBrains' own forums, "quasi non-existent."

**Why it happens:**
JSR-45 was designed for JSP. Outside the JSP ecosystem it has limited tooling investment. Most profilers and heap analyzers were written before SMAP was standardized.

**How to avoid:**
1. Populate **both** `LineNumberTable` (with the *best single source for that class*) and SMAP (with the multi-source mapping). Decision 21 explicitly says "both" — preserve this.
2. Generate the Idris source file *as the primary* in `LineNumberTable` (so async profilers see Idris lines), with SMAP mapping back to original Idris column/row plus any embedded Java/macro origin.
3. Build a **Palisade-aware async-profiler patch or wrapper** (or contribute SMAP support upstream to async-profiler — likely a lift but high-value).
4. **Test the full debugging path in CI** — set a breakpoint in an Idris file via JDWP from a script, verify it triggers. Don't trust documentation; verify the artifact.
5. Decision 23's "Semantic Naming with ADT Reconstruction" partially compensates — at least class/method names are human-readable even if line mapping fails. Make this redundancy explicit.
6. Provide a "Palisade Stack Trace Decoder" CLI tool that takes raw stack traces (as profilers/agents emit them) and produces Idris-source-located ones via SMAP lookup. This makes the contract robust to incomplete tool support.

**Warning signs:**
- Profiler output showing `Idris$Compiled$0x123` rather than `MyModule.someFunction`.
- IDE breakpoints not triggering on Idris source.
- Heap dump tools showing ADTs as opaque `Object[]` arrays.
- Stack traces in production logs showing JVM-internal classes rather than Idris source.
- Customer reports of "I can't debug this in IntelliJ Community."

**Phase to address:**
Phase for codegen foundation — populate `LineNumberTable` and `LocalVariableTable` from day one. JSR-45 SMAP and Decision 23 ADT reconstruction in a slightly later phase. **Build the stack-trace decoder CLI by v1** — it's the safety net.

**PROJECT.md Decisions affected:** **Decision 21** (Telemetry-Native Mapping — directly), **Decision 23** (Semantic Naming — the redundancy layer), **Decision 32** (dev loop — REPL must show Idris source on errors).

Sources: [JetBrains JSR 45 Support thread](https://intellij-support.jetbrains.com/hc/en-us/community/posts/206340179-JSR-45-Support), [Haxe JSR-45 request issue #10111](https://github.com/HaxeFoundation/haxe/issues/10111), [JaCoCo SourceDebugExtension HowTo #293](https://github.com/jacoco/jacoco/issues/293)

---

### Pitfall 8: JFR Custom Events at High Cardinality Hit the Throttling Wall (or Worse, Don't)

**What goes wrong:**
Decision 22 emits Idris-vocabulary events: `LinearFutureCompleted`, `EffectPerformed`, `LinearityViolationDetected`, `OrderedContainerAccess`. **`EffectPerformed` and `OrderedContainerAccess` are inherently high-cardinality** — they fire potentially every method call. Two failure modes:
1. JFR overhead exceeds the 1-2% target. JFR aims for <1% but high-frequency instant events without throttling can blow this. JDK-8257602 introduced JFR event throttling specifically because high-frequency unregulated events were a problem.
2. JFR recordings grow unbounded — multi-GB JFR files in a few minutes of trading load.
3. Category metadata is wrong (`@Category(...)` paths conflict, get sorted alphabetically by JMC into wrong folders, or use spaces that JMC strips).

**Why it happens:**
JFR was designed for sampled or rare events (GC, allocation samples). Adding a custom event for *every method call* (which `EffectPerformed` essentially is, if effects are ubiquitous) violates the intended usage. Even with `@Period`, instant events at high frequency overwhelm the per-thread JFR buffer.

**How to avoid:**
1. **Tier JFR events by frequency:**
   - Always-on (low-frequency): `LinearityViolationDetected`, `LinearFutureCompleted` (only on completion, not creation), `EffectStreamStart`/`End` (effect transactions, not individual effects).
   - On-demand (high-frequency, off by default): `EffectPerformed`, `OrderedContainerAccess`. Enabled via `-XX:StartFlightRecording=settings=palisade-detail.jfc`.
2. **Use JFR throttling** (`@Throttle("100/s")` for high-frequency events).
3. Standardize the `@Category` taxonomy — document it as a public contract; never break categories between versions (Decision 34 versioning includes JFR vocabulary).
4. Establish a JFR overhead budget in the CI benchmark suite (Decision 28) — running with all Palisade events enabled must add <2% overhead vs running without.
5. Test JFR file growth rate under load — fail if >100MB/min at default settings.
6. Provide JMC plugin or templates so JFR events render correctly in Mission Control.

**Warning signs:**
- JFR-enabled benchmark runs slower than non-JFR by >2%.
- JFR files growing faster than expected in production.
- JMC unable to open Palisade JFR files due to size.
- JFR recording corruption (lost events on overflow).
- Category labels showing as raw strings rather than tree structure in JMC.

**Phase to address:**
Phase for telemetry/JFR integration. Define the event taxonomy + throttling tiers *before* emitting events from codegen. The "transparent box" promise depends on JFR being trustworthy under load.

**PROJECT.md Decisions affected:** **Decision 22** (JFR vocabulary — directly), **Decision 24** (Observability-as-Proof — overhead budget), **Decision 28** (benchmarks — must include JFR-on overhead), **Decision 34** (versioning — JFR vocabulary becomes public API).

Sources: [Custom JDK Flight Recorder Events (Inside.java)](https://inside.java/2022/04/25/sip48/), [JDK-8257602 (JFR throttling)](https://bugs.openjdk.org/browse/JDK-8257602), [Monitoring REST APIs with Custom JFR Events (morling.dev)](https://www.morling.dev/blog/rest-api-monitoring-with-custom-jdk-flight-recorder-events/)

---

### Pitfall 9: Trampoline-Based Tail Calls Blow Up Stack Traces and Break Profilers

**What goes wrong:**
Decision 2: "direct self-recursion → JVM loop; mutual → trampoline." Trampolines work, but they introduce systematic problems:
1. **Stack traces from inside trampolined functions show the trampoline frame, not the logical caller.** Idris programmers see `Trampoline.run(...)` instead of their own recursive call chain. Decision 21's source maps don't help — the stack frame is genuinely synthetic.
2. **Profilers attribute time to the trampoline runner, not to the called function.** This makes performance analysis of trampolined code essentially impossible.
3. **Exceptions thrown inside trampolined calls have stack traces that have been "compressed" — intermediate calls are erased.** Recovery context is lost.
4. **Trampolines defeat JIT inlining.** The HotSpot JIT cannot inline through a trampoline dispatch loop, so what was logically a tight tail-call loop becomes a megamorphic dispatch site.
5. **`recur`-equivalent (Clojure's solution) requires the call to be *syntactically* in tail position, but Idris's `let` and `case` desugar in ways that may not preserve tail position obviously.** A function the programmer thinks is tail-recursive may not be.
6. **Trampolined code that throws inside `try`/`catch` interacts badly** — Clojure documents that you can't use `recur` inside `try`. Trampolines have a related issue: the exception unwinds to the trampoline frame, not the logical try.

**Why it happens:**
JVM lacks general TCO. Every workaround (trampolines, `recur`, MethodHandle adaptation) has known issues; the question is which set of issues is acceptable.

**How to avoid:**
1. **Prefer self-loop → JVM-loop (Decision 2's first leg) aggressively.** Detect tail-call self-recursion in NamedCExp and turn it into a `goto` emit. Reserve trampolines for genuinely mutual recursion.
2. **Decision 16's `@hot` pragma should also imply "monomorphize and inline through the trampoline" via specialized direct-dispatch bytecode emission** — no trampoline for hot paths.
3. Emit a **synthetic Idris-aware stack trace decoder** (Decision 21 redundancy with Pitfall 7) that reconstructs logical recursion from trampoline frame data stored in the trampoline's local state.
4. Test trampoline correctness against `try`/`catch`/`finally` — if any case is unsupported, *fail to compile*, don't silently miscompile.
5. Add a pragma `@no-trampoline` that fails compilation if mutual recursion would require trampolining (forces programmer to refactor).
6. JFR event for trampoline iterations to make trampoline-heavy paths visible in production.

**Warning signs:**
- Stack overflow in mutual-recursion code that "should be" tail-call optimized.
- Profiler hot-spots showing `Trampoline.run` rather than user functions.
- JIT deopt events near trampoline dispatch (use `-XX:+PrintCompilation` or JFR).
- "I can't tell why this function is slow" reports from users.

**Phase to address:**
Phase for codegen foundation (initial NamedCExp → bytecode lowering must implement Decision 2 correctly from the start — retrofit is painful).

**PROJECT.md Decisions affected:** **Decision 2** (tail calls — directly), **Decision 16** (JIT shape — trampolines defeat inline-cache assumptions), **Decision 21** (source maps — trampolines defeat naive line attribution), **Decision 27** (verification — trampolines exercise StackMapFrame edge cases per Pitfall 1).

Sources: [On Recursion, Continuations and Trampolines (Eli Bendersky)](https://eli.thegreenplace.net/2017/on-recursion-continuations-and-trampolines/), [PurelyFunctional.tv: Tip: trampoline your tail recursion](https://ericnormand.me/issues/purelyfunctional-tv-newsletter-361-tip-trampoline-your-tail-recursion), [Clojure problems with the JVM (Eric Normand)](https://ericnormand.me/article/problems-with-the-jvm), [Bytecode generation for tailrec methods uses temporary variables (scala/scala3 #14773)](https://github.com/lampepfl/dotty/issues/14773)

---

### Pitfall 10: Hybrid Metadata Erasure Fails the External Verifier

**What goes wrong:**
Decision 15 says erased-arg metadata is preserved at FFI boundaries via "Ghost Annotations" and linearity markers retained as JVM attributes "for external static analysis (SonarQube, custom JPMC linters)." Two failure modes:
1. **Ghost Annotations are emitted but their semantics are not documented as a stable contract** — SonarQube rules and custom linters break across Palisade versions because the annotation schema changes.
2. **Aggressive internal erasure leaves "ghost" metadata only at FFI boundaries — but Spring AOP, Hibernate, AspectJ, and Mockito *all* synthesize subclasses dynamically, and those synthesized classes don't carry the ghost annotations.** External analyzers see Spring proxies of Idris classes and miss the linearity/totality guarantees.
3. **Linearity-as-JVM-attribute** uses custom `Attribute` subclasses; ASM `ClassReader` strips unknown attributes by default (`SKIP_FRAMES | SKIP_DEBUG | SKIP_CODE` flags). Anything that re-reads the class with default ASM settings *destroys* the linearity attribute.

**Why it happens:**
Custom JVM attributes are spec-allowed but tooling-hostile. The JVM verifier ignores unknown attributes (good), but the rest of the toolchain often strips them (bad). Spring/Hibernate/Mockito generate runtime subclasses without preserving custom metadata.

**How to avoid:**
1. **Publish the Ghost Annotation schema as part of Decision 34 versioning** — semver applies to the annotation contract.
2. Use *standard JVM annotations* (`@Retention(RUNTIME)`, in a documented package like `palisade.runtime.annotations`) rather than custom `Attribute` subclasses — annotations survive ASM round-trips that strip custom attributes.
3. Provide a Spring `BeanPostProcessor` and Mockito `MockMaker` extension that copy Ghost Annotations onto generated proxies/mocks. Document that without these extensions, external static analysis on AOP'd classes is unsupported.
4. Provide a SonarQube rule pack and Error Prone bug pattern as part of v1 deliverables — if Decision 15 promises "external static analysis works," ship the analyzers.
5. Test annotation survival across: ASM round-trip, JaCoCo instrumentation, Spring AOP wrapping, AspectJ weaving, Mockito mocking, GraalVM native-image.

**Warning signs:**
- External linter shows "no linearity info" on Idris-emitted classes after Spring startup.
- Annotations missing from `Class.getAnnotations()` after running through any agent.
- SonarQube rules silently report 0 violations (because they can't find the metadata to check).

**Phase to address:**
Phase that introduces Decision 15 — design the annotation schema *before* committing to a Ghost Annotation format. Re-validate in Spring integration phase.

**PROJECT.md Decisions affected:** **Decision 15** (Hybrid Metadata Erasure — directly), **Decision 5** (Spring DI — proxy generation strips annotations), **Decision 26** (Lifted Semantic Coverage — JaCoCo instrumentation strips annotations), **Decision 34** (versioning).

Sources: [ASM ClassWriter docs](https://asm.ow2.io/javadoc/org/objectweb/asm/ClassWriter.html), [Stop Guessing Your GraalVM Native Image Metadata](https://stevenpg.com/posts/graalvm-native-metadata-from-tests/)

---

### Pitfall 11: NamedCExp Vendor Snapshot Drifts Away From Upstream and Becomes Unmaintainable

**What goes wrong:**
Decision 38: "vendor snapshot in Palisade tree with controlled upstream merges." Standard fork-divergence pattern. Six months in:
1. Upstream Idris fixes a critical bug in NamedCExp lowering. Palisade vendor snapshot doesn't get the fix because the merge would conflict with Palisade-specific changes.
2. Palisade's local NamedCExp diverges in subtle ways from what `idris2 --check` produces. Code that passes `idris2` typecheck fails Palisade compilation (or worse, miscompiles silently).
3. Decision 12's "tagged release + periodic rebase" requires *actually doing the rebases*. Personal-project bus factor.
4. Idris core team makes a backwards-incompatible IR change. Palisade has to choose: skip the upgrade (stay frozen), or do a heroic rebase (months of work).

**Why it happens:**
Vendoring is the right call for stability. But vendoring without a disciplined sync process is the path to abandonment. Every fork that died (idris-jvm-hs, original Eta) had a related dynamic.

**How to avoid:**
1. **Automate the vendor sync** — CI job that monthly attempts to merge upstream Idris HEAD into the vendor snapshot, opens a PR with conflicts annotated, and runs the full Palisade test suite against the merged tree.
2. **Maintain an "extension surface" doc** — every divergence from upstream is documented with rationale. If a divergence becomes unjustifiable, prioritize upstream-ing it.
3. **Test against `idris2 --check` output as oracle** — every NamedCExp Palisade processes is also processed by upstream Idris's other backends (Chez, RefC) and behavior compared. Discrepancy = bug.
4. **Define a "Palisade is N versions behind upstream" SLA** publicly (Decision 34). E.g. "Palisade tracks within 3 Idris releases of latest tagged."
5. **Upstream-first culture** — when something needs to change in NamedCExp, propose it upstream first. Vendor only if upstream rejects.
6. **Publish a compatibility matrix** (Decision 34) and never ship a Palisade release that doesn't pin to a specific Idris commit hash, reproducible from source.

**Warning signs:**
- More than 2 months without a vendor sync.
- Vendor snapshot has files modified that aren't justified by Palisade-specific changes.
- "We can't upgrade Idris because of vendor changes" appears in any planning conversation.
- Palisade test failures that pass under `idris2 --check`.

**Phase to address:**
Phase 1 (vendor snapshot setup) — establish the sync automation *before* the snapshot drifts. Re-validate every release.

**PROJECT.md Decisions affected:** **Decision 38** (NamedCExp vendor snapshot), **Decision 12** (Idris pinning + rebase), **Decision 34** (versioning + compatibility matrix), **Decision 1** (consume NamedCExp directly).

Sources: General fork-maintenance literature; Eta and idris-jvm precedents above.

---

### Pitfall 12: Reproducible Builds Promise Broken by JFR Class Auto-Generation, Annotation Processor Order, or Native-Image Determinism

**What goes wrong:**
Decision 36: "byte-identical JAR." Sounds simple. Killers:
1. **JFR `EventFactory` generates classes at runtime** — not in the JAR — but if Palisade pre-generates JFR event classes at compile time (per Pitfall 4), the generation must be deterministic. Class name collisions if two events have the same name in different packages can produce different classes depending on classpath order.
2. **Annotation processor ordering** — Spring's annotation processors, Lombok-style processors, and Palisade's own emit-time processors interact. Maven plugin execution order is partial; Gradle's task-graph DAG is partial. Different orders can produce different bytecode.
3. **GraalVM native-image is not deterministic by default** — same JAR, two builds, different native images. (Reproducible native-image is a research problem, not a solved one.)
4. **`Map`/`Set` iteration order in codegen** — if Palisade is implemented in Idris, and any IR transformation iterates an unordered collection (which Idris stdlib has — see CONCERNS.md noting "List Name should be Set"), output bytecode order can vary.
5. **Timestamps embedded in class files** (`SourceFile` attribute, `ConstantValue` for compile-time constants like `LocalDate.now()` if used).

**Why it happens:**
Reproducibility requires *all* steps to be deterministic. Any single nondeterministic step (one set iteration, one timestamp, one classpath order) breaks the chain.

**How to avoid:**
1. **CI gate on `diffoscope` of two independently-built JARs from clean checkout** (Decision 36 already says this — make it a hard gate).
2. **Use sorted collections everywhere in Palisade codegen** — never iterate a `Map`/`Set` unsorted. Audit codegen for this.
3. **`SOURCE_DATE_EPOCH` discipline** — pass through the Maven/Gradle build (post-2025 Maven supports this).
4. **Fix annotation processor ordering** — declare explicit dependencies between processors, or run them in a single defined order via a unified Palisade-controlled processor.
5. **Defer reproducible native-image promise to a later phase** — don't promise byte-identical native-image at v1; promise byte-identical JVM JAR.
6. **Test reproducibility on different OSes** (Linux + macOS) and different file system orderings.

**Warning signs:**
- `diffoscope` ever shows differences in CI rebuild check.
- Build output filename or timestamp embedded in class files.
- Inconsistent test failures that disappear on rebuild ("flaky").
- Different file ordering inside JAR between two builds.

**Phase to address:**
Phase 1 (early infrastructure) — establish reproducibility CI gate from day one. Adding it later means hunting down dozens of nondeterminism sources.

**PROJECT.md Decisions affected:** **Decision 36** (reproducible builds — directly), **Decision 7** (GraalVM — native-image reproducibility caveat), **Decision 22** (JFR — pre-gen class determinism), **Decision 35** (security — SLSA provenance depends on reproducibility).

Sources: [Reproducible JVM builds (reproducible-builds.org)](https://reproducible-builds.org/docs/jvm/), [Configuring for Reproducible Builds (Apache Maven)](https://maven.apache.org/guides/mini/guide-reproducible-builds.html), [Canonical Builds and Reproducibility in Java (Java Code Geeks)](https://www.javacodegeeks.com/2025/08/canonical-builds-and-reproducibility-in-java-ensuring-deterministic-artifacts-with-tools-like-chains-rebuild/)

---

### Pitfall 13: JPMS `module-info.class` Auto-Emission Conflicts With Spring/Reflection-Heavy Frameworks

**What goes wrong:**
Decision 40: "auto-emit `module-info.class` per Idris package, derived from `.ipkg` dependencies; modulepath-clean by default." This collides with reality:
1. Spring needs `--add-opens` to access internals via reflection. Auto-emitted `module-info` will not export those packages by default.
2. Hibernate needs reflective access to entity classes; same issue.
3. Mockito needs to break encapsulation to mock final classes.
4. **Split-package conflicts**: if two Idris packages happen to map to the same Java package name (or worse, conflict with a Java/Kotlin/Scala package on the modulepath), JPMS rejects the module graph at startup with a non-actionable error.
5. **Automatic-module-name conflicts** with vendored libraries — the rest of the JVM ecosystem is *still* not fully modular (Wikipedia note: "many libraries are still automatic modules in 2026").

**Why it happens:**
JPMS prohibits split packages absolutely (no negotiation). The "modulepath-clean by default" promise of Decision 40 collides with Spring/Hibernate/AOP requiring `--add-opens` to function.

**How to avoid:**
1. **Default `module-info` to `open` modules** for Idris packages exposed to Spring DI — `open` modules permit reflective access without `--add-opens`. Document the trade-off.
2. **Validate that no two `.ipkg` files produce conflicting Java package names** — fail to compile if they do, with a clear error message.
3. **Provide a "Spring profile" for `module-info` generation** that pre-emits `opens` directives for Spring/Hibernate/Mockito reflection needs.
4. **Test the full Spring Boot startup on the modulepath** (not just classpath) in CI for the `palisade spring` template.
5. **Document the modulepath vs classpath choice for users** — many Spring shops are still classpath-only in 2026; auto-emit modulepath-clean only when the user opts in.
6. Allow per-`.ipkg` `module-info` overrides so users can hand-tune when needed.

**Warning signs:**
- Spring startup failing with `IllegalAccessException`.
- Hibernate "cannot access entity field via reflection" errors.
- "Module X reads more than one module called Y" errors at startup.
- Test that uses Mockito throwing `InaccessibleObjectException`.

**Phase to address:**
Phase for module emission (after basic codegen). Cannot be cleanly retrofitted — module structure is wired into the build system.

**PROJECT.md Decisions affected:** **Decision 40** (JPMS auto-emit — directly), **Decision 5** (Spring DI — needs `opens`), **Decision 25** (test framework — Mockito needs reflection), **Decision 30** (hybrid-module — package name conflicts likely).

Sources: [JPMS Cheatsheet (tfesenko)](https://github.com/tfesenko/Java-Modules-JPMS-CheatSheet/blob/master/README.md), [Handling Split Packages (prgrmmng.com)](https://prgrmmng.com/handling-split-packages-and-avoiding-conflicts-in-modules), [Stephen Colebourne on JPMS automatic modules](https://blog.joda.org/2017/05/java-se-9-jpms-automatic-modules.html)

---

### Pitfall 14: ScopedValue Lifecycle Bugs From Unbounded Scopes or Cross-Scope Capture

**What goes wrong:**
Decision 19's Type-Bound `ScopedValues` and Decision 17's LIMM rely on ScopedValue for safe shared state. ScopedValue was finalized only in JDK 25 (JEP 506) after multiple preview rounds (446 → 464 → 481 → 487). Failure modes:
1. **Scope binding outlives the `ScopedValue.run` block** — typically by capturing the bound value into a closure that escapes the scope. The captured value is now stale (the next `run` rebinds, but the closure holds the old binding).
2. **Cross-thread inheritance** — `StructuredTaskScope` inherits ScopedValues; legacy `ForkJoinPool` does not. If Palisade code mixes both (likely for Spring's existing thread pools), ScopedValue is silently absent in the legacy-thread path.
3. **Idris's effect system materializes ScopedValues lazily** — if effects are erased aggressively (Decision 15), the `ScopedValue.run` call may be elided entirely, and access throws `NoSuchElementException`.
4. **ScopedValue is per-thread** — sharing across virtual threads in the same scope works only via `StructuredTaskScope` inheritance. If Palisade spawns virtual threads via `Thread.startVirtualThread` directly (bypassing `StructuredTaskScope`), inheritance is lost.

**Why it happens:**
ScopedValue is genuinely new — JDK 25 finalization, limited production track record. The lifetime/inheritance semantics are subtle and the API contract is strict.

**How to avoid:**
1. **Compiler-enforced "no escape" check** — Idris codegen verifies that no closure capturing a ScopedValue escapes its `run` block. (This is exactly the kind of static guarantee Idris's QTT can provide.)
2. **Force all virtual-thread spawns through `StructuredTaskScope`** (Decision 18 already implies this) — ban direct `Thread.startVirtualThread` from emitted code.
3. **Test ScopedValue in legacy-thread interop scenarios** — Spring's `@Async`, JDBC connection pools, Tomcat thread pools all use platform threads with `ForkJoinPool`-style semantics. Palisade must either prohibit ScopedValue access from these contexts (compile error) or provide a fallback bridge.
4. **Erasure of `ScopedValue.run` is forbidden** — Decision 15's aggressive erasure must blacklist effect-system constructs that materialize ScopedValue.
5. **JDK version pinning** — Decision 12 says JDK 25+; verify against JDK 25 specifically (where ScopedValue is final), not just preview JDKs.

**Warning signs:**
- `NoSuchElementException` from `ScopedValue.get()`.
- "Stale" data observed via ScopedValue (capture-and-escape bug).
- Behavioral difference between `StructuredTaskScope` and direct `Thread.startVirtualThread`.
- Legacy Spring `@Async` code unable to access Idris ScopedValue context.

**Phase to address:**
Phase for concurrency primitives (Decision 19). After basic Loom integration but before LIMM (Decision 17) which depends on ScopedValue semantics.

**PROJECT.md Decisions affected:** **Decision 19** (concurrency primitives — directly), **Decision 17** (LIMM — depends on ScopedValue), **Decision 18** (Structured Linear Runtime), **Decision 15** (erasure must not eliminate ScopedValue setup).

Sources: [JEP 506: Scoped Values (final)](https://openjdk.org/jeps/506), [JEP 446 (preview)](https://openjdk.org/jeps/446), [JEP 487 (4th preview)](https://openjdk.org/jeps/487), [Looking at Java 21: Scoped Values](https://belief-driven-design.com/looking-at-java-21-scoped-values-a78f1/)

---

### Pitfall 15: Hot-Reload of Idris Classes Breaks LIMM Invariants and Ghost Annotations

**What goes wrong:**
Decision 32 promises "Spring DevTools-style hot reload of Idris classes." Hot reload via `Instrumentation.redefineClasses` has constraints:
1. **Cannot change class shape** (add/remove methods, change signatures, change supertypes). Idris codegen often produces structural changes for what looks like a "small" source change (e.g. adding a constructor to an ADT regenerates the entire ADT class hierarchy).
2. **LIMM's barrier-omission proofs become invalid** if the redefined class changes its memory model. A field that was `@Linear` (no barrier) and becomes shared on reload will be accessed without barriers — *silent data race*.
3. **Ghost Annotations on the old class instance vs the new** — running threads hold old-class instances; new code uses new-class. Cross-class equality and instanceof checks behave unexpectedly.
4. **JFR event class redefinition** is not supported — once registered, JFR event classes are immutable for the JVM lifetime.
5. **Spring DevTools restart-by-default** uses a separate classloader, which is more permissive but has its own gotchas — bean caches, EntityManager, prototype scope all break in ways that vary by Spring version.

**Why it happens:**
JVM hot-reload is shallow by design (security, soundness). Anything beyond method-body changes requires JVMTI tricks or full classloader restart. Verified-types semantics interact badly with shallow reload.

**How to avoid:**
1. **Default to classloader-restart hot-reload** (Spring DevTools-style), not JVMTI redefinition. Faster than full JVM restart, doesn't require shape preservation.
2. **Detect "verification-affecting changes"** in incremental compile — if a change affects LIMM proofs, force restart instead of redefine.
3. **Provide a clear contract: "hot reload preserves source-level type equivalence; structural source changes require restart."** Document.
4. **Test hot reload against LIMM-using code** — verify no race condition can be introduced by reload.
5. **Pre-load all JFR event classes at startup** (Pitfall 8) — never lazy-load.
6. **Spring DevTools integration test** in the `palisade spring` template — first-class CI gate.

**Warning signs:**
- "Hot reload didn't pick up my change" reports.
- Race conditions appearing only after hot reload.
- Spring beans not refreshed after reload.
- `LinkageError` after hot reload.

**Phase to address:**
Late phase (developer experience polish). After core compile and Spring integration are stable — hot reload is a nice-to-have that can break correctness if rushed.

**PROJECT.md Decisions affected:** **Decision 32** (hot reload — directly), **Decision 17** (LIMM — invariant preservation), **Decision 15** (Ghost Annotations — class identity), **Decision 22** (JFR — class lifetime).

Sources: General Spring DevTools and JVMTI documentation (no single canonical source).

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Skip JaCoCo round-trip in `-Xverify:all` gate (Pitfall 1) | Faster CI | `VerifyError` discovered in production by users running observability agents | Never |
| Defer GraalVM native-image CI to "after v1" | Velocity in early phases | Native-image becomes a months-long retrofit; community can't deploy in modern stacks | Never (Decision 7 explicitly forbids this) |
| Emit annotations only in `RuntimeVisibleAnnotations` (skip `RuntimeInvisibleAnnotations`) | Smaller class files | Spring AOT misses some annotations; component scan inconsistent | Never for annotation-using decisions (5, 15, 24, 25) |
| Use `ThreadLocal` for OTel context bridge (Pitfall 6) | Works in JDK 21 with no special handling | Loses context across virtual threads; defeats Decision 24 | MVP demo only; must be ScopedValue by v1 |
| Manually maintain reachability metadata via agent tracing (Pitfall 4) | Initial setup faster | Production gaps discovered by users; trust in native-image collapses | Bootstrap-only; codegen must emit by v1 |
| Skip vendor-snapshot sync for "just one release cycle" (Pitfall 11) | Avoid merge conflicts now | Snapshot diverges, never re-syncs, becomes effectively-forked | Never; weekly automation required |
| Use single `LineNumberTable` only (skip JSR-45 SMAP) (Pitfall 7) | Simpler emit logic | Idris source positions invisible to profilers; debugging defeated | Until phase implementing Decision 21 |
| Trampoline everything (skip self-loop optimization) (Pitfall 9) | Uniform codegen path | Profilers attribute time wrong; JIT inlining defeated; perf 2-3× worse | Never (Decision 2 explicitly demands self-loop) |
| Implement linear membrane only at FFI return values (skip parameter checks) | Faster initial impl | Java callers can pass linear values into reflection bypassing membrane | Never (compromises Decision 20's central thesis) |
| Defer reproducible-build CI to post-v1 | Quicker initial CI | Hunting nondeterminism months later; SLSA L3 attestation impossible | Never (Decision 36 + Decision 35 chain) |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Spring Boot DI | Emit `@Component` annotation but skip `META-INF/spring.components` index | Emit annotation AND update index file from build plugin (Decision 10) |
| Spring AOP / CGLIB proxies | Assume Idris classes can be proxied without changes | `@Linear` classes must be `final` or proxy generation breaks linearity; provide non-proxy alternative |
| Hibernate JPA | Idris ADTs as entities — ORM expects mutable JavaBean shape | Provide an entity-projection layer; do not pretend ADTs are entities |
| Mockito | Mock a `LinearFuture<T>` — mock instance has no consumed-bit semantics | Provide a `PalisadeMockMaker` with linearity-aware mocking; document |
| GraalVM native-image | Trust the agent-traced metadata (Pitfall 4) | Emit metadata from codegen; agent is fallback only |
| OpenTelemetry | Use ThreadLocal-backed context across LinearFutures (Pitfall 6) | ScopedValue-backed bridge integrated with effect system |
| JaCoCo coverage | Run JaCoCo on Idris-emitted bytecode without source-map awareness | JaCoCo + JSR-45 SMAP integration (Decision 26); custom report generator |
| JFR + JMC | Emit high-frequency events without throttling (Pitfall 8) | Event taxonomy with throttle tiers; `@Throttle` annotations |
| Maven lifecycle | Run Palisade compile before/after `kotlinc`/`javac` arbitrarily | Define explicit phase order in plugin (`generate-sources` → `compile` integrating all three) |
| Gradle multi-source-set | Use default sourceSet conventions | Define `idris` source set with explicit dependsOn ordering vs `java`/`kotlin` |
| Spring Boot DevTools | Hot reload Idris classes without checking LIMM safety (Pitfall 15) | Classloader-restart by default; JVMTI redefine only for safe changes |
| JPMS modulepath | Auto-emit `module-info` without testing Spring reflection (Pitfall 13) | Default to `open` modules; Spring profile for `opens` directives |
| Micrometer | Export raw JFR events to Prometheus | Aggregate via Micrometer custom meters; bound cardinality |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Megamorphic dispatch from over-eager `invokedynamic` (Decision 16) | JIT deopt storms in JFR; tight loops 2-5× slower | Decision 16's `@hot`/`@specialize` pragma forces monomorphization; closed-world analysis on link | When >3 implementations of a callsite type exist |
| Carrier thread pinning under load (Pitfall 3) | P99 latency cliff at ~256 concurrent FFI calls; throughput plateau | Bounded blocking-pool dispatch for FFI; eager class init | At ~256 concurrent FFI-blocking operations |
| JFR overhead exceeding budget (Pitfall 8) | Throughput drops 2-5% with JFR enabled | Throttling tiers; default-off for high-frequency events | Trading workloads with millions of effect performances/sec |
| Trampoline defeating JIT inlining (Pitfall 9) | Profiler shows hotspot in trampoline runner | Self-loop optimization for self-recursion; `@hot` for mutual | All tail-recursive code with non-trivial body |
| ZGC pointer overhead vs G1 (Decision 13) | 15-30% higher heap memory consumption | Acceptable for trading-grade pause profiles; document trade-off | When throughput-bound workloads adopt without measuring |
| `BigInteger` boxing on `Integer` arithmetic (Decision 3) | Allocation rate spikes; GC pressure | Decision 3's `Int`/`Double` specialization catches `Integer (small)` cases at compile time | When `Integer` arguments cross polymorphic boundaries |
| Linearity guard atomic CAS contention (Decision 20) | Lock-like behavior on hot linear values | Membrane should be zero-overhead on internal Idris paths (Decision 20 says "signature-based dispatch") — *test this* | If atomic CAS appears on internal Idris-to-Idris hot paths |
| OpenTelemetry span creation overhead | Latency budget consumed by span instrumentation | Sampling at the trace level; no per-effect spans by default | Trading paths where 10μs span overhead matters |
| Class-loading storm at startup (Pitfall 3) | Cold-start P99 spike; slow first-request | Eager-loading via `module-info` + warm-up sequence | Idris-heavy programs with many small ADT classes |
| `EventFactory` reflection during JFR enable | Slow JFR enable/disable cycles | Pre-generate JFR event classes at compile time | If JFR is dynamically enabled/disabled in production |

---

## Security Mistakes

Domain-specific issues beyond OWASP basics. Decision 35 specifies full enterprise process; these are issues *within* that envelope.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Linearity violation as exception (silent in catch-all) | Java code catches `PalisadeLinearityViolation` and continues; bypasses guarantee | Mark exception as `Error` (not `Exception`); document never to catch |
| Reproducible-build CI broken (Pitfall 12) | SLSA L3 attestation unverifiable; supply-chain trust collapses | Hard CI gate; never merge with broken reproducibility |
| Custom JVM attribute used for security claims (Pitfall 10) | Stripped by ASM round-trip; security claim invisible | Use standard runtime annotations, not custom attributes |
| FFI exception bridge swallows exceptions silently (Decision 8 + CompletableFuture pitfall) | Errors lost across async boundary; production debugging impossible | Always `handle()` not `whenComplete()`; structured logging on FFI exceptions |
| Hot reload bypasses LIMM verification (Pitfall 15) | Race condition introduced post-deploy | Detect verification-affecting changes; force restart |
| GraalVM `serialization-config` over-broad | Deserialization gadget chain attack surface | Generate serialization-config from `Serializable` annotations only; not blanket |
| ScopedValue holds secret across scope boundary (Pitfall 14) | Secret captured into long-lived closure; logged inadvertently | Compiler check on closure-escape from ScopedValue scope |
| Idris-emitted class names collide with `java.*` package | Reflection attacks via package squatting | Reserve `palisade.*` namespace; refuse to emit anything in `java.*`/`javax.*`/`jdk.*` |
| JFR events leak sensitive data into recordings | Heap dumps / JFR files contain secrets | Decision 22 vocabulary must specify which fields are PII-safe; opt-in for sensitive fields |
| Vendored Idris snapshot patches not reviewed | Supply-chain risk in vendor sync (Pitfall 11) | Every vendor sync diff goes through code review; CycloneDX SBOM tracks vendor commit hash |
| Native-image `--initialize-at-build-time` initializes secret-loading code | Secrets baked into binary | Enumerate build-time-init classes explicitly; never blanket |

---

## UX Pitfalls

The "users" here are a small set: enterprise JVM engineers (Java/Kotlin/Clojure background) considering Idris on JVM; Idris programmers wanting JVM deployment.

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| First error message references Idris-internal types | Java engineer abandons after 2 minutes | Decision 23 + 21 — error messages in Java/Kotlin vocabulary when crossing FFI |
| `palisade spring` template requires manual config to work (Decision 31) | Adoption blocked by 30-min "make it run" | Template runs `mvn spring-boot:run` → working endpoint with zero edits |
| LSP missing JVM-type hover information (Decision 11) | Idris IDE feels worse than Kotlin IntelliJ | Idris LSP must show JVM-side types on FFI signatures |
| REPL cannot eval against running JVM (Decision 32) | Dev loop slower than Clojure REPL → Idris feels "dead" | nREPL-equivalent with running-JVM attach as v1 must-have |
| Stack traces show JVM-internal frames (Pitfall 7) | "I have no idea where this error came from" | Stack-trace decoder CLI; SMAP-aware tooling |
| First incremental compile slow (no warm cache) | Onboarding feels slow | Document expected first-compile time; speed up cold start in tooling |
| Coverage reports show JVM bytecode coverage (Pitfall 7's cousin) | Coverage gates meaningless for verification claims | Decision 26 lifted coverage; reports show Idris source coverage |
| JFR events not viewable in JMC without plugin | "Where do I see these promised events?" | JMC plugin shipped alongside Palisade; or render in stock JMC categories |
| Hot reload "works" but state corrupts (Pitfall 15) | "I can't trust hot reload, restarting every time" | Default conservative reload mode; clear "must restart" warnings |
| Diataxis docs (Decision 29) heavy on reference, light on tutorials | Engineers can't get started; only reference manual | 5 tutorials *first*, ramp-up complexity; reference fills in |
| Error messages in QTT-speak ("multiplicity 1 expected, 0 found") | JVM engineer doesn't know what QTT means | Tutorial-grade error message variants; link to docs from error |
| Spring annotation on Idris class refused with parser error | Adoption regression vs Java | Idris syntax for Spring annotations must be ergonomic; Decision 4/5 |

---

## "Looks Done But Isn't" Checklist

Verification checks during execution. Especially for end-of-phase sign-offs.

- [ ] **Codegen "complete":** Often missing JaCoCo round-trip verification — verify `mvn test jacoco:report` runs without `VerifyError`.
- [ ] **Codegen "complete":** Often missing GraalVM native-image build — verify `./gradlew nativeCompile && ./build/native/nativeCompile/app` runs.
- [ ] **Spring DI "works":** Often missing `META-INF/spring.components` index — verify `cat target/classes/META-INF/spring.components` lists Idris beans.
- [ ] **Spring DI "works":** Often missing AOT processing — verify `mvn spring-boot:process-aot` succeeds with Idris beans.
- [ ] **Linear types enforced at FFI:** Often missing reflection bypass test — verify `Method.invoke(linearFuture, "consume")` is detected.
- [ ] **Linear types enforced at FFI:** Often missing serialization defense — verify `ObjectOutputStream.writeObject(linearFuture)` throws.
- [ ] **JFR events emit correctly:** Often missing throughput benchmark with events on — verify <2% overhead in JMH suite.
- [ ] **JFR events emit correctly:** Often missing JMC visualization test — verify events render in stock JMC.
- [ ] **OpenTelemetry context propagates:** Often missing virtual-thread test — verify trace stays connected across `LinearFuture.spawn`.
- [ ] **Source maps "work":** Often missing breakpoint test — verify IntelliJ Community can break on Idris source line.
- [ ] **Source maps "work":** Often missing async-profiler test — verify profiler shows Idris function names.
- [ ] **Reproducible build:** Often missing two-host check — verify `diffoscope` agrees on JARs from two CI runners.
- [ ] **Hot reload "works":** Often missing LIMM-changing test — verify race-condition-introducing change triggers restart not redefine.
- [ ] **GraalVM native-image "supported":** Often missing JFR-in-native test — verify JFR events emit from native image.
- [ ] **JPMS modules "clean":** Often missing Spring-on-modulepath test — verify `java -p ...` starts Spring with Idris beans.
- [ ] **Tail calls "work":** Often missing mutual-recursion test through trampoline — verify no stack overflow at depth 1M.
- [ ] **Vendor snapshot "synced":** Often missing weekly sync CI — verify automated PR cadence.
- [ ] **Test framework "integrated":** Often missing Spring TestContext test — verify `@SpringBootTest` works with Idris-defined beans.
- [ ] **Coverage "lifted":** Often missing Idris-source threshold gate — verify SonarQube/Codecov enforces per-Idris-function thresholds.
- [ ] **Defensive Membrane "complete":** Often missing CGLIB-proxy test — verify Spring `@Transactional` proxy preserves linearity.

---

## Recovery Strategies

When pitfalls occur despite prevention.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| `VerifyError` from emitted class (Pitfall 1) | MEDIUM | (1) Reduce to minimal `.idr` reproducer; (2) compare ASM vs ClassFile API output; (3) fix StackMapFrame computation; (4) add to regression corpus |
| Membrane bypass discovered (Pitfall 2) | HIGH | (1) Document the exact bypass; (2) close the hole if possible; (3) if structurally impossible, document as a capability boundary; (4) audit all production users |
| Carrier thread pinning storm (Pitfall 3) | MEDIUM | (1) JFR-identify pinning sites; (2) annotate FFI as `@Blocking`; (3) reroute to bounded blocking pool; (4) increase carrier pool as bandage |
| GraalVM `ClassNotFoundException` in production (Pitfall 4) | HIGH | (1) Add reachability metadata for missing class; (2) republish image; (3) audit codegen for similar gaps; (4) post-mortem and add to test corpus |
| Spring bean missing (Pitfall 5) | MEDIUM | (1) Diff `spring.components` against expected; (2) regenerate index file; (3) verify annotation is in correct format; (4) add CI check |
| OTel trace split (Pitfall 6) | LOW-MEDIUM | (1) Add ScopedValue-based context bridge; (2) backfill trace correlation IDs; (3) add CI test for trace continuity |
| JSR-45 source map invisible to tool (Pitfall 7) | MEDIUM | (1) Use stack-trace decoder CLI to translate; (2) document tool limitation; (3) consider upstream contribution to tool |
| JFR overhead too high (Pitfall 8) | LOW | (1) Add `@Throttle` to high-frequency events; (2) move events to "detail" config; (3) measure and document new overhead |
| Trampoline-induced perf cliff (Pitfall 9) | MEDIUM | (1) Identify hot trampolined paths via JFR; (2) add `@hot` pragma; (3) refactor mutual recursion if possible |
| Ghost annotations stripped (Pitfall 10) | MEDIUM | (1) Switch to standard runtime annotations; (2) add `BeanPostProcessor` for Spring proxies; (3) verify across all agents |
| Vendor snapshot diverged unmaintainably (Pitfall 11) | HIGH | (1) Triage divergences (rationale check); (2) upstream what can be upstreamed; (3) heroic rebase or fork-of-fork; (4) automate sync going forward |
| Reproducibility broken (Pitfall 12) | MEDIUM | (1) `diffoscope` to identify nondeterminism source; (2) fix sort ordering / timestamps; (3) re-attest SLSA |
| JPMS Spring failure (Pitfall 13) | MEDIUM | (1) Switch module to `open`; (2) add explicit `opens` directives; (3) document Spring profile for `module-info` |
| ScopedValue bug (Pitfall 14) | MEDIUM | (1) Identify scope-escape; (2) refactor to keep capture inside scope; (3) compiler check upgrade |
| Hot-reload corruption (Pitfall 15) | HIGH | (1) Restart all affected JVMs; (2) tighten "verification-affecting" detection; (3) make reload more conservative by default |

---

## Pitfall-to-Phase Mapping

This is the most actionable artifact for roadmap planning. Each pitfall is paired with the phase that should prevent it.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| 1. Stack Map Frame bugs | **Phase 1: Codegen Foundation** | Dual-verifier CI gate (ASM + ClassFile API + JaCoCo round-trip); regression corpus |
| 2. Defensive Membrane holes | **Phase: FFI / Linear Types Boundary** | Adversarial test corpus (reflection, serialization, `invokedynamic`, lambda capture, AOP proxies) |
| 3. Virtual thread pinning | **Phase: Loom / Concurrency** (after FFI importer) | JFR `VirtualThreadPinned` gate in benchmark CI; FFI annotation taxonomy |
| 4. GraalVM metadata gaps | **Phase 1 + Phase: Spring Integration** | Native-image CI from day one; reachability metadata emitted by codegen |
| 5. Spring bean discovery | **Phase: Spring Integration** | `spring.components` index test; hybrid-module DI test |
| 6. OTel context loss | **Phase: Telemetry / Effects** (after ScopedValue) | Trace continuity CI test through `LinearFuture` boundaries |
| 7. JSR-45 invisible to tools | **Phase: Codegen Foundation + Phase: Dev Tools** | Breakpoint test, profiler-output test, stack-trace-decoder CLI |
| 8. JFR cardinality / throttling | **Phase: Telemetry / Observability** | Overhead budget in JMH suite; throttling discipline |
| 9. Trampoline perf cliffs | **Phase 1: Codegen Foundation** | Self-loop detection from start; profiler tests |
| 10. Ghost annotation stripping | **Phase: Hybrid Erasure (Decision 15)** | Annotation survival test across agents |
| 11. Vendor snapshot drift | **Phase 1: Project Setup** | Automated weekly sync PR; compatibility matrix gate |
| 12. Reproducible build break | **Phase 1: Project Setup** | `diffoscope` CI gate from first commit |
| 13. JPMS Spring conflicts | **Phase: Module Emission (Decision 40)** | Spring-on-modulepath integration test |
| 14. ScopedValue lifecycle bugs | **Phase: Concurrency Primitives (Decision 19)** | Compile-time scope-escape check; JDK 25-specific test |
| 15. Hot-reload correctness | **Phase: Dev Loop Polish (Decision 32)** | LIMM-affecting change detection; classloader-restart default |

---

## Sources

### Historical / Comparable Languages

- [GitHub - mmhelloworld/idris-jvm](https://github.com/mmhelloworld/idris-jvm) — current state of the prior art
- [Idris JVM 0.7.0 Release notes](https://mmhelloworld.github.io/blog/2024/07/15/idris-jvm-0-7-0-release/)
- [The Story of Eta — Rahul Muttineni](https://medium.com/@rahulmuttineni/the-story-of-eta-pure-love-pure-functional-programming-2a690f3082b4)
- [Frege vs. Eta (DEV Community)](https://dev.to/awwsmm/haskell-on-the-jvm-frege-vs-eta-5238)
- [Frege: a Haskell-like Language for the JVM (InfoQ)](https://www.infoq.com/news/2015/08/frege-haskell-for-jvm/)
- [Frege, a JVM Haskell (taylor.fausak.me)](https://taylor.fausak.me/2015/06/25/frege-a-jvm-haskell/)
- [Java Champion James Ward on State of Java and JVM Languages (InfoQ)](https://www.infoq.com/articles/james-ward-james-jvm-languages/)
- [Bytecode generation for tailrec methods (scala/scala3 #14773)](https://github.com/lampepfl/dotty/issues/14773)

### ASM and Bytecode Verification

- [JaCoCo issue #1009 — VerifyErrors from un-recomputed stack frames](https://github.com/jacoco/jacoco/issues/1009)
- [ASM #317986 — COMPUTE_MAXS may generate non-valid frames](https://gitlab.ow2.org/asm/asm/-/issues/317986)
- [Spock #2080 — Stack map does not match exception handler](https://github.com/spockframework/spock/issues/2080)
- [Eclipse 545567 — switch expression with try VerifyError](https://bugs.eclipse.org/bugs/show_bug.cgi?id=545567)
- [JEP 484: Class-File API](https://openjdk.org/jeps/484)
- [Always up to date — Class-File API (INNOQ)](https://www.innoq.com/en/articles/2025/04/java-class-file-api/)

### Loom / Virtual Threads / ScopedValue

- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [JEP 506: Scoped Values (final)](https://openjdk.org/jeps/506)
- [How to solve the pinning problem (TheServerSide)](https://www.theserverside.com/tip/How-to-solve-the-pinning-problem-in-Java-virtual-threads)
- [The Carrier Pinning Trap (Azguards)](https://azguards.com/distributed-systems/the-carrier-pinning-trap-diagnosing-virtual-thread-starvation-in-spring-boot-3-migrations/)
- [Java 25 Virtual Threads — Pitfalls (springjavalab)](https://www.springjavalab.com/2025/12/java-25-virtual-threads-benchmarks-pitfalls.html)
- [Java 24 Stops Pinning Virtual Threads (nipafx)](https://nipafx.dev/inside-java-newscast-80/)
- [Migrating Project Loom Code from Java 21 to Java 25 (engnotes.dev)](https://engnotes.dev/blog/project-loom/project-loom-java-25-after-java-21-migration-part-9)

### GraalVM Native-Image

- [Reachability Metadata (graalvm.org)](https://www.graalvm.org/latest/reference-manual/native-image/metadata/)
- [graalvm-reachability-metadata #655 — Spring Boot 4.0.0-M1 + Java 24](https://github.com/oracle/graalvm-reachability-metadata/issues/655)
- [Spring Boot Native Image Support](https://docs.spring.io/spring-boot/reference/packaging/native-image/introducing-graalvm-native-images.html)
- [Native Images with Spring Boot and GraalVM (Baeldung)](https://www.baeldung.com/spring-native-intro)
- [Stop Guessing Your GraalVM Native Image Metadata (Coding Steve)](https://stevenpg.com/posts/graalvm-native-metadata-from-tests/)

### Spring Boot / AOT / Annotation Processing

- [Spring Boot AOT documentation](https://docs.spring.io/spring-boot/reference/packaging/aot.html)
- [Spring Boot #28046 — KAPT and ConfigurationProcessor](https://github.com/spring-projects/spring-boot/issues/28046)
- [Spring Classpath Scanning](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html)
- [Annotation Processing best practices (kt.academy)](https://kt.academy/article/ak-annotation-processing)
- [Spring Boot 3 component scanning bug #34379](https://github.com/spring-projects/spring-boot/issues/34379)

### JSR-45 Source Maps

- [JetBrains JSR 45 Support discussion](https://intellij-support.jetbrains.com/hc/en-us/community/posts/206340179-JSR-45-Support)
- [Haxe JSR-45 source maps request #10111](https://github.com/HaxeFoundation/haxe/issues/10111)
- [JaCoCo SourceDebugExtension HowTo #293](https://github.com/jacoco/jacoco/issues/293)

### JFR

- [Custom JDK Flight Recorder Events (Inside.java)](https://inside.java/2022/04/25/sip48/)
- [JDK-8257602 — JFR Event Throttling](https://bugs.openjdk.org/browse/JDK-8257602)
- [Monitoring REST APIs with custom JFR Events (morling.dev)](https://www.morling.dev/blog/rest-api-monitoring-with-custom-jdk-flight-recorder-events/)

### OpenTelemetry / Tracing

- [Propagating OpenTelemetry context with Virtual Threads (Softwaremill)](https://softwaremill.com/propagating-opentelemetry-context-when-using-virtual-threads-and-structured-concurrency/)
- [opentelemetry-java-instrumentation #11950 — Thread.startVirtualThread context](https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/11950)
- [OTel Java context propagation discussion #2884](https://github.com/open-telemetry/opentelemetry-java/discussions/2884)

### JPMS

- [JPMS Cheatsheet (tfesenko)](https://github.com/tfesenko/Java-Modules-JPMS-CheatSheet/blob/master/README.md)
- [Handling Split Packages (prgrmmng.com)](https://prgrmmng.com/handling-split-packages-and-avoiding-conflicts-in-modules)
- [Stephen Colebourne — JPMS automatic modules](https://blog.joda.org/2017/05/java-se-9-jpms-automatic-modules.html)

### Tail Calls / Trampolines

- [On Recursion, Continuations and Trampolines (Eli Bendersky)](https://eli.thegreenplace.net/2017/on-recursion-continuations-and-trampolines/)
- [PurelyFunctional.tv — trampoline tail recursion](https://ericnormand.me/issues/purelyfunctional-tv-newsletter-361-tip-trampoline-your-tail-recursion)
- [Clojure problems with the JVM (Eric Normand)](https://ericnormand.me/article/problems-with-the-jvm)

### Linear Types

- [Tweag — Safe memory management in inline-java using linear types](https://www.tweag.io/blog/2020-02-06-safe-inline-java/)
- [ACM — Safe-by-default Concurrency for Modern Programming Languages](https://dl.acm.org/doi/fullHtml/10.1145/3462206)

### CompletableFuture / Async Errors

- [Working with Exceptions in Java CompletableFuture (Baeldung)](https://www.baeldung.com/java-exceptions-completablefuture)
- [JDK-8254350 — CompletableFuture.get may swallow InterruptedException](https://bugs.openjdk.org/browse/JDK-8254350)
- [whenComplete vs handle (dempkow.ski)](https://dempkow.ski/blog/java-completablefuture-exception-handling/)

### invokedynamic / JIT

- [Too Fast, Too Megamorphic (DZone)](https://dzone.com/articles/too-fast-too-megamorphic-what)
- [The Black Magic of Java Method Dispatch (Shipilev)](https://shipilev.net/blog/2015/black-magic-method-dispatch/)
- [Indy deep dive (Nuno Caro)](https://nuno-caro.medium.com/indy-deep-dive-578e0c29af9)

### Reproducible Builds

- [Reproducible JVM builds (reproducible-builds.org)](https://reproducible-builds.org/docs/jvm/)
- [Configuring for Reproducible Builds (Apache Maven)](https://maven.apache.org/guides/mini/guide-reproducible-builds.html)
- [Reproducible Java Builds: Why Your JARs Lie (Medium)](https://medium.com/@ayushgupta228/reproducible-java-builds-why-your-jars-lie-and-how-to-fix-it-e32e4942acf6)

### ZGC

- [JEP 439: Generational ZGC](https://openjdk.org/jeps/439)
- [Bending pause times to your will with Generational ZGC (Netflix Tech Blog)](https://netflixtechblog.com/bending-pause-times-to-your-will-with-generational-zgc-256629c9386b)
- [Pauseless GC in Java 25: ZGC Guide (Andrew Baker)](https://andrewbaker.ninja/2025/12/03/deep-dive-pauseless-garbage-collection-in-java-25/)

### Inherited Idris 2 Concerns (from `.planning/codebase/CONCERNS.md`)

- TTC binary format version-checking gaps — affects Decision 38 vendor snapshot interaction with cache invalidation
- List-where-Set patterns in compiler — affects Decision 36 reproducibility (iteration order)
- Quadratic compile transformation in `CompileExpr.idr:375` — Palisade NamedCExp ingestion may amplify
- Coverage checker untested edges — affects Decision 27 verification claims if Idris itself accepts incomplete patterns

---

*Pitfalls research for: Palisade — JVM compiler backend for Idris 2*
*Researched: 2026-04-21*
*Quality bar context: "Jane Street happy" — pitfalls catalog calibrated to that production-confidence threshold.*
