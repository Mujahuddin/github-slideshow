# CHAPTER 1 — SQL FUNDAMENTALS: JOINS, SUBQUERIES, CTES

---

## 1.1 CONCEPT: Join Types — Precise Semantics, Not Just Syntax

### TELUGU EXPLANATION

```sql
-- Sample tables: customers (id, name), orders (id, customer_id, amount)
```

| Join | అర్థం | ఎప్పుడు వాడాలి |
|---|---|---|
| **INNER JOIN** | రెండు tables లోనూ **match అయిన** rows మాత్రమే | "ఆర్డర్లు పెట్టిన customers మాత్రమే కావాలి" |
| **LEFT JOIN** | ఎడమ table యొక్క **అన్ని rows**, కుడి table లో match లేకపోతే `NULL` | "అన్ని customers, వాళ్ళకి ఆర్డర్లు ఉన్నా లేకపోయినా" |
| **RIGHT JOIN** | LEFT JOIN యొక్క మిర్రర్ (అరుదుగా వాడతారు — LEFT JOIN తో table order మార్చితే సరిపోతుంది) | — |
| **FULL OUTER JOIN** | రెండు tables యొక్క **అన్ని rows**, match అయినా కాకపోయినా | "అనాథ records రెండు వైపులా కనుక్కోవడం" (data reconciliation) |

**⚠️ అత్యంత frequently-made mistake: LEFT JOIN + WHERE clause combination:**

```sql
-- ❌ ఇది LEFT JOIN ని INNER JOIN గా మార్చేస్తుంది!
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.amount > 100; -- ❌ o.amount NULL ఉన్న rows (orders లేని customers) ఇక్కడే తొలగించబడతాయి!

-- ✅ సరైన పద్ధతి — filter condition ని JOIN లోనే పెట్టాలి
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.amount > 100;
```

**ఎందుకు ఇది జరుగుతుంది:** `WHERE` clause **join పూర్తయిన తర్వాత**
apply అవుతుంది. Orders లేని customer కి, `o.amount` value `NULL` —
`NULL > 100` **ఎప్పుడూ `NULL`** (true కాదు, false కూడా కాదు — SQL లో
మూడు-విలువల logic, section 1.2 తర్వాత chapters లో వివరంగా), కాబట్టి ఆ
row **filtered out** అవుతుంది — ఫలితంగా, LEFT JOIN యొక్క "ఖాళీ ఉన్నా
అన్ని customers చూపించు" అనే ఉద్దేశ్యం **విఫలమవుతుంది**, ఇది
అనుకోకుండా INNER JOIN లా ప్రవర్తిస్తుంది.

### ENGLISH INTERVIEW ANSWER

"The join type I get asked to debug most often is this exact LEFT JOIN
plus WHERE clause interaction. A WHERE clause is applied *after* the
join completes, so filtering on a column from the right-hand table — like
`WHERE o.amount > 100` — silently discards exactly the rows the LEFT JOIN
was meant to preserve, namely customers with no matching order, because
`NULL > 100` evaluates to unknown, not true, in SQL's three-valued logic.
The fix is moving that condition into the `ON` clause instead, so it's
applied *during* the join, before rows without a match are already
guaranteed to be kept with NULLs. This is a genuinely common, subtle bug
I specifically look for in code review whenever I see a LEFT JOIN
combined with a WHERE filter on the joined table's columns."

---

## 1.2 CONCEPT: Subqueries — Correlated vs Non-Correlated, and `IN` vs `EXISTS`

### TELUGU EXPLANATION

**Non-correlated subquery:** Outer query తో సంబంధం లేకుండా, **ఒక్కసారి
మాత్రమే** execute అవుతుంది:
```sql
SELECT * FROM products WHERE price > (SELECT AVG(price) FROM products);
```

**Correlated subquery:** Outer query యొక్క **ప్రతి row కి ఒకసారి**
execute అవుతుంది (inner query, outer row ని reference చేస్తుంది):
```sql
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id); -- ప్రతి customer row కి, ఈ subquery run అవుతుంది
```

**`IN` vs `EXISTS` — senior-level performance distinction:**
- **`IN`:** inner subquery **మొత్తం result set** ని ముందుగా materialize
  చేస్తుంది, తర్వాత outer query దానితో compare చేస్తుంది. Inner result
  set **పెద్దది** అయితే, ఇది expensive.
- **`EXISTS`:** database, ప్రతి outer row కి, **మొదటి matching row
  దొరకగానే ఆగిపోతుంది** (short-circuit) — మొత్తం inner result అవసరం
  లేదు, కేవలం "ఏదైనా ఉందా" అని మాత్రమే తెలుసుకోవాలి.

