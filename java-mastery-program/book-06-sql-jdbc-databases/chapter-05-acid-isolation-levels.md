# CHAPTER 5 — ACID & TRANSACTION ISOLATION LEVELS

---

## 5.1 CONCEPT: ACID — Precise Definitions, Not Buzzwords

### TELUGU EXPLANATION

- **Atomicity:** ఒక transaction లోని అన్ని operations, **అన్నీ succeed
  అవుతాయి, లేదా అన్నీ fail అవుతాయి** — partial completion సాధ్యం కాదు
  (Book 4 Chapter 5 `@Transactional` రోల్‌బ్యాక్ behavior ఇదే
  guarantee చేయడానికి ప్రయత్నిస్తుంది).
- **Consistency:** Transaction ముందు, తర్వాత, database **ఎప్పుడూ ఒక
  valid state** లో ఉండాలి (constraints, triggers అన్నీ maintain
  అవ్వాలి) — ఇది **application-level invariant** కి కూడా సంబంధించినది,
  కేవలం DB constraints కి మాత్రమే కాదు.
- **Isolation:** Concurrent transactions, ఒకదానికొకటి **ఎంతవరకు
  కనిపిస్తాయో** నియంత్రించడం — ఇదే ఈ chapter యొక్క ప్రధాన అంశం
  (section 5.2).
- **Durability:** Transaction commit అయిన తర్వాత, ఆ మార్పు **శాశ్వతంగా
  ఉంటుంది** (server crash అయినా) — ఇది write-ahead logging (WAL) ద్వారా
  achieve అవుతుంది (database, commit చేయడానికి ముందు, changes ని ఒక
  durable log కి flush చేస్తుంది).

### ENGLISH INTERVIEW ANSWER

"Atomicity means a transaction is all-or-nothing — no partial commits,
which is exactly what Book 4's `@Transactional` rollback behavior exists
to guarantee at the application layer. Consistency means the database
moves from one valid state to another, respecting all constraints —
this is broader than just DB-level constraints; it includes the
application's own business invariants. Isolation controls how much of one
in-progress transaction's changes another concurrent transaction can see
— this is the property with real, nuanced trade-offs, covered in depth in
this chapter. Durability guarantees that once a transaction commits, the
change survives even a server crash, typically implemented via
write-ahead logging, where changes are durably logged before the
transaction is considered committed."

---

## 5.2 CONCEPT: The Four Isolation Levels and the Three Phenomena They Prevent

### TELUGU EXPLANATION

**మూడు "phenomena" (concurrency సమస్యలు), ఇవి isolation levels
address చేస్తాయి:**

1. **Dirty Read:** Transaction A, ఇంకా **commit కాని** Transaction B
   యొక్క changes ని చదవడం — B rollback అయితే, A **ఎప్పుడూ existed
   లేని data** ని చదివినట్టు అవుతుంది.
2. **Non-Repeatable Read:** Transaction A, ఒకే row ని **రెండుసార్లు**
   చదివితే, మధ్యలో B ఆ row ని **update+commit** చేస్తే, A కి **రెండు
   వేర్వేరు values** కనిపిస్తాయి — అదే transaction లోపల.
3. **Phantom Read:** Transaction A, ఒక condition తో query చేసి,
   మధ్యలో B ఆ condition కి సరిపోయే ఒక **కొత్త row insert** చేసి
   commit చేస్తే, A అదే query **మళ్ళీ run చేస్తే కొత్త row (phantom)**
   కనిపిస్తుంది.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | ఎప్పుడు వాడాలి |
|---|---|---|---|---|
| **READ UNCOMMITTED** | ✅ సాధ్యం | ✅ సాధ్యం | ✅ సాధ్యం | దాదాపు ఎప్పుడూ వాడకూడదు (కేవలం approximate analytics) |
| **READ COMMITTED** (PostgreSQL/Oracle default) | ❌ నివారించబడింది | ✅ సాధ్యం | ✅ సాధ్యం | చాలా applications కి డిఫాల్ట్, సరిపోతుంది |
| **REPEATABLE READ** (MySQL InnoDB default) | ❌ నివారించబడింది | ❌ నివారించబడింది | ✅ సాధ్యం (కానీ InnoDB actually gap locks తో phantom ని కూడా నివారిస్తుంది) | Read-modify-write patterns, రిపోర్ట్‌ల consistency కావాలంటే |
| **SERIALIZABLE** | ❌ నివారించబడింది | ❌ నివారించబడింది | ❌ నివారించబడింది | Financial transactions, correctness అత్యంత critical అయినప్పుడు |

