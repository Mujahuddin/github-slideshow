# BOOK 9 — FINAL ASSESSMENT, MESSAGING MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 8 CHAPTERS)

1. A team wants to send a "welcome email" after user registration
   without blocking the registration response. Should this use
   synchronous or asynchronous communication, and why? *(Ch. 1)*
2. Explain why "exactly-once delivery" is not something a broker alone
   can guarantee end to end, using a consumer that calls an external
   payment gateway as your example. *(Ch. 1, 4)*
3. A consumer needs all events for a given `orderId` processed in
   strict order, but the Kafka topic has 20 partitions. How do you
   achieve this without reducing parallelism for other orders? *(Ch. 2)*
4. Your consumer group's throughput drops sharply and repeatedly. You
   suspect rebalancing. What's the most likely root cause, and how do
   you fix it? *(Ch. 3)*
5. Why does `acks=all` alone not guarantee strong durability, and what
   additional setting is required? *(Ch. 4)*
6. Design the exchange and binding strategy in RabbitMQ for routing
   `payment.success`, `payment.failed`, and `payment.refunded` events
   to three different specialized consumers, plus one audit consumer
   that needs all payment events. *(Ch. 5)*
7. A message fails processing repeatedly due to a permanent data
   problem. Describe the full lifecycle you'd design for it in
   RabbitMQ, from first failure to final resolution. *(Ch. 6)*
8. Why is "JMS Topic" a potentially confusing term next to "Kafka
   topic," and what does each actually mean? *(Ch. 7)*
9. A workload needs 30 days of replay and handles 3 million
   events/second. A different workload needs complex, priority-based
   task routing at 50 messages/minute. Which technology fits each, and why? *(Ch. 8)*
10. Trace a single OrderPlaced event from Book 8's Outbox pattern relay
    through to three independent downstream consumers (billing,
    analytics, notifications) — naming which chapter's concept governs
    each part of that journey.

<details>
<summary>Answer Key</summary>

1. Asynchronous — the registration endpoint doesn't need to wait for
   the email to be sent to respond successfully to the user, and
   publishing a `UserRegistered` event lets a separate consumer handle
   email sending at its own pace, including gracefully handling a
   slow or briefly unavailable email provider without affecting
   registration latency at all.
2. Even if the broker guarantees exactly-once delivery to the
   consumer's application code (e.g., via Kafka transactions for a
   Kafka-to-Kafka flow), that guarantee stops at the boundary of the
   broker itself — it cannot make an external payment gateway call
   happen exactly once, since that side effect is entirely outside the
   broker's transactional scope. A crash between calling the gateway
   and committing the consumer's progress can still cause the gateway
   to be called twice on redelivery; only an idempotent design at the
   payment-call level (e.g., an idempotency key the gateway itself
   deduplicates on) actually closes this gap.
3. Use `orderId` as the Kafka message key — Kafka's hashing guarantees
   every message with the same key lands in the same partition and is
   therefore strictly ordered relative to other events for that same
   order, while different orders (different keys) still spread across
   all 20 partitions and are processed in parallel, so overall
   parallelism for the topic as a whole is unaffected.
4. Most likely, consumer processing time per poll batch is exceeding
   `max.poll.interval.ms`, causing Kafka to consider the consumer dead
   and evict it from the group, which triggers a rebalance; when the
   consumer catches up and rejoins, that triggers another one. Fix:
   reduce `max.poll.records` to shrink batch processing time, move
   genuinely slow work off the poll thread to run asynchronously, and
   set `max.poll.interval.ms` to a value realistic for actual
   processing time.
5. `acks=all` only requires all *current in-sync replicas* to
   acknowledge a write — if `min.insync.replicas` is left at a low
   value (e.g., 1) relative to the replication factor, a write can be
   considered durable after only one replica confirms it, which is a
   much weaker guarantee than it appears. Setting `min.insync.replicas`
   to at least 2 (with replication factor 3) is required for genuinely
   strong durability.
6. A topic exchange with routing keys like `payment.success`,
   `payment.failed`, `payment.refunded`. Bind a `success-queue` with
   binding key `payment.success`, a `failed-queue` with binding key
   `payment.failed`, a `refunded-queue` with binding key
   `payment.refunded` for the three specialized consumers, and bind an
   `audit-queue` with binding key `payment.#` to catch every payment
   event regardless of its specific type.
