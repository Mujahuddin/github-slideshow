# CHAPTER 6 — LOCKS & DEADLOCKS

---

## 6.1 CONCEPT: Pessimistic Locking — `SELECT ... FOR UPDATE`

### TELUGU EXPLANATION

Chapter 5 లో మనం "row ని lock చేయాలి" అని ప్రస్తావించాము — ఇక్కడ
ఖచ్చితంగా ఎలా చేయాలో చూద్దాం. **Pessimistic Locking** అంటే, "ఎవరో
conflict చేస్తారు అని ముందుగానే అనుకుని, row ని lock చేయడం":

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE; -- ఈ row ని లాక్ చేస్తుంది
-- ఇప్పుడు ఇంకో transaction, ఈ row ని update చేయడానికి (లేదా FOR UPDATE తో చదవడానికి)
-- ప్రయత్నిస్తే, ఈ transaction commit/rollback అయ్యేవరకు WAIT చేస్తుంది
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
COMMIT; -- ఇప్పుడు లాక్ విడుదల అవుతుంది
```

**ఇది Book 1 Chapter 9 `synchronized` తో direct సారూప్యత:** `FOR
UPDATE` ఖచ్చితంగా ఒక **critical section** ని create చేస్తుంది —
ఒక్క transaction మాత్రమే ఆ row ని ఏకకాలంలో modify చేయగలదు, మిగతావి
**wait** చేస్తాయి (Java లో threads `synchronized` block కోసం wait
చేసినట్టే).

### ENGLISH INTERVIEW ANSWER

"`SELECT ... FOR UPDATE` is pessimistic locking — I acquire an exclusive
lock on the row at read time, assuming a conflict is likely enough to
guard against proactively. Any other transaction trying to modify (or
also `FOR UPDATE`-read) that same row blocks until my transaction commits
or rolls back. This is structurally identical to Book 1's `synchronized`
block — I'm creating a critical section, just at the database row level
instead of an in-process object monitor, with the same fundamental
trade-off: correctness under concurrency, at the cost of making
conflicting transactions wait rather than proceed in parallel."

---

## 6.2 CONCEPT: Optimistic Locking — The `@Version` Column Approach

### TELUGU EXPLANATION

**Pessimistic locking limitation:** Lock hold చేసిన సమయం అంతా,
throughput తగ్గుతుంది — **conflict అరుదుగా జరిగే** పరిస్థితులకి ఇది
అనవసరమైన overhead. **Optimistic Locking** వేరే approach తీసుకుంటుంది:
"**conflict అరుదుగా జరుగుతుంది అని అనుకుని, lock వాడకుండా, update
సమయంలో మాత్రమే conflict జరిగిందో లేదో check చేయడం**":

```sql
-- ప్రతి row కి ఒక version column
ALTER TABLE accounts ADD COLUMN version INT DEFAULT 0;

-- Update చేసేటప్పుడు, version ని కూడా check చేయండి
UPDATE accounts SET balance = balance - 1000, version = version + 1
WHERE id = 1 AND version = 5; -- ఇది మనం చదివినప్పుడు చూసిన version