**Senior rule:** "**ఏదైనా match ఉందా**" అని check చేయాలంటే `EXISTS`
సాధారణంగా faster (ముఖ్యంగా inner table పెద్దది అయితే). "**ఈ values
జాబితాలో ఉందా**" (చిన్న, static list) అయితే `IN` ఇంకా చదవడానికి
సులభం, పనితీరులో పెద్ద తేడా ఉండదు. **ఆధునిక query optimizers
(PostgreSQL, MySQL 8+) చాలా cases లో వీటిని సమానంగా optimize
చేస్తాయి** — కానీ ఈ theoretical differenceని అర్థం చేసుకోవడం, execution
plan (Chapter 3) చదివేటప్పుడు ఇంకా ముఖ్యం.

### ENGLISH INTERVIEW ANSWER

"A non-correlated subquery runs once, independent of the outer query — like
computing an average to compare against. A correlated subquery references
a column from the outer query, so conceptually it runs once per outer
row. On `IN` vs `EXISTS`: `EXISTS` can short-circuit the moment it finds
one matching row for the current outer row, while `IN` conceptually needs
the subquery's full result set materialized for comparison — so for
'does any match exist' semantics against a potentially large inner table,
I default to `EXISTS`. That said, I'd verify with an actual execution
plan (Chapter 3) rather than assuming, since modern optimizers in
PostgreSQL and MySQL 8+ often rewrite `IN` and `EXISTS` into equivalent
execution plans anyway — the theoretical distinction matters most when
reading a plan that shows they *didn't* get optimized equivalently."

---

## 1.3 CONCEPT: CTEs (Common Table Expressions) — Readability, and Recursive Queries

### TELUGU EXPLANATION

**CTE (`WITH` clause):** ఒక complex query ని **named, readable steps**
గా విభజించడానికి:

```sql
WITH high_value_customers AS (
    SELECT customer_id, SUM(amount) AS total_spent
    FROM orders
    GROUP BY customer_id
    HAVING SUM(amount) > 10000
)
SELECT c.name, hvc.total_spent
FROM customers c
JOIN high_value_customers hvc ON c.id = hvc.customer_id
ORDER BY hvc.total_spent DESC;
```

ఇదే logic ఒక **nested subquery** గా రాయవచ్చు, కానీ CTE **చదవడానికి,
maintain చేయడానికి చాలా సులభం** — ముఖ్యంగా multiple CTEs ఒకదాని తర్వాత
ఒకటి build అయ్యేటప్పుడు (ఒక "query లో pipeline" లా).

**Recursive CTEs — hierarchical data కి (ఉదా: org chart, category
tree):**
```sql
WITH RECURSIVE employee_hierarchy AS (
    SELECT id, name, manager_id, 0 AS level
    FROM employees WHERE manager_id IS NULL -- base case: top-level (CEO)

    UNION ALL

    SELECT e.id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.id -- recursive case
)
SELECT * FROM employee_hierarchy ORDER BY level;
```

**ఇది Book 2 Chapter 9 (Recursion) తో direct సారూప్యత:** base case
(`manager_id IS NULL`) + recursive case (`JOIN` తనతో తానే) — ఖచ్చితంగా
Java recursion యొక్క base case/recursive case structure, SQL లో express
చేయబడింది.

### ENGLISH INTERVIEW ANSWER

"I reach for CTEs primarily for readability — breaking a complex query
into named, logical steps that read top-to-bottom, especially when
multiple CTEs build on each other like a pipeline. Performance-wise, a
non-recursive CTE is often equivalent to an inlined subquery once the
optimizer processes it, though this varies by database — PostgreSQL, for
instance, historically treated CTEs as an optimization fence in older
versions, materializing them rather than inlining, which is worth
knowing and verifying via execution plan if performance is critical.
Recursive CTEs are the real power feature — they're structurally
identical to recursion in application code: a base case (the anchor
member, like finding top-level employees with no manager) and a
recursive case (joining the CTE to itself to walk down each level),
exactly the pattern from Book 2's recursion chapter, just expressed in SQL."

---

## 1.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| LEFT JOIN with a filter on the joined table | Puts the filter in WHERE, unknowingly breaks the LEFT JOIN | Puts the filter in the ON clause |
| "Does any match exist" check | Always uses `IN` | Considers `EXISTS` for large inner tables, verifies with an execution plan |
| Complex multi-step query | Writes deeply nested subqueries | Uses CTEs for readability, especially multi-step pipelines |
| Hierarchical data (org chart, categories) | Writes application-level recursive loops fetching one level at a time | Uses a recursive CTE to fetch the whole hierarchy in one query |

---

## 1.5 COMMON MISTAKES

1. Filtering a LEFT JOIN's right-side columns in WHERE instead of ON,
   silently converting it to an INNER JOIN.
2. Using `IN` with a large, expensive subquery when `EXISTS` would
   short-circuit and perform better.
3. Writing deeply nested subqueries where a CTE would be far more
   readable and maintainable.
4. Fetching hierarchical data with N sequential application-level queries
   (one per level) instead of a single recursive CTE.
5. Forgetting NULL's three-valued logic — assuming `NULL > 100` is
   `false` when it's actually `unknown`, filtering out rows unexpectedly.

---

## 1.6 INTERVIEW QUESTION BANK — CHAPTER 1

**Basic:** 1. What's the difference between INNER JOIN and LEFT JOIN? 2.
What is a correlated subquery?

