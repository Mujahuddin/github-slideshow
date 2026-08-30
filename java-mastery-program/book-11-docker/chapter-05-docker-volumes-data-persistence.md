# CHAPTER 5 — DOCKER VOLUMES & DATA PERSISTENCE

---

## 5.1 CONCEPT: Container Ephemerality — Why the Writable Layer Isn't Enough

### TELUGU EXPLANATION

**Chapter 1.2 గుర్తుచేసుకోండి:** ఒక container, image మీద ఒక **thin,
writable layer** — container delete అయితే, ఆ writable layer లోని
**అన్ని data శాశ్వతంగా పోతుంది**. ఇది, database container ల
కోసం (ఉదా: Book 10 యొక్క MongoDB/PostgreSQL containers), ఒక **critical
సమస్య**: container restart అయినా, redeploy అయినా, **మొత్తం
data పోతుంది** — ఇది ఖచ్చితంగా, Book 10 Chapter 6 లో చూసిన "Redis
ని sole source of truth గా వాడకూడదు" హెచ్చరికకి, container level
లో సారూప్యమైన సమస్య.

**పరిష్కారం — Volumes:** Container యొక్క lifecycle నుండి **స్వతంత్రంగా**
ఉండే, ఒక ప్రత్యేక storage mechanism — container delete అయినా,
volume లోని data **అలాగే ఉంటుంది**, కొత్త container (అదే, లేదా
వేరే image నుండి) ఆ volume ని **మళ్ళీ mount** చేసుకోగలదు, పాత
data తో.

### ENGLISH INTERVIEW ANSWER

"Recall from Chapter 1 that a container is just a thin, writable layer
on top of an immutable image — when the container is deleted, everything
in that writable layer is permanently lost. For anything stateful, like
the database containers we'd use for Book 10's MongoDB or PostgreSQL,
this is a critical problem: a routine container restart or redeploy
would wipe out all the data, which is exactly the container-level
version of Book 10 Chapter 6's warning against making Redis the sole
source of truth for critical data — here, the writable layer itself is
even more ephemeral than that. The fix is volumes: a storage mechanism
that exists independently of any single container's lifecycle. When a
container is deleted, data in a volume survives, and a new container —
the same image or a different one — can mount that same volume and pick
up right where the data left off."

---

## 5.2 CONCEPT: Named Volumes vs Bind Mounts — Two Different Mechanisms

### TELUGU EXPLANATION

**Named Volume:** Docker, తనే volume ని create చేసి, manage చేస్తుంది
(`docker volume create mydata`, లేదా `docker run -v mydata:/var/lib/postgresql/data`)
— actual storage location, Docker యొక్క own managed area లో ఉంటుంది
(host filesystem నేరుగా తెలుసుకోవాల్సిన అవసరం లేదు). **Production
కోసం recommended** — Docker, backup/migration tooling తో బాగా
integrate అవుతుంది.

**Bind Mount:** Host filesystem లోని ఒక **specific path** ని, నేరుగా
container లోకి mount చేయడం (`docker run -v /host/path:/container/path`)
— host మరియు container, **అదే files ని share** చేస్తాయి, real-time
గా. ఇది **local development** కి బాగా సరిపోతుంది (ఉదా: source code
ని bind mount చేసి, host మీద edit చేసిన మార్పులు, container లో
వెంటనే కనిపించేలా — hot reload workflows కి).

**Senior-level caution, bind mounts గురించి:** Host path, **hardcoded**
గా ఉంటుంది కాబట్టి, ఇది **portability ని దెబ్బతీస్తుంది** — ఒక
`docker run` command, ఒక specific host path మీద ఆధారపడితే, అది
**వేరే machine మీద పనిచేయకపోవచ్చు** (ఆ path లేకపోతే). Production
లో, bind mounts ని configuration files కోసమే (కాదు, application
data కోసం) పరిమితం చేయడం సాధారణం.

### ENGLISH INTERVIEW ANSWER

"A named volume is created and managed entirely by Docker — you
reference it by name, and Docker handles where it actually lives on the
host, integrating well with Docker's own backup and migration tooling.
This is what I'd recommend for production stateful data. A bind mount
instead maps a specific host filesystem path directly into the
container, so the host and container share the exact same files in
real time — this is excellent for local development, like bind-mounting
source code so edits on the host are immediately visible inside a
running container for hot-reload workflows. The caution I'd raise about
bind mounts is that they hardcode a specific host path, which hurts
portability — a `docker run` command depending on a specific host path
might simply fail on a different machine where that path doesn't
exist. In production, I'd generally restrict bind mounts to specific
configuration files rather than using them for actual application data,
where named volumes are the more portable, Docker-managed choice."

