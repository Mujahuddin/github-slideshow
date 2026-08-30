# CHAPTER 3 — MULTI-STAGE BUILDS & IMAGE OPTIMIZATION

---

## 3.1 CONCEPT: The Problem Multi-Stage Builds Solve — Build Tools Don't Belong in Production

### TELUGU EXPLANATION

**సమస్య:** ఒక Java application ని build చేయడానికి, **JDK, Maven/Gradle,
build cache** అవసరం — కానీ ఆ application ని **run** చేయడానికి,
**JRE మాత్రమే** సరిపోతుంది (compiler అవసరం లేదు). ఒకే Dockerfile
లో, build మరియు run రెండూ చేస్తే, **final image లో, JDK + Maven +
build cache + source code అన్నీ శాశ్వతంగా ఉండిపోతాయి** — ఇది image
size ని (వందల MBs, GB కూడా) unnecessary గా పెంచుతుంది, మరియు
**attack surface** ని కూడా పెంచుతుంది (Chapter 7 లో దీని security
angle చూస్తాం) — production లో ఎప్పుడూ వాడని build tools, potential
vulnerabilities తో పాటు అక్కడే ఉండిపోతాయి.

**Multi-Stage Build — పరిష్కారం:** ఒకే Dockerfile లో, **అనేక `FROM`
stages** define చేయడం — ఒక stage (`AS builder`), JDK+Maven తో
application ని compile చేస్తుంది; **చివరి stage**, ఒక **తేలికైన JRE
base image** తో మొదలై, మునుపటి stage నుండి **compiled artifact
మాత్రమే** (`COPY --from=builder`) copy చేసుకుంటుంది — build tools,
source code, intermediate files, **ఏవీ final image లో చేరవు**.

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /build/target/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### ENGLISH INTERVIEW ANSWER

"Building a Java application needs a JDK, Maven or Gradle, and a build
cache, but running it only ever needs a JRE — no compiler required. If
you build and run in a single Dockerfile stage, all of that build
tooling, plus source code and intermediate build artifacts, permanently
ends up in the final image — bloating it, often by hundreds of
megabytes, and expanding the attack surface with tools and files that
serve no purpose in production, which we'll dig into further as a
security concern in Chapter 7. A multi-stage build solves this directly:
you define multiple `FROM` stages in one Dockerfile — a `builder` stage
using a JDK-and-Maven base image to compile the application, and a
final stage starting fresh from a minimal JRE-only base image that
copies over just the compiled JAR artifact using `COPY --from=builder`.
None of the build tooling, source code, or intermediate files make it
into the final image at all — only the one artifact actually needed to run."

---

## 3.2 CONCEPT: Choosing a Minimal Base Image — Alpine, Distroless, and the Trade-Offs

### TELUGU EXPLANATION

**Base image ఎంపిక, image size మరియు security ని నేరుగా
ప్రభావితం చేస్తుంది:**

- **Full OS base (ఉదా: `ubuntu`, `debian`):** అనేక utilities,
  package manager, shell — debugging కి సౌకర్యం, కానీ size పెద్దది,
  attack surface ఎక్కువ.
- **Alpine-based (ఉదా: `eclipse-temurin:17-jre-alpine`):** చాలా
  తక్కువ size (musl libc వాడుతుంది, glibc కాదు) — **senior-level
  caution:** musl libc, glibc తో పూర్తి compatible కాదు, కొన్ని
  native libraries (JNI ఆధారిత) unexpected గా fail అవ్వొచ్చు, ఇది
  ఒక "surprising in production" bug category.