7. First failure: process, fail, and NACK/reject without requeue,
   routing to a retry queue with a short TTL (e.g., 10s) that
   dead-letters back to the original queue after expiry, incrementing a
   `x-retry-count` header on each attempt. After a capped number of
   attempts (e.g., 3), route to a dedicated dead-letter queue instead of
   retrying again, with monitoring/alerting on that DLQ's depth so the
   team is notified. Resolution: a human investigates the root cause
   (the "permanent data problem"), fixes it if possible, and manually
   reprocesses (or discards, with justification) the message from the DLQ.
8. A JMS Topic specifically denotes publish-subscribe (broadcast to all
   subscribers) semantics — one of JMS's two explicit messaging
   domains, alongside point-to-point queues. A Kafka topic is simply a
   named, partitioned category of messages with no inherent
   broadcast-versus-load-balanced behavior baked into the name — that
   depends entirely on how many consumer groups subscribe and how many
   instances each group has. The shared word "topic" describes
   genuinely different concepts in each system.
9. The 30-day-replay, 3-million-events/second workload fits Kafka —
   its retained log with configurable retention natively supports
   long-window replay, and partition-based parallelism scales to that
   throughput. The low-volume, complex priority-based routing workload
   fits RabbitMQ — its exchange/binding model naturally expresses
   conditional and priority-based routing at a scale where Kafka's
   throughput advantage provides no benefit and its routing model would
   be awkward to force into this shape.
10. The Outbox relay reads an unpublished row and publishes it with
    at-least-once semantics, marking it published only after a broker
    ack (Ch. 1, and Book 8 Ch. 6) → the event lands in a Kafka topic,
    keyed by `orderId` for ordering within that order's own event
    sequence (Ch. 2) → billing, analytics, and notifications each run
    their own consumer group subscribed to the same topic, each
    receiving a full independent copy of the event — broadcast via
    separate consumer groups (Ch. 3) → each consumer's actual
    processing should be idempotent, since relay retries or consumer
    redelivery can duplicate the event at either hop (Ch. 1, Ch. 4) →
    if any of these three downstream systems were RabbitMQ-based
    instead, the same fan-out would be expressed via a fanout or topic
    exchange to three independently bound queues (Ch. 5), with manual
    ack and dead-lettering protecting each consumer's reliability (Ch. 6).

</details>

---

## PART B — MOCK INTERVIEW: MESSAGING ROUND

**Interviewer:** "Your team currently uses RabbitMQ for everything. A
new requirement calls for ingesting 500,000 clickstream events per
second for real-time analytics, with a need to reprocess the last 7
days of data whenever the analytics model changes. Would you keep using
RabbitMQ, or introduce Kafka? Walk me through your reasoning."

**Model answer:** "I'd introduce Kafka specifically for this workload,
and I'd want to be clear this isn't a wholesale migration away from
RabbitMQ — it's recognizing that this particular requirement has two
signals pointing directly at Kafka's strengths: the throughput, 500,000
events per second, is well within Kafka's partition-based parallel
design but would strain a RabbitMQ setup whose per-message routing
overhead makes that scale harder to sustain efficiently; and the 7-day
reprocessing requirement is exactly what Kafka's retained log with
configurable retention is built for — RabbitMQ's consume-and-delete
model has no native way to replay data that's already been acknowledged
and removed. I'd keep RabbitMQ for whatever existing task-queue or
complex-routing use cases it's already serving well, since there's no
reason to migrate those just because a new, different workload favors
a different tool — this becomes a deliberate polyglot messaging
decision, not a broker replacement."

**Follow-up:** "The team is worried about the operational cost of
running two messaging systems. How do you respond?"
(That's a legitimate concern worth weighing explicitly — additional
on-call expertise, monitoring, and tooling are real costs — but the
alternative, force-fitting a 500K-events/sec replay-heavy workload into
RabbitMQ, would likely cost more in the long run through operational
strain and awkward custom replay engineering; recommend evaluating
whether this is a one-off need or the first of several future
event-streaming use cases, since that affects whether the investment is
justified.)

---

**Interviewer:** "A consumer in your payment-processing pipeline is
occasionally double-charging customers. Walk me through how you'd
diagnose this, given the pipeline uses Kafka with `acks=all` and a
manually-committed consumer offset."

**Model answer:** "The `acks=all` setting tells me producer-to-broker
durability is likely solid, so I'd focus on the consumer side first.
The most likely cause, given manual offset commits, is the classic
at-least-once duplicate scenario from Chapter 1 and 2: the consumer
processes the message — calling the payment gateway and charging the
customer — but crashes or is evicted from the group before it commits
the offset. On restart or rebalance, it (or another instance) receives
the same message again and, if the actual charge call isn't idempotent,
charges the customer a second time. I'd verify this by checking for
rebalance events or restarts correlated with the double-charge
incidents in the logs. The fix isn't about Kafka configuration at all —
it's making the payment gateway call idempotent, typically via an
idempotency key derived from the order or payment ID that the payment
gateway itself deduplicates on, so a redelivered message can't result
in a second real charge regardless of how many times it's reprocessed."

