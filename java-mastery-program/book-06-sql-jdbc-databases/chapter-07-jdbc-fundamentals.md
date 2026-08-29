# CHAPTER 7 — JDBC FUNDAMENTALS

---

## 7.1 CONCEPT: The JDBC Building Blocks

### TELUGU EXPLANATION

JPA/Hibernate (Book 7) వాడేటప్పుడు కూడా, అవి అంతర్గతంగా **JDBC**
నే వాడతాయి — కాబట్టి JDBC fundamentals అర్థం చేసుకోవడం, ఆ abstraction
layer కింద ఏం జరుగుతుందో అర్థం చేసుకోవడానికి కీలకం.

```java
try (Connection conn = dataSource.getConnection(); // Book 1 Chapter 6: try-with-resources, AutoCloseable
     PreparedStatement stmt = conn.prepareStatement(
             "SELECT id, name, balance FROM accounts WHERE customer_id = ?")) {

    stmt.setLong(1, customerId); // parameter binding — placeholder ki value
    try (ResultSet rs = stmt.executeQuery()) {
        while (rs.next()) {
            long id = rs.getLong("id");
            String name = rs.getString("name");
            BigDecimal balance = rs.getBigDecimal("balance");
            // ... process row ...
        }
    }
} catch (SQLException e) {
    throw new DataAccessException("Failed to fetch accounts", e); // Book 1 Chapter 6: exception chaining!
}
```

**కీలక objects:**
- **`Connection`:** ఒక database session — **ఖరీదైనది create చేయడానికి**
  (network handshake, authentication) — section 7.3 లో దీన్ని ఎందుకు
  pool చేయాలో చూద్దాం.
- **`PreparedStatement`:** ఒక **precompiled** SQL statement, `?`
  placeholders తో — section 7.2 లో దీని **critical security ప్రయోజనం**
  చూద్దాం.
- **`ResultSet`:** Query ఫలితం, ఒక cursor లా — `rs.next()` ప్రతి row
  కి ముందుకు వెళ్తుంది.

**ఇక్కడ Book 1 Chapter 6 సూత్రాలు ఎలా వర్తిస్తున్నాయో గమనించండి:**
`try-with-resources` (multiple resources — Connection, Statement,
ResultSet, అన్నీ **reverse order** లో close అవుతాయి), మరియు
`SQLException` (checked exception) ని catch చేసి, **chain చేస్తూ**
ఒక unchecked `DataAccessException` గా wrap చేయడం.

### ENGLISH INTERVIEW ANSWER

"Even when using JPA/Hibernate, understanding raw JDBC matters because
that's the actual mechanism underneath the abstraction. A `Connection`
represents a database session and is genuinely expensive to create — a
network round trip plus authentication — which is exactly why connection
pooling exists, covered in section 7.3. A `PreparedStatement` is a
precompiled SQL statement with placeholder parameters, which has a
critical security benefit beyond performance, covered next. A
`ResultSet` is a cursor over the query results, advanced row by row with
`next()`. I always wrap these in try-with-resources — multiple resources
closed in reverse order automatically — and I always catch the checked
`SQLException` and chain it into an appropriate unchecked exception,
exactly the exception handling discipline from Book 1 Chapter 6."

---

## 7.2 CONCEPT: `PreparedStatement` vs `Statement` — SQL Injection Prevention

### TELUGU EXPLANATION

**ఇది కేవలం performance optimization కాదు — ఇది critical security
practice.**

```java
// ❌ SQL Injection VULNERABILITY — ఎప్పుడూ ఇలా చేయకూడదు
String query = "SELECT * FROM users WHERE username = '" + username + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(query);

// ఒక attacker, username గా ఇలా పంపితే:
// username = "' OR '1'='1"
// Final query అవుతుంది: SELECT * FROM users WHERE username = '' OR '1'='1'
// ఇది ప్రతి user ని return చేస్తుంది — authentication bypass!
```

```java
// ✅ PreparedStatement — SQL structure మరియు DATA పూర్తిగా వేరుగా ఉంటాయి
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE username = ?");
stmt.setString(1, username); // ఇది ఎప్పుడూ ఒక LITERAL VALUE గానే treat అవుతుంది, SQL code గా కాదు
ResultSet rs = stmt.executeQuery();
// attacker అదే "' OR '1'='1" పంపినా, ఇది కేవలం ఒక (దొరకని) username string గానే treat అవుతుంది
```

