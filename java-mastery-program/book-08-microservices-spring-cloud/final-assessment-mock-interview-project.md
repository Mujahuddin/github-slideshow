# BOOK 8 — FINAL ASSESSMENT, MICROSERVICES MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 8 CHAPTERS)

1. A team splits a monolith along technical layers (a "UserService",
   an "OrderService", a "PaymentService" that all share one database)
   instead of along business capabilities. Explain why this isn't real
   decomposition and what a bounded-context-driven split would look
   like instead. *(Ch. 1)*
2. Why does hardcoding a downstream service's host:port break the moment
   you need to run two instances of it, and how do Eureka + a client-side
   load balancer solve this without the caller ever knowing an IP
   changed? *(Ch. 2)*
3. A Feign client call to Inventory Service is wrapped in a
   `@CircuitBreaker`, `@Retry`, and `@TimeLimiter` all at once. Which
   order should they compose in, and what goes wrong if `@Retry` sits
   outside `@CircuitBreaker`? *(Ch. 3, 4)*
4. Why is a blind, unconditional retry on every failed downstream call
   dangerous during a partial outage — connect this to a concept from
   Book 1's concurrency chapters. *(Ch. 4)*
5. Design a Saga for "place order" spanning Order, Payment, and
   Inventory services. Choose choreography or orchestration, justify it,
   and write the compensating action for a failed payment step. *(Ch. 5)*
6. Why is a compensating transaction not a real "undo," and what
   customer-facing consequence does this have for a Saga that has to
   compensate after an email confirmation was already sent? *(Ch. 5)*
7. Explain the dual-write problem for "save order to DB, then publish
   OrderPlaced to a message broker" and how the Transactional Outbox
   pattern eliminates it without a distributed transaction. *(Ch. 6)*
8. A request fails somewhere across 6 services and nobody can tell
   which one. What specifically does distributed tracing add that
   centralized logging alone doesn't, and what is the role of the
   W3C Trace Context header in making it work across service boundaries? *(Ch. 7)*
9. A team adds CQRS to a simple internal admin CRUD tool "for future
   scalability." Explain why this is likely the wrong call, referencing
   the same architectural caution from Book 1 Chapter 2. *(Ch. 8)*
10. Trace a single "place order" request through this entire book's
    architecture — service discovery, Feign call, resilience wrapping,
    Saga coordination, outbox-based event publishing, and a trace
    span — naming which chapter governs each hop.

<details>
<summary>Answer Key</summary>

1. Sharing one database across "services" split by technical layer means
   none of them can be deployed, scaled, or changed independently — a
   schema change for orders can break the shared UserService, and there's
   no real ownership boundary. A bounded-context split groups by business
   capability (Order Management, Payments, Inventory), each owning its
   own data and schema, so a change inside one context never requires
   coordinating a deploy with another.
2. Hardcoding an address assumes exactly one instance exists at a fixed
   location forever; the moment a second instance starts (for scaling or
   failover) or the existing one restarts on a new IP, the hardcoded
   reference silently breaks. Eureka lets instances register themselves
   and lets a client-side load balancer (e.g. Spring Cloud LoadBalancer)
   query the current instance list by logical service name at call time,
   so the caller only ever addresses a name — the actual instance
   resolution and distribution happens underneath, invisibly.
3. `@CircuitBreaker` should wrap `@Retry`, which wraps `@TimeLimiter`
   (outermost to innermost). If `@Retry` sits outside `@CircuitBreaker`,
   each retry attempt is a fresh call that can still reach a downstream
   service the circuit breaker would otherwise be protecting — so instead
   of failing fast once the breaker opens, the caller keeps retrying
   into an already-known-bad dependency, amplifying load on a service
   that's already struggling.
4. Blind retries during a partial outage multiply the request rate every
   downstream call receives — if 10 callers each retry 3 times, a
   struggling service now receives 40 requests instead of 10, which is
   the same amplification-under-contention failure mode as unbounded
   thread creation from Book 1's concurrency chapters: both turn a
   resource that's already stressed into one that's overwhelmed, because
   nothing bounds how much additional load gets added on top of failure.
5. Choreography (each service reacts to the previous service's event
   independently, no central coordinator) suits this if the steps are
   simple and services should stay decoupled; orchestration (a central
   saga orchestrator issuing commands and tracking state) suits it if the
   flow has many steps, needs central visibility, or has complex
   conditional branching. Either way, the compensating action for a
   failed payment step is not "delete the order" — it's issuing a
   `ReleaseInventoryReservation` command to Inventory Service to reverse
   its earlier reservation, since the Saga can only run compensating
   business actions, not roll back another service's database.
