# BOOK 11 — FINAL ASSESSMENT, DOCKER MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 8 CHAPTERS)

1. Explain why a container is not "a lightweight VM," and why this
   distinction matters when reasoning about isolation guarantees. *(Ch. 1)*
2. A Dockerfile copies the entire project directory before running
   `mvn package`. Why does this hurt build speed, and how would you fix
   it? *(Ch. 2)*
3. Design a multi-stage Dockerfile for a Gradle-based Spring Boot
   service, explaining why the final image doesn't contain the JDK. *(Ch. 3)*
4. Two containers on the default Docker bridge network can't resolve
   each other by name, but the same setup works fine in Docker Compose.
   Why? *(Ch. 4, 6)*
5. A team removes a stateful container as part of a routine redeploy
   and loses all its data. What was misconfigured? *(Ch. 5)*
6. Explain why `depends_on` in Compose is insufficient on its own for a
   service depending on a database, and design the correct fix. *(Ch. 6)*
7. Why is running a container as root a meaningful security risk even
   though namespaces provide isolation? *(Ch. 7)*
8. Explain why deploying `latest` to production makes incident response
   and rollback harder, using a concrete scenario. *(Ch. 8)*
9. Design the full local Docker Compose environment for a system built
   across Books 8-10 (three microservices, Kafka, PostgreSQL, MongoDB,
   Redis), including volumes, healthchecks, and networking. *(Ch. 4, 5, 6)*
10. Trace a single Java microservice's journey from source code to a
    healthy, running production container — naming which chapter's
    concept governs each step, and explicitly naming what Docker/Compose
    alone cannot do that the next book (Kubernetes) will solve.

<details>
<summary>Answer Key</summary>

1. A container is a process sharing the host's kernel, isolated via
   namespaces and limited via cgroups — it does not run its own kernel
   the way a VM does. This matters because container isolation is
   inherently weaker than VM isolation (a kernel-level vulnerability
   could theoretically be exploited across container boundaries in ways
   a VM's hardware-virtualized boundary wouldn't allow), which is why
   Chapter 7's non-root-user practice and image scanning are genuine,
   necessary defenses rather than redundant caution.
2. Copying the whole project before the dependency-installation step
   means the dependency-download layer gets invalidated by any source
   code change, even though dependencies themselves haven't changed —
   forcing a full re-download on every build. Fix: copy just the
   dependency descriptor (`pom.xml`/`build.gradle`) first, run
   dependency resolution, then copy source code afterward, so only the
   layers after that point need to rebuild on a code-only change.
3. A `builder` stage `FROM gradle:8-jdk17` compiles and packages the
   application; a final stage `FROM eclipse-temurin:17-jre` copies only
   the compiled JAR via `COPY --from=builder`. The final image doesn't
   contain the JDK because intermediate build stages are discarded
   after their needed artifacts are copied out — only the last stage's
   layers, plus explicitly copied files, become part of the final image.
4. Docker's *default* bridge network doesn't support container-name DNS
   resolution — only custom, user-defined bridge networks do. Docker
   Compose automatically creates a custom bridge network for every
   project and attaches all defined services to it, which is why name
   resolution "just works" there, while two containers manually started
   without an explicit custom network fall back to the
   non-DNS-supporting default bridge.
5. The stateful container almost certainly had no named volume mounted
   to its data directory — since a container's writable layer is
   ephemeral and discarded when the container is removed, any data not
   stored in an independently-lifecycled volume is lost the moment the
   container is deleted, regardless of how routine or well-intentioned
   the redeploy was.
6. `depends_on` only guarantees the dependency container has *started*,
   not that it's actually ready to accept connections (a database can
   report "started" before its engine finishes initializing). Fix:
   combine `depends_on` with `condition: service_healthy`, backed by an
   actual `HEALTHCHECK` on the database service (e.g., `pg_isready`),
   plus application-level connection retry as a defense-in-depth
   second layer.
7. Namespace isolation is real but not absolute — it's weaker than a
   VM's hardware-virtualized isolation. If an attacker achieves code
   execution inside a container running as root, they have unrestricted
   privileges within that container's namespace, which meaningfully
   increases the risk and ease of privilege escalation or a container
   escape attempt, compared to the same exploit occurring under a
   restricted, non-root user.
