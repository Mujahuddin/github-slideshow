# BOOK 8 — MICROSERVICES + SPRING CLOUD
## Distributed Systems Thinking (Telugu + English)

---

## COVER

**Program:** Java Backend / Microservices / Cloud / AI Engineering — Complete Mastery Series
**Book 8 of 25:** Decomposition, Service Discovery, Resilience, Sagas, Event-Driven Architecture, Distributed Tracing

## LEARNING OBJECTIVES

By the end of this book you will be able to:

- Decide, with real justification, when to decompose a monolith into
  microservices — and when NOT to.
- Explain service discovery and centralized configuration mechanics, not
  just that "Eureka does discovery."
- Design resilient inter-service communication using circuit breakers,
  retries, timeouts, and bulkheads — and explain how they interact.
- Explain why distributed transactions don't work like local ones, and
  design a Saga for a real multi-service business process.
- Solve the dual-write problem with the Transactional Outbox pattern.
- Trace a request across multiple services using correlation IDs and
  distributed tracing.
- Reason about CQRS, scalability, and high availability trade-offs at
  an architect level.

## PREREQUISITES

Books 4-7 (Spring Boot, REST APIs, SQL/JDBC, JPA/Hibernate) — this book
assumes a single service is already well-built, and addresses what
changes when there are many of them talking to each other.

## SKILL MAP (Master Skill Matrix, Row 11)

| Level | What you should be able to do |
|---|---|
| Beginner | Explain what a microservice is and isn't |
| Intermediate | Use service discovery, Feign, basic resilience annotations |
| Professional | Design a Saga, implement the Outbox pattern |
| Senior | Debug distributed failures using tracing, reason about consistency trade-offs |
| Lead/Architect | Decide decomposition boundaries, design org-wide resilience/observability standards |

---

## TABLE OF CONTENTS

- ✅ **Chapter 1** — [Decomposition & Bounded Contexts](./chapter-01-decomposition-bounded-contexts.md)
- ✅ **Chapter 2** — [Service Discovery & Config Server](./chapter-02-service-discovery-config-server.md)
- ✅ **Chapter 3** — [Inter-Service Communication & Feign](./chapter-03-inter-service-communication-feign.md)
- ✅ **Chapter 4** — [Resilience Patterns: Circuit Breaker, Retry, Timeout, Bulkhead](./chapter-04-resilience-patterns.md)
- ✅ **Chapter 5** — [Distributed Transactions & the Saga Pattern](./chapter-05-distributed-transactions-saga.md)
- ✅ **Chapter 6** — [Event-Driven Architecture & the Outbox Pattern](./chapter-06-event-driven-outbox-pattern.md)
- ✅ **Chapter 7** — [Distributed Tracing & Observability](./chapter-07-distributed-tracing-observability.md)
- ✅ **Chapter 8** — [CQRS, Scalability & High Availability](./chapter-08-cqrs-scalability-high-availability.md)
- ✅ **[Final Assessment, Microservices Mock Interview Round, Capstone Project](./final-assessment-mock-interview-project.md)**

## SCOPE NOTE

Kafka/RabbitMQ mechanics are Book 9's dedicated territory — this book
uses messaging conceptually (Chapter 6's Outbox pattern, Chapter 5's
choreographed Sagas) without requiring broker-specific knowledge yet.
API Gateway and rate limiting were covered in Book 5 Chapter 6 and are
referenced, not repeated.

---

## BOOK 8 STATUS: COMPLETE

All 8 chapters, the final assessment, mock interview round, and capstone
project are written. This book took the single, well-built service from
Books 4-7 and addressed what changes once many of them talk to each
other: bounded-context decomposition, service discovery, resilient
inter-service calls, distributed transaction coordination via Sagas, the
Outbox pattern for reliable event publishing, distributed tracing for
cross-service debugging, and CQRS/scalability/HA trade-offs at an
architect level.

**Next in the program: Book 9 — Kafka + RabbitMQ + Messaging.**