---

## 5.3 CONCEPT: What Should Never Be Baked Into an Image

### TELUGU EXPLANATION

**ఇది Chapter 1 యొక్క "image = immutable template" concept యొక్క
practical consequence:** Image, **build time లో** create అవుతుంది,
**అన్ని environments (dev, staging, production) కి ఒకేలా share**
అవుతుంది — కాబట్టి, environment-specific, లేదా sensitive data,
image లో **ఎప్పుడూ బేక్ చేయకూడదు**:

- **Secrets (database passwords, API keys):** Image లో baked చేస్తే,
  image ని access చేయగలిగిన **ఎవరైనా** ఆ secrets ని చూడగలరు (image
  layers, `docker history`/`docker save` ద్వారా inspect చేయొచ్చు) —
  Book 8 CI/CD security సూత్రాలకి direct విరుద్ధం. Secrets, **runtime
  లో** (environment variables, mounted secret files, లేదా Kubernetes
  Secrets — Book 12) inject చేయాలి.
- **Environment-specific configuration** (ఉదా: production database
  URL): Image, ఒకేసారి build అయ్యి, **ప్రతి environment లో ఒకేలా**
  run అవ్వాలి (Chapter 1 యొక్క "works on my machine" పరిష్కారం
  యొక్క core idea) — environment-specific values, `ENV` ద్వారా
  build-time లో fix చేయకుండా, **runtime configuration** (Book 4
  Chapter 2 యొక్క Spring profiles/externalized config కి direct
  సారూప్యం) ద్వారా inject చేయాలి.
- **Persistent application data:** Section 5.1 లో చూసినట్టు, ఇది
  volumes లో ఉండాలి, image లో కాదు.

### ENGLISH INTERVIEW ANSWER

"This follows directly from Chapter 1's image-as-immutable-template
concept — an image is built once and shared identically across every
environment, so anything environment-specific or sensitive should never
be baked into it. Secrets like database passwords or API keys must
never be baked into an image, because anyone who can pull or inspect
that image — via `docker history` or `docker save` — can potentially
recover them from the layers; they should instead be injected at
runtime, through environment variables, mounted secret files, or a
platform's dedicated secrets mechanism like Kubernetes Secrets in Book
12. Environment-specific configuration, like a production database URL,
shouldn't be fixed into the image via `ENV` at build time either — the
whole point of Chapter 1's 'works on my machine' solution is that the
exact same image runs everywhere, so environment differences need to be
supplied through runtime configuration instead, the same principle as
Spring's externalized configuration and profiles from Book 4 Chapter 2.
And persistent application data, as covered in section 5.1, belongs in
volumes, never baked into or written directly onto the image's own
writable layer."

---

## 5.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Running a database container | Doesn't configure a volume, loses data on restart | Always mounts a named volume for any stateful container |
| Local dev source-code editing | Rebuilds the image on every code change | Bind-mounts source code for hot-reload workflows |
| Storing secrets | Bakes them into the Dockerfile via `ENV` | Injects them at runtime, never building them into the image |
| Environment-specific config | Builds a different image per environment | Builds one image, injecting config at runtime per environment |

---

## 5.5 COMMON MISTAKES

1. Running a stateful container (database, message broker) without a
   volume, losing all data on restart or redeploy.
2. Using bind mounts for production application data instead of
   Docker-managed named volumes.
3. Hardcoding host-specific paths in bind mounts, breaking portability
   across machines.
4. Baking secrets or environment-specific configuration into the image
   via `ENV` or `COPY`.
5. Building a separate image per environment instead of one image
   configured differently at runtime.

---

## 5.6 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. What happens to data in a container's writable layer when
the container is deleted? 2. What's the difference between a named
volume and a bind mount?

**Intermediate:** 3. Why are bind mounts good for local development but
risky for production data? 4. Why shouldn't secrets be baked into a
Docker image?

**Senior:** 5. Design the volume strategy for a local Compose-based
development environment running PostgreSQL and MongoDB (Book 10)
alongside application services.

**Architect:** 6. Your organization has found database credentials in
several teams' Docker images via `docker history`. Propose a
remediation and prevention strategy.

**Scenario:** 7. A team rebuilt and redeployed their PostgreSQL
container after a routine security patch, and all data was lost. Diagnose.

**Trick:** 8. "Since a volume is 'attached' to a container, deleting
that container also deletes the volume and its data." True or false?

<details><summary>Key answers</summary>