8. Concrete scenario: a bad deployment goes out under the `latest` tag;
   the on-call engineer wants to roll back to "the previous working
   version," but since every build overwrote `latest`, there's no
   record of exactly what was running before — unlike an immutable
   tag (commit SHA or semver), which would let them redeploy the exact
   known-good prior artifact immediately instead of having to
   reconstruct or rebuild it under time pressure during an active incident.
9. Custom Compose network (created automatically). Named volumes for
   postgres-data and mongo-data (Redis optional, workload-dependent).
   Healthchecks (`pg_isready`, Mongo/Redis equivalents, Kafka broker
   readiness) gating each dependent microservice's `depends_on`. The
   three microservices reach postgres/mongo/redis/kafka by service
   name over the automatically-created network, with only necessary
   ports (e.g., the services' own API ports) published to the host.
10. Source code committed → CI triggers a multi-stage Docker build
    (Ch. 2-3, dependency-first layer ordering, minimal final image) →
    automated tests run against the built image (Book 4 Ch8) →
    vulnerability scan gate (Ch. 7), blocking critical findings →
    image tagged with an immutable commit SHA and pushed to a registry
    (Ch. 8) → deployed with resource limits and a `HEALTHCHECK` tied to
    Actuator (Ch. 7) → runs on a custom bridge network, reachable by
    service name from dependent services (Ch. 4). What Docker/Compose
    alone cannot do: automatically reschedule this container if its
    host fails, perform a zero-downtime rolling update across multiple
    replicas on multiple hosts, or autoscale based on load — all of
    which Book 12's Kubernetes is built to solve.

</details>

---

## PART B — MOCK INTERVIEW: DOCKER ROUND

**Interviewer:** "Your team's Java service Docker image is 780MB, and a
teammate says 'that's just how big Java images are.' Do you agree, and
what would you investigate?"

**Model answer:** "I wouldn't accept that at face value — a
well-optimized Java production image should typically be well under
200MB, often under 100MB, so 780MB tells me something specific is
wrong, not that this is inherent to Java. I'd first check whether the
Dockerfile uses a multi-stage build at all — a single-stage build that
includes the full JDK, Maven, and source code alongside the compiled
artifact is the single most common cause of this kind of bloat. I'd
also check the base image choice — a full OS base like `ubuntu` with a
manually installed JDK is much larger than a purpose-built minimal JRE
image like `eclipse-temurin:17-jre`. And I'd check whether cache
cleanup, if the Dockerfile installs any OS packages, happens in the
same `RUN` instruction as the install — cleanup in a separate
instruction leaves those bytes in the image regardless, since layers
are immutable. Restructuring around a proper multi-stage build with a
minimal JRE base image would very likely get this down to a fraction
of its current size."

**Follow-up:** "The team is hesitant to change the Dockerfile because
'it works, why touch it.' How do you make the case?"
(Frame it in terms of concrete costs beyond aesthetics: larger images
mean slower pulls during every deployment and every horizontal scale-out
event, more storage cost in the registry, and — critically — a larger
attack surface carrying unnecessary build tools and OS packages into
production, per Chapter 7's security reasoning; propose making the
change behind a feature-flagged or parallel build first to prove
equivalent behavior before fully cutting over, addressing the "it
works" risk aversion directly rather than dismissing it.)

---

**Interviewer:** "A production incident: a Spring Boot service
container keeps restarting every 30 seconds. `docker logs` shows the
application starts, logs 'Started Application in 8.2 seconds,' and then
the container exits. Walk me through your diagnosis."

**Model answer:** "The fact that it logs a successful startup and then
still exits points away from an application crash and toward something
about how the container's main process or health checking is
configured. First, I'd check the `HEALTHCHECK` configuration and
whatever orchestration is managing restarts — if a healthcheck is
misconfigured (checking the wrong port, wrong path, or too short a
timeout relative to actual startup time) and the orchestrator treats
failed healthchecks as a signal to kill and restart the container, that
would exactly match this symptom: the app finishes starting, but the
healthcheck fails anyway and triggers a restart before it can even
serve its first real request. Second, I'd check whether the process
running as PID 1 inside the container is actually the Java process —
if the `ENTRYPOINT`/`CMD` is set up in a way where a shell wraps the
Java process (e.g., shell form instead of exec form) and something
causes that shell to exit, the container could stop even though the
JVM logged a successful startup message moments before. I'd check both
in parallel, since they're the two most likely explanations for
'successful startup log, followed by unexplained exit.'"

**Follow-up:** "It turns out the healthcheck was checking
`http://localhost:8080/actuator/health` but the app was configured to
listen only on `https`. What does this teach about designing healthchecks?"
(A healthcheck needs to be validated against the application's actual
runtime configuration, not assumed defaults — this is exactly the kind
of environment-specific detail from Chapter 5's "runtime configuration,
not baked into the image" principle; a healthcheck command referencing
a hardcoded protocol/port should be verified whenever that
configuration changes, ideally with the healthcheck itself parameterized
or tested as part of the CI pipeline's test stage from Chapter 8.)

---

**Interviewer:** "Explain to me, as if I'm a new engineer joining the
team, why we never deploy the `latest` tag to production, and why
Docker Compose isn't what we use for our actual production deployments."

**Model answer:** "On tagging: `latest` isn't a specific version, it's
a moving pointer that gets reassigned with every build. If we deployed
`latest` and something went wrong, we'd have no reliable way to know
exactly what code was running, and no clean way to roll back to a known
previous version — every image we build gets tagged with its Git commit
SHA specifically so we always have an exact, permanent, traceable
record of what's running anywhere, at any time. On Compose versus
production: Compose is fantastic for local development — one command
spins up our whole stack — but it's fundamentally built around running
on a single host. It has no built-in way to reschedule a container if
the host it's on fails, no rolling-update mechanism across multiple
hosts, and no autoscaling. Production needs all of those things, which
is exactly why we run on Kubernetes there — it takes the same container
images we build with Docker and orchestrates them properly across many
machines, with self-healing and zero-downtime deployments built in."

---

## PART C — CAPSTONE PROJECT: "CONTAINERIZED MICROSERVICES PLATFORM"

**Goal:** A full Docker-based build and local orchestration setup
demonstrating every chapter of Book 11, containerizing the
microservices architecture from Book 8.

**Requirements:**

1. Write multi-stage Dockerfiles (Ch. 2-3) for at least two Java
   microservices (e.g., order-service, inventory-service), using
   dependency-first layer ordering and a minimal JRE final base image;
   verify final image size is under 200MB.
2. Configure each service to run as a non-root user, with no secrets or
   environment-specific configuration baked into the image (Ch. 5, 7).
3. Add a `HEALTHCHECK` to each service's Dockerfile, tied to its Spring
   Boot Actuator health endpoint (Ch. 7).
4. Write a `docker-compose.yml` (Ch. 6) defining both microservices,
   PostgreSQL, and Redis, with: a custom network (automatic via
   Compose), named volumes for stateful services, and
   `condition: service_healthy` gating each microservice's database
   dependency.
5. Demonstrate container-to-container communication by name (Ch. 4) —
   order-service calling inventory-service via its Compose service name.
6. Run an image vulnerability scan (Trivy or equivalent) against both
   built images and document any findings and remediation (Ch. 7).
7. Set up a tagging convention (Ch. 8) using commit-SHA-based tags for
   both services, and write (or diagram, if a live registry isn't
   available) the CI/CD pipeline stages that would build, test, scan,
   tag, and push these images.
8. Write a one-page reflection naming three specific things
   Docker/Compose does NOT solve for this system that Book 12's
   Kubernetes will need to address (e.g., multi-host failover, rolling
   updates, autoscaling).

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Image size | Final images are under 200MB via correct multi-stage builds |
| Layer caching | Rebuilding after a source-only change doesn't re-trigger dependency download |
| Security | Containers run as non-root; scan results show no unaddressed critical findings |
| Compose correctness | `docker compose up` starts the full stack correctly with no manual intervention or race-condition failures |
| Networking | Services communicate by Compose service name, with only necessary ports published to the host |
| Tagging discipline | No `latest` tag used for anything resembling a "production" deployment path |

---

*(This completes BOOK 11 — DOCKER. Book 12 — KUBERNETES — takes the
containers and images this book taught how to build correctly and adds
multi-host scheduling, self-healing, rolling updates, and autoscaling —
exactly the gaps named in this book's closing chapter.)*
