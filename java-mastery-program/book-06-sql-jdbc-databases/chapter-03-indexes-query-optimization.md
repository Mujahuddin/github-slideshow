# CHAPTER 3 — INDEXES & QUERY OPTIMIZATION

---

## 3.1 CONCEPT: B-Tree Indexes — Why `O(log n)`, Connecting Back to DSA

### TELUGU EXPLANATION

Index లేకుండా, `WHERE email = 'x@example.com'` వంటి query, table యొక్క
**ప్రతి row ని scan చేయాలి** (**Full Table Scan**, O(n)) — Book 2
Chapter 1 లో మనం చూసిన "brute force linear search" యొక్క database
రూపం.

**B-Tree Index** (దాదాపు అన్ని relational databases డిఫాల్ట్ index
type) ఒక **balanced tree structure** (Book 2 Chapter 10 BST content
గుర్తు చేసుకోండి, కానీ **ఒక్కో node లో multiple keys** ఉండే **wider**
tree — disk-page-optimized) — ఇది values ని **sorted order** లో
maintain చేస్తుంది, search ని **O(log n)** కి తగ్గిస్తుంది — **ఖచ్చితంగా
Book 2 Chapter 7 binary search సూత్రమే**, ఇప్పుడు disk-based structure
గా.

**ఎందుకు B-Tree (Binary Tree కాదు):** Disk I/O ఖరీదైనది — ప్రతి "node"
access ఒక disk page read. Binary tree (ఒక్కో node కి 2 children)
`log₂(n)` depth కి వెళ్తుంది — B-Tree (ఒక్కో node కి వందలాది keys,
disk page size కి సరిపోయేలా) చాలా **తక్కువ depth** తో అదే n elements
ని hold చేస్తుంది — తక్కువ disk reads.

### ENGLISH INTERVIEW ANSWER

"Without an index, a query filtering on a column requires a full table
scan — checking every row, O(n), exactly the brute-force linear search
from Book 2's first chapter. A B-Tree index maintains values in sorted
order in a balanced tree structure, turning lookups into O(log n) — the
same binary search principle from Book 2, just applied to a persistent,
disk-based structure. The reason databases use a *wide* B-Tree rather
than a binary tree specifically is disk I/O cost: each tree node access
is potentially a disk page read, so packing many keys per node
(sized to fit a disk page) minimizes tree depth and therefore the number
of expensive disk reads needed to find a value, compared to a binary
tree's depth for the same number of elements."

---

## 3.2 CONCEPT: Composite Indexes — Column Order Is Not Arbitrary

### TELUGU EXPLANATION

**ఇది అత్యంత frequently misunderstood, production-critical topic.** ఒక
composite index `(status, created_at)` మీద — column order **ఖచ్చితంగా
ముఖ్యం**, ఎందుకంటే composite index అనేది **"leftmost prefix"** సూత్రం
మీద పని చేస్తుంది — ఇది ఒక **multi-level sorted structure**, phone
book లా (surname ముందు, first name తర్వాత — surname తెలియకుండా
first name తో వెతకలేరు).

