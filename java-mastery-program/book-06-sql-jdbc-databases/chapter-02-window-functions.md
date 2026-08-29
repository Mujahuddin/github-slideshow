# CHAPTER 2 — WINDOW FUNCTIONS

---

## 2.1 CONCEPT: Why Window Functions Exist — Beyond `GROUP BY`

### TELUGU EXPLANATION

`GROUP BY` ఒక fundamental limitation కలిగి ఉంది: ఇది rows ని **collapse**
చేస్తుంది — group కి ఒక్క output row మాత్రమే. "**ప్రతి customer యొక్క
total spend చూపించు, కానీ ప్రతి individual order row ని కూడా ఉంచు**"
అనేది `GROUP BY` తో సాధ్యం కాదు నేరుగా.

**Window functions** ఇది పరిష్కరిస్తాయి — ఇవి ఒక "window" (related
rows యొక్క సమూహం) మీద compute చేస్తాయి, **కానీ rows ని collapse
చేయవు** — ప్రతి original row అలాగే ఉంటుంది, ఒక కొత్త computed column
తో:

```sql
SELECT
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (PARTITION BY customer_id) AS customer_total -- row ఒక్కటి కూడా కోల్పోదు!
FROM orders;
```

**`OVER (PARTITION BY ...)`** — ఇదే key syntax — "**దేని ప్రకారం
groups విభజించాలో**" చెప్తుంది (`GROUP BY` లాగే), కానీ actual rows ని
merge చేయకుండా.

### ENGLISH INTERVIEW ANSWER

"`GROUP BY` collapses rows into one row per group, which means you lose
row-level detail the moment you need an aggregate. Window functions solve
exactly this: `SUM(amount) OVER (PARTITION BY customer_id)` computes each
customer's total while still returning every individual order row intact,
with the total attached as an additional column on each. The `PARTITION
BY` clause defines the grouping for the computation without performing
the actual row-collapsing that `GROUP BY` does — this is the single
concept that unlocks the entire category of 'aggregate context but
row-level detail' queries that are otherwise awkward or impossible with
plain aggregation."

---

## 2.2 CONCEPT: Ranking Functions — `ROW_NUMBER`, `RANK`, `DENSE_RANK`

### TELUGU EXPలanaTION

మూడు ranking functions, **tie-handling లో మాత్రమే వేరు**:

```sql
SELECT
    name, score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num,
    RANK()       OVER (ORDER BY score DESC) AS rank_val,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank_val
FROM students;
```

| name | score | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|---|
| Alice | 95 | 1 | 1 | 1 |
| Bob | 90 | 2 | 2 | 2 |
| Carol | 90 | 3 | 2 | 2 |
| Dave | 85 | 4 | 4 | 3 |

**తేడా (Bob, Carol tie అయినప్పుడు):**
- **`ROW_NUMBER`:** ఎప్పుడూ **unique**, sequential (1,2,3,4) — ties
  ని కూడా ఏకపక్షంగా విడదీస్తుంది.
- **`RANK`:** Tie ఉన్నవాటికి **ఒకే rank** ఇస్తుంది, కానీ **తర్వాతి
  rank ని skip చేస్తుంది** (2, 2, **4** — 3 లేదు).
- **`DENSE_RANK`:** Tie ఉన్నవాటికి ఒకే rank, **skip చేయదు** (2, 2, **3**).

**Senior గా, ఏది వాడాలో నిర్ణయించడం:** "**Top N per group**" pattern
కి (ఉదా: "ప్రతి department కి highest-paid 3 ఉద్యోగులు") `ROW_NUMBER`
సాధారణంగా సరైనది — ఖచ్చితంగా N rows కావాలంటే (ties ఉన్నా). `RANK`/
`DENSE_RANK` "**ఈ score కి సమానమైన వాళ్ళందరినీ చేర్చు**" అనే
semantics కి సరిపోతాయి (ఉదా: "top 3 scores" లో ties ఉంటే, 3+ మంది
ఉండొచ్చు).

### ENGLISH INTERVIEW ANSWER

"These three ranking functions only differ in how they handle ties.
`ROW_NUMBER` always assigns unique, sequential numbers, breaking ties
arbitrarily — useful when you need exactly N rows regardless of ties.
`RANK` gives tied rows the same rank but then skips the next rank number
entirely, reflecting 'these two are both in 2nd place, so the next one is
4th.' `DENSE_RANK` gives tied rows the same rank without skipping — 'both
2nd, next one is 3rd.' The choice matters concretely for a 'top N per
group' query: if I need exactly 3 rows per department no matter what,
`ROW_NUMBER` is correct; if I want 'everyone tied for a top-3 score,'
even if that's actually 4 or 5 people, `RANK` or `DENSE_RANK` reflects
that intent correctly instead of arbitrarily cutting one of the tied
values."

