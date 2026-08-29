# BOOK 6 — FINAL ASSESSMENT, SQL/JDBC MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 8 CHAPTERS)

1. A report query uses `WHERE o.status = 'SHIPPED'` in the WHERE clause
   after a LEFT JOIN from customers to orders, and customers with no
   orders are missing from results. Fix it and explain why. *(Ch. 1)*
2. Design a query showing each employee's salary rank within their
   department, without collapsing any employee rows. Which SQL feature,
   and why not GROUP BY? *(Ch. 2)*
3. A composite index exists on `(customer_id, order_date)`, but a query
   filtering only on `order_date` doesn't use it. Explain precisely why. *(Ch. 3)*
4. A `products` table stores `category_name` as a plain string, and
   renaming a category requires updating thousands of rows. Diagnose and
   fix using normalization principles. *(Ch. 4)*
5. Explain why Read Committed doesn't prevent a report from seeing two
   different totals when querying the same aggregate twice within one
   transaction. *(Ch. 5)*
6. Design the locking approach for a "reserve the last item in stock"
   race condition between two concurrent checkouts. *(Ch. 6)*
7. Why does string-concatenating a WHERE clause value expose a SQL
   injection vulnerability, and why doesn't parameter binding fully
   protect a dynamically-chosen ORDER BY column? *(Ch. 7)*
8. A query was fast last month and is slow today, with no code changes.
   List three hypotheses, in the order you'd check them. *(Ch. 8)*
9. Trace how a single JDBC `PreparedStatement` with `FOR UPDATE`,
   running inside a `REPEATABLE_READ` transaction, prevents a lost-update
   race condition — connect Chapters 5, 6, and 7 together.
10. Your team is deciding between pessimistic locking and optimistic
    locking for a new feature. What single question would you ask first
    to guide the decision?

<details>
<summary>Answer Key</summary>

1. Move `o.status = 'SHIPPED'` into the JOIN's ON clause:
   `LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'SHIPPED'`.
   In WHERE, it's applied after the join, and since customers with no
   orders have `NULL` for `o.status`, `NULL = 'SHIPPED'` is unknown (not
   true), filtering those customers out entirely — silently turning the
   LEFT JOIN into an INNER JOIN.
2. `RANK()` or `DENSE_RANK() OVER (PARTITION BY department ORDER BY
   salary DESC)` — a window function, not GROUP BY, because GROUP BY
   would collapse all employees in a department into one row, losing the
   individual employee detail the requirement needs alongside the rank.
3. The composite index's leftmost column is `customer_id` — per the
   leftmost prefix rule, an index on `(customer_id, order_date)` can't be
   used to efficiently search by `order_date` alone, since the index's
   sort order is anchored on `customer_id` first.
4. This is a 3NF violation — `category_name` is a transitive
   dependency, duplicated across many product rows. Fix: extract a
   `categories` table with its own surrogate key, referenced via
   `category_id` in `products`; renaming now updates exactly one row.
5. Read Committed only prevents dirty reads (seeing uncommitted data);
   it explicitly does not prevent non-repeatable reads — if another
   transaction commits a change to the underlying data between the
   report's two reads, seeing different totals is the correct, documented
   behavior of Read Committed, not a bug.
6. `SELECT ... FOR UPDATE` on the stock row (or an atomic conditional
   `UPDATE ... WHERE stock > 0`) during the reservation check — ensuring
   the second concurrent checkout's read/update blocks until the first's
   transaction completes, preventing both from seeing "1 available" and
   both succeeding.
7. String concatenation lets attacker-supplied characters be
   reinterpreted as SQL syntax rather than data (the classic `' OR
   '1'='1` bypass). Parameter binding fixes this for *values* bound as
   parameters, but a column/table name used in a dynamic ORDER BY can't
   be parameterized the same way (parameters bind to values, not
   identifiers) — it must instead be validated against a fixed allowlist
   of legitimate column names server-side.
8. In order: (1) run `EXPLAIN ANALYZE` to see the current plan and
   estimated-vs-actual row counts (cheapest, most informative first
   step); (2) check whether a recent bulk data change happened without a
   subsequent `ANALYZE`/statistics refresh; (3) check whether an index
   was dropped, altered, or became unusable due to a schema change.
9. The `PreparedStatement` (Ch. 7) safely executes the parameterized
   `SELECT ... FOR UPDATE` query; `FOR UPDATE` (Ch. 6) acquires a
   pessimistic lock on the row, blocking concurrent modification;
   `REPEATABLE_READ` (Ch. 5) ensures that within this same transaction,
   any subsequent read of that same row is guaranteed consistent with
   the first read — together, no other transaction can modify the row
   between this transaction's read and its eventual write, eliminating
   the lost-update race entirely.
10. "How frequently will concurrent conflicts on the same record
    actually occur?" — frequent/high-contention favors pessimistic
    locking; rare/low-contention favors optimistic locking, per Chapter
    6's core decision framework.

</details>

---

## PART B — MOCK INTERVIEW: SQL/JDBC ROUND