-- affected rows == 0 అయితే, ఎవరో మధ్యలో update చేశారని అర్థం — conflict!
-- Application code ఇది detect చేసి, retry చేయాలి (లేదా user కి error చూపించాలి)
```

**ఇది Book 7 (JPA/Hibernate) లో `@Version` annotation తో automatic గా
implement అవుతుంది** — Hibernate ఈ version check ని ప్రతి update
కి automatic గా జోడిస్తుంది, mismatch అయితే
`OptimisticLockException` throw చేస్తుంది.

**ఎప్పుడు ఏది వాడాలి (senior-level decision):**
- **Pessimistic:** Conflict **frequent** గా జరిగే అవకాశం ఉన్నప్పుడు
  (ఉదా: ఒకే popular item మీద అనేక concurrent updates), లేదా conflict
  జరిగితే **retry చేయడం ఖరీదైనది/కష్టం** అయినప్పుడు.
  **Optimistic:** Conflict **అరుదుగా** జరిగే అవకాశం ఉన్నప్పుడు (ఉదా:
  ఒక user తన own profile ని edit చేయడం — ఇద్దరూ ఏకకాలంలో ఒకే profile
  edit చేయడం అరుదు), **throughput ముఖ్యమైనప్పుడు**.

### ENGLISH INTERVIEW ANSWER

"Pessimistic locking holds a lock for the duration of the transaction,
which is correct but costs throughput even when conflicts are rare.
Optimistic locking instead assumes conflicts are rare and only checks for
one at commit time, via a version column incremented on every update —
the update's WHERE clause includes the version it was read with, and if
no rows match (because someone else already incremented it), zero rows
are affected, signaling a conflict the application must handle, typically
by retrying. This is exactly what JPA/Hibernate's `@Version` annotation
automates, throwing an `OptimisticLockException` on a version mismatch —
Book 7's territory. I choose pessimistic locking when conflicts are
genuinely likely and retrying is expensive or awkward — like reserving
the last unit of a hot-sale item — and optimistic locking when conflicts
are rare and I want to maximize throughput, like a user editing their own
profile, where two simultaneous edits to the same record are an edge case
worth handling gracefully but not worth constant locking overhead for."

---

## 6.3 CONCEPT: Database Deadlocks — Automatic Detection, Unlike Java Threads

### TELUGU EXPLANATION

Book 1 Chapter 9 లో మనం Java-level deadlock చూశాము — JVM దాన్ని
**automatic గా detect చేయదు** (`jstack` manual గా run చేసి చూడాలి).
**Database deadlocks వేరుగా ఉంటాయి** — దాదాపు అన్ని production
databases **automatic deadlock detection** కలిగి ఉంటాయి:

```
Transaction A: locks account 1, waits for account 2
Transaction B: locks account 2, waits for account 1
-- Database ఇది గుర్తించి, ఒక transaction ని (usually "victim" — తక్కువ
-- work చేసినది, లేదా configurable policy ప్రకారం) ROLLBACK చేస్తుంది
-- ఆ transaction కి ఒక error వస్తుంది (ఉదా: "Deadlock found; try restarting transaction")
```

**Application-level handling తప్పనిసరి:** Database deadlock ని
"victim" గా ఎంచుకున్న transaction కి **ఎప్పుడూ error వస్తుంది** —
application code ఈ specific exception ని **catch చేసి, transaction
ని retry చేయాలి** (Book 2 Chapter 16 retry సూత్రం, ఇక్కడ specific
గా deadlock exceptions కి వర్తింపజేయడం):

```java
@Retryable(retryFor = DeadlockLoserDataAccessException.class, maxAttempts = 3)
@Transactional
void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    // ... transfer logic, potential deadlock candidate ...
}
```

**Prevention (Book 1 Chapter 9 సూత్రమే, DB స్థాయిలో):** **Consistent
lock ordering** — అన్ని transactions, multiple rows లాక్ చేయాల్సి
వస్తే, **ఎప్పుడూ ఒకే క్రమంలో** (ఉదా: account ID ఆధారంగా ఎప్పుడూ చిన్న
ID ముందు) లాక్ చేయాలి — ఇది circular wait ని structurally అసాధ్యం
చేస్తుంది, ఖచ్చితంగా Book 1 Chapter 9 లో మనం చూసిన పరిష్కారమే.

### ENGLISH INTERVIEW ANSWER

"Unlike JVM-level deadlocks, which just hang forever until someone
manually diagnoses them with a thread dump, virtually every production
database has automatic deadlock detection — it identifies the circular
wait and picks a 'victim' transaction to roll back automatically,
returning a specific deadlock exception to that transaction rather than
hanging indefinitely. This means application code has a real
responsibility: catch that specific exception — like Spring's
`DeadlockLoserDataAccessException` — and retry the transaction, applying
exactly the retry principles from Book 2's production coding chapter,
scoped specifically to this recoverable, expected failure type. For
prevention, the fix is identical in principle to Book 1's application-level
deadlock prevention: consistent lock ordering — every transaction that
needs to lock multiple rows does so in the same globally agreed order,
like always locking the lower account ID first — which makes a circular
wait structurally impossible, exactly the same reasoning applied at a
different layer of the stack."

---

## 6.4 CONCEPT: Gap Locks and Phantom Prevention (Brief, MySQL-Specific Awareness)

### TELUGU EXPLANATION

Chapter 5 లో table లో REPEATABLE READ, phantom reads ని పూర్తిగా
నివారించదు అని చెప్పాము — **కానీ MySQL InnoDB ఒక exception** — ఇది
**Gap Locks** మరియు **Next-Key Locks** వాడి, REPEATABLE READ లోనే
phantom reads ని కూడా నివారిస్తుంది (SQL standard కి మించి). Gap Lock
అనేది, existing rows మధ్య **"gap" (ఖాళీ స్థలం)** ని లాక్ చేస్తుంది —
కొత్త row ఆ gap లోకి insert అవ్వకుండా.

**Senior-level practical takeaway:** ఇది **database-specific behavior**
— "REPEATABLE READ phantom reads ని నివారించదు" అనేది **SQL standard**
ప్రకారం నిజం, కానీ **MySQL InnoDB ఇది default గా override చేస్తుంది**
— ఇంటర్వ్యూలో ఈ nuance ప్రస్తావించడం (standard vs vendor-specific
behavior) genuine depth చూపిస్తుంది.

### ENGLISH INTERVIEW ANSWER

"One nuance worth knowing: the SQL standard says Repeatable Read doesn't
prevent phantom reads, but MySQL's InnoDB actually goes beyond the
standard using gap locks and next-key locks, which lock the space
*between* existing rows so a new row can't be inserted into a range
another transaction has already queried — meaning InnoDB's Repeatable
Read effectively also prevents phantoms in practice, unlike a strict
standard-conformant implementation. I'd bring this up specifically to
show that isolation-level guarantees aren't purely portable across
database vendors — the SQL standard sets a floor, but a specific engine's
actual behavior can exceed it, and it's worth verifying against the
specific database in use rather than assuming standard-only guarantees."

---

## 6.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Preventing a lost-update race | Doesn't lock at all, hopes for the best | Chooses pessimistic (`FOR UPDATE`) or optimistic (`@Version`) locking deliberately |
| Frequent vs rare conflict scenarios | Uses the same locking strategy everywhere | Matches locking strategy to actual conflict frequency |
| A deadlock exception in production | Treats it as an unrecoverable bug | Catches the specific exception and retries, per Book 2's retry principles |
| Multi-row locking order | Locks rows in whatever order the code happens to process them | Enforces consistent lock ordering to prevent deadlocks structurally |

---

## 6.6 COMMON MISTAKES

1. Not locking at all for a genuine read-modify-write race condition,
   relying on hope rather than correctness.
2. Using pessimistic locking everywhere, needlessly reducing throughput
   for low-conflict operations.
3. Not catching and retrying database deadlock exceptions, letting them
   surface as unhandled application errors.
4. Inconsistent lock ordering across different code paths, creating
   avoidable deadlock risk.
5. Assuming isolation-level guarantees are identical across all database
   vendors without checking vendor-specific behavior (like InnoDB's gap locks).

---

## 6.7 INTERVIEW QUESTION BANK — CHAPTER 6

**Basic:** 1. What does `SELECT ... FOR UPDATE` do? 2. What's the
difference between pessimistic and optimistic locking?

**Intermediate:** 3. How does a version column detect a conflict in
optimistic locking? 4. Why must application code catch and retry
database deadlock exceptions?

**Senior:** 5. Design the locking strategy for a high-contention "flash
sale" checkout (many users competing for limited stock) versus a
low-contention "update my profile" feature. 6. Explain how consistent
lock ordering prevents deadlocks structurally, using a concrete two-account transfer example.

**Architect:** 7. You're seeing intermittent deadlocks in production
between an order-creation flow and an inventory-reporting flow that
touch overlapping tables in different orders. How would you diagnose the
exact lock-acquisition order causing this, and redesign to prevent it?

**Scenario:** 8. A team implements optimistic locking with a version
column but doesn't handle the case where the UPDATE affects zero rows.
What's the resulting bug?

**Trick:** 9. "Optimistic locking is always better than pessimistic
locking because it doesn't block other transactions." True or false?

<details><summary>Key answers</summary>

- Q6: If Transaction A always locks the lower account ID first and
  Transaction B does the same (regardless of which account each
  transaction's transfer logically originates from), neither can ever be
  simultaneously holding one lock while waiting for the other's — one of
  them will always acquire both locks in sequence without contention, or
  wait cleanly for the other to finish, never forming the circular
  wait-for relationship required for a deadlock.
- Q7: Use the database's deadlock diagnostic output (most databases log
  the exact queries/lock order involved when a deadlock is detected) to
  identify which tables/rows each flow locks and in what order; redesign
  by enforcing the same lock acquisition order across both flows (e.g.,
  always lock the order table before the inventory table, in both code
  paths), or reduce lock scope/duration in one flow (e.g., making the
  reporting flow read with a weaker isolation level that doesn't need
  locks at all, if approximate reporting data is acceptable).
- Q8: The bug: silently proceeding as if the update succeeded, when
  actually zero rows were affected because another transaction already
  changed the version — the application never learns the update didn't
  happen, and the user's change is silently lost (a lost-update bug,
  precisely what optimistic locking is meant to catch and surface, not
  silently swallow).
- Q9: False — "always better" ignores that optimistic locking's
  no-blocking benefit comes at the cost of needing to handle conflicts
  reactively (detect-and-retry) rather than proactively preventing them,
  which is worse for high-contention scenarios where conflicts are
  frequent — repeatedly retrying failed optimistic updates under heavy
  contention can perform worse than a well-scoped pessimistic lock that
  simply serializes access to the hot row.

</details>

---

## 6.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does an optimistic locking conflict manifest as "zero rows affected" rather than a database-level error?
- **Coding Check:** Implement a Spring service method using `@Retryable` to automatically retry on a deadlock exception, following section 6.3's pattern.
- **Explanation Check:** Explain in English why database deadlock handling differs fundamentally from Java thread deadlock handling (automatic detection vs. manual diagnosis).
- **Real-World Check:** Your team's checkout flow uses optimistic locking on the inventory table, and under a flash-sale traffic spike, the retry rate becomes so high that effective throughput collapses. Diagnose and redesign.
- **Senior Check:** When would you combine BOTH pessimistic and optimistic locking in the same system, for different operations?
- **Master Check:** Design the complete locking and retry strategy for a multi-step order-fulfillment process (reserve inventory, charge payment, create shipment record) where each step touches a different table, minimizing both deadlock risk and unnecessary blocking.

<details><summary>Answers</summary>

- Real-World Check: This is exactly the "optimistic locking under
  high contention performs worse" scenario from the Trick question —
  under flash-sale-level contention, switch the hot inventory-decrement
  operation specifically to pessimistic locking (or an atomic conditional
  UPDATE), while potentially keeping optimistic locking for less-contended
  parts of the same checkout flow (like updating a shipping address) —
  matching the locking strategy to actual measured contention rather than
  using one strategy uniformly.
- Senior Check: A system might use pessimistic locking for known
  high-contention hot paths (inventory reservation during a sale) while
  using optimistic locking for the vast majority of low-contention
  operations (general record updates, user profile edits) throughout the
  rest of the same application — this is a very common real-world mixed
  strategy, not an either-or architectural choice.
- Master Check: Consistent lock/operation ordering across the whole flow
  (always: inventory table, then payment, then shipment — same order
  every time, preventing circular waits per section 6.3); pessimistic
  locking (or atomic conditional updates) specifically for the
  inventory-reservation step (high-contention, correctness-critical);
  optimistic locking or no special locking for shipment record creation
  (low-contention, mostly append-only); deadlock retry logic wrapping the
  whole transaction boundary per section 6.3, since even with consistent
  ordering, deadlocks with unrelated flows touching the same tables in
  different orders remain possible and must be handled gracefully.

</details>

---

## 6.9 CHEAT SHEET

| Concept | Rule |
|---|---|
| `SELECT ... FOR UPDATE` | Pessimistic lock — same idea as `synchronized`, at the row level |
| Optimistic locking (`@Version`) | No lock held — conflict detected via version mismatch at commit |
| Pessimistic vs optimistic | Frequent conflict / expensive retry → pessimistic. Rare conflict / throughput matters → optimistic |
| DB deadlocks | Automatically detected — a "victim" transaction is rolled back with a specific exception |
| App responsibility | Catch the deadlock exception, retry (Book 2 Ch. 16 principles) |
| Deadlock prevention | Consistent lock ordering across all code paths (same as Book 1 Ch. 9) |
| Vendor nuance | MySQL InnoDB's gap locks prevent phantoms even at Repeatable Read, beyond the SQL standard |

---

*(Continues to Chapter 7 — JDBC Fundamentals.)*
