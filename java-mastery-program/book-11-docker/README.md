# BOOK 11 — DOCKER
## Containerizing Java Applications for Consistent Deployment (Telugu + English)

---

## COVER

**Program:** Java Backend / Microservices / Cloud / AI Engineering — Complete Mastery Series
**Book 11 of 25:** Containerization Fundamentals, Dockerfile Authoring, Multi-Stage Builds, Networking, Volumes, Compose, Production Practices

## LEARNING OBJECTIVES

By the end of this book you will be able to:

- Explain what a container actually is (and isn't) and why it solves
  the "works on my machine" problem more fundamentally than a VM does.
- Author a correct, efficient Dockerfile for a Java/Spring Boot
  application, understanding image layers and layer caching.
- Build optimized, minimal production images using multi-stage builds.
- Reason about Docker networking modes and design container-to-container
  communication correctly.
- Use volumes correctly to persist data beyond a container's lifecycle,
  and know what should never be baked into an image.
- Use Docker Compose to define and run a multi-service local
  development environment matching Book 8's microservices architecture.
- Apply container security and production best practices, and
  understand how a container image becomes a deployable artifact in a
  CI/CD pipeline.

## PREREQUISITES

Book 8 (Microservices + Spring Cloud) — this book packages the
individually-designed services from that book into deployable
containers; Book 9's messaging services and Book 10's polyglot data
stores are also referenced as things a Compose-based local environment
needs to run alongside application containers.

## SKILL MAP (Master Skill Matrix, Row 18)

| Level | What you should be able to do |
|---|---|
| Beginner | Run and build a basic Docker image |
| Intermediate | Author a correct Dockerfile, understand image layers |
| Professional | Design multi-stage builds and optimize image size |
| Senior | Design container networking and volume strategy for a multi-service system |
| Lead/Architect | Set container platform standards (base images, security scanning, registry policy) |

---

## TABLE OF CONTENTS

- ✅ **Chapter 1** — [Containerization Fundamentals: Containers vs VMs](./chapter-01-containerization-fundamentals.md)
- ✅ **Chapter 2** — [Dockerfile Authoring & Image Layers](./chapter-02-dockerfile-authoring-image-layers.md)
- ✅ **Chapter 3** — [Multi-Stage Builds & Image Optimization](./chapter-03-multi-stage-builds-image-optimization.md)
- ✅ **Chapter 4** — [Docker Networking](./chapter-04-docker-networking.md)
- ✅ **Chapter 5** — [Docker Volumes & Data Persistence](./chapter-05-docker-volumes-data-persistence.md)
- ✅ **Chapter 6** — [Docker Compose: Multi-Container Applications](./chapter-06-docker-compose-multi-container-applications.md)
- ✅ **Chapter 7** — [Container Security & Production Best Practices](./chapter-07-container-security-production-best-practices.md)
- ✅ **Chapter 8** — [Container Registries, CI/CD & The Path to Kubernetes](./chapter-08-container-registries-cicd-path-to-kubernetes.md)
- ✅ **[Final Assessment, Docker Mock Interview Round, Capstone Project](./final-assessment-mock-interview-project.md)**

## SCOPE NOTE

This book covers Docker as a standalone containerization and local
multi-container orchestration tool (via Compose) — cluster-scale
orchestration, scheduling, and self-healing across many machines is
Book 12's dedicated territory (Kubernetes). Cloud-provider-managed
container services (ECS, GKE, EKS) are Book 13's territory. CI/CD
pipeline mechanics beyond "how an image becomes a deployable artifact"
are Book 14's dedicated territory (Git/Maven/Gradle/CI-CD/Terraform).

---

## BOOK 11 STATUS: COMPLETE

All 8 chapters, the final assessment, mock interview round, and
capstone project are written. This book took the individually-designed
services from Book 8 and packaged them into deployable containers:
containerization fundamentals corrected against the "lightweight VM"
misconception, Dockerfile authoring and layer-caching discipline,
multi-stage builds for minimal production images, networking and
volumes for correct inter-service communication and data persistence,
Compose for local multi-container orchestration mirroring Books 8-10's
stack, and production security/registry/CI-CD practices — closing by
explicitly naming the multi-host orchestration gaps Book 12's
Kubernetes exists to fill.

**Next in the program: Book 12 — Kubernetes.**