- Q5: Create named volumes for both databases (e.g.,
  `postgres-data` and `mongo-data`), mounted to each database
  container's respective data directory (`/var/lib/postgresql/data`
  and `/data/db`). Application service source code, if hot-reload is
  desired for local development, can use bind mounts, but the databases
  themselves should always use named volumes so that stopping/restarting
  the local environment (a common, frequent local dev action) never
  loses local test data unexpectedly.
- Q6: Remediation: audit and rebuild affected images with credentials
  removed from all layers (note: simply adding a later layer that
  "deletes" the credential file doesn't remove it from history, per
  Chapter 3's layer immutability — affected images need to be rebuilt
  from a corrected Dockerfile, and old images with embedded credentials
  should be considered compromised, with those credentials rotated).
  Prevention: add a CI step scanning images for secret patterns before
  they're pushed to a registry, and train teams to inject secrets only
  at runtime.
- Q7: The PostgreSQL container almost certainly wasn't running with a
  named volume mounted to its data directory — a routine redeploy
  replaces the container itself (per Chapter 1's image/container
  distinction), and without a volume, the entire writable layer,
  including all database files, was discarded along with the old
  container. Fix going forward: always mount a named volume for the
  data directory before this can happen again, and separately, restore
  from backup if one existed.
- Q8: False — this is a common and consequential misconception. A
  volume's lifecycle is independent of any specific container; deleting
  a container that had a volume mounted leaves the volume and its data
  completely intact, ready to be mounted by a new container. A volume
  is only actually deleted by an explicit `docker volume rm` (or
  `docker system prune` with volume-pruning flags), never as a
  side effect of removing a container that used it.

</details>

---

## 5.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain why a container's writable layer alone is insufficient for stateful applications, connecting it to Chapter 1's image/container model.
- **Coding Check:** Write the `docker run` command to start a PostgreSQL container with a named volume mounted to its data directory, and explain what happens to the data if the container is later removed.
- **Explanation Check:** Explain to a teammate why baking a database password into a Dockerfile via `ENV` is a security risk, even if the image is never made public.
- **Real-World Check:** Your team wants local developers to see their source code changes reflected instantly in a running container without rebuilding. What volume mechanism would you use, and what's the portability trade-off?
- **Senior Check:** Why is "one image, runtime-injected configuration" preferred over "one image per environment" as a deployment philosophy?
- **Master Check:** Design the full data-persistence and configuration strategy for a Book 8-style microservices stack being containerized: each service's database needs durable storage, each service needs environment-specific configuration (dev/staging/prod), and no secrets should ever appear in any image layer.

<details><summary>Answers</summary>

- Real-World Check: Use a bind mount, mapping the host's source code
  directory into the container's expected source path — this gives the
  real-time file-sharing needed for hot reload. The portability
  trade-off is that this `docker run` (or Compose) configuration now
  depends on a specific host path existing, which is fine for local
  development (where the path is the developer's own project checkout)
  but should never be relied upon for anything meant to run
  identically across arbitrary machines, like a CI or production deployment.
- Senior Check: Building a separate image per environment reintroduces
  exactly the risk Chapter 1's "works on my machine" solution was meant
  to eliminate — if the staging and production images are built
  differently (even slightly), you're no longer guaranteed the same
  artifact is what's actually running in each place, undermining the
  core reliability benefit of containerization. One image with runtime
  configuration guarantees the literal same tested artifact moves through
  every environment unchanged, with only externally-supplied values differing.
- Master Check: Each database (per service, per Book 8's bounded
  contexts) gets its own named volume mounted to its data directory,
  independent of that service's application container lifecycle.
  Application services receive environment-specific configuration
  (database URLs, feature flags) via environment variables or mounted
  config files supplied at container start time — never baked into the
  image — following Book 4 Chapter 2's externalized configuration
  principle. Secrets (database credentials, API keys) are injected via
  a secrets mechanism (environment variables sourced from a secrets
  manager, or mounted secret files) at runtime, with CI pipeline
  scanning (per Q6) verifying no image layer ever contains a credential
  before it's pushed to a registry (Chapter 8).

</details>

---

## 5.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Container writable layer | Ephemeral — lost when the container is deleted |
| Named volume | Docker-managed, portable, recommended for production stateful data |
| Bind mount | Host-path-dependent, great for local dev hot-reload, risky for production portability |
| Volume lifecycle | Independent of any container — surviving container deletion by default |
| Secrets | Never bake into an image — inject at runtime only |
| Environment-specific config | Never bake into an image — use runtime configuration, one image everywhere |
| Stateful containers | ALWAYS mount a volume for their data directory |
| Volume deletion | Only happens via explicit `docker volume rm`, never as a side effect of container removal |

---

*(Continues to Chapter 6 — Docker Compose: Multi-Container Applications.)*