```sql
CREATE INDEX idx_status_created ON orders (status, created_at);

-- ✅ ఈ index ని పూర్తిగా వాడుతుంది
SELECT * FROM orders WHERE status = 'PLACED' AND created_at > '2024-01-01';

-- ✅ ఇది కూడా వాడుతుంది (leftmost column మాత్రమే)
SELECT * FROM orders WHERE status = 'PLACED';

-- ❌ ఇది ఈ index ని వాడలేదు! (leftmost column, status, missing)
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**Senior rule — Equality columns ముందు, Range columns తర్వాత:** ఒక
composite index లో, **equality (`=`) filter అయ్యే columns ని ముందు**,
**range (`>`, `<`, `BETWEEN`) filter అయ్యే వాటిని తర్వాత** పెట్టాలి.
ఎందుకంటే B-Tree, equality column మీద ఒక **specific point** కి
జంప్ చేసి, అక్కడి నుండి **సమీపంలో ఉన్న (sorted) range** ని scan చేయగలదు
— కానీ ఒక range column ముందు ఉంటే, తర్వాత column యొక్క sorting
**ఆ range లోపల maintain అవ్వదు** (అది కూడా sort అవుతుంది, కానీ overall
useful గా ఉండదు అదే స్థాయిలో).

### ENGLISH INTERVIEW ANSWER

"Composite indexes follow the leftmost prefix rule — think of a phone
book sorted by surname then first name; you can find everyone with a
given surname, or a given surname-plus-first-name, but you can't
efficiently find everyone with a given first name alone without scanning
the whole book, since the sort order is anchored on surname first. So a
query filtering only on the second column of a composite index generally
can't use that index at all. The column-ordering rule I always apply:
equality-filtered columns go first, range-filtered columns go last. The
reasoning is that the B-Tree can jump directly to the exact point for an
equality match and then scan a contiguous, still-sorted range from there
for the next column — but if a range column comes first, everything
after it in the index isn't usefully sorted for the remaining filter,
since the range itself spans many different 'starting points' in the tree."

---

## 3.3 CONCEPT: Covering Indexes and the "Function on a Column Kills the Index" Trap

### TELUGU EXPLANATION

**Covering Index:** ఒక index, query కి అవసరమైన **అన్ని columns** ని
కలిగి ఉంటే (WHERE, SELECT రెండింటికీ), database **actual table రో ని
చదవాల్సిన అవసరమే లేదు** — **Index-Only Scan** (అత్యంత fast, extra
disk I/O లేకుండా):

```sql
CREATE INDEX idx_covering ON orders (status, created_at) INCLUDE (customer_id, amount);
-- SELECT customer_id, amount FROM orders WHERE status = 'PLACED' -- index లోనే అన్నీ ఉన్నాయి!
```

**⚠️ "Function on an indexed column kills the index" — extremely
common, costly mistake:**

```sql
-- ❌ Index ఉన్నా వాడదు! — YEAR(created_at) ఒక్కో row కి compute చేయాలి,
--    ఇది index యొక్క sorted structure ని పనికిరాకుండా చేస్తుంది
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- ✅ సరైన పద్ధతి — column ని "as-is" గా compare చేయండి, index వాడుతుంది
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

**ఎందుకు ఇది జరుగుతుంది:** Index, `created_at` యొక్క **actual (raw)
values** ని sorted order లో store చేస్తుంది — `YEAR(created_at)` అనేది
ఒక **derived value**, index లో store అవ్వలేదు. Database, ఈ derived
value ని compute చేయాలంటే, **ప్రతి row ని చదివి** compute చేయాలి —
ఇది index యొక్క ప్రయోజనాన్నే negate చేస్తుంది (**Full Table Scan** కి
తిరిగి వెళ్తుంది).

### ENGLISH INTERVIEW ANSWER

"A covering index includes every column a query needs — both filter and
select columns — letting the database satisfy the entire query from the
index alone, without a separate read of the actual table row; this is
called an index-only scan and it's meaningfully faster since it avoids
extra disk I/O. The mistake I watch for constantly in code review is
applying a function to an indexed column in a WHERE clause — `WHERE
YEAR(created_at) = 2024` — because the index stores the raw column
values in sorted order, not the result of applying a function to them.
The database can't use the index's sort order to jump to matching rows
when it first has to compute a derived value per row, so it silently
falls back to a full table scan. The fix is always to rewrite the
condition to compare the raw indexed column directly — a date range
instead of a function call — which is one of the highest-value, lowest-effort
optimizations I look for when reviewing slow queries."

---

## 3.4 CONCEPT: Reading an Execution Plan (`EXPLAIN`)