6. Compensation runs a new, forward-moving business operation that
   reverses the *effect* of an earlier step, not a database rollback —
   the earlier step's side effects that already left the system's
   boundary (an email sent, a webhook fired, a third-party charge
   already visible on a bank statement) cannot be un-sent. Practically,
   if payment fails after an "Order Confirmed" email already went out,
   the compensating flow must also send a "sorry, your order was
   cancelled" follow-up — the customer experience has to account for the
   fact that the first message already happened and can't be recalled.
7. If code saves the order row and then separately calls the broker to
   publish `OrderPlaced`, either step can succeed while the other fails
   (a crash between them, or a broker timeout) — no single transaction
   spans both a database and a message broker, so the two states can
   drift apart with no way to detect it. The Outbox pattern writes the
   event into an `outbox` table in the *same* local database transaction
   as the order save, so both either commit together or neither does;
   a separate poller/relay process then reads unpublished outbox rows
   and publishes them to the broker asynchronously, retrying safely
   since publishing is idempotent against the already-durable row.
8. Centralized logging aggregates each service's own logs, but each
   service only knows about itself — nothing inherently links "this log
   line in Payment Service" to "the request that started in the API
   Gateway five hops earlier." Distributed tracing adds a trace ID that
   is generated once at the entry point and propagated through every
   downstream call, plus per-service span IDs recording each hop's
   timing and outcome, so a single trace ID reconstructs the entire
   request's path and pinpoints exactly which span failed or was slow.
   The W3C Trace Context `traceparent` header standardizes how that
   trace ID and parent span ID are carried in HTTP headers across
   service and even vendor boundaries, so instrumentation from different
   libraries/languages can still participate in the same trace.
9. Book 1 Chapter 2 warned against building abstractions for
   requirements that don't exist yet ("speculative generality"). CQRS
   adds real, ongoing complexity — separate write and read models,
   eventual consistency between them, more moving parts to operate — that
   only pays off under a specific pressure: read and write workloads
   with very different scaling or modeling needs. An internal admin CRUD
   tool has neither high read/write asymmetry nor complex read models,
   so the added complexity has no corresponding benefit — the same
   "don't build for a future that may never arrive" mistake, this time
   at the architecture-pattern level instead of the code level.
10. Client resolves the target service name via Eureka (Ch. 2) → a
    Feign client, backed by resilience annotations composed
    CircuitBreaker→Retry→TimeLimiter (Ch. 3, 4), makes the call → if the
    flow spans multiple services with compensating steps, a Saga
    coordinates it, choreography or orchestration (Ch. 5) → each
    service's local state change and any resulting event is written via
    the Outbox pattern in one local transaction, then relayed
    asynchronously (Ch. 6) → every hop carries a propagated trace context
    header, producing spans that reconstruct the full request path for
    observability (Ch. 7) → if the read side of any of this is modeled
    separately from the write side for scaling reasons, that's CQRS,
    applied only where the asymmetry actually justifies it (Ch. 8).

</details>

---

## PART B — MOCK INTERVIEW: MICROSERVICES ROUND

**Interviewer:** "Your Order Service calls Inventory Service via Feign.
During a deploy, Inventory Service is unavailable for 45 seconds. What
happens to Order Service, and what should you have built in advance to
prevent it from cascading?"

**Model answer:** "Without protection, every request to Order Service
that needs Inventory Service blocks or fails during that window, and if
those calls hold threads waiting on a timeout, Order Service's own
thread pool can exhaust — which means Order Service starts failing
requests that have nothing to do with Inventory Service at all, purely
because it ran out of capacity waiting on a dependency. That's the
cascading failure this book keeps coming back to. What should already be
in place: a `@CircuitBreaker` around the Feign call so that after a
threshold of failures it stops calling Inventory Service entirely for a
cooldown window and fails fast with a fallback instead of piling up
blocked threads; a `@TimeLimiter` so individual calls don't hang
indefinitely; and a `@Bulkhead` isolating the thread pool used for this
specific downstream call so exhaustion there can't consume the threads
other endpoints need. The fallback itself should be a genuine business
decision — maybe 'show cached inventory levels with a staleness
warning' — not just swallowing the error silently."

**Follow-up:** "The circuit breaker's fallback for inventory availability
returns 'assume in stock.' Is that a good idea?"
(Depends entirely on the business cost of being wrong in each direction —
for a high-value or limited-stock item, "assume in stock" risks
overselling and a worse customer experience than a clear "please try
again" error; for a high-availability item, it's probably fine. This is
exactly the kind of question that should never be answered unilaterally
by whoever wires up the annotation — it needs product/business input.)

---

**Interviewer:** "Walk me through how you'd add distributed tracing to
an existing 6-service system that currently only has per-service logs,
without a big-bang rewrite."