**Trade-off, ఎప్పుడూ గుర్తుంచుకోవాల్సింది:** Isolation level **పెరిగే
కొద్దీ**, correctness guarantee పెరుగుతుంది, కానీ **concurrency
(throughput) తగ్గుతుంది** (ఎక్కువ locking/versioning overhead) —
ఇది Book 1 Chapter 9 (concurrency vs correctness) సూత్రం, database
స్థాయిలో.

### ENGLISH INTERVIEW ANSWER

"The three phenomena build on each other in severity. A dirty read means
seeing another transaction's uncommitted changes — if that transaction
rolls back, you've read data that never actually existed. A
non-repeatable read means the same row gives different values on two
reads within the same transaction, because another transaction modified
and committed it in between. A phantom read means a repeated query
returns a different *set* of rows, because another transaction inserted
matching rows in between. Each isolation level prevents an increasing
subset of these: Read Committed — the default in PostgreSQL and Oracle —
prevents dirty reads but allows the other two; Repeatable Read — MySQL's
InnoDB default — additionally prevents non-repeatable reads; Serializable
prevents all three, effectively making concurrent transactions behave as
if they ran one at a time. The trade-off is fundamental: stronger
isolation means stronger correctness guarantees but more locking or
versioning overhead, directly reducing throughput — this is the same
concurrency-versus-correctness tension from Book 1's threading chapters,
just expressed at the database level."