### TELUGU EXPLANATION

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'PLACED';
```

**చూడాల్సిన కీలక విషయాలు:**
- **`Seq Scan` (Sequential/Full Table Scan)** vs **`Index Scan`** వs
  **`Index Only Scan`** — ఏది వాడుతున్నారో (పెద్ద table మీద `Seq Scan`
  కనిపిస్తే, index missing/unused అని సూచన).
  Full Table Scan **ఎప్పుడూ చెడ్డది కాదు** — చిన్న tables కి, లేదా
  query result **table లో ఎక్కువ శాతం** rows తిరిగి ఇస్తుంటే (ఉదా:
  50%+), Full Table Scan **actually faster** కావొచ్చు index scan కంటే
  (index overhead, random I/O కంటే sequential read faster) — **optimizer
  ఇది deliberately ఎంచుకోవచ్చు, bug కాదు**.
- **Estimated rows vs Actual rows:** పెద్ద తేడా ఉంటే (`ANALYZE` వాడితే
  ఇది కనిపిస్తుంది), database **statistics stale** గా ఉన్నాయని సూచన
  (ఉదా: recent bulk insert తర్వాత `ANALYZE`/`UPDATE STATISTICS` run
  చేయలేదు) — ఇది తప్పు execution plan choices కి దారితీయవచ్చు.
- **Cost:** Optimizer యొక్క internal estimate (actual milliseconds
  కాదు) — వేర్వేరు plans ని compare చేయడానికి ఉపయోగపడేది.

### ENGLISH INTERVIEW ANSWER

"When reading an execution plan, I first check the scan type — Sequential
Scan versus Index Scan versus Index-Only Scan — but I don't automatically
treat a sequential scan on a large table as a problem; if the query is
genuinely returning a large fraction of the table's rows, a full scan can
legitimately be faster than an index scan's random I/O overhead, and a
good optimizer will choose it deliberately. What I actually look for is a
sequential scan on a highly selective query — one that should return a
tiny fraction of rows — which signals a missing or unusable index. I also
compare estimated versus actual row counts from `EXPLAIN ANALYZE`; a
large discrepancy usually means the database's statistics are stale,
often after a large bulk insert without a subsequent `ANALYZE`/`UPDATE
STATISTICS`, which can mislead the optimizer into a poor plan choice even
when the right index exists."

---

## 3.5 CONCEPT: When NOT to Index

### TELUGU EXPLANATION

Indexes **ఉచితం కాదు** — ప్రతి index, **ప్రతి INSERT/UPDATE/DELETE**
కి extra work add చేస్తుంది (index ని కూడా update చేయాలి) — Book 1
Chapter 2 (trade-offs) సూత్రం ఇక్కడ వర్తిస్తుంది:

- **Low-cardinality columns** (ఉదా: `is_active boolean`, కేవలం 2
  values) — index పెద్దగా ఉపయోగపడదు, ఎందుకంటే ఇది query ని చాలా
  తక్కువ narrow చేస్తుంది (50% rows ఇప్పటికీ match అవుతాయి).
- **Write-heavy tables**, ఎక్కువ indexes తో — ప్రతి write, అన్ని
  indexes ని update చేయాలి, write throughput తగ్గుతుంది.
- **చిన్న tables** — Full Table Scan అప్పటికే fast, index overhead
  అనవసరం.

### ENGLISH INTERVIEW ANSWER

"Indexes aren't free — every index adds write overhead, since every
insert, update, or delete has to update every affected index too. I avoid
indexing low-cardinality columns like a boolean flag, where an index
barely narrows the search since roughly half the table still matches
either value. For write-heavy tables, I'm deliberate about how many
indexes exist, since each one is a tax on every write. And for small
tables, a full scan is often already fast enough that an index adds
maintenance cost without a meaningful read-performance benefit. Indexing
decisions are a real trade-off between read and write performance, not
a 'more indexes are always better' default."

---

## 3.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Slow query | Adds an index on any column mentioned in WHERE, hoping it helps | Checks execution plan first, designs composite index with correct column order |
| Composite index column order | Puts columns in whatever order they appear in the WHERE clause | Equality columns first, range columns last, per the leftmost prefix rule |
| Seeing `Seq Scan` in a plan | Assumes it's always bad, adds an index reflexively | Checks selectivity — a full scan can be the optimizer's correct choice for low-selectivity queries |
| Applying a function to a filtered column | `WHERE YEAR(date_col) = 2024` without a second thought | Rewrites as a range comparison on the raw column to preserve index usability |

---

## 3.7 COMMON MISTAKES

1. Applying a function to an indexed column in a WHERE clause, silently
   disabling index usage.
2. Getting composite index column order wrong (range before equality).
3. Adding an index to every column mentioned anywhere in a WHERE clause
   without considering selectivity or write overhead.
4. Assuming `Seq Scan` in an execution plan is always a problem.
5. Not re-running `ANALYZE`/`UPDATE STATISTICS` after a large bulk data
   change, leaving the optimizer working from stale statistics.

---

## 3.8 INTERVIEW QUESTION BANK — CHAPTER 3

**Basic:** 1. Why is a B-Tree index O(log n) instead of O(n)? 2. What is
a covering index?

**Intermediate:** 3. Explain the leftmost prefix rule with a concrete
example. 4. Why does `WHERE UPPER(name) = 'JOHN'` disable an index on `name`?

**Senior:** 5. Design the composite index for a query filtering on
`status = 'ACTIVE'` and `created_at > X`, explaining the correct column
order and why. 6. When would the query optimizer correctly choose a full
table scan over using an available index?

**Architect:** 7. You're reviewing a schema with 15 indexes on a
high-write orders table, and write latency has degraded. How do you
determine which indexes are actually necessary versus removable?

**Scenario:** 8. After a bulk data migration inserting 10 million rows,
a previously-fast query becomes extremely slow, even though the relevant
index still exists. Diagnose.

**Trick:** 9. "Adding more indexes to a table can only help query
performance, never hurt it." True or false?

<details><summary>Key answers</summary>

- Q4: `UPPER(name)` computes a derived value not present in the index,
  which stores raw `name` values in sorted order — the database would
  need to compute `UPPER()` per row to compare, defeating the index's
  sorted-lookup benefit; fix with a case-insensitive collation on the
  column, or a functional index on `UPPER(name)` specifically if the
  database supports it (PostgreSQL does), rather than relying on a plain
  index on the raw column.
- Q6: When the query's selectivity is low — it's expected to return a
  large fraction of the table's rows — a full scan's efficient sequential
  I/O can beat an index scan's random I/O pattern, especially as the
  fraction of matching rows grows past a threshold (often cited around
  5-30% depending on the database and hardware) — the optimizer uses
  table statistics to estimate this and choose accordingly.
- Q7: Query the database's own index usage statistics (e.g.,
  PostgreSQL's `pg_stat_user_indexes`, tracking how often each index is
  actually scanned) to identify indexes with near-zero usage — these are
  prime candidates for removal, directly reducing write overhead without
  sacrificing any actually-used read performance.
- Q8: The bulk migration likely didn't trigger updated statistics
  (`ANALYZE`/`UPDATE STATISTICS`) — the optimizer's row-count estimates
  are now badly stale relative to the new 10-million-row reality, causing
  it to choose a poor execution plan (e.g., picking a full scan or a
  suboptimal join order) even though the index itself is intact and
  usable; running `ANALYZE` (or the database's equivalent statistics
  refresh) typically resolves this.
- Q9: False — every index adds overhead to every write operation
  (insert/update/delete must maintain the index too), and an excessive
  number of indexes on a write-heavy table can measurably degrade write
  throughput even while providing no meaningful additional read benefit,
  especially for redundant or low-selectivity indexes.

</details>

---

## 3.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does a composite index on `(a, b)` not help a query filtering only on `b`?
- **Coding Check:** Given a query filtering on `customer_id = ? AND order_date BETWEEN ? AND ?` and selecting `status`, design the optimal composite/covering index.
- **Explanation Check:** Explain in English why "add an index on every WHERE-clause column" is not a sound optimization strategy.
- **Real-World Check:** Your team's `orders` table has grown to 50 million rows, and a previously-instant query filtering on `email` (a low-cardinality-seeming but actually high-cardinality column) has become slow. Diagnose the likely cause and fix.
- **Senior Check:** When would you deliberately choose NOT to add an index even though it would measurably speed up a specific slow query?
- **Master Check:** Design the complete indexing strategy for a `transactions` table supporting three query patterns: (1) lookup by transaction ID, (2) list all transactions for an account within a date range, (3) a rare monthly report scanning nearly the entire table. Which patterns get dedicated indexes, and which might rely on a full scan deliberately?

<details><summary>Answers</summary>

- Real-World Check: Most likely a missing index on `email` entirely (if
  it was never indexed, believing it was "just another column"), or an
  existing index rendered unusable by a function applied to it in the
  query (e.g., `WHERE LOWER(email) = ?`) — check both: does an index on
  `email` exist, and does the actual query preserve the raw column
  comparison the index can use.
- Senior Check: When the query is rare enough (run once a quarter, say)
  that the cumulative write overhead of maintaining the index for
  potentially years outweighs the infrequent read benefit — a rarely-run
  report might be better served by accepting a slower, occasional full
  scan than by paying a permanent write-time cost for it.
- Master Check: (1) Transaction ID lookup → a unique index on
  `transaction_id` (likely already the primary key). (2) Account +
  date-range listing → a composite index on `(account_id, transaction_date)`,
  equality column first, range column second, per section 3.2 — possibly
  covering additional frequently-selected columns. (3) The rare monthly
  report scanning nearly the whole table → deliberately rely on a full
  table scan rather than adding a dedicated index purely for this
  infrequent, low-selectivity query, per the Senior Check reasoning —
  the write-overhead cost of a permanent index isn't justified for a
  once-a-month, near-full-table query.

</details>

---

## 3.10 CHEAT SHEET

| Concept | Rule |
|---|---|
| B-Tree index | O(log n) lookup — same binary search principle as Book 2, disk-optimized |
| Composite index column order | Equality columns first, range columns last (leftmost prefix rule) |
| Covering index | Includes all needed columns — enables index-only scan, no table read |
| Function on indexed column | Disables index usage — rewrite as a raw-column comparison |
| `Seq Scan` in a plan | Not automatically bad — can be the optimizer's correct choice for low selectivity |
| Stale statistics | Run `ANALYZE`/`UPDATE STATISTICS` after large bulk data changes |
| Index cost | Every index taxes every write — don't index low-cardinality or rarely-queried columns reflexively |

---

*(Continues to Chapter 4 — Normalization & Schema Design.)*
