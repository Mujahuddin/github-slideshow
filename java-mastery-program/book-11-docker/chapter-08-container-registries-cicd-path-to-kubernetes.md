# CHAPTER 8 — CONTAINER REGISTRIES, CI/CD & THE PATH TO KUBERNETES

---

## 8.1 CONCEPT: Container Registries — The Distribution Mechanism

### TELUGU EXPLANATION

**Registry:** ఒక image ని build చేసిన తర్వాత, దాన్ని **ఎక్కడైనా
run చేయాలంటే** (CI server, staging, production), ఆ image ని ఒక
**central location** కి push చేసి, అక్కడి నుండి pull చేసుకోగలగాలి
— ఇది registry యొక్క role (ఉదా: Docker Hub, Amazon ECR, Google
Artifact Registry, GitHub Container Registry, private/self-hosted
registries).

**Workflow:** `docker build -t myregistry.com/order-service:v1.2.0 .`
→ `docker push myregistry.com/order-service:v1.2.0` → ఏ machine
అయినా (CI, production server), `docker pull myregistry.com/order-service:v1.2.0`
ద్వారా, **ఖచ్చితంగా అదే image** ని fetch చేసుకోగలదు.

**Senior-level connection, Book 9/10 కి direct సారూప్యం:** ఇది,
Book 9 Chapter 1 యొక్క message broker concept కి, structurally
సారూప్యం — producer (build పైప్‌లైన్), registry (broker) కి
publish చేస్తుంది, అనేక consumers (deployment targets), అక్కడి
నుండి **స్వతంత్రంగా, తమ own సమయంలో** pull చేసుకోగలరు.

### ENGLISH INTERVIEW ANSWER

"Once an image is built, it needs to be runnable anywhere — a CI
server, staging, production — which requires a central distribution
point: a container registry, like Docker Hub, Amazon ECR, Google
Artifact Registry, or a private self-hosted registry. The workflow is
straightforward: build and tag the image with a registry address, push
it, and then any machine that can authenticate to that registry can
pull the exact same image by that tag. I'd draw a structural parallel to
Book 9's message broker concept here: a build pipeline acts like a
producer publishing to the registry, which acts like the broker, and
multiple deployment targets act like independent consumers, each
pulling the same artifact on their own schedule — the same
producer-broker-consumer decoupling idea, just applied to distributing
container images instead of messages."

---

## 8.2 CONCEPT: Image Tagging Strategy — `latest` Is Not a Deployment Strategy

### TELUGU EXPLANATION

**అత్యంత సాధారణ, production-affecting mistake:** ప్రతి build ని,
`myimage:latest` గా tag చేయడం, మరియు production లో కూడా `latest`
నే వాడటం. **ఇది ఎందుకు ప్రమాదకరం:**
- **Non-deterministic:** "`latest`" అనేది, ఒక **నిర్దిష్ట version**
  ని సూచించదు — ఇది, build అయిన ప్రతిసారి **మారుతుంది**. ఒక production
  server, `latest` ని pull చేస్తే, **ఏ code exactly run అవుతుందో
  తెలియదు** — ఇది Book 6 Chapter 5 లో చూసిన **non-repeatable read**
  కి, deployment level లో సారూప్యమైన సమస్య.
  - **Rollback impossible:** ఒక bad deployment జరిగితే, "మునుపటి,
  పనిచేసిన version కి వెళ్ళు" అని చెప్పలేం, ఎందుకంటే **ఏ specific
  version ముందు run అవుతుందో record లేదు**.

**సరైన పద్ధతి — Semantic Versioning + immutable tags:** `order-service:v1.4.2`
(లేదా, commit SHA తో: `order-service:a1b2c3d`) — ప్రతి build, ఒక
**unique, immutable tag** పొందుతుంది; `latest` ని (ఉపయోగిస్తే కూడా)
**production లో ఎప్పుడూ వాడకూడదు**, కేవలం local development
convenience కోసమే.

### ENGLISH INTERVIEW ANSWER

"One of the most common, genuinely production-affecting mistakes is
tagging every build as `myimage:latest` and deploying `latest` to
production. The problem is that `latest` isn't a specific version at
all — it's a moving target that changes with every build. If a
production server pulls `latest`, there's no way to know exactly which
code is actually running, which is structurally the same kind of
problem as a non-repeatable read from Book 6 Chapter 5, just at the
deployment level instead of the transaction level. Worse, rollback
becomes genuinely difficult, since there's no record of exactly which
specific version was running before a bad deployment. The correct
practice is semantic versioning or commit-SHA-based immutable tags —
something like `order-service:v1.4.2` or `order-service:a1b2c3d` —
where every build gets a unique, permanent tag, and `latest`, even if
it still exists, is never what production actually deploys; it's at
most a local development convenience."

---

## 8.3 CONCEPT: How an Image Becomes a Deployable Artifact — The CI/CD Bridge

### TELUGU EXPLANATION

