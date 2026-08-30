# CHAPTER 2 — DOCKERFILE AUTHORING & IMAGE LAYERS

---

## 2.1 CONCEPT: Every Instruction Is a Layer — And Layers Are Cached

### TELUGU EXPLANATION

**ఇది ఈ chapter యొక్క కేంద్ర mental model:** ఒక Dockerfile లోని
**ప్రతి instruction** (`FROM`, `COPY`, `RUN`, మొదలైనవి), ఒక **కొత్త,
read-only layer** ని create చేస్తుంది — image, ఈ layers అన్నిటి
**stack** గా ఉంటుంది. Docker, ప్రతి layer ని **cache** చేస్తుంది —
ఒక layer, మునుపటి build తో **identical** గా ఉంటే (అదే instruction,
అదే input files), Docker, ఆ layer ని **మళ్ళీ build చేయకుండా, cache
నుండి reuse** చేస్తుంది.

**దీని practical, senior-level consequence:** Dockerfile లో
instructions ఉన్న **order**, build speed ని **నాటకీయంగా** ప్రభావితం
చేస్తుంది — Book 6 Chapter 3 లో చూసిన **index ordering** ఎంత
ముఖ్యమో, ఇక్కడ **layer ordering** అంతే ముఖ్యం. **తక్కువగా మారే
వాటిని ముందు, తరచుగా మారే వాటిని చివర** పెట్టాలి:

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline    # Dependencies: మారడం అరుదు
COPY src ./src                    # Source code: తరచుగా మారుతుంది
RUN mvn package
```

ఇక్కడ, `pom.xml` (dependencies) ముందు copy చేయడం వలన, source code
మారినప్పుడు కూడా, **dependency download layer, cache నుండి reuse**
అవుతుంది — source code మార్పు, ఆ తర్వాత layers నే మళ్ళీ build చేయాల్సి
వస్తుంది.

### ENGLISH INTERVIEW ANSWER

"The central mental model here is that every instruction in a
Dockerfile — `FROM`, `COPY`, `RUN`, and so on — creates a new, read-only
layer, and the image is really just a stack of these layers. Docker
caches each layer, and if a layer is identical to a previous build's
layer — same instruction, same input files — Docker reuses it from
cache instead of rebuilding it. The practical, senior-level consequence
is that instruction order dramatically affects build speed, the same
way index field ordering mattered in Book 6 Chapter 3 — the rule here is
to put what changes rarely first, and what changes often last. In a
typical Java Dockerfile, that means copying just the dependency
descriptor (`pom.xml`) and running the dependency download before
copying the actual source code — since dependencies change far less
often than source code, that expensive download layer stays cached
across most builds, and only the layers after the source code copy need
to rebuild when you're just iterating on application code."

---

## 2.2 CONCEPT: Key Dockerfile Instructions and Their Roles

### TELUGU EXPLANATION

**Core instructions:**
- **`FROM`:** Base image ని నిర్దేశిస్తుంది — Dockerfile యొక్క
  మొదటి line, ప్రతి తర్వాతి instruction, ఈ base మీద build అవుతుంది.
- **`WORKDIR`:** తర్వాతి instructions కి working directory సెట్
  చేస్తుంది — `cd` కి సారూప్యం, కానీ persistent గా.
- **`COPY`:** Host నుండి image లోకి files copy చేస్తుంది.
- **`RUN`:** Build-time command execute చేస్తుంది (ఉదా: dependencies
  install చేయడం, compile చేయడం) — ఫలితం, ఆ layer లో **permanently**
  బేక్ అవుతుంది.
- **`ENV`:** Environment variables సెట్ చేస్తుంది — image లో, runtime
  లో కూడా అందుబాటులో ఉంటాయి.
- **`EXPOSE`:** ఏ port మీద container listen చేస్తుందో **document**
  చేస్తుంది (ఇది **actual port publishing కాదు** — అది Chapter 4 లో
  చూసే `docker run -p` ద్వారా జరుగుతుంది; `EXPOSE`, కేవలం metadata).
- **`ENTRYPOINT` vs `CMD`:** రెండూ, container start అయినప్పుడు
  run అయ్యే command ని నిర్దేశిస్తాయి — `ENTRYPOINT`, **override
  చేయడం కష్టం** (fixed executable కోసం, ఉదా: `java -jar app.jar`);
  `CMD`, **default arguments** ఇస్తుంది, ఇది `docker run` command
  line ద్వారా **సులభంగా override** అవుతుంది.

### ENGLISH INTERVIEW ANSWER

"A few instructions do most of the work. `FROM` sets the base image
everything else builds on top of. `WORKDIR` sets the working directory
for subsequent instructions, like a persistent `cd`. `COPY` brings files
from the host into the image. `RUN` executes a build-time command,
baking its result permanently into that layer — installing
dependencies, compiling code. `ENV` sets environment variables available
both during build and at runtime. `EXPOSE` is often misunderstood — it
only documents which port the container listens on; it doesn't actually
publish that port to the host, which is what `docker run -p` does at
runtime, covered in Chapter 4. And `ENTRYPOINT` versus `CMD` is a
frequent point of confusion: `ENTRYPOINT` defines the fixed executable
the container runs — something like `java -jar app.jar` that you don't
want casually overridden — while `CMD` supplies default arguments that
are easy to override from the `docker run` command line, useful when
you want a sensible default that callers can still customize."

---

## 2.3 CONCEPT: `.dockerignore` and Avoiding Cache-Busting Mistakes

### TELUGU EXPLANATION

**`.dockerignore`:** `.gitignore` కి సారూప్యం — build context నుండి
(Docker daemon కి పంపే files) ఏ files/directories exclude చేయాలో
నిర్దేశిస్తుంది. దీన్ని సరిగ్గా వాడకపోతే:

- **Build context size పెరుగుతుంది** (ఉదా: `target/`, `.git/`,
  `node_modules/` లాంటి unnecessary directories, ప్రతి build కి
  Docker daemon కి పంపబడతాయి) — build slow అవుతుంది.
- **Cache-busting risk:** ఒక అనవసరమైన file (ఉదా: IDE metadata,
  build artifacts) మారితే, అది ఒక `COPY . .` layer లో భాగం అయితే,
  ఆ layer **అనవసరంగా invalidate** అవుతుంది, తర్వాతి అన్ని layers
  కూడా మళ్ళీ build అవ్వాల్సి వస్తుంది.

**సాధారణ mistake, section 2.1 యొక్క layer ordering సూత్రాన్ని
ఉల్లంఘించేది:** `COPY . .` ని **ముందుగానే** (dependencies install
చేసేముందు) పెట్టడం — ఇది, source code లో **ఏ చిన్న మార్పు** అయినా,
dependency-download layer ని కూడా invalidate చేస్తుంది, ప్రతి
build లో dependencies మళ్ళీ download అవ్వాల్సి వస్తుంది.

### ENGLISH INTERVIEW ANSWER

"`.dockerignore` works like `.gitignore` — it excludes files and
directories from the build context sent to the Docker daemon. Getting
this wrong has two costs: an unnecessarily large build context — sending
`target/`, `.git/`, or `node_modules/` to the daemon on every build
slows things down — and a subtler cache-busting risk, where an
irrelevant file change, like IDE metadata, gets swept into a broad
`COPY . .` layer and invalidates it unnecessarily, forcing every
subsequent layer to rebuild too. The most common mistake I'd flag,
directly violating the layer-ordering principle from section 2.1, is
placing a broad `COPY . .` before the dependency-installation step —
that means any tiny source code change invalidates the dependency
download layer as well, so every single build re-downloads
dependencies from scratch instead of only rebuilding the layers that
actually depend on the changed source files."

---

## 2.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Ordering Dockerfile instructions | Copies everything, then installs dependencies | Copies dependency descriptors first, installs, then copies source code |
| Using `EXPOSE` | Assumes it publishes the port | Knows it's documentation only; publishing happens at `docker run -p` |
| Choosing `ENTRYPOINT` vs `CMD` | Uses whichever example they copied | Chooses `ENTRYPOINT` for the fixed executable, `CMD` for overridable defaults |
| Writing `.dockerignore` | Skips it or copies a generic template blindly | Tailors it to actually exclude build-context bloat and cache-busting files |

---

## 2.5 COMMON MISTAKES

1. Copying the entire project before installing dependencies, busting
   the dependency-cache layer on every source code change.
2. Assuming `EXPOSE` actually publishes a port to the host.
3. Using `CMD` for a fixed executable that should never be casually
   overridden, or `ENTRYPOINT` where flexible default arguments would
   be more appropriate.
4. Skipping `.dockerignore`, bloating build context with `.git`,
   build artifacts, or IDE files.
5. Running unnecessary commands in separate `RUN` instructions when
   they could be combined, creating more layers than needed (a minor
   but real image-size consideration, expanded on in Chapter 3).

---

## 2.6 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. What does each instruction in a Dockerfile create? 2.
What is the difference between `ENTRYPOINT` and `CMD`?

**Intermediate:** 3. Why does instruction order affect build speed? 4.
What does `.dockerignore` do, and why does it matter?

**Senior:** 5. A team's Dockerfile copies the whole project directory
before running `mvn install`. Every build takes 3 minutes even for a
one-line code change. Diagnose and fix.

**Architect:** 6. Design a Dockerfile authoring standard for your
organization's Java services to maximize build cache effectiveness
across dozens of repositories.

**Scenario:** 7. A container starts, but `docker run myimage --debug`
doesn't seem to pass `--debug` to the application at all. What Dockerfile
configuration would you check?

**Trick:** 8. "Using `EXPOSE 8080` in a Dockerfile means the
application is now accessible on port 8080 from the host machine." True or false?

<details><summary>Key answers</summary>

- Q5: The Dockerfile is violating the layer-ordering principle — copying
  the whole project (including source code) before running the
  dependency install means every source code change invalidates that
  layer and forces a full dependency re-download. Fix: split into two
  `COPY` steps — copy just `pom.xml` (or `build.gradle`) first, run
  the dependency resolution step, then copy the actual source code
  afterward, so only the layers after the source copy need to rebuild
  on a code-only change.
- Q6: Standardize a shared base Dockerfile pattern/template with
  dependency-descriptor-first copying, a documented `.dockerignore`
  baseline (excluding `.git`, build output directories, IDE files),
  required multi-stage build usage (Chapter 3), and a linting or
  review checklist item verifying instruction ordering — applied
  consistently via a shared template repository or a Dockerfile
  linter in CI, rather than leaving each team to rediscover these
  practices independently.
- Q7: Most likely, the Dockerfile uses `CMD` in "shell form" or a fixed
  `ENTRYPOINT` that doesn't incorporate arguments passed at
  `docker run`. Check whether `ENTRYPOINT` is set to something like
  `["java", "-jar", "app.jar"]` without accepting additional args, or
  whether `CMD` was used for the fixed executable — if the goal is for
  `docker run myimage --debug` to append `--debug` to the running
  command, `ENTRYPOINT` should be the fixed executable in exec form,
  with `CMD` providing default (overridable) arguments that
  `docker run`'s trailing arguments actually replace.
- Q8: False — `EXPOSE` is purely documentation/metadata about which
  port the containerized application listens on; it has no effect on
  actual network accessibility. The port is only actually published to
  the host when `docker run` is given an explicit `-p` (or `-P` to
  auto-publish exposed ports) flag, which we cover in Chapter 4.

</details>

---

## 2.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain why Dockerfile instruction order affects build speed, connecting it to the caching mechanism.
- **Coding Check:** Write a Dockerfile snippet for a Maven-based Spring Boot app that correctly orders dependency installation before source code copying, and explain each instruction's role.
- **Explanation Check:** Explain to a teammate why `EXPOSE` alone doesn't make a service reachable from outside the container.
- **Real-World Check:** Your team's CI pipeline rebuilds a Docker image on every commit, and dependency installation (usually cached) is re-running every time despite no dependency changes. What would you check in the Dockerfile?
- **Senior Check:** Why might combining several `RUN` commands into one instruction (using `&&`) sometimes be preferred over separate `RUN` instructions?
- **Master Check:** Design a Dockerfile authoring checklist for a Java microservices team that ensures every service's image builds are cache-efficient, correctly documents its port, and has a sensible, appropriately-overridable startup command.

<details><summary>Answers</summary>

- Real-World Check: Check whether the Dockerfile copies source code (or
  the whole project directory) before the dependency-installation `RUN`
  step — if so, any commit touching source files, even without touching
  dependency descriptors, invalidates the dependency layer and forces a
  full re-download; reordering to copy dependency descriptors first
  fixes this.
- Senior Check: Combining `RUN` commands reduces the total number of
  layers, which can meaningfully reduce final image size (each layer
  adds some overhead, and intermediate state from an uncombined `RUN`
  — like downloaded package caches removed in a *later* `RUN` — would
  otherwise persist in an earlier layer even after later cleanup)."
  The trade-off is a coarser cache granularity — one combined `RUN`
  either fully hits or fully misses cache, versus finer-grained
  individual `RUN` steps that can partially hit cache.
- Master Check: (1) Order instructions with dependency descriptors
  before source code. (2) Maintain a `.dockerignore` excluding `.git`,
  build artifacts, and IDE files. (3) Use `EXPOSE` to document the
  actual listening port, understanding it's metadata only. (4) Use
  `ENTRYPOINT` in exec form for the fixed Java executable
  (`["java", "-jar", "app.jar"]`), with `CMD` supplying default,
  overridable JVM or application arguments. (5) Combine related `RUN`
  steps where it meaningfully reduces image size without harming cache
  granularity for genuinely independent steps.

</details>

---

## 2.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Every instruction | Creates a new, cached, read-only layer |
| Layer ordering | Rarely-changing instructions first, frequently-changing ones last |
| `FROM` | Sets the base image |
| `COPY` | Brings host files into the image |
| `RUN` | Executes and permanently bakes a build-time command's result |
| `EXPOSE` | Documentation only — does NOT publish the port |
| `ENTRYPOINT` vs `CMD` | Fixed executable vs overridable default arguments |
| `.dockerignore` | Excludes build-context bloat and cache-busting files |
| Common anti-pattern | `COPY . .` before dependency installation, busting the dependency cache on every change |

---

*(Continues to Chapter 3 — Multi-Stage Builds & Image Optimization.)*
