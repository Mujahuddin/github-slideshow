# BOOK 9 — KAFKA + RABBITMQ + MESSAGING
## Asynchronous, Event-Driven Communication (Telugu + English)

---

## COVER

**Program:** Java Backend / Microservices / Cloud / AI Engineering — Complete Mastery Series
**Book 9 of 25:** Message Brokers, Kafka Internals, RabbitMQ Internals, Delivery Semantics, JMS/Pub-Sub

## LEARNING OBJECTIVES

By the end of this book you will be able to:

- Explain why asynchronous, broker-mediated communication solves problems
  synchronous Feign calls (Book 8) cannot, and where it introduces new
  ones.
- Reason precisely about Kafka's topic/partition/offset model and how it
  determines ordering and scalability.
- Design consumer groups correctly, including rebalancing behavior and
  its failure modes.
- Explain delivery semantics (at-most-once, at-least-once,
  exactly-once) concretely, in terms of what a producer/consumer must
  actually do to achieve each one.
- Design RabbitMQ topologies using exchanges, bindings, and routing keys
  for real fan-out, routing, and topic-based use cases.
- Build production-grade reliability with manual ACK/NACK, dead-letter
  queues, and retry-with-backoff patterns.
- Compare Kafka, RabbitMQ, JMS, and cloud Pub/Sub systems and justify a
  choice for a given workload, not from familiarity but from trade-offs.

## PREREQUISITES

Book 8 (Microservices + Spring Cloud), especially Chapter 5 (Sagas) and
Chapter 6 (the Transactional Outbox pattern) — this book goes deep into
the broker mechanics that the Outbox pattern's "relay" step and
choreographed Sagas' events depend on, which Book 8 deliberately used
without requiring broker-specific knowledge.

## SKILL MAP (Master Skill Matrix, Rows 15-16)

| Level | What you should be able to do |
|---|---|
| Beginner | Explain what a message broker is and why async beats sync here |
| Intermediate | Produce/consume on Kafka and RabbitMQ, configure consumer groups/bindings |
| Professional | Choose and tune delivery semantics correctly for a given guarantee |
| Senior | Design DLQ/retry topologies, diagnose rebalancing and ordering bugs |
| Lead/Architect | Choose the right messaging technology for a given org-wide architecture |

---

## TABLE OF CONTENTS

- ✅ **Chapter 1** — [Messaging Fundamentals: Why Asynchronous Communication](./chapter-01-messaging-fundamentals.md)
- ✅ **Chapter 2** — [Kafka Architecture: Topics, Partitions, Offsets](./chapter-02-kafka-architecture-topics-partitions-offsets.md)
- ✅ **Chapter 3** — [Kafka Producers & Consumers, Consumer Groups](./chapter-03-kafka-producers-consumers-groups.md)
- ✅ **Chapter 4** — [Kafka Delivery Semantics & Reliability](./chapter-04-kafka-delivery-semantics-reliability.md)
- ✅ **Chapter 5** — [RabbitMQ Architecture: Exchanges, Queues, Routing](./chapter-05-rabbitmq-architecture-exchanges-routing.md)
- ✅ **Chapter 6** — [RabbitMQ Reliability: ACK/NACK, Dead-Letter Queues, Retry](./chapter-06-rabbitmq-reliability-ack-dlq-retry.md)
- ✅ **Chapter 7** — [JMS & Pub/Sub Models](./chapter-07-jms-pubsub-models.md)
- ✅ **Chapter 8** — [Kafka vs RabbitMQ: Choosing the Right Tool](./chapter-08-kafka-vs-rabbitmq-choosing.md)
- ✅ **[Final Assessment, Messaging Mock Interview Round, Capstone Project](./final-assessment-mock-interview-project.md)**

## SCOPE NOTE

This book covers messaging conceptually and at the Java/Spring
integration level (Spring Kafka, Spring AMQP) — it does not cover
Kafka/RabbitMQ cluster administration, broker deployment topology, or
Kubernetes operators for either, which belong to Books 11-13
(Docker/Kubernetes/Cloud). MongoDB/Redis and other NoSQL/caching
technologies are Book 10's territory, referenced here only where a
consumer needs idempotency-tracking storage.

---

## BOOK 9 STATUS: COMPLETE

All 8 chapters, the final assessment, mock interview round, and
capstone project are written. This book opened up the message broker
as a mechanism Book 8's Outbox pattern and choreographed Sagas had
already assumed: temporal decoupling and delivery semantics as first
principles, Kafka's retained-log architecture (topics/partitions/
offsets, consumer groups, delivery reliability), RabbitMQ's
consume-and-delete queue architecture (exchanges/routing, ACK/NACK,
dead-letter/retry patterns), JMS/cloud pub-sub as portable abstractions
over these same ideas, and a workload-driven framework for choosing
between them.

**Next in the program: Book 10 — MongoDB + Redis + NoSQL.**