**ఇది ఈ book యొక్క closing concept, Book 14 (CI/CD) కి bridge గా
పనిచేస్తుంది:** ఒక typical CI/CD pipeline, container context లో,
ఈ దశలు కలిగి ఉంటుంది:

1. **Build:** Code commit → CI trigger → Dockerfile ఆధారంగా image
   build (Chapter 2-3 యొక్క multi-stage builds ఇక్కడ కీలకం, build
   speed కోసం).
2. **Test:** Built image ని run చేసి, automated tests (Book 4 Chapter
   8 యొక్క testing pyramid) run చేయడం.
3. **Scan:** Chapter 7 యొక్క vulnerability scanning — critical
   findings ఉంటే, pipeline ఇక్కడే fail అవ్వాలి.
4. **Tag & Push:** ఒక unique, immutable tag (section 8.2) తో,
   registry కి push చేయడం.
5. **Deploy:** Target environment (Compose-based staging, లేదా Book
   12 Kubernetes cluster), ఆ నిర్దిష్ట tag ని pull చేసి, run చేయడం.

**ఈ book నుండి Book 12 కి transition, senior-level framing:**
Docker/Compose, **single-host, manual-ish** orchestration ఇస్తుంది —
ఒక్క host fail అయితే, ఏమవుతుంది? Scaling ఎలా చేయాలి (ఒక్క service
ని, అనేక hosts మీద)? Rolling updates ఎలా చేయాలి, zero downtime తో?
ఈ ప్రశ్నలకి సమాధానం, **Kubernetes** — ఇది, ఈ book లో నేర్చుకున్న
containers/images concepts నే వాడుకుంటూ, **multi-host, automatic,
self-healing orchestration** ని అందిస్తుంది.

### ENGLISH INTERVIEW ANSWER

"A typical container-based CI/CD pipeline has five stages. Build:
a code commit triggers CI, which builds the image from the Dockerfile —
this is exactly where Chapters 2 and 3's multi-stage builds and layer
caching matter most for keeping build times reasonable. Test: run
automated tests against the built image, following Book 4 Chapter 8's
testing pyramid. Scan: run Chapter 7's vulnerability scan, failing the
pipeline outright on critical findings. Tag and push: apply a unique,
immutable tag, per section 8.2, and push to the registry. Deploy: the
target environment — whether a Compose-based staging setup or a
Kubernetes cluster — pulls that specific tag and runs it. As a closing
thought bridging into Book 12, I'd point out what Docker and Compose
genuinely don't solve well: what happens if a single host fails? How do
you scale one service across many hosts? How do you roll out an update
with zero downtime? Those are exactly the questions Kubernetes answers —
it takes the same containers and images this book taught how to build
correctly, and adds multi-host, automated, self-healing orchestration
on top, which is where the program goes next."

---

## 8.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Tagging images | Uses `latest` for everything, including production | Uses immutable, unique tags (semver or commit SHA) for every deployable build |
| Deploying to production | Pulls whatever `latest` currently resolves to | Deploys a specific, known, previously-tested tag |
| Rolling back a bad deploy | Has no clear "previous version" to revert to | Redeploys the last known-good immutable tag |
| Thinking about scaling beyond Docker/Compose | Tries to manually script multi-host orchestration | Recognizes this is exactly what Kubernetes (Book 12) solves |

---

## 8.5 COMMON MISTAKES

1. Using `latest` as the tag deployed to production.
2. Having no CI pipeline step that fails the build on critical
   vulnerability scan findings.
3. Not being able to identify exactly which image tag is running in a
   given environment during an incident.
4. Skipping the test stage against the actual built container image,
   testing only in a separate, non-containerized environment.
5. Trying to hand-roll multi-host orchestration, failover, and scaling
   with plain Docker/Compose instead of adopting Kubernetes when that
   need genuinely arises.

---

## 8.6 INTERVIEW QUESTION BANK — CHAPTER 8

**Basic:** 1. What is a container registry, and why is it needed? 2.
Why is `latest` a risky tag to deploy to production?

**Intermediate:** 3. Name the typical stages of a container-based
CI/CD pipeline. 4. Why is testing the actual built container image
important, rather than testing only in a separate dev environment?

**Senior:** 5. Design an image tagging strategy for a team practicing
continuous deployment, ensuring every production deployment is
traceable back to a specific commit.

**Architect:** 6. Your organization's CI/CD pipeline builds, tests, and
pushes images, but has no automated way to roll back a bad production
deployment. Propose a fix.

**Scenario:** 7. During an incident, the on-call engineer can't
determine which exact version of a service is currently running in
production, delaying diagnosis by 20 minutes. Diagnose the process gap
and propose a fix.

**Trick:** 8. "As long as our CI/CD pipeline pushes images to a
registry automatically, our deployment process is safe from human error."
True or false?

<details><summary>Key answers</summary>

