# BOOK 7 — FINAL ASSESSMENT, JPA/HIBERNATE MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 7 CHAPTERS)

1. Explain why `order.getItems().add(item)` alone doesn't persist the
   relationship, and how the owning side determines the actual fix. *(Ch. 2)*
2. A list endpoint returning 100 orders makes 101 SQL queries. Name two
   valid fixes and one invalid "fix" that would make things worse. *(Ch. 3)*
3. Why does a JPQL constructor expression sidestep the N+1 problem
   entirely rather than just reducing it? *(Ch. 3, 4)*
4. Design the `@Version` and retry strategy for a "like count" increment
   endpoint under moderate concurrent load. *(Ch. 5)*
5. Your team enables second-level caching on a `User` entity across 5
   service instances. A user updates their profile picture, and it
   doesn't show up consistently across page loads. Diagnose. *(Ch. 6)*
6. Why does `hibernate.jdbc.batch_size` silently fail to batch inserts
   for an entity using `GenerationType.IDENTITY`? *(Ch. 6)*
7. A controller receives a `LazyInitializationException` when
   serializing an entity to JSON. Name the best fix and explain why
   extending `@Transactional` to the controller is not the recommended one. *(Ch. 7)*
8. Why is Lombok's `@Data` dangerous specifically for `@Entity` classes,
   even though it's fine for DTOs? *(Ch. 7)*
9. Trace the complete lifecycle of an entity from `new Order()` through
   `persist()`, a field mutation, transaction commit, and eventual
   detachment — naming which chapter's concept governs each transition.
10. Why does this book's guidance consistently point toward DTOs over
    entities for reads — connect this to at least three separate
    chapters' reasoning.

<details>
<summary>Answer Key</summary>

1. The `@OneToMany` side (`items`) is the inverse (`mappedBy`) side —
   Hibernate only persists relationship state from the owning
   `@ManyToOne` side; adding to the inverse collection alone never sets
   the child's foreign key. Fix: also call `item.setOrder(order)`
   (ideally via a helper method keeping both sides in sync).
2. Valid fixes: `JOIN FETCH`/`@EntityGraph` for the specific association
   causing the extra queries, or a JPQL constructor expression / DTO
   projection avoiding entity loading of that association entirely.
   Invalid "fix": switching the association to `FetchType.EAGER`, which
   doesn't eliminate the extra queries, just makes them unconditional and
   global across every place that entity is loaded.
3. A constructor expression selects only specific columns via a single
   SQL query with explicit JOINs for navigated fields — it never
   materializes full entities with lazy associations at all, so there's
   no possibility of triggering per-row follow-up queries; N+1
   fundamentally requires touching a lazy association per row, which
   simply doesn't happen here.
4. `@Version` on the entity holding the like count, with
   `@Retryable(retryFor = OptimisticLockException.class, maxAttempts =
   3)` around the increment operation — appropriate for moderate
   (not extreme) concurrency; under very high contention (a viral
   post), this specific counter might warrant pessimistic locking or an
   atomic database-level increment instead.
5. This is the second-level cache multi-instance staleness problem —
   each of the 5 instances has its own local cache; updating the profile
   picture on whichever instance handled that request doesn't invalidate
   the other 4 instances' cached copies, so subsequent requests
   load-balanced to different instances see stale data inconsistently.
6. `IDENTITY` generation requires the database to assign the ID at
   insert time, and Hibernate needs to know that ID immediately after
   each individual insert for subsequent entity operations — this
   conflicts with batching, which requires deferring the actual INSERT
   execution to send many together; `SEQUENCE` allows pre-allocating IDs
   in blocks, which is compatible with batching.
7. Best fix: map the entity to a DTO within the transactional service
   method, before the entity would otherwise leave the transaction
   boundary, and return the DTO from the controller. Extending
   `@Transactional` to the controller is discouraged because it blurs
   the web layer and persistence layer's separation of concerns, forcing
   the controller to understand transaction/session lifecycle management
   it shouldn't need to know about.
8. Lombok `@Data` generates `equals`/`hashCode` using all fields by
   default — for an entity, this means a mutable field can change the
   hash code after the entity is already stored in a `HashSet`/`HashMap`
   (breaking lookup), and the generated `equals` typically uses
   `getClass()` comparison, which fails against Hibernate proxies. DTOs
   are immutable and never proxied, so neither risk applies to them.
9. `new Order()` → Transient (Ch. 1) → `persist()` → Managed, tracked
   by the persistence context / first-level cache (Ch. 1) → field
   mutation → detected via dirty checking against the loaded snapshot
   (Ch. 1) → commit → flush sends the UPDATE, transaction finalizes (Ch.
   1) → persistence context closes at the transaction boundary → entity
   becomes Detached (Ch. 1); if later `merge()`'d, its `@Version` (Ch. 5)
   is checked against the current database state.
10. Book 4 Chapter 3: entities carry lazy-loading and over-exposure
    risks when serialized directly. Book 7 Chapter 3: DTO projections
    eliminate N+1 and fetch-strategy concerns entirely. Book 7 Chapter 7:
    entities that leave the transaction boundary risk
    `LazyInitializationException`, and DTOs sidestep the entity
    equals/hashCode trap entirely by being simple, immutable, unproxied
    objects — three independent chapters converging on the same
    architectural recommendation from different angles.

</details>

---

## PART B — MOCK INTERVIEW: JPA/HIBERNATE ROUND

**Interviewer:** "A code review shows a REST endpoint returning a JPA
entity directly, annotated `@OneToMany(fetch = FetchType.EAGER)` on one
of its collections 'to avoid LazyInitializationException.' Walk me
through everything you'd flag."