**Real-world "Top N per group" pattern (extremely common interview
question):**
```sql
WITH ranked_employees AS (
    SELECT
        name, department, salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT name, department, salary FROM ranked_employees WHERE rn <= 3;
```
**ఇది Book 2 Chapter 8 (Heap, Top-K pattern) యొక్క SQL-native
equivalent** — అక్కడ bounded min-heap వాడాము, ఇక్కడ `PARTITION BY` +
`ROW_NUMBER` + filter అదే ఫలితం ఇస్తాయి, database engine లోపలే.

---

## 2.3 CONCEPT: Running Totals and `LAG`/`LEAD` — Comparing Adjacent Rows

### TELUGU EXPLANATION

**Running total (cumulative sum) — Book 2 Chapter 6 (Prefix Sum) యొక్క
SQL రూపం:**
```sql
SELECT
    order_date, amount,
    SUM(amount) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;
```
`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — "**ఇప్పటివరకు
అన్ని rows**" అనే **frame** ని define చేస్తుంది — ఇది ఖచ్చితంగా Book
2 Chapter 6 లో మనం చూసిన `prefix[i] = prefix[i-1] + nums[i]` యొక్క
SQL-native computation, database engine ఒక్క pass లో efficiently
చేస్తుంది.

**`LAG`/`LEAD` — గత/రాబోయే row తో compare చేయడానికి:**
```sql
SELECT
    order_date, amount,
    LAG(amount, 1) OVER (ORDER BY order_date) AS previous_amount,
    amount - LAG(amount, 1) OVER (ORDER BY order_date) AS change_from_previous