**ఎందుకు ఇది పని చేస్తుంది:** `PreparedStatement`, SQL statement ని
**ముందుగానే database కి పంపి, compile చేయిస్తుంది** (structure fix
అయిపోతుంది) — తర్వాత parameters, **data గా మాత్రమే** బైండ్ అవుతాయి,
అవి ఎప్పటికీ SQL syntax లో భాగం గా parse అవ్వవు. ఇది Book 15 (Security)
లో వివరంగా చూసే **OWASP Top 10 #1 historically — Injection** కి
direct, fundamental పరిష్కారం.

### ENGLISH INTERVIEW ANSWER

"This isn't just a performance best practice — it's a fundamental
security control. Building a query via string concatenation with
`Statement` lets an attacker inject SQL syntax through user input — the
classic `' OR '1'='1` pattern can turn a login check into 'return every
user,' bypassing authentication entirely. `PreparedStatement` sends the
SQL structure to the database first, compiled and fixed, before any
parameter values are bound — those values are then always treated as
literal data, never re-parsed as SQL syntax, no matter what characters
they contain. This single practice — always using parameterized queries,
never string-concatenating user input into SQL — is the direct,
foundational defense against SQL injection, historically OWASP's most
cited vulnerability class, which Book 15 covers in full depth."

---

## 7.3 CONCEPT: Connection Pooling — Why HikariCP, and How to Size It

### TELUGU EXPLANATION

Section 7.1 లో చెప్పినట్టు, `Connection` create చేయడం **ఖరీదైనది**.
ప్రతి request కి కొత్త connection create చేసి, request తర్వాత close
చేయడం — Book 1 Chapter 10 లో మనం చూసిన "**ప్రతి task కి కొత్త Thread
create చేయడం**" యొక్క సమస్యే, database connections కి.

**Connection Pool** (HikariCP — Spring Boot యొక్క default, industry
fastest గా considered) ఒక **fixed సంఖ్యలో connections ని ముందుగానే
create చేసి, reuse చేస్తుంది** — ఇది Book 1 Chapter 10 `ThreadPoolExecutor`
యొక్క **direct సారూప్యత**, connections కి.

**Pool sizing (Book 1 Chapter 10 సూత్రాలే ఇక్కడ వర్తిస్తాయి):**
HikariCP యొక్క సొంత documentation ఒక సూత్రం సూచిస్తుంది:
```
connections = ((core_count * 2) + effective_spindle_count)
```
కానీ **ఆచరణలో, ముఖ్యమైన insight**: DB connections **CPU-bound resources
కాదు** — ఇవి **I/O wait** మీద ఆధారపడతాయి (query execute అవ్వడానికి
DB response కోసం wait). కాబట్టి, chapter 10 లో మనం చూసిన "I/O-bound
work కి ఎక్కువ threads/connections" సూత్రం వర్తిస్తుంది — **కానీ**
ఒక limit ఉంది: **DB server తనే** ఒక max connections limit కలిగి ఉంటుంది
— మీ pool size, DB యొక్క capacity దాటకూడదు, ముఖ్యంగా **multiple
service instances** (Book 8) ఒకే DB ని share చేస్తుంటే (ప్రతి instance
తన own pool కలిగి ఉంటుంది కాబట్టి, total connections = pool_size ×
instance_count).

### ENGLISH INTERVIEW ANSWER

"Creating a new database connection per request is exactly the same
anti-pattern as creating a new thread per task from Book 1's concurrency
chapter — connections are genuinely expensive to establish. HikariCP,
Spring Boot's default connection pool, pre-creates a fixed number of
connections and hands them out for reuse, directly parallel to
`ThreadPoolExecutor` managing threads. Sizing the pool follows similar
I/O-bound reasoning from Book 1 — since a connection spends most of its
time waiting on the database rather than consuming CPU, a larger pool
than the CPU core count often makes sense — but there's a hard ceiling I
always account for: the database server itself has a maximum connection
limit, and in a microservices architecture where multiple instances of
the same service each maintain their own pool, the *total* connections
across all instances — pool size times instance count — must stay well
under that database-side limit, or you risk exhausting the database's own
capacity even though each individual service's pool looks reasonably sized."

