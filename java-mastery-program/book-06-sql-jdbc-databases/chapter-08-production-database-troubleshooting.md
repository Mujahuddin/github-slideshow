# CHAPTER 8 — PRODUCTION DATABASE TROUBLESHOOTING

> This chapter applies the Symptom → Hypothesis → Investigation → Logs →
> Metrics → Root Cause → Fix → Prevention framework (fully generalized in
> Book 16) to database-specific production incidents, pulling together
> every concept from Chapters 1-7.

---

## 8.1 SCENARIO: A Previously-Fast Query Becomes Slow

**SYMPTOM:** A query that used to run in milliseconds now takes seconds,
with no code deployment in between.

**HYPOTHESES (in order of likelihood, cheapest to check first):**
1. Table growth — the query was always doing more work than it looked
   like, and data volume finally crossed a threshold.
2. Stale statistics after a bulk data change (Chapter 3).
3. An index was dropped or became unusable (a migration, a schema change).
4. A query plan regression — the optimizer switched strategies (perhaps
   correctly, given changed data distribution, or a database version
   upgrade changed planner behavior).

**INVESTIGATION:**
```sql
EXPLAIN ANALYZE <the slow query>;
```
Look for: `Seq Scan` where an `Index Scan` was previously used; a large
gap between estimated and actual row counts (Chapter 3's stale-statistics
signal); an unexpectedly high `cost` value.

**LOGS/METRICS:** Database slow query log (most databases have one,
configurable by a duration threshold); compare row counts/table size now
versus historically.

**ROOT CAUSE (example):** A bulk import last week added 20 million rows;
`ANALYZE` was never re-run, so the optimizer still estimates row counts
based on old statistics and chooses a plan that was fine for the old
data volume but is now catastrophically wrong.