**Model answer:** "Several things, in order of how much I'd prioritize
them. First, returning the entity directly at all — per Book 4 and this
book's Chapter 7, entities shouldn't cross the controller boundary; a DTO
avoids the entire lazy-loading question. Second, even setting aside the
DTO issue, switching to EAGER as a fix for `LazyInitializationException`
doesn't actually solve the underlying problem well — it just forces that
association to load every single time this entity loads anywhere in the
codebase, which can silently add overhead to unrelated code paths and,
if there's ever a second EAGER collection on the same entity, risks a
cartesian product from Chapter 3. Third, `LazyInitializationException`
itself was a useful signal here — it was telling the team the entity was
being serialized outside its transaction boundary, and the correct fix
is mapping to a DTO within the transaction, not silencing the symptom
with a global fetch-strategy change. I'd propose replacing the whole
approach with a `SELECT NEW` constructor expression or a manually-mapped
DTO, built while the persistence context is still open."

**Follow-up:** "What if the team says 'we need the full entity graph
for many different views, a DTO for each is too much duplication'?"
(A shared base DTO/projection interface for common fields, with
view-specific extensions, or Spring Data's dynamic projections — this is
a legitimate design conversation, but the answer is still "design better
DTOs," not "expose the entity.")

---

**Interviewer:** "Explain, as if teaching a junior developer, why they
never need to call `save()` after modifying a loaded entity's field
inside a `@Transactional` method — and what could go wrong if they
misunderstand this."

**Model answer:** "I'd explain dirty checking directly: when Hibernate
loads an entity, it keeps a snapshot of its state. When you change a
field on that entity — as long as it's still managed, meaning still
inside an active persistence context — Hibernate compares the current
state to that snapshot when the transaction is about to commit, and
automatically generates the UPDATE for whatever changed. So
`order.setStatus(SHIPPED)` inside a `@Transactional` method is genuinely
enough. What goes wrong with misunderstanding: I've seen developers
who don't trust this call `entityManager.merge()` defensively on an
already-managed entity — usually harmless but unnecessary and sometimes
subtly wrong if the entity somehow isn't the same instance Hibernate is
tracking. The more dangerous misunderstanding runs the other way: modifying
a *detached* entity's field and expecting it to save automatically — since
there's no active persistence context tracking that specific instance,
dirty checking simply doesn't apply, and the change silently does
nothing without an explicit `merge()`."

---

**Interviewer:** "Design the fetching and DTO strategy for an order
management dashboard's three views: (1) a paginated list showing 20
orders with id, customer name, total, status; (2) an order detail view
showing full items, customer info, and payment history; (3) an analytics
report showing revenue by category using a window function."

**Model answer:** "View 1 is a textbook case for a JPQL constructor
expression — `SELECT NEW OrderSummaryDto(o.id, o.customer.name, o.total,
o.status) FROM Order o`, one query, no entity loading, no fetch strategy
decisions needed at all, and naturally paginated via `Pageable`. View 2
genuinely needs the richer object graph, so I'd fetch-join the customer
and one collection — say items — in a single query
(`JOIN FETCH o.customer JOIN FETCH o.items`), and handle payment history
via a second, separate query or batch fetching, specifically to avoid the
cartesian product from stacking two collection fetch-joins together, per
Chapter 3. View 3 needs a window function and possibly a CTE for
category-level revenue aggregation — JPQL can't express window functions
well, so I'd use native SQL here, accepting the portability trade-off,
mapped directly into a reporting DTO."

---

## PART C — CAPSTONE PROJECT: "ORDER MANAGEMENT PERSISTENCE LAYER"

**Goal:** A Spring Boot + JPA project demonstrating every chapter of
Book 7 working together correctly, building directly on Book 6's schema
design.

**Requirements:**

1. Model `Customer`, `Order`, `OrderItem`, `Product` with correct owning
   sides, cascade settings (composition vs reference relationships
   distinguished explicitly, with comments justifying each), and
   `@Version` on `Order` and `Product` (Ch. 1, 2, 5).
2. Implement and demonstrate the N+1 problem deliberately (a naive
   endpoint), then fix it three ways in three separate endpoints: `JOIN
   FETCH`, `@EntityGraph`, and a JPQL constructor expression — with a
   test asserting query count for each (Ch. 3).
3. Implement a dynamic order search using Spring Data `Specification`
   with at least three optional filters (Ch. 4).
4. Implement a stock-reservation method using `@Lock(PESSIMISTIC_WRITE)`
   and a profile-update method relying on `@Version`, with tests
   demonstrating each under simulated concurrency (Ch. 5).
5. Configure `hibernate.jdbc.batch_size` correctly (verify your ID
   generation strategy is compatible) for a bulk order-import feature (Ch. 6).
6. Set `spring.jpa.open-in-view=false`, and ensure every endpoint uses
   DTOs — no `LazyInitializationException` should be possible by
   construction. Implement correct `equals()`/`hashCode()` on every
   entity following section 7.3's pattern (Ch. 7).

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Relationship correctness | Helper methods keep both sides of every bidirectional relationship in sync |
| N+1 fixes | All three fix approaches genuinely reduce query count, verified by tests |
| Locking correctness | Concurrency tests demonstrate both `@Version` conflicts and `@Lock` serialization actually occurring |
| OSIV discipline | `open-in-view=false` with zero `LazyInitializationException`s anywhere |
| Entity equals/hashCode | Uses `instanceof` + ID + constant hashCode, not Lombok `@Data` |

---

*(This completes BOOK 7 — JPA + HIBERNATE. Book 8 — Microservices +
Spring Cloud — moves from a single service's persistence layer to
distributed systems spanning many services.)*