**Interviewer follow-up:** "What happens if the pool is exhausted?" —
New requests needing a connection **block/wait** (with a configurable
timeout, after which they fail with a `SQLTransientConnectionException`
or similar) — this is exactly Book 16's "connection pool exhausted"
production incident, previewed here: symptoms include request latency
spiking, then timeouts, tracing back to either genuine load exceeding
pool capacity, or — more insidiously — connections being leaked (never
returned to the pool, usually from a missing `close()`/try-with-resources
somewhere in the code).

---

## 7.4 CONCEPT: Batch Updates — Reducing Round Trips

### TELUGU EXPLANATION

1000 rows insert చేయాల్సి వస్తే, 1000 individual `INSERT` statements
(1000 network round trips) చేయడం **చాలా inefficient**. **Batch
updates** వీటిని group చేసి, **ఒక్క (లేదా కొన్ని) round trips** లో
పంపుతాయి:

```java
try (PreparedStatement stmt = conn.prepareStatement(
        "INSERT INTO audit_log (event, timestamp) VALUES (?, ?)")) {
    for (AuditEvent event : events) {
        stmt.setString(1, event.getName());
        stmt.setTimestamp(2, event.getTimestamp());
        stmt.addBatch(); // ఒక్కో row ని batch కి add చేయండి, execute చేయకుండా
    }
    stmt.executeBatch(); // అన్నింటినీ ఒకేసారి పంపండి
}
```

### ENGLISH INTERVIEW ANSWER

"Batch updates address the round-trip cost of many individual statements
— instead of 1000 separate network round trips for 1000 inserts, `addBatch()`
accumulates them and `executeBatch()` sends them together, dramatically
reducing network overhead for bulk operations. This is a standard
optimization I reach for whenever inserting or updating many rows in one
logical operation — data migrations, bulk imports, audit log batching —
rather than looping with individual `executeUpdate()` calls."

---

## 7.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Building a query with user input | String concatenation with `Statement` | `PreparedStatement` with parameter binding, always |
| Getting a DB connection | Creates a new one per request/operation | Uses a pooled connection (HikariCP) |
| Sizing a connection pool | Picks an arbitrary large number "to be safe" | Reasons about I/O-bound workload characteristics AND the database's own connection limit |
| Inserting many rows | Loops with individual `executeUpdate()` calls | Uses `addBatch()`/`executeBatch()` |

---

## 7.6 COMMON MISTAKES

1. String-concatenating user input into SQL instead of using
   `PreparedStatement` parameter binding — a critical SQL injection vulnerability.
2. Creating a new `Connection` per operation instead of using a pool.
3. Sizing a connection pool without considering the database's own
   connection limit across all service instances combined.
4. Not closing `Connection`/`Statement`/`ResultSet` via try-with-resources,
   leaking connections and eventually exhausting the pool.
5. Looping individual inserts/updates instead of using batch operations
   for bulk data changes.

---

## 7.7 INTERVIEW QUESTION BANK — CHAPTER 7

**Basic:** 1. Why is `PreparedStatement` preferred over `Statement`? 2.
Why is creating a database connection expensive?

**Intermediate:** 3. Explain exactly how SQL injection works and why
parameter binding prevents it. 4. What happens when a connection pool is
exhausted?

**Senior:** 5. Design connection pool sizing for a service with 10
instances sharing a database with a max of 200 connections. 6. Diagnose
a production symptom of "requests slow, then timing out, correlating
with rising active-connection metrics" — what are the two most likely
root causes?

**Architect:** 7. You're designing a system where a batch job needs to
insert 10 million rows without exhausting memory or overwhelming the
database. How would you combine batching, connection management, and
transaction boundaries?

**Scenario:** 8. A code review finds a method that opens a `Connection`
but the `close()` call is inside a code path that can be skipped by an
early return on an exception. What's the risk, and what's the fix?

**Trick:** 9. "PreparedStatement prevents all SQL injection risk
automatically, regardless of how the query is constructed." True or false?

<details><summary>Key answers</summary>

- Q5: Each instance's pool size, multiplied by 10 instances, must stay
  safely under 200 — e.g., 15 connections per instance (150 total),
  leaving headroom for other clients/admin connections and avoiding
  running the database at its absolute connection ceiling.
- Q6: Either (1) genuine load has grown beyond the pool's capacity to
  serve requests promptly (a capacity problem, may need pool size or
  database scaling review), or (2) a connection leak — some code path
  isn't returning connections to the pool (missing `close()`, an
  exception path skipping cleanup), so the pool slowly fills with
  "checked out but never returned" connections until nothing remains
  available — the fix and severity differ completely, so distinguishing
  them (checking pool metrics like active vs idle connections over time)
  is the first diagnostic step.