**Interviewer:** "Write a query to find the second-highest salary in
each department, and explain your approach."

**Model answer:** "I'd use `DENSE_RANK()` rather than a subquery-based
MAX-excluding-MAX approach, since it handles ties correctly and reads
clearly: `WITH ranked AS (SELECT *, DENSE_RANK() OVER (PARTITION BY
department ORDER BY salary DESC) AS rnk FROM employees) SELECT * FROM
ranked WHERE rnk = 2`. I chose `DENSE_RANK` over `RANK` here
deliberately — if there's a 3-way tie for highest salary, `RANK` would
jump straight to rank 4 for the next distinct value, meaning `rnk = 2`
would return nothing; `DENSE_RANK` correctly treats the next distinct
salary as rank 2 regardless of how many people tied for rank 1. I'd
verify this reasoning by walking through a small example with a tie
before considering the query done."

**Follow-up:** "What if the interviewer wants exactly one row per
department, even if there's a tie for second-highest?" (Switch to
`ROW_NUMBER()` instead, accepting that ties get arbitrarily broken, since
`ROW_NUMBER` guarantees uniqueness the way `DENSE_RANK` doesn't.)

---

**Interviewer:** "Your production database is experiencing intermittent
deadlocks since yesterday's release. Walk me through your live
investigation, right now."

**Model answer:** "First, I pull the database's deadlock log — most
databases record the exact queries and lock chain for each deadlock
event. I'd look for a pattern: are the same two transaction types always
involved? I'd then check what changed in yesterday's release that
touches those same tables — a new code path might be acquiring locks on
the same tables in a different order than existing code. Once I've
identified the specific mismatch — say, a new batch job locks `inventory`
then `orders`, while checkout locks `orders` then `inventory` — the fix
is standardizing that order across both, not just adding a retry, which
would mask the problem without fixing the actual conflict. I'd also add a
`@Retryable` for deadlock exceptions as a safety net for the remaining,
much-rarer collision window, since even correct lock ordering can't
eliminate every possible contention scenario, just the systematic ones."

**Follow-up:** "How would you have caught this before it reached
production?" (A staging/load test exercising both code paths
concurrently under realistic load, or a documented lock-ordering
convention that code review would have flagged the new batch job against.)

---

**Interviewer:** "Design the schema and query for an 'audit trail' that
must record every change to a `orders` table, and explain your indexing
strategy for querying it."

**Model answer:** "I'd create a separate `order_audit_log` table
(id, order_id, changed_field, old_value, new_value, changed_at,
changed_by), populated either via application-level logic within the
same transaction as the actual order change, or via a database trigger,
depending on whether I need to guarantee capture even for changes made
outside the application. For indexing, the dominant query pattern is
likely 'show audit history for order X' — a composite index on
`(order_id, changed_at)`, equality column first per Chapter 3's rule,
letting me efficiently retrieve one order's history in chronological
order. Since audit logs are append-only and rarely updated, I wouldn't
over-index it — just the access patterns actually needed, since every
additional index still taxes every insert into what's likely a
high-volume table."

---

## PART C — CAPSTONE PROJECT: "E-COMMERCE ANALYTICS AND INTEGRITY ENGINE"

**Goal:** A project combining every chapter of Book 6 around one
realistic schema, both querying it and diagnosing problems in it.

**Requirements:**

1. Design a normalized (3NF) schema for customers, orders, order_items,
   products, and categories, with surrogate keys throughout (Ch. 4).
2. Write a report query combining a CTE, a window function
   (top-3-products-per-category by revenue), and a JOIN, following the
   Chapter 1/2 patterns (Ch. 1, 2).
3. Design and justify the composite indexes needed for your three most
   frequent query patterns, with the correct column order (Ch. 3).
4. Implement, in Java/JDBC (or Spring's `JdbcTemplate`), an inventory
   reservation method using `SELECT ... FOR UPDATE` inside a
   `REPEATABLE_READ` transaction, with a test simulating two concurrent
   reservations for the last unit of stock (Ch. 5, 6, 7).
5. Implement the same reservation using optimistic locking instead
   (a version column), and write a short comparison of when you'd choose
   each approach for this specific feature (Ch. 6).
6. Write a "database health checklist" document applying Chapter 8's
   four incident scenarios to your own schema — for each, state what
   monitoring/prevention you'd put in place before launch, not after an incident.

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Schema normalization | No transitive dependencies; category rename touches one row |
| Index design | Composite indexes justified with actual query patterns, correct column order |
| Concurrency correctness | The concurrent-reservation test actually demonstrates the race condition being prevented |
| Locking comparison | Recommendation is tied to actual contention characteristics, not a coin flip |
| Production readiness | Health checklist references specific chapter mechanisms, not generic advice |

---

*(This completes BOOK 6 — SQL + JDBC + DATABASES. Book 7 — JPA + Hibernate
— builds the object-relational mapping layer directly on top of this
book's SQL, transaction, and locking foundation.)*