**FIX:** Run `ANALYZE` (or the database's equivalent) to refresh
statistics; verify with `EXPLAIN ANALYZE` that the plan improves.

**PREVENTION:** Ensure `ANALYZE` runs automatically after bulk data
operations (many databases have auto-analyze/auto-vacuum features —
verify they're enabled and appropriately tuned, not just assumed default).

---

## 8.2 SCENARIO: Connection Pool Exhausted

**SYMPTOM:** Requests start timing out; application logs show
`SQLTransientConnectionException: Connection is not available, request
timed out` (or the equivalent for your pool library).

**HYPOTHESES:**
1. Genuine load increase beyond pool capacity.
2. A connection leak — code somewhere isn't returning connections
   (Chapter 7's try-with-resources discipline missed somewhere).
3. Slow queries holding connections longer than normal, reducing
   effective pool availability even at the same request rate.
4. A downstream dependency (the database itself) is slow/degraded,
   causing every query to hold its connection longer.

**INVESTIGATION:** Check the connection pool's metrics (HikariCP exposes
these via Actuator, Book 4 Chapter 7): active connections, idle
connections, pending threads waiting for a connection, over time.

- **Leak signature:** Active connections climb steadily and never return
  to baseline, even as request load fluctuates normally.
- **Genuine load signature:** Active connections track request volume
  closely, maxing out during traffic peaks and returning to normal after.
- **Slow query signature:** Active connections spike in correlation with
  a specific slow query's execution time, not overall request volume.

**LOGS:** Enable HikariCP's `leakDetectionThreshold` — it logs a warning
with a full stack trace showing exactly where a connection was checked
out, when it's held suspiciously long, pinpointing the leaking code path
directly.

**ROOT CAUSE (example):** A recently-added method uses manual
`Connection` handling (not try-with-resources) with an early-return path
on validation failure that skips the `close()` call.

**FIX:** Refactor to try-with-resources (Chapter 7), guaranteeing closure
on every exit path.

**PREVENTION:** A static analysis rule/linter flagging manual JDBC
resource management outside try-with-resources; code review checklist
item; `leakDetectionThreshold` enabled permanently in production as an
early-warning system, not just for incident response.

---

## 8.3 SCENARIO: Sudden Spike in Deadlocks

**SYMPTOM:** Database logs show a sharp increase in deadlock errors;
application logs show a corresponding spike in retried/failed transactions.

**HYPOTHESES:**
1. A new code path was deployed that acquires locks in a different order
   than existing code (Chapter 6's lock-ordering principle violated).
2. Increased concurrency/traffic simply made a pre-existing,
   low-probability deadlock scenario statistically much more likely to occur.
3. A long-running transaction (perhaps a new reporting query, or a batch
   job) is holding locks far longer than typical request-scoped
   transactions, increasing collision windows.

**INVESTIGATION:** Most databases log full deadlock details, including
the exact queries and lock chain involved (PostgreSQL: `deadlock_timeout`
+ logs; MySQL: `SHOW ENGINE INNODB STATUS` for the latest deadlock).
Identify which two (or more) code paths were involved and in what lock order.

**ROOT CAUSE (example):** A newly-deployed batch reconciliation job
updates `orders` then `inventory` in that order, while the existing
checkout flow updates `inventory` then `orders` — a direct lock-ordering
mismatch (Chapter 6).

**FIX:** Standardize lock acquisition order across both code paths
(always `inventory` before `orders`, or vice versa — the specific order
matters less than universal consistency).

**PREVENTION:** Document the canonical lock ordering for the schema's
frequently-co-locked tables; add it to onboarding/architecture
documentation so new code paths follow the same convention by default,
not by accident.

---

## 8.4 SCENARIO: N+1 Query Pattern (Preview — Full Treatment in Book 7)

**SYMPTOM:** A single logical operation (e.g., "list 50 orders with
customer names") generates 51 queries instead of 1-2 — one to fetch the
orders, then one *per order* to fetch its customer.

**QUICK DIAGNOSIS:** Enable SQL logging temporarily (or use a tool like
Hibernate's statistics, or a proxy like p6spy) and count actual queries
executed for one logical request — if it scales linearly with result set
size, that's the N+1 signature.

**WHY THIS BELONGS HERE, NOT JUST BOOK 7:** N+1 is fundamentally a JOIN
problem (Chapter 1) hidden behind an ORM abstraction — the fix, at the
SQL level, is always some form of fetching the related data in the same
query (a JOIN) or in one batched follow-up query, instead of one query
per parent row. Book 7 covers exactly how Hibernate's fetch strategies
control this, but the underlying SQL principle is this book's material.

---

## 8.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Slow query investigation | Guesses and tries random index additions | Runs `EXPLAIN ANALYZE` first, forms a hypothesis from the actual plan |
| Connection pool exhaustion | Increases pool size to "fix" it | Distinguishes leak vs genuine load vs slow queries via metrics first |
| Deadlock spike after a deploy | Treats it as a database "flakiness" issue | Checks the new code path's lock acquisition order against existing conventions |
| N+1 symptom | Doesn't recognize it, just says "the page is slow" | Recognizes the query-count-scales-with-result-size signature immediately |

---

## 8.6 COMMON MISTAKES

1. Adding indexes speculatively without first reading an execution plan
   to confirm the actual bottleneck.
2. Treating "increase the pool size" as a fix for connection exhaustion
   without first ruling out a leak.
3. Not enabling connection leak detection until after an incident forces
   the question.
4. Fixing a deadlock by adding a retry without also fixing the
   underlying lock-ordering inconsistency, leaving the root cause in place.
5. Not recognizing the N+1 pattern until it's already caused a
   production incident under real load, rather than catching it via query
   count assertions in tests.

---

## 8.7 INTERVIEW QUESTION BANK — CHAPTER 8

**Basic:** 1. What's the first tool you reach for when a query is slow?
2. What does a connection pool leak look like in metrics, versus genuine load?

**Intermediate:** 3. How do you distinguish a stale-statistics problem
from a missing-index problem when a query is slow? 4. What's the N+1
query pattern, and how do you detect it?

**Senior:** 5. Walk through your complete investigation process for
"deadlocks suddenly increased after today's deployment." 6. Design a
production monitoring/alerting strategy that would catch a connection
leak before it causes a full outage.

**Architect:** 7. You're setting database observability standards across
50 services. What metrics, logs, and automated checks would you mandate
to catch the four categories of incidents in this chapter before they
become severe?

**Scenario:** 8. A team's "fix" for recurring deadlocks was adding
`@Retryable` with unlimited retries. Explain why this is an incomplete
fix and what's still at risk.

**Trick:** 9. "If EXPLAIN shows an index is being used, the query is
already optimal." True or false?

<details><summary>Key answers</summary>

- Q3: Stale statistics: the execution plan's estimated row count differs
  drastically from `EXPLAIN ANALYZE`'s actual row count, while the
  correct index still appears in the plan. Missing index: the plan shows
  a `Seq Scan` on a highly selective query with no index-based
  alternative even considered — running `ANALYZE` won't fix this; a new
  index is needed.
- Q6: Permanently enable HikariCP's `leakDetectionThreshold` in
  production (not just during incidents), alert on active-connection
  count trending upward without a corresponding drop during
  low-traffic periods (the leak signature from section 8.2), and
  incorporate a connection-pool-metrics dashboard into standard
  on-call/observability tooling rather than something only consulted
  reactively.
- Q7: Mandate: slow query logging enabled with a defined threshold;
  `EXPLAIN` plan review as part of PR review for any new/changed query
  touching a large table; connection pool metrics (active/idle/pending)
  exported to a shared monitoring platform with standard alert
  thresholds; deadlock logging enabled and alerted on; automated test
  assertions on query count for critical endpoints to catch N+1
  regressions before production.
- Q8: Unlimited retries on a deadlock without fixing the underlying
  lock-ordering inconsistency means the same deadlock will keep
  recurring indefinitely under the same conditions, consuming resources
  on repeated retry attempts and potentially never actually succeeding if
  the conflicting code paths run at similar frequency — it masks the
  symptom without addressing the root cause, and can degrade throughput
  significantly under sustained load compared to actually fixing the
  lock ordering.
- Q9: False — an index being used doesn't mean it's the *right* index,
  used *efficiently* (e.g., only partially matching a composite index's
  leftmost columns, still requiring significant additional filtering), or
  that the overall query structure (joins, subqueries) is optimal — "an
  index is used" is a necessary check, not a sufficient one, for
  concluding a query is fully optimized.

</details>

---

## 8.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does distinguishing "connection leak" from "genuine load increase" require looking at trends over time, not just a single point-in-time metric snapshot?
- **Coding Check:** Write a test asserting that fetching a list of 10 orders with their customers executes at most 2 SQL queries (a basic N+1 regression guard).
- **Explanation Check:** Explain in English why "add a retry" is treating a symptom, not a cause, for a recurring deadlock.
- **Real-World Check:** Your team's nightly batch job started causing daytime query slowness after being rescheduled to run during business hours. Diagnose using this chapter's framework.
- **Senior Check:** When would repeated, ongoing deadlocks between two specific transaction types be an acceptable, tolerated trade-off rather than something to eliminate entirely?
- **Master Check:** Design a complete "database health" runbook for on-call engineers covering all four scenarios in this chapter — for each, list the exact first three diagnostic queries/commands to run before escalating or making a change.

<details><summary>Answers</summary>

- Real-World Check: The batch job likely holds long-running transactions
  and/or heavy table scans that now compete with regular daytime query
  traffic for locks and I/O bandwidth — diagnose via `EXPLAIN ANALYZE` on
  affected daytime queries during the batch window, and check for lock
  contention/blocking correlating with the batch job's schedule; fix by
  either reverting the schedule change, optimizing the batch job's
  resource footprint (smaller transactions, off-peak throttling), or
  running it against a read replica instead of the primary.
- Senior Check: When the deadlock is genuinely rare (occurring far less
  often than, say, once per hour under normal load) and the cost of a
  redesign to eliminate the specific lock-ordering conflict exceeds the
  cost of an occasional retry — not every deadlock justifies an
  architecture change if retry-based recovery is cheap, fast, and
  infrequent enough to have negligible user impact; this is a judgment
  call based on actual measured frequency, not a default assumption.
- Master Check: Slow query: (1) `EXPLAIN ANALYZE` the query, (2) check
  table row count/last `ANALYZE` time, (3) check for recent schema/index
  changes. Connection exhaustion: (1) pool active/idle/pending metrics
  trend, (2) `leakDetectionThreshold` logs if enabled, (3) recent
  deploys touching connection-handling code. Deadlock spike: (1) database
  deadlock log for exact lock chain, (2) correlate timing with recent
  deployments, (3) compare lock acquisition order across the involved
  code paths. N+1: (1) enable/check SQL query logging for the affected
  endpoint, (2) count queries per request, (3) check the ORM's fetch
  strategy configuration for the involved entity relationships (Book 7 preview).

</details>

---

## 8.9 CHEAT SHEET

| Incident | First diagnostic step | Common root cause |
|---|---|---|
| Slow query, previously fast | `EXPLAIN ANALYZE` | Stale statistics after bulk change, or missing index |
| Connection pool exhausted | Pool metrics trend (active/idle/pending over time) | Connection leak, or genuine load beyond capacity |
| Deadlock spike after deploy | DB deadlock log (exact lock chain) | Lock ordering inconsistency between two code paths |
| N+1 query pattern | Count actual queries per logical request | Missing JOIN/fetch strategy — full fix in Book 7 |
| General framework | Symptom → Hypothesis → Investigation → Logs/Metrics → Root Cause → Fix → Prevention | Always confirm root cause before fixing — don't guess |

---

## BOOK 6 — CHAPTER 8 COMPLETE

*(All 8 chapters of Book 6's chapter content are now complete. See
`final-assessment-mock-interview-project.md` in this directory for the
cumulative Final Assessment, the SQL/JDBC Mock Interview round, and the
Book 6 capstone Project Assignment.)*