- Q7: Read/process the 10 million rows in chunks (not all in memory at
  once), use `addBatch()`/`executeBatch()` for each chunk (e.g., 1000
  rows per batch) to minimize round trips, and commit periodically
  (e.g., once per chunk or every few thousand rows) rather than one giant
  transaction — a single 10-million-row transaction risks excessive lock
  duration/redo log growth and makes failure recovery much harder than
  smaller, periodically-committed batches.
- Q8: The risk is a connection leak — if the exception path returns
  early without reaching the `close()` call, that connection is never
  returned to the pool, and repeated occurrences will eventually exhaust
  the pool. Fix: use try-with-resources instead of a manual `close()`
  call, guaranteeing closure on every exit path, exception or not.
- Q9: False — `PreparedStatement` prevents injection specifically for
  properly parameterized *values*; if a developer still concatenates
  untrusted input directly into the query string itself (e.g., for a
  dynamic table/column name, which can't be parameterized the same way),
  injection risk remains — parameter binding must actually be used for
  every piece of untrusted input, not just having a `PreparedStatement`
  object present in the code.

</details>

---

## 7.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why can't parameter binding be used for a dynamic table or column name, and what does this mean for how such inputs must be handled safely?
- **Coding Check:** Implement a batch insert of 10,000 records using `PreparedStatement.addBatch()`, chunked into batches of 1,000 with a commit after each chunk.
- **Explanation Check:** Explain in English, using the `' OR '1'='1` example, exactly why this input breaks a string-concatenated query but not a parameterized one.
- **Real-World Check:** Your team's admin panel lets users choose which column to sort a report by via a dropdown, passed as a query parameter to the backend, which builds `ORDER BY <column>` dynamically. Is this a SQL injection risk, and how do you fix it given that column names can't be parameterized?
- **Senior Check:** When would you deliberately use a smaller connection pool than the theoretical I/O-bound sizing formula suggests?
- **Master Check:** Design a complete connection-leak detection and prevention strategy for a large codebase where try-with-resources isn't consistently used yet — what would you check for, and how would you verify the fix worked in production?

<details><summary>Answers</summary>

- Real-World Check: Yes, this is a real injection risk if the column
  name comes directly from user input without validation — since it
  can't be parameterized, the correct fix is an allowlist: validate the
  requested column name against a fixed, known set of legitimate sortable
  columns server-side, rejecting or ignoring anything not on that list,
  rather than ever directly interpolating raw user input into the SQL string.
- Senior Check: When the database's own connection limit is tightly
  constrained relative to the number of service instances (Q5's
  scenario), or when the database server itself has limited resources
  (memory per connection) such that even I/O-bound-optimal pool sizing
  would overwhelm the database — the database's actual capacity is
  always the hard ceiling, overriding the theoretical per-service formula.
- Master Check: Use HikariCP's built-in leak detection
  (`leakDetectionThreshold` configuration, which logs a warning with a
  stack trace when a connection is checked out longer than the
  threshold); audit the codebase for any `Connection`/`Statement`/
  `ResultSet` usage not wrapped in try-with-resources and migrate them;
  monitor pool metrics (active vs idle connections) in production before
  and after the fix — a leak shows a slow, steady climb in active
  connections over time even under stable load, while a properly-fixed
  pool should show active connections fluctuating with actual request
  load and returning to baseline.

</details>

---

## 7.9 CHEAT SHEET

| Concept | Rule |
|---|---|
| `PreparedStatement` | Always use for queries with any user-supplied value — prevents SQL injection |
| Dynamic column/table names | Can't be parameterized — validate against an allowlist instead |
| Connection creation | Expensive — always use a pool (HikariCP), never create per-request |
| Pool sizing | I/O-bound reasoning (Book 1 Ch. 10), bounded by the database's own connection limit × instance count |
| Resource cleanup | Always try-with-resources — a missed `close()` leaks connections |
| Bulk inserts/updates | Use `addBatch()`/`executeBatch()`, chunked, with periodic commits |
| Connection leak symptom | Active connections climb steadily over time even under stable load |

---

*(Continues to Chapter 8 — Production Database Troubleshooting.)*