- Q5: Tag every built image with the Git commit SHA (e.g.,
  `order-service:a1b2c3d`), giving an unambiguous, permanent link
  between a running image and the exact source code that produced it;
  optionally also apply a semantic version tag for human readability at
  release boundaries, but treat the commit-SHA tag as the source of
  truth for traceability. Never deploy `latest` to production.
- Q6: Ensure every deployment records exactly which immutable tag was
  previously running (e.g., via deployment history in the CI/CD tool or
  orchestration platform); build a rollback mechanism that redeploys
  that specific previous tag rather than requiring a new build; and
  ensure the registry retains prior image versions long enough to
  support this (not garbage-collecting recent tags too aggressively).
- Q7: The process gap is almost certainly a missing or poorly
  surfaced record of which specific image tag is deployed to
  production at any given time — fix by ensuring the deployment
  process logs/exposes the exact running tag (e.g., via a
  `/actuator/info` endpoint reporting build metadata, Book 4 Chapter 7,
  or a deployment dashboard), so an on-call engineer can immediately see
  which version is running without needing to reconstruct it from
  scattered CI logs during an active incident.
- Q8: False — an automated pipeline reduces certain classes of human
  error (manual build/push mistakes) but doesn't eliminate deployment
  risk on its own; using `latest` in that automated pipeline, skipping
  the scan-gate step, or having no rollback strategy are all still
  serious risks that automation alone doesn't address — automation
  needs to be paired with the specific practices from this chapter
  (immutable tagging, mandatory scan gates, traceable deployments) to
  actually be safe.

</details>

---

## 8.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain why deploying `latest` to production makes rollback difficult, connecting it to the concept of a non-repeatable, non-deterministic reference.
- **Coding Check:** Write the `docker build`, `docker tag`, and `docker push` commands for building and publishing an image tagged with both a commit SHA and a semantic version.
- **Explanation Check:** Explain to a teammate why a CI pipeline should fail the build on a critical vulnerability finding rather than just logging a warning.
- **Real-World Check:** Your team wants zero-downtime deployments and automatic recovery if a service crashes on one of several hosts. Would plain Docker/Compose achieve this, or is a different tool needed? Explain why.
- **Senior Check:** Why does immutable tagging matter even in an organization that deploys many times per day and rarely needs to look back at old versions?
- **Master Check:** Design the full CI/CD pipeline for a Book 8-style microservices system, from commit to production deployment, incorporating every relevant concept from this book (multi-stage builds, scanning, tagging, healthchecks) and explicitly naming where Kubernetes (Book 12) would take over responsibilities Docker/Compose can't fulfill alone.

<details><summary>Answers</summary>

- Real-World Check: Plain Docker/Compose would not achieve this well —
  Compose is fundamentally single-host oriented; it has no built-in
  concept of automatically rescheduling a container onto a healthy host
  if the host it was running on fails, and no native rolling-update
  mechanism across multiple hosts. This is exactly the gap Kubernetes
  (Book 12) is built to fill, with its scheduler, self-healing
  controllers, and rolling deployment strategies operating across a
  cluster of many hosts.
- Senior Check: Even with frequent deployments, an incident can occur
  at any time referencing "what was running an hour ago" or "what
  changed between these two deployments" — immutable tags give an exact,
  reliable answer to both questions at any point in time; without them,
  reconstructing "what was actually running" during a past incident
  becomes guesswork based on approximate deployment timestamps rather
  than a precise, verifiable record.
- Master Check: Commit → CI triggers a multi-stage Docker build
  (Chapters 2-3, minimal final image) → automated tests run against the
  built image (Book 4 Ch8) → vulnerability scan gate (Chapter 7),
  failing on critical/high findings → image tagged with commit SHA and
  pushed to the registry (this chapter) → deployment to a target
  environment pulls that specific tag, with `HEALTHCHECK`/readiness
  verification (Chapter 7) confirming successful startup. Where
  Kubernetes takes over: multi-host scheduling and bin-packing, rolling
  updates with automatic rollback on failed health checks across many
  replicas, automatic rescheduling if a host fails, and horizontal
  autoscaling based on load — none of which plain Docker/Compose
  provides natively.

</details>

---

## 8.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Container registry | Central distribution point — build once, pull anywhere |
| `latest` tag | Never deploy to production — non-deterministic, breaks rollback |
| Immutable tagging | Commit SHA or semver — every build traceable to exact source |
| CI/CD pipeline stages | Build → Test → Scan → Tag & Push → Deploy |
| Testing scope | Test the actual built container image, not just a separate dev environment |
| Rollback requirement | Requires knowing the exact previous immutable tag that was running |
| Docker/Compose limitation | Single-host oriented — no built-in multi-host failover or rolling updates |
| Kubernetes' role (Book 12) | Multi-host scheduling, self-healing, rolling updates, autoscaling |

---

*(This completes BOOK 11 — DOCKER's chapter content. Continue to the
Final Assessment, Docker Mock Interview Round, and Capstone Project.)*