**Interviewer follow-up:** "How does this connect to Spring's
`@Transactional`?" — Book 4 Chapter 5's `@Transactional(isolation =
Isolation.SERIALIZABLE)` attribute sets exactly this database-level
isolation for that transaction, overriding the connection/database
default — a direct, practical link between the annotation and the
underlying database mechanism.

---

## 5.3 CONCEPT: Choosing the Right Isolation Level — A Real Scenario

### TELUGU EXPLANATION

**Bank transfer ఉదాహరణ:** Account A నుండి Account B కి ₹1000 transfer
చేయాలి — ఇది రెండు updates (A నుండి తీసేయడం, B కి add చేయడం). **Race
condition risk:** ఒకేసారి రెండు transfers జరిగితే (A→B, మరియు A→C),
**non-repeatable read** phenomenon వల్ల, రెండు transactions A యొక్క
balance ని **stale** గా చదివి, రెండూ **ఒకే base balance** మీద deduct
చేయవచ్చు — ఫలితంగా, A యొక్క balance **తప్పుగా** అవుతుంది (ఒక
deduction "పోతుంది").

**పరిష్కారం:** **REPEATABLE READ** (లేదా SERIALIZABLE) + **explicit
row locking** (`SELECT ... FOR UPDATE`, Chapter 6 లో వివరంగా) — ఇది
A యొక్క row ని, transaction పూర్తయ్యేవరకు **lock** చేస్తుంది, రెండో
transaction wait చేయాల్సి వస్తుంది.

**Senior గా, correct reasoning process:**
1. "ఏ phenomenon actually నా data ని corrupt చేయగలదు?" అని identify
   చేయండి.
2. దాన్ని నివారించే **కనిష్ట** isolation level ఎంచుకోండి (అనవసరంగా
   SERIALIZABLE వాడకండి, throughput cost ఉంటుంది).
3. అవసరమైతే, isolation level తో పాటు **explicit locking** జోడించండి
   (Chapter 6).

### ENGLISH INTERVIEW ANSWER

"For a bank transfer, the risk is two concurrent transfers both reading
the same stale source-account balance before either commits — a
non-repeatable read scenario that can cause one deduction to be silently
lost, corrupting the balance. My approach is to identify precisely which
phenomenon threatens correctness here, then choose the minimal isolation
level that prevents it, rather than defaulting to Serializable everywhere
and taking an unnecessary throughput hit. For financial transfers,
Repeatable Read combined with explicit row-level locking — `SELECT ...
FOR UPDATE` on the source account, which Chapter 6 covers in depth — is
the standard, practical solution: it ensures the second transaction
blocks until the first completes, rather than both operating on the same
stale read. Reaching for Serializable everywhere 'to be safe' is a common
overcorrection that costs real throughput without necessarily buying more
correctness than a properly-scoped Repeatable Read plus explicit locking."

---

## 5.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Choosing an isolation level | Uses whatever the database default is, without knowing what it is | Explicitly reasons about which phenomena threaten this specific operation |
| "This needs to be really safe" | Sets SERIALIZABLE everywhere | Identifies the minimal isolation level that prevents the actual risk, adds explicit locking if needed |
| Debugging inconsistent reads within one transaction | Confused why the same query gave different results | Recognizes non-repeatable read, checks isolation level |
| Financial/critical operations | Assumes `@Transactional` alone guarantees correctness under concurrency | Explicitly considers isolation level + row locking together |

---

## 5.5 COMMON MISTAKES

1. Not knowing the database's default isolation level, assuming it
   prevents phenomena it doesn't.
2. Reflexively using SERIALIZABLE everywhere "to be safe," paying an
   unnecessary throughput cost.
3. Assuming `@Transactional` alone (without considering isolation level
   and locking) prevents concurrent-update race conditions.
4. Confusing "atomicity" (all-or-nothing) with "isolation" (visibility
   between concurrent transactions) — they solve different problems.
5. Not testing concurrent-access scenarios at all, only discovering
   isolation-related bugs under real production load.

---

## 5.6 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. What are the four ACID properties? 2. What is a dirty read?

**Intermediate:** 3. What's the difference between a non-repeatable read
and a phantom read? 4. What isolation level does PostgreSQL default to,
and what does it prevent?

**Senior:** 5. Design the isolation level and locking strategy for an
inventory system where two concurrent orders must not both successfully
reserve the last unit of stock. 6. Why does increasing isolation level
reduce throughput — what's actually happening under the hood (locking or
versioning)?

**Architect:** 7. You're designing a high-throughput e-commerce checkout
system. Where would you use Serializable-level guarantees (or explicit
locking) versus where you'd deliberately accept a weaker isolation level
for performance?

**Scenario:** 8. A report-generation transaction reads the same
aggregate query twice (for a consistency check) and gets two different
totals within the same transaction, even though isolation is set to Read
Committed. Explain why this is expected behavior, not a bug.

**Trick:** 9. "Serializable isolation guarantees your application logic
is free of race conditions." True or false?

<details><summary>Key answers</summary>

- Q5: Repeatable Read (or Read Committed with explicit locking) plus
  `SELECT ... FOR UPDATE` on the stock row when checking/reserving
  inventory — this ensures the second concurrent order's read of the
  stock count blocks until the first order's transaction (which is
  modifying that same row) completes, preventing both from seeing "1
  unit available" and both successfully reserving it.
- Q6: Higher isolation levels typically implement their guarantees via
  either more extensive locking (holding locks longer, on more rows, or
  using range/gap locks to prevent phantoms) or additional versioning
  overhead (MVCC systems maintaining more row versions and doing more
  work to determine visibility) — both reduce the number of transactions
  that can proceed truly concurrently without blocking or additional
  computation, which is the actual mechanism behind reduced throughput.
- Q7: Serializable/explicit locking for the inventory-reservation and
  payment-charging steps specifically, where a race condition has direct
  financial/correctness consequences (overselling, double-charging);
  weaker isolation (Read Committed) is fine for less critical reads, like
  displaying a product's "estimated" stock level on a browsing page,
  where slightly stale data has no real correctness consequence and
  strict isolation there would only cost throughput for no benefit.
- Q8: This is expected — Read Committed only prevents dirty reads
  (seeing uncommitted data); it does NOT prevent non-repeatable reads, so
  if another transaction committed a change to the underlying data
  between the two reads within this transaction, seeing different totals
  is exactly the documented, correct behavior of Read Committed, not a bug.
- Q9: False — Serializable prevents specific *database-level* concurrency
  phenomena (dirty/non-repeatable/phantom reads), but it doesn't protect
  against race conditions in application-level logic that don't map
  directly to those phenomena — e.g., a check-then-act sequence spanning
  multiple separate transactions, or a race condition in in-memory
  application state entirely outside the database (Book 1 Chapter 9's
  material) — isolation level is a database concurrency-control tool, not
  a universal race-condition solution for the whole application.

</details>

---

## 5.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does Read Committed prevent dirty reads but not non-repeatable reads — what specifically changes between the two guarantees?
- **Coding Check:** Configure a Spring `@Transactional` method with `isolation = Isolation.REPEATABLE_READ` and explain, in a code comment, exactly which race condition this prevents for the specific operation.
- **Explanation Check:** Explain in English why "just use Serializable everywhere" is not free — what's actually being traded away?
- **Real-World Check:** Your team's flash-sale feature (limited quantity, high concurrent demand) is overselling inventory under load. Diagnose the likely isolation/locking gap and design the fix.
- **Senior Check:** When would READ UNCOMMITTED ever be an acceptable choice despite allowing all three phenomena?
- **Master Check:** Design the complete concurrency-correctness strategy for a seat-booking system (concert tickets) under extremely high concurrent demand for a limited number of seats — combining isolation level, explicit locking, and application-level design (e.g., optimistic locking preview from Chapter 6) to both prevent double-booking AND maximize throughput under load.

<details><summary>Answers</summary>

- Real-World Check: Likely running at Read Committed (or even
  effectively unprotected) without explicit row locking on the inventory
  check-and-decrement — multiple concurrent orders read the same
  "available" stock count before any of them commits their decrement,
  all believing stock is available. Fix: `SELECT ... FOR UPDATE` on the
  inventory row during the check-and-reserve step (or an atomic
  conditional UPDATE like `UPDATE inventory SET stock = stock - 1 WHERE
  product_id = ? AND stock > 0`, checking the affected row count),
  ensuring only as many orders succeed as there is actual stock.
- Senior Check: For genuinely approximate, non-critical analytics or
  monitoring queries where seeing occasionally-inconsistent or even
  uncommitted data is an acceptable trade-off for avoiding any locking
  overhead on a busy production table — e.g., a rough real-time
  "requests per second" dashboard metric where perfect accuracy is far
  less important than not adding load/contention to the production system.
- Master Check: A hybrid approach is typical: an atomic conditional
  UPDATE or `SELECT ... FOR UPDATE` with a short-held lock for the actual
  seat-reservation moment (correctness-critical, brief), combined with
  optimistic locking (Chapter 6 preview — a version column checked at
  commit time) for less contended parts of the booking flow (e.g.,
  updating a user's profile during the same checkout), and
  application-level design choices like a short reservation hold/timeout
  window (reserve a seat for 5 minutes while payment completes, then
  release it) to avoid seats being locked indefinitely by abandoned
  checkouts — balancing strict correctness exactly where double-booking
  risk exists with throughput-friendly approaches everywhere else.

</details>

---

## 5.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Atomicity | All-or-nothing — Book 4's `@Transactional` rollback behavior |
| Isolation | Controls visibility between concurrent transactions — this chapter's focus |
| Durability | Survives a crash — via write-ahead logging |
| Dirty read | Seeing uncommitted data — prevented from Read Committed upward |
| Non-repeatable read | Same row, different value on re-read — prevented from Repeatable Read upward |
| Phantom read | Same query, different row SET on re-run — prevented only at Serializable (mostly) |
| Isolation level choice | Pick the minimum level that prevents the actual risk — not reflexively Serializable |
| `@Transactional(isolation=...)` | Sets the DB-level isolation for that transaction (Book 4 Ch. 5 connection) |

---

*(Continues to Chapter 6 — Locks & Deadlocks.)*