FROM orders;
```
`LAG(amount, 1)` — "1 row వెనుక ఉన్న amount value" — **self-join
అవసరం లేకుండా** "month-over-month growth," "previous value తో compare"
లాంటి queries రాయడానికి — ఇది self-join తో రాస్తే చాలా awkward,
window function తో సూటిగా ఉంటుంది.

### ENGLISH INTERVIEW ANSWER

"Running totals via `SUM(...) OVER (ORDER BY ... ROWS BETWEEN UNBOUNDED
PRECEDING AND CURRENT ROW)` are the SQL-native version of Book 2's prefix
sum pattern — the database computes the cumulative sum in a single
efficient pass rather than me needing a self-join or application-level
loop. `LAG` and `LEAD` let me reference a previous or following row's
value directly within the same row's computation — this replaces what
would otherwise require an awkward self-join on a row-offset condition,
and it's the natural tool for month-over-month comparisons, detecting
value changes between consecutive readings, or computing a difference
from the prior row — all without leaving SQL or pulling data into
application code to do the comparison."

---

## 2.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Total per group, but keep row detail" | Runs a separate GROUP BY query and joins it back | Uses a window function directly |
| Top N per group | Fetches all rows into application code and filters there | Uses `ROW_NUMBER() OVER (PARTITION BY ...)` in SQL |
| Month-over-month comparison | Self-joins the table on a date offset | Uses `LAG`/`LEAD` |
| Choosing a ranking function | Always uses `RANK` by habit | Chooses based on exact tie-handling semantics needed |

---

## 2.5 COMMON MISTAKES

1. Running a separate aggregate query and joining it back when a window
   function would do it in one pass, more efficiently.
2. Using `RANK` when `ROW_NUMBER` was actually needed (or vice versa),
   producing an unexpected number of "top N" results due to tie handling.
3. Fetching all rows into application code to compute a top-N-per-group
   result instead of using `ROW_NUMBER` + `PARTITION BY` in SQL.
4. Using a self-join for previous/next-row comparisons instead of `LAG`/`LEAD`.
5. Forgetting that a window function's `WHERE` clause can't directly
   reference the window function's result (needing a CTE or subquery
   wrapper, since `WHERE` is evaluated before window functions in query
   processing order).

---

## 2.6 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. What's the key difference between `GROUP BY` and a window
function? 2. What does `PARTITION BY` do?

**Intermediate:** 3. Explain the difference between `RANK` and
`DENSE_RANK` with a tie example. 4. Why can't you filter directly on a
window function's result in the same query's WHERE clause?

**Senior:** 5. Write a query to find the top 3 highest-value orders per
customer, and explain why `ROW_NUMBER` is the right choice over `RANK`
here. 6. Design a query computing a 7-day moving average of daily sales
using a window function frame clause.

**Architect:** 7. A reporting dashboard needs running totals, month-over-
month deltas, and top-N-per-category rankings, computed over a large
sales table, refreshed every few minutes. Would you compute this live
via window functions on every dashboard load, or pre-aggregate? What
factors decide this?

**Scenario:** 8. A "top 5 products per category" query using `RANK()`
sometimes returns 7 or 8 products for a category. Explain why and how
you'd fix it if the requirement is truly "exactly 5."

**Trick:** 9. "Window functions always make a query slower than the
equivalent GROUP BY approach." True or false?

<details><summary>Key answers</summary>

- Q4: WHERE is logically evaluated before window functions are computed
  in SQL's query processing order (window functions are part of the
  SELECT-list evaluation, occurring after WHERE/GROUP BY/HAVING but
  before ORDER BY) — so a window function's result isn't yet available to
  filter on inside the same query block; you wrap the query in a CTE or
  subquery and filter in the outer query instead, which is why the
  ranked-employees pattern from section 2.2 uses a CTE.
- Q6: `AVG(daily_sales) OVER (ORDER BY sale_date ROWS BETWEEN 6 PRECEDING
  AND CURRENT ROW)` — the frame clause `ROWS BETWEEN 6 PRECEDING AND
  CURRENT ROW` defines a 7-row window (the current row plus the 6 before
  it), computing the average within that sliding window as it moves down
  the ordered result set.
- Q7: Depends on data volume and dashboard load frequency/concurrency —
  for a large table queried by many concurrent dashboard viewers,
  pre-aggregating (e.g., via a scheduled job populating a summary table,
  or a materialized view refreshed periodically) avoids recomputing
  expensive window function queries on every page load; for smaller data
  or infrequent access, computing live is simpler and avoids staleness —
  the "how fresh does this need to be vs. how expensive is it to compute
  live" trade-off decides it.
- Q8: This is exactly the `RANK` tie-handling behavior — if there's a tie
  for 5th place, `RANK` gives multiple rows the same rank value, and
  filtering `WHERE rank <= 5` includes all of them, producing more than 5
  results. If the requirement is genuinely "exactly 5, no more, no
  less," switch to `ROW_NUMBER`, which breaks ties arbitrarily to
  guarantee exactly 5 rows.
- Q9: False — window functions typically compute a value in a single
  pass over sorted/partitioned data, which can be more efficient than an
  equivalent approach using a separate aggregate query plus a join or a
  self-join; the database's query planner optimizes window function
  execution similarly to other query constructs, and the "single query,
  single pass" nature is often faster than a multi-query or self-join
  alternative achieving the same result.

</details>

---

## 2.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why can a window function query return the same number of rows as its input, while a GROUP BY query returns one row per group?
- **Coding Check:** Write a query showing each employee's salary alongside their department's average salary and their salary's rank within the department, all in one query.
- **Explanation Check:** Explain in English why `LAG`/`LEAD` are preferable to a self-join for "compare to the previous row" queries.
- **Real-World Check:** Your team needs a report showing, for each customer, their most recent order date and how many days have passed since their previous order (to identify re-engagement opportunities). Design this query using window functions.
- **Senior Check:** When would you choose NOT to use a window function even though it could technically solve the problem?
- **Master Check:** Design a query identifying "customers whose most recent order was larger than all of their previous orders combined" — describe which window functions and framing you'd combine to express this.

<details><summary>Answers</summary>

- Real-World Check: `SELECT customer_id, order_date, order_date -
  LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS
  days_since_previous FROM orders` then filter (in an outer query/CTE)
  for the most recent row per customer (via `ROW_NUMBER() OVER
  (PARTITION BY customer_id ORDER BY order_date DESC) = 1`) — combining
  `LAG` for the gap calculation with `ROW_NUMBER` for "most recent only."
- Senior Check: When the equivalent GROUP BY query is simpler, more
  readable, and doesn't need row-level detail preserved — reaching for a
  window function when a plain aggregate query would do is unnecessary
  complexity; use the simplest tool that expresses the actual requirement.
- Master Check: For each customer, compute the running sum of all
  previous orders (`SUM(amount) OVER (PARTITION BY customer_id ORDER BY
  order_date ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING)` —
  excluding the current row) alongside the current order's amount, then
  filter for rows (via a CTE) where the current order's amount exceeds
  that running total of everything before it — combining a running-total
  window function with a comparison against the current row's own value.

</details>

---

## 2.8 CHEAT SHEET

| Need | Function |
|---|---|
| Aggregate value + keep row detail | `SUM()/AVG()/COUNT() OVER (PARTITION BY ...)` |
| Exactly N rows per group | `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` |
| "Top N" allowing ties | `RANK()` or `DENSE_RANK()` |
| Running/cumulative total | `SUM() OVER (ORDER BY ... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` |
| Compare to previous/next row | `LAG()` / `LEAD()` — avoids a self-join |
| Filtering on a window function's result | Wrap in a CTE/subquery — can't filter directly in the same WHERE |

---

*(Continues to Chapter 3 — Indexes & Query Optimization.)*