**Follow-up:** "What if the payment gateway doesn't support idempotency
keys?"
(Then implement idempotency locally: before calling the gateway, check
a durable "already charged for this payment ID" record; only call the
gateway if no such record exists, and write the record atomically with,
or immediately after, a successful charge — accepting a small window of
risk during that write that would need its own careful handling, e.g. a
database unique constraint on payment ID preventing a concurrent duplicate insert.)

---

**Interviewer:** "Explain to me, as if I'm a new engineer on your team,
when I should reach for RabbitMQ's dead-letter queue versus just
retrying a failed message immediately."

**Model answer:** "I'd start with the distinction between transient and
permanent failures. If a message fails because a downstream dependency
is momentarily overloaded or briefly unreachable, retrying makes sense —
but not immediately in a tight loop, since that can pile more load onto
an already-struggling dependency, which is the same retry-storm problem
we saw with synchronous calls back in the microservices book. Instead,
I'd use a delayed retry — routing the message to a retry queue with a
TTL that dead-letters back to the original queue after a short wait,
giving the dependency time to recover. If a message keeps failing after
a capped number of these delayed attempts — say, three — that's a
strong signal the failure isn't transient at all; maybe the message
itself has malformed data, or there's a bug in processing logic that
retrying will never fix. At that point, sending it to a dead-letter
queue for a human to investigate is the right move, rather than
retrying forever and letting a permanently broken message loop
indefinitely and consume resources."

---

## PART C — CAPSTONE PROJECT: "EVENT-DRIVEN NOTIFICATION AND ANALYTICS PLATFORM"

**Goal:** A messaging system demonstrating every chapter of Book 9
working together, building directly on Book 8's microservices
foundation.

**Requirements:**

1. Produce `OrderPlaced`, `OrderShipped`, and `OrderCancelled` events to
   a Kafka topic keyed by `orderId`, and demonstrate that all events for
   a single order arrive in order in the same partition even with
   multiple partitions configured (Ch. 2).
2. Implement three independent consumer groups (billing, analytics,
   notifications) reading the same topic, each processing the full
   event stream independently; demonstrate that stopping one group
   doesn't affect the others (Ch. 3).
3. Configure the producer with `acks=all`, `min.insync.replicas=2`, and
   `enable.idempotence=true`; write a test that forces a simulated
   broker acknowledgment failure and confirms no duplicate is written to
   the partition (Ch. 4).
4. Build a RabbitMQ-based customer-support ticket router using a topic
   exchange, with bindings routing `ticket.urgent.*` to a
   high-priority queue, `ticket.billing.*` to a billing-support queue,
   and `ticket.#` to a catch-all audit queue (Ch. 5).
5. Implement manual acknowledgment, a delayed retry (TTL + DLX, capped
   at 3 attempts), and a monitored dead-letter queue for the ticket
   router's consumers; demonstrate a permanently failing message ending
   up in the DLQ after exactly 3 attempts (Ch. 6).
6. Add one JMS-based (or documented conceptual design, if a JMS broker
   isn't available in your environment) integration point simulating a
   legacy warehouse system that only speaks JMS point-to-point queues,
   and explain in a comment why JMS was chosen there specifically (Ch. 7).
7. Write a one-page architecture decision record (ADR) justifying why
   Kafka was used for order events and RabbitMQ for ticket routing,
   using the workload-driven decision framework from Chapter 8 (Ch. 8).

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Ordering correctness | All events for one order are provably in order despite multiple partitions |
| Consumer group independence | Stopping one consumer group has zero effect on the others' processing |
| Producer reliability | The forced-failure test proves no duplicate write occurs |
| Routing correctness | Each RabbitMQ binding pattern correctly routes only its intended message types |
| Retry/DLQ correctness | The permanently-failing message reaches the DLQ after exactly 3 attempts, not before or endlessly |
| Decision justification | The ADR reasons from workload requirements, not from tool familiarity |

---

*(This completes BOOK 9 — KAFKA + RABBITMQ + MESSAGING. Book 10 — MongoDB
+ Redis + NoSQL — moves from asynchronous communication to the data
storage technologies that often sit on the other side of these
messaging pipelines: consumers reading events and writing into NoSQL
stores, or using Redis for caching and idempotency-tracking.)*