**Intermediate:** 3. Why does putting a right-table filter in WHERE break
a LEFT JOIN's intent? 4. When would you choose a CTE over a subquery?

**Senior:** 5. Explain when `EXISTS` outperforms `IN`, and when the
difference is negligible in modern databases. 6. Design a recursive CTE
to find all descendants of a given category in a category tree, and
explain the base case / recursive case structure.

**Architect:** 7. You're designing a reporting query that joins 6 tables
with several correlated subqueries, and it's become unreadable and slow.
How would you approach refactoring it (CTEs, indexes, materialized views)?

**Scenario:** 8. A report showing "all customers and their order totals"
is missing customers who have never placed an order, even though the
query uses a LEFT JOIN. Diagnose using this chapter's material.

**Trick:** 9. "A LEFT JOIN and a RIGHT JOIN with the tables swapped
always produce identical results." True or false?

<details><summary>Key answers</summary>

- Q6: Base case: `SELECT id FROM categories WHERE id = :rootCategoryId`.
  Recursive case: `SELECT c.id FROM categories c JOIN category_tree ct ON
  c.parent_id = ct.id` — unioned together, this walks down from the given
  category through all descendant levels, structurally identical to a
  tree traversal in application code (Book 2 Chapter 10).
- Q7: Break the query into readable CTEs first (readability and
  debuggability), verify each CTE's contributing sub-result with
  `EXPLAIN` (Chapter 3) to find missing indexes, and consider a
  materialized view if the same expensive aggregation is computed
  repeatedly by many queries/reports rather than needing to be real-time
  on every request.
- Q8: Almost certainly the classic LEFT JOIN + WHERE-on-right-table-column
  bug from section 1.1 — the query likely filters on an `orders` column
  in the WHERE clause, silently discarding customers with no orders
  (NULL in that column) exactly as described; fix by moving that
  condition into the JOIN's ON clause.
- Q9: True in terms of the *logical result set* (assuming the same join
  condition and table roles swapped correctly) — `A LEFT JOIN B` is
  equivalent to `B RIGHT JOIN A` — but this is often a source of
  confusion precisely because it's easy to swap the tables incorrectly
  or forget to also flip which columns are selected/expected to be NULL.

</details>

---

## 1.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does `NULL > 100` evaluate to unknown rather than false, and why does that matter for a WHERE clause filtering a LEFT JOIN's result?
- **Coding Check:** Write a query using a CTE to find the top 3 highest-spending customers per region (hint: this previews Chapter 2's window functions — note where a CTE alone isn't quite enough).
- **Explanation Check:** Explain in English, using a concrete example, why moving a filter from WHERE to the JOIN's ON clause changes a LEFT JOIN's actual results, not just its readability.
- **Real-World Check:** Your team's customer churn report needs "all customers who have NOT placed an order in the last 90 days." Design this query using a LEFT JOIN correctly (not an easy-to-get-wrong NOT IN).
- **Senior Check:** When would a recursive CTE be the wrong tool, even for hierarchical data?
- **Master Check:** Design a query finding all "orphaned" records in two directions (customers with no orders, AND orders referencing a non-existent customer_id due to a data integrity issue) in a single query — what join type enables checking both directions at once?

<details><summary>Answers</summary>

- Real-World Check: `SELECT c.* FROM customers c LEFT JOIN orders o ON
  c.id = o.customer_id AND o.created_at > NOW() - INTERVAL '90 days'
  WHERE o.id IS NULL` — the date condition goes in the ON clause (so it
  doesn't eliminate customers with old-but-existing orders from the join
  entirely), and `WHERE o.id IS NULL` correctly identifies customers with
  no matching *recent* order row after the join.
- Senior Check: When the hierarchy is extremely deep and the database's
  recursive CTE implementation has performance/depth limitations, or when
  the hierarchy needs to be queried so frequently that a denormalized
  materialized path/closure table (Book 6 Chapter 4 schema design
  territory) would perform significantly better than recomputing the
  recursive traversal on every query.
- Master Check: A FULL OUTER JOIN between customers and orders, then
  filtering for `WHERE c.id IS NULL OR o.customer_id IS NULL` — customers
  with no matching order show a NULL on the orders side, and orphaned
  orders (referencing a deleted/non-existent customer) show a NULL on the
  customers side, both caught by one query using the "keep everything
  from both sides" semantics of FULL OUTER JOIN.

</details>

---

## 1.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| LEFT JOIN + filter on right table | Put it in ON, not WHERE, or you silently get an INNER JOIN |
| `IN` vs `EXISTS` | `EXISTS` short-circuits — better for "does any match exist" on large tables |
| CTE vs nested subquery | CTE for readability, especially multi-step pipelines |
| Recursive CTE | Base case + recursive case — same structure as application-level recursion |
| NULL comparisons | Three-valued logic — `NULL > x` is unknown, not false |
| FULL OUTER JOIN | Use for finding orphans/mismatches in both directions at once |

---

*(Continues to Chapter 2 — Window Functions.)*