- **Distroless (Google's `gcr.io/distroless/java`):** **shell కూడా
  లేదు**, కేవలం application + runtime dependencies మాత్రమే —
  అత్యంత తక్కువ attack surface, కానీ **debugging కష్టం** (`docker
  exec -it container sh` పనిచేయదు, shell లేదు కాబట్టి).

**Senior-level framework:** Size మరియు security, debugging convenience
తో trade-off అవుతాయి — production కి, minimal (distroless/alpine),
కానీ team యొక్క debugging tooling maturity (Book 8 Chapter 7 distributed
tracing, centralized logging) ని బట్టి, "shell లేకపోతే డీబగ్ చేయగలమా"
అని ముందుగా నిర్ధారించుకోవాలి.

### ENGLISH INTERVIEW ANSWER

"Base image choice directly affects both size and security. A full OS
base like Ubuntu or Debian includes many utilities, a package manager,
and a shell — convenient for debugging, but large and with a wider
attack surface. An Alpine-based image is much smaller because it uses
musl libc instead of glibc, but I'd flag a real caution here: musl isn't
fully compatible with glibc, and some native, JNI-based libraries can
fail in surprising, hard-to-reproduce-locally ways specifically because
of this difference — it's a legitimate 'works in one image, breaks in
another' bug category to watch for. A distroless image, like Google's
`gcr.io/distroless/java`, goes further, shipping no shell at all — the
smallest attack surface, but it also means `docker exec -it container
sh` simply won't work, since there's no shell to exec into, making
interactive debugging harder. My framework here is that minimizing size
and attack surface trades off against debugging convenience, so before
committing to a shell-less distroless image in production, I'd confirm
the team already has strong observability — centralized logging,
distributed tracing from Book 8 Chapter 7 — good enough that
interactive shell debugging isn't the primary diagnostic tool being relied on."

---

## 3.3 CONCEPT: Layer-Level Optimization — Combining, Cleaning, and Ordering

### TELUGU EXPLANATION

**ఇది Chapter 2 యొక్క layer ordering సూత్రాలకి, image size
optimization dimension add చేసేది:**

- **Package manager cache cleanup, అదే `RUN` instruction లో:**
  `RUN apt-get update && apt-get install -y curl && rm -rf
  /var/lib/apt/lists/*` — cleanup ని **వేరే `RUN`** లో పెడితే,
  మునుపటి layer లో ఇప్పటికే బేక్ అయిన cache files, **ఇంకా image
  లోనే ఉండిపోతాయి** (layers, immutable — తర్వాతి layer, మునుపటి
  layer యొక్క files ని "delete" చేసినా, image size లో ఆ layer
  physically ఉంటూనే ఉంటుంది).
- **Multi-stage build, ఇక్కడ కూడా అదనపు advantage ఇస్తుంది:**
  Build stage లో ఎంత "మురికిగా" (cache files, temp artifacts తో)
  ఉన్నా పర్వాలేదు — అది **final image లో కనిపించదు**, ఎందుకంటే
  final stage, కేవలం అవసరమైన artifact ని మాత్రమే copy చేసుకుంటుంది.

**Layered JAR (Spring Boot-specific optimization):** Spring Boot
3.x+, `layertools` ఇస్తుంది — ఒక fat JAR ని, **dependencies (అరుదుగా
మారేవి), application code (తరచుగా మారేది)** అనే వేర్వేరు layers
గా విభజించడానికి — ఇది Chapter 2 యొక్క layer caching ఆలోచనని,
JAR level కి కూడా extend చేస్తుంది, ఒక్క fat JAR ని ఒకే layer గా
copy చేయడం కంటే, చాలా efficient గా.

### ENGLISH INTERVIEW ANSWER

"This extends Chapter 2's layer-ordering principles into the
image-size dimension. A common mistake is cleaning up package manager
cache files in a separate `RUN` instruction from the install step —
since layers are immutable, files already baked into an earlier layer
physically remain in the image even if a later layer 'deletes' them;
the cleanup has to happen in the *same* `RUN` instruction as the
install to actually keep those bytes out of the image. Multi-stage
builds give an additional advantage here too: the build stage can be as
messy as it needs to be, full of cache files and temporary artifacts,
because none of that ever reaches the final image — only the
explicitly copied artifact does. For Spring Boot specifically, version
3.x and later's `layertools` feature splits a fat JAR into separate
layers — dependencies, which change rarely, versus application code,
which changes often — extending Chapter 2's layer-caching idea down
into the JAR itself, which is considerably more efficient than copying
one fat JAR as a single monolithic layer that has to be entirely
re-copied on every code change."

---

## 3.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Building a production Java image | Uses one JDK-based stage for build and run | Uses multi-stage builds, running on a minimal JRE-only final image |
| Choosing a base image | Defaults to a full OS image for convenience | Weighs Alpine/distroless against debugging needs and native library compatibility |
| Cleaning up build artifacts | Adds a separate `RUN rm -rf ...` instruction | Combines install and cleanup in the same `RUN`, or relies on multi-stage isolation |
| Packaging a Spring Boot JAR | Copies the fat JAR as one layer | Uses layertools to split dependencies and application code into separate cacheable layers |

---

## 3.5 COMMON MISTAKES

1. Building and running in a single stage, leaving JDK/Maven/source
   code in the production image.
2. Choosing Alpine without testing native/JNI-dependent libraries for
   musl compatibility issues.
3. Committing to a shell-less distroless image without adequate
   external observability tooling to compensate for lost interactive
   debugging.
4. Placing package cache cleanup in a separate `RUN` instruction,
   leaving the cache bytes baked into an earlier, still-present layer.
5. Copying a Spring Boot fat JAR as a single layer instead of using
   layertools to separate dependency and application code layers.

---

## 3.6 INTERVIEW QUESTION BANK — CHAPTER 3

**Basic:** 1. What problem do multi-stage builds solve? 2. Name one
trade-off of using a distroless base image.

**Intermediate:** 3. Why doesn't deleting a file in a later layer
reduce the final image size? 4. What does Spring Boot's `layertools`
feature do?

**Senior:** 5. A team's production image is 850MB, built from a single
JDK-based stage with Maven, full source code, and no cleanup. Redesign
it and estimate the improvement.

**Architect:** 6. Design an organization-wide base image standard for
Java services, balancing image size, security, and debuggability
across teams with varying observability maturity.

**Scenario:** 7. After switching to an Alpine-based image, a service
that previously worked fine now intermittently throws
`UnsatisfiedLinkError` for a native library. Diagnose.

**Trick:** 8. "Since multi-stage builds create multiple stages, the
final image includes all of them, so it should be roughly the same
size as building everything in one stage." True or false?

<details><summary>Key answers</summary>

- Q5: Redesign as a multi-stage build: a `builder` stage using a
  Maven+JDK base image to compile and package, and a final stage on a
  minimal JRE base (or distroless, if debugging tooling permits),
  copying only the compiled JAR. This typically drops image size from
  the JDK+Maven+source baseline (often 700-900MB) down to well under
  200MB, sometimes under 100MB with distroless, since none of the build
  tooling or source code carries into the final image.
- Q6: Standardize on a small set of approved minimal base images
  (e.g., a JRE-based image as the safe default, distroless as an
  opt-in for teams with mature observability), require multi-stage
  builds in a shared Dockerfile template/linter, and pair any move
  toward distroless with a documented minimum observability bar
  (centralized logging, tracing) so teams don't lose their only
  debugging tool without a replacement in place.
- Q7: This is the classic Alpine/musl libc compatibility issue — a
  native library built against glibc can behave unexpectedly or fail
  to load correctly under Alpine's musl libc, and such failures are
  often intermittent or code-path-dependent rather than immediate,
  since only specific native calls may be affected. Fix: either use a
  glibc-compatible minimal base image instead of Alpine for this
  specific service, or verify/rebuild the native dependency for musl
  compatibility if that's supported.
- Q8: False — only the final stage's layers end up in the final image;
  intermediate stages (like a `builder` stage) exist only during the
  build process and are discarded once their needed artifacts are
  copied out via `COPY --from=`. The whole point of multi-stage builds
  is that the final image size reflects only the last stage plus
  whatever was explicitly copied into it, not the sum of every stage.

</details>

---

## 3.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain why a multi-stage build's final image doesn't include the build stage's JDK and Maven installation.
- **Coding Check:** Write a two-stage Dockerfile for a Gradle-based Spring Boot app, using layertools in the final stage to separate dependency and application layers.
- **Explanation Check:** Explain to a teammate the trade-off between a distroless image's security benefit and its debugging cost.
- **Real-World Check:** Your team is deciding between Alpine and a standard JRE base image for a service with several native (JNI) dependencies. What would you check before choosing Alpine?
- **Senior Check:** Why does combining `apt-get install` and cache cleanup into one `RUN` instruction matter for final image size, when a later instruction "removes" the same files?
- **Master Check:** Design the full multi-stage Dockerfile strategy for a Spring Boot microservice that needs to be small (for fast deployment/scaling), secure (minimal attack surface), and still debuggable in production incidents.

<details><summary>Answers</summary>

- Real-World Check: Check each native dependency's documentation or
  test suite specifically against musl libc/Alpine compatibility —
  many JNI libraries either explicitly support Alpine, explicitly don't,
  or are untested; if any are untested or known-incompatible, either
  avoid Alpine for this service or budget time to verify/rebuild them
  against musl before committing to it in production.
- Senior Check: Docker layers are immutable and additive — a later
  layer's "deletion" is itself just a marker recorded in that layer;
  the actual bytes from the earlier layer still exist in the image's
  layer stack and are counted in the final image size, since Docker
  images are the union of all their layers' changes, not just the
  final visible filesystem state. Only cleaning up within the same
  layer as the installation avoids ever writing those bytes to a
  persisted layer in the first place.
- Master Check: A `builder` stage on a JDK+Gradle image producing a
  layered Spring Boot JAR (via `layertools`); a final stage on a
  minimal (not necessarily fully shell-less) JRE base image, copying
  the JAR's layers in dependency-then-application order for cache
  efficiency; retaining a minimal shell (a slim JRE image rather than
  full distroless) specifically to preserve `docker exec` debugging
  capability, provided this is a deliberate, documented trade-off
  against a fully distroless approach given the team's current
  observability maturity — moving to distroless later once tracing/
  logging tooling is confirmed sufficient to not need interactive shell access.

</details>

---

## 3.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Multi-stage build | Build stage (JDK/Maven) discarded; final stage copies only the needed artifact |
| Full OS base | Convenient for debugging, large size, wider attack surface |
| Alpine base | Small, but musl libc can break glibc-dependent native libraries |
| Distroless base | Smallest attack surface, but no shell — requires strong external observability |
| Layer immutability | Deleting a file in a later layer does NOT remove its bytes from the image |
| Cache cleanup rule | Install and cleanup must be in the SAME `RUN` instruction |
| Spring Boot layertools | Splits fat JAR into dependency/application layers for better caching |
| Decision framework | Trade image size/security against team's debugging tooling maturity |

---

*(Continues to Chapter 4 — Docker Networking.)*
