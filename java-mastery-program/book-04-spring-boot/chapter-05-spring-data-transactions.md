# CHAPTER 5 — SPRING DATA BASICS & TRANSACTIONS DEEP DIVE

> **Scope note:** This chapter covers `@Transactional` in real depth
> (propagation, isolation, rollback rules) and just enough Spring Data
> repository usage to build working endpoints. Full JPA/Hibernate
> internals — entity lifecycle, N+1 queries, fetch strategies, locking —
> are Book 7's dedicated territory.

---

## 5.1 CONCEPT: Spring Data Repositories — The Interface-Only Magic, Demystified

### TELUGU EXPLANATION

```java
interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerIdAndStatus(String customerId, OrderStatus status); // implementation ఏమీ రాయలేదు!
    Optional<Order> findByOrderNumber(String orderNumber);
}
```

**ఇక్కడ implementation ఎక్కడ ఉంది?** — ఇది Book 3 Chapter 1 లో మనం
నేర్చుకున్న **proxy mechanism** యొక్కే మరో application: Spring Data
runtime లో ఈ interface కి ఒక **dynamic proxy** create చేస్తుంది.
Method పేరు ని **parse** చేసి (`findByCustomerIdAndStatus` → "customerId
field తో, status field తో filter చేయి"), దాన్నుండి ఒక JPQL/SQL query
**automatic గా generate** చేస్తుంది. `JpaRepository<Order, Long>`
extend చేయడం వల్ల, `save()`, `findById()`, `findAll()`, `deleteById()`
లాంటి CRUD methods కూడా **ఉచితంగా** వస్తాయి.

**ఇది "మీరు రాయని code" అనేది గుర్తుపెట్టుకోండి** — method పేరు తప్పుగా
రాస్తే (`findByCustmerId` typo), **compile-time లో ఏ error రాదు**
(interface method గా valid గానే ఉంటుంది), కానీ **startup వద్ద** Spring
Data ఆ field ని entity లో వెతికి, దొరకకపోతే `PropertyReferenceException`
throw చేస్తుంది — ఇది కూడా Book 4 Chapter 1 "fail fast at startup"
సూత్రమే.

### ENGLISH INTERVIEW ANSWER

"Spring Data repositories work through the same proxy mechanism as Spring
AOP — at startup, Spring Data creates a dynamic proxy implementing the
repository interface, and for query-derivation methods like
`findByCustomerIdAndStatus`, it parses the method name to derive a query
against the entity's fields automatically. Extending `JpaRepository`
provides standard CRUD operations for free without writing any
implementation. The interesting failure mode here is that a typo in a
derived query method name — `findByCustmerId` — compiles fine, since
it's just a valid interface method signature, but fails at application
startup with a `PropertyReferenceException` once Spring Data tries to
match it against the entity's actual fields, which is exactly the 'fail
fast, not silently wrong' pattern we've seen elsewhere in Spring Boot."

---

## 5.2 CONCEPT: `@Transactional` Propagation — What Actually Happens Across Method Boundaries

### TELUGU EXPLANATION

**Propagation** అంటే, ఒక `@Transactional` method, ఇప్పటికే **transaction
active గా ఉన్న context** లో call అయినప్పుడు, ఏం జరగాలో నిర్ణయించడం.

| Propagation | ప్రవర్తన | ఎప్పుడు వాడాలి |
|---|---|---|
| **REQUIRED** (default) | Existing transaction ఉంటే, దాన్నే వాడు; లేకపోతే కొత్తది create చేయి | చాలా cases కి డిఫాల్ట్, సరైనది |
| **REQUIRES_NEW** | ఎప్పుడూ **కొత్త, independent** transaction create చేయి — existing దాన్ని suspend చేసి | Audit logging లాంటివి — outer transaction rollback అయినా, ఈ log entry ఉండాలి అనుకున్నప్పుడు |
| **NESTED** | Existing transaction లోపల ఒక **savepoint** create చేయి — ఈ భాగం fail అయితే, ఈ savepoint వరకు మాత్రమే rollback | Partial rollback కావాల్సినప్పుడు, outer transaction మొత్తం fail అవ్వకుండా |
| **MANDATORY** | Existing transaction తప్పకుండా ఉండాలి, లేకపోతే exception | "ఈ method ఎప్పుడూ ఒక transaction లోపలే call అవ్వాలి" అని enforce చేయడానికి |
| **NOT_SUPPORTED** | Transaction ని suspend చేసి, non-transactional గా run అవ్వు | Long-running, read-only reporting queries |

**అత్యంత ముఖ్యమైన practical ఉదాహరణ — `REQUIRES_NEW` ఎందుకు అవసరం:**

```java
@Service
class OrderService {
    private final AuditLogService auditLogService; // వేరే bean — Book 3 Chapter 4 self-invocation సూత్రం గుర్తుంచుకోండి

    @Transactional
    void placeOrder(Order order) {
        try {
            // ... order placement logic, DB writes ...
            processPayment(order); // fail అవ్వొచ్చు
        } finally {
            auditLogService.logAttempt(order); // ఇది ఎప్పుడూ persist అవ్వాలి, placeOrder rollback అయినా సరే!
        }
    }
}

@Service
class AuditLogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW) // ఇండిపెండెంట్ transaction
    void logAttempt(Order order) {
        auditRepository.save(new AuditEntry(order)); // ఇది commit అవుతుంది, placeOrder rollback అయినా
    }
}
```

**ఎందుకు ఇది `REQUIRES_NEW` లేకుండా పని చేయదు:** `placeOrder()` fail
అయితే (payment fail అయితే), దాని transaction rollback అవుతుంది — అదే
transaction లో ఉన్న ఏ DB write అయినా (audit log సహా) rollback
అవుతుంది. `REQUIRES_NEW` ఈ audit log write ని **వేరే, independent
transaction** లోకి తీసుకువెళ్తుంది — outer transaction rollback
అయినా, ఈ inner transaction commit అయ్యే అవకాశం ఉంటుంది.

### ENGLISH INTERVIEW ANSWER

"Propagation determines how a transactional method behaves when called
from within an already-active transaction. `REQUIRED`, the default, joins
the existing transaction — most methods want this. `REQUIRES_NEW` is the
one I reach for deliberately when I need something to survive regardless
of the outer transaction's outcome — audit logging is the textbook case:
if an order placement fails and rolls back, I still want a record that the
attempt happened, which requires suspending the outer transaction and
running the audit write in a genuinely separate, independently-committed
transaction. Getting this wrong — using `REQUIRED` when you needed
`REQUIRES_NEW` — is a real, subtle production bug: your audit trail
silently loses exactly the failure cases you most need visibility into,
since they roll back along with everything else in the same transaction."

---

## 5.3 CONCEPT: Rollback Rules — The Default That Surprises Everyone

### TELUGU EXPLANATION

**ఇది అత్యంత frequently-missed production gotcha:** `@Transactional`
**default గా, checked exceptions మీద rollback చేయదు** — కేవలం
**unchecked (RuntimeException) exceptions** మీద మాత్రమే automatic గా
rollback చేస్తుంది (Book 1 Chapter 6 checked/unchecked distinction ఇక్కడ
నేరుగా, ఆచరణాత్మకంగా వర్తిస్తుంది).

```java
@Transactional
void placeOrder(Order order) throws InvalidOrderException { // ❌ checked exception
    if (!isValid(order)) {
        throw new InvalidOrderException("Invalid order"); // ❌ ROLLBACK జరగదు! Transaction commit అవుతుంది!
    }
    orderRepository.save(order); // ఇది save అవుతుంది, exception వచ్చినా!
}
```

**Fix:** `rollbackFor` explicitly specify చేయాలి:
```java
@Transactional(rollbackFor = InvalidOrderException.class) // ఇప్పుడు checked exception మీద కూడా rollback
void placeOrder(Order order) throws InvalidOrderException { ... }
```

**Senior గా, ఈ surprise ని ఎందుకు avoid చేయవచ్చు:** Book 1 Chapter 6
లో మనం చెప్పినట్టు, **చాలా modern Spring codebases unchecked exceptions
నే default గా వాడతాయి** — ఈ rule ని precisely గుర్తుపెట్టుకోవాల్సిన
అవసరం చాలావరకు తగ్గుతుంది, ఎందుకంటే మీ domain exceptions అన్నీ
`RuntimeException` extend చేస్తే, default rollback behavior మీరు
expect చేసినట్టే పని చేస్తుంది.

### ENGLISH INTERVIEW ANSWER

"This is one of the most commonly missed `@Transactional` details:
Spring's default rollback rule only covers unchecked exceptions —
`RuntimeException` and its subtypes. A checked exception thrown from a
transactional method does NOT trigger a rollback by default; the
transaction commits anyway, potentially leaving partial, invalid data
persisted. The fix is explicit — `@Transactional(rollbackFor =
SomeCheckedException.class)`. In practice, this is exactly one more reason
I favor unchecked domain exceptions for business logic failures, as
discussed in Book 1's exception handling chapter — it means the default
rollback behavior already matches what I actually want, without needing
to remember `rollbackFor` on every single transactional method that might
throw a checked exception."

---

## 5.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Audit logging inside a transactional method | Uses default `REQUIRED` propagation, wonders why logs vanish on failure | Uses `REQUIRES_NEW` for logs that must survive rollback |
| Checked exception in a `@Transactional` method | Assumes any exception triggers rollback | Knows the default only covers unchecked exceptions; uses `rollbackFor` or switches to unchecked |
| Repository method typo | Assumes it would be caught immediately | Knows it fails at startup with `PropertyReferenceException`, not compile time |
| `@Transactional` on a method called via `this` | Doesn't connect this to Book 3's AOP material | Immediately recognizes the self-invocation problem still applies here |

---

## 5.5 COMMON MISTAKES

1. Assuming `@Transactional` rolls back on any exception — it doesn't, by
   default, for checked exceptions.
2. Using default `REQUIRED` propagation for logging/auditing that needs
   to survive the outer transaction's rollback.
3. Forgetting that `@Transactional` is AOP-proxy-based (Book 3 Chapter 4)
   — self-invocation silently disables it here just as much as for any
   other aspect.
4. Overusing `REQUIRES_NEW`, creating many small independent transactions
   where one larger transaction would be simpler and more consistent.
5. Not understanding that Spring Data query-derivation method name typos
   fail at startup, not compile time — assuming a passing compile means
   correctness.

---

## 5.6 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. What does `JpaRepository` give you for free? 2. What
exception types trigger automatic rollback by default?

**Intermediate:** 3. Explain the difference between `REQUIRED` and
`REQUIRES_NEW` with a concrete example. 4. Why does a Spring Data query
method with a typo compile successfully but fail at runtime?

**Senior:** 5. Design a scenario requiring `NESTED` propagation and
explain why `REQUIRES_NEW` wouldn't be the right fit for it. 6.
Explain, precisely, why `@Transactional` on a method called via `this`
from within the same class doesn't apply — tie this back to Book 3 Chapter 4.

**Architect:** 7. You're reviewing a service with `@Transactional`
methods calling other `@Transactional` methods across five different
beans, several using `REQUIRES_NEW`. How would you audit this for
correctness, and what documentation/diagramming approach would help the
team reason about the actual transaction boundaries?

**Scenario:** 8. A production bug report: after a failed order,
inconsistent partial data is found in the database — some related rows
were saved, others weren't, and the developer swears `@Transactional` was
applied to the whole method. What are the two most likely root causes
given this chapter?

**Trick:** 9. "Adding `@Transactional` to a method guarantees that method
is atomic with respect to all its database operations, regardless of
what else calls it or how." True or false?

<details><summary>Key answers</summary>

- Q5: A scenario processing a batch of independent sub-items where each
  sub-item's failure should roll back only that sub-item's changes, but
  the overall batch should still commit whatever succeeded, all as part of
  one logical unit of work — `NESTED` uses savepoints within the same
  underlying transaction/connection, allowing partial rollback to a
  savepoint, whereas `REQUIRES_NEW` would use an entirely separate
  transaction/connection, losing the "still part of one overall unit of
  work, can be fully rolled back together if needed" property that
  `NESTED` preserves.
- Q6: `@Transactional` is implemented via the exact same AOP proxy
  mechanism as any other aspect (Book 3 Chapter 4) — a call via `this`
  goes directly to the target object, bypassing the proxy that would
  otherwise start/commit/rollback the transaction, so the annotation is
  silently ignored in that call path, identical in mechanism to the
  general self-invocation problem, just applied specifically to transactions.
- Q7: Diagnose which propagation setting each method actually needs
  given its real requirement (survive outer rollback? part of one atomic
  unit? require an existing transaction?), and produce a call-graph
  diagram annotated with each method's propagation setting — this makes
  otherwise-invisible transaction boundaries explicit and reviewable,
  since the actual behavior can't be inferred just by reading business
  logic without also tracking propagation settings across the whole call chain.
- Q8: Most likely: (1) self-invocation — some of the "transactional"
  methods were actually called via `this` and their `@Transactional`
  never applied at all, or (2) a checked exception was thrown somewhere
  in the flow without `rollbackFor` specified, so the transaction
  committed the already-completed writes instead of rolling them back.
- Q9: False — atomicity only holds within the actual transactional
  boundary Spring establishes via the proxy, which depends entirely on
  how the method is called (self-invocation bypasses it), what propagation
  setting is used (REQUIRES_NEW deliberately creates a separate atomic
  unit), and whether the specific exception thrown is covered by the
  rollback rules in effect — "atomic no matter what" is precisely the
  false assumption this chapter is built to correct.

</details>

---

## 5.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does the default `@Transactional` rollback behavior only cover unchecked exceptions, and how does this connect to Book 1's checked-vs-unchecked design philosophy?
- **Coding Check:** Implement the audit-logging example from section 5.2 with correct `REQUIRES_NEW` usage, and write a test demonstrating the audit entry survives even when the outer transaction rolls back.
- **Explanation Check:** Explain in English why a Spring Data repository method typo is a startup-time failure, not a compile-time or runtime-during-normal-operation failure.
- **Real-World Check:** Your team's payment reconciliation job needs each individual reconciliation record processed independently — one bad record shouldn't roll back the other 999 good ones in the same batch. Design the transaction boundaries.
- **Senior Check:** When would you deliberately AVOID wrapping a large multi-step operation in one single `@Transactional` method, even though it would guarantee atomicity?
- **Master Check:** Design the propagation and rollback strategy for a complete order-checkout flow: charge payment, reserve inventory, create order record, send confirmation event — specify which steps share a transaction, which need `REQUIRES_NEW`, and what happens on partial failure at each step.

<details><summary>Answers</summary>

- Real-World Check: Process each record in its own `@Transactional`
  method call (e.g., a separate bean method invoked in a loop from a
  non-transactional orchestrating method) so each gets its own independent
  transaction boundary — a failure in one record's transaction rolls back
  only that record, leaving the other 999 unaffected, rather than one
  giant transaction around the whole batch.
- Senior Check: When the operation is long-running and holding one
  database transaction/connection open for its entire duration would tie
  up connection pool resources excessively (Book 1 Chapter 10's connection
  pool exhaustion material) — for genuinely long multi-step workflows,
  breaking into smaller transactional units with compensating actions
  (Saga pattern, Book 8) is often more appropriate than one giant atomic transaction.
- Master Check: Charge payment and reserve inventory and create the order
  record ideally share one transaction (REQUIRED) since they represent one
  atomic "did the order succeed" unit — if payment succeeds but inventory
  reservation fails, the whole thing rolls back including the payment
  charge (or, in a real system, the payment charge is an external call
  needing compensation/refund logic since it can't literally be part of
  the DB transaction — a preview of Book 8's Saga pattern for exactly this
  cross-system-boundary problem); sending the confirmation event should be
  `REQUIRES_NEW` or, better, published only after successful commit (e.g.,
  via a transactional outbox pattern, Book 8) so it never fires for an
  order that ultimately rolled back.

</details>

---

## 5.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| `JpaRepository` CRUD | Free, no implementation needed — proxy-generated |
| Derived query method typo | Fails at STARTUP (`PropertyReferenceException`), not compile time |
| `REQUIRED` (default) | Join existing transaction, or create one |
| `REQUIRES_NEW` | Always a new, independent transaction — for things that must survive outer rollback |
| `NESTED` | Savepoint within the same transaction — for partial rollback within one unit of work |
| Default rollback | Unchecked exceptions ONLY — checked exceptions need explicit `rollbackFor` |
| `@Transactional` + self-invocation | Same AOP proxy bypass as any other aspect (Book 3 Ch. 4) — silently doesn't apply |

---

*(Continues to Chapter 6 — Spring Security Basics.)*