**Model answer:** "I'd start by adding tracing instrumentation
(Micrometer Tracing with a W3C-compliant propagator) to each service one
at a time, since it's additive — a service without it yet simply doesn't
contribute spans, it doesn't break anything for services that already
have it. The key thing to get right early is making sure every outbound
call — Feign clients, message publishers — actually propagates the
`traceparent` header instead of silently dropping context at a hop,
because a single un-instrumented hop breaks the trace chain right there.
I'd also correlate this with existing logs by injecting the trace ID
into each service's log format, so during the transition teams can still
cross-reference a trace ID against the older log-based debugging
workflow they're used to. Once all 6 are instrumented and exporting to a
collector, I'd wire up a tracing backend for the actual visualization
and start using it for real incidents to build trust in it before
retiring any old workflow."

**Follow-up:** "How does this connect to the observability pillars from
Chapter 7 — is tracing enough on its own?"
(No — tracing shows a single request's path and timing; metrics show
aggregate system health and trends (error rates, latency percentiles);
logs show detailed context for a specific event. They answer different
questions and are complementary, not substitutes for each other.)

---

**Interviewer:** "A Saga for order placement has 4 steps across 3
services. After step 3, a bug causes the compensating transaction for
step 2 to also fail. What are your options, and what would you actually
recommend?"

**Model answer:** "This is the hard edge case Sagas have to plan for
explicitly — a failed compensation. Options: retry the compensation with
backoff, since many compensation failures are transient (a temporary
network issue, a downstream restart); if it keeps failing, escalate to a
dead-letter/manual-intervention queue rather than silently giving up,
since at that point the system is in a genuinely inconsistent state that
needs a human or an automated reconciliation job to resolve; and,
critically, make sure the Saga's own state is durably tracked throughout
so it's possible to know exactly which steps succeeded, which failed,
and which compensations are still outstanding — without that, there's no
way to even know what needs manual attention. What I wouldn't recommend
is treating this as rare enough to skip planning for — any Saga design
review should explicitly ask 'what happens if the compensation itself
fails,' because assuming compensations always succeed is the same class
of mistake as assuming the primary action always succeeds."

---

## PART C — CAPSTONE PROJECT: "DISTRIBUTED ORDER PLATFORM"

**Goal:** A multi-service system demonstrating every chapter of Book 8
working together, building on the single-service persistence layer from
Book 7.

**Requirements:**

1. Decompose into at least 3 services (Order, Payment, Inventory) along
   bounded contexts, each with its own database — no shared schema
   (Ch. 1).
2. Register all services with Eureka and use a Config Server for shared
   configuration; demonstrate a config change propagating without a
   redeploy (Ch. 2).
3. Order Service calls Inventory and Payment via Feign clients using
   service-name resolution, never hardcoded addresses (Ch. 3).
4. Wrap every Feign call in composed Resilience4j annotations
   (CircuitBreaker → Retry → TimeLimiter, plus a Bulkhead on at least one
   call), with meaningful business-level fallbacks — and a test that
   forces the circuit open and verifies fail-fast behavior (Ch. 4).
5. Implement the order-placement flow as a Saga (choreography or
   orchestration — justify your choice in a comment) with a working
   compensating transaction for a simulated payment failure (Ch. 5).
6. Use the Transactional Outbox pattern for publishing `OrderPlaced` and
   `OrderCancelled` events — no direct dual-write from a service method
   to the broker (Ch. 6).
7. Instrument all 3 services with distributed tracing propagating a
   trace context across every Feign call and event publish/consume;
   produce one complete trace spanning all 3 services for a single order
   (Ch. 7).
8. Identify one genuine read/write asymmetry in this system (e.g. an
   order-history read view aggregating data from all 3 services) and
   implement it as a separate read model — with a one-paragraph
   justification for why this specific case warrants CQRS while the rest
   of the system correctly doesn't use it (Ch. 8).

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Decomposition | Each service's database is genuinely private; no cross-service SQL joins exist anywhere |
| Resilience | Forcing a downstream failure demonstrates fail-fast behavior, not cascading thread exhaustion |
| Saga correctness | A forced mid-saga failure results in all prior steps being correctly compensated, not a half-applied state |
| Outbox correctness | Killing the process between the DB write and the publish step never loses or duplicates the event after restart |
| Tracing completeness | One trace ID reconstructs the full cross-service path with no missing hops |
| CQRS restraint | CQRS is used in exactly the one justified place, not applied uniformly "for consistency" |

---

*(This completes BOOK 8 — MICROSERVICES + SPRING CLOUD. Book 9 — Kafka +
RabbitMQ + Messaging — goes deeper into the asynchronous, event-driven
communication this book's Outbox pattern only introduced the surface of.)*
