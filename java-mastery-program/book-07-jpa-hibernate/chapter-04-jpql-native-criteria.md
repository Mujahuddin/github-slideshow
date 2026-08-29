# CHAPTER 4 — JPQL, NATIVE QUERIES, AND CRITERIA API

---

## 4.1 CONCEPT: JPQL — Querying Entities, Not Tables

### TELUGU EXPLANATION

**JPQL (Java Persistence Query Language)** SQL లా కనిపిస్తుంది, కానీ
ఇది **entities మరియు their fields** మీద పనిచేస్తుంది, **tables/columns**
మీద కాదు:

```java
// JPQL — entity name "Order", field name "status" (table/column పేర్లు కాదు!)
@Query("SELECT o FROM Order o WHERE o.status = :status AND o.customer.email = :email")
List<Order> findByStatusAndCustomerEmail(@Param("status") OrderStatus status, @Param("email") String email);
```

గమనించండి: `o.customer.email` — ఇది **object navigation** (Java లో
`order.getCustomer().getEmail()` లాగే), SQL JOIN syntax కాదు — Hibernate
ఇంటర్నల్ గా దీన్ని సరైన SQL JOIN గా మారుస్తుంది. ఇదే JPQL యొక్క
ప్రధాన ప్రయోజనం — **database-agnostic** గా (ఇదే JPQL ఏ database
అయినా, PostgreSQL, MySQL, Oracle — మార్పు లేకుండా పని చేస్తుంది,
అంతర్లీన SQL dialect తేడాలు Hibernate handle చేస్తుంది).

### ENGLISH INTERVIEW ANSWER

"JPQL looks like SQL but operates on the entity object model — entity
names and field names, not table and column names — and object
navigation like `o.customer.email` compiles down to the correct SQL JOIN
automatically. This is what makes JPQL database-agnostic: the same JPQL
query works unchanged whether the underlying database is PostgreSQL,
MySQL, or Oracle, since Hibernate translates it to the appropriate SQL
dialect. I use JPQL as my default for anything that maps cleanly onto the
entity model — most business queries fall into this category."

---

## 4.2 CONCEPT: DTO Projections via Constructor Expressions — Closing the Loop with Book 4

### TELUGU EXPLANATION

Book 4 Chapter 3 లో మనం "entities ని ఎప్పుడూ నేరుగా return చేయకూడదు"
అని నేర్చుకున్నాం, Book 7 Chapter 3 లో "complex views కి DTO projection
అనేది fetch-strategy సమస్యలన్నింటినీ sidestep చేస్తుంది" అని చెప్పాం.
JPQL **constructor expression** ఈ రెండు సూత్రాలని **ఒకే query** లో
combine చేస్తుంది:

```java
public record OrderSummary(String orderId, String customerName, BigDecimal total) {}

@Query("SELECT NEW com.example.OrderSummary(o.id, o.customer.name, o.total) FROM Order o")
List<OrderSummary> findAllSummaries();
```

**ఇది ఎందుకు అత్యుత్తమం (అనేక cases కి):**
1. **N+1 సమస్యే లేదు** — ఒక్క SQL query, అవసరమైన columns మాత్రమే
   SELECT చేస్తుంది (`orders.id, customers.name, orders.total` —
   JOIN automatic గా, ఎందుకంటే `o.customer.name` navigation).
2. **Entity fetch అవసరం లేదు** — persistence context లో ఏ entity
   track అవ్వదు (dirty checking overhead లేదు, ఎందుకంటే `OrderSummary`
   ఒక plain, unmanaged Java record).
3. **DTO discipline ఉచితంగా వస్తుంది** — Book 4 Chapter 3 సూత్రం,
   ఇక్కడ **query level లోనే** enforce అవుతుంది.

**Senior గమనిక:** Constructor expression లో, DTO class యొక్క
**fully-qualified name** అవసరం, మరియు దానికి **సరిపోయే constructor**
ఉండాలి — ఇది refactoring-sensitive (class package మారితే, query string
కూడా మారాలి) — ఇది ఒక చిన్న maintenance cost, కానీ performance benefit
దానికి తగినది చాలా cases లో.

### ENGLISH INTERVIEW ANSWER

"JPQL's `SELECT NEW` constructor expression is where the DTO-not-entity
principle from Book 4 and the N+1/fetch-strategy concerns from this
book's Chapter 3 converge into one clean solution. It generates a single
SQL query selecting exactly the needed columns, joining automatically
based on the object navigation in the query, and returns plain,
unmanaged DTO instances — not tracked entities — so there's zero
persistence-context or dirty-checking overhead, and zero fetch-strategy
decisions to make at all, since no lazy associations are ever touched.
For any read-heavy view that doesn't need to modify the data afterward, I
consider this my default approach rather than fetching full entities and
mapping them to a DTO in application code afterward — it's both more
efficient and structurally guarantees the DTO discipline at the query
level rather than relying on someone remembering to map correctly downstream."

---

## 4.3 CONCEPT: Native SQL — When JPQL Isn't Enough

### TELUGU EXPLANATION

JPQL, entity model మీద పరిమితం — కొన్ని అవసరాలు దీనికి మించి ఉంటాయి:

- **Database-specific functions:** PostgreSQL యొక్క `jsonb` operators,
  window functions (Book 6 Chapter 2) కొన్ని complex forms — JPQL
  వీటిని అన్నింటినీ support చేయకపోవచ్చు.
- **Highly-optimized, hand-tuned queries:** ఒక critical, performance-sensitive
  query కి, DBA/senior engineer specific execution plan hints, specific
  join order కావాలంటే.
- **Bulk/complex reporting queries:** Book 6 Chapter 1-2 (CTEs, window
  functions) ని పూర్తిగా వాడాలంటే.

```java
@Query(value = "SELECT * FROM orders WHERE status = :status " +
               "ORDER BY (data->>'priority')::int DESC", // PostgreSQL jsonb-specific
       nativeQuery = true)
List<Order> findByStatusOrderByJsonPriority(@Param("status") String status);
```

**Trade-off:** Native SQL **database-specific** గా మారుతుంది (JPQL
యొక్క portability కోల్పోతారు) — ఇది **acceptable trade-off**, మీరు
ఇప్పటికే ఒక specific database technology కి committed అయితే (చాలా
production systems ఇలానే ఉంటాయి — "portable across all databases"
అనేది వాస్తవానికి అరుదుగా నిజమైన requirement).

### ENGLISH INTERVIEW ANSWER

"I reach for native SQL when JPQL's entity-model abstraction genuinely
can't express what's needed — database-specific functions like
PostgreSQL's JSONB operators, complex window functions and CTEs from Book
6, or a query where I need to hand-tune the exact SQL for a
performance-critical path. The trade-off is losing JPQL's
database-portability, but in practice, most production systems are
already committed to one specific database technology, so pure
portability is often a theoretical concern rather than a real
requirement — I don't avoid native SQL out of portability purism when the
query genuinely needs a database-specific capability."

---

## 4.4 CONCEPT: Criteria API — Type-Safe, Programmatic Queries for Dynamic Conditions

### TELUGU EXPLANATION

**Criteria API** JPQL ని **Java code గా** (string literals బదులు)
programmatically build చేయడానికి వీలు కల్పిస్తుంది — ఇది **compile-time
type safety** ఇస్తుంది (typo ఉన్న field name, string-based JPQL లో
runtime వరకు కనిపించదు, Criteria API లో compile error).

**ఎప్పుడు ఇది నిజంగా విలువైనది — Dynamic, Optional Filters:**
```java
// ఒక search endpoint, అనేక optional filters తో (status, customer, date range — ఏవైనా ఉండొచ్చు, ఏవైనా ఉండకపోవచ్చు)
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Order> query = cb.createQuery(Order.class);
Root<Order> order = query.from(Order.class);

List<Predicate> predicates = new ArrayList<>();
if (status != null) predicates.add(cb.equal(order.get("status"), status));
if (customerId != null) predicates.add(cb.equal(order.get("customer").get("id"), customerId));
if (fromDate != null) predicates.add(cb.greaterThanOrEqualTo(order.get("createdAt"), fromDate));

query.where(predicates.toArray(new Predicate[0]));
List<Order> results = entityManager.createQuery(query).getResultList();
```

**ఇది string-based JPQL తో ఎందుకు కష్టం:** ఒక dynamic search endpoint
కి, "ఏ filters apply అవుతాయో" ముందుగానే తెలియదు — string concatenation
తో JPQL build చేయడం **verbose, error-prone, SQL-injection-adjacent
risk** (parameters సరిగ్గా బైండ్ చేయకపోతే) — Criteria API ఇది
**programmatically, type-safe గా** చేస్తుంది.

**Senior గమనిక — practical popularity:** Criteria API **verbose** గా
విమర్శించబడుతుంది — చాలా teams **QueryDSL** (ఒక third-party library,
Criteria API కంటే readable syntax ఇచ్చేది) వాడతాయి అదే సమస్యని
పరిష్కరించడానికి, లేదా Spring Data యొక్క **`Specification`** interface
(Criteria API మీదే build అయ్యింది, కానీ Spring-idiomatic గా wrap
చేయబడింది).

### ENGLISH INTERVIEW ANSWER

"Criteria API builds queries programmatically in Java rather than as
string literals, giving compile-time type safety — a typo in a field name
becomes a compile error instead of a runtime surprise. Its real value
shows up for dynamic queries with optional filters — a search endpoint
where any combination of status, customer, or date range might or might
not be provided. Building that with string-concatenated JPQL is verbose
and error-prone; Criteria API lets me conditionally add predicates in
plain Java code. That said, I'll be honest that raw Criteria API is
genuinely verbose, which is exactly why many teams use Spring Data's
`Specification` interface — built on top of Criteria API but far more
ergonomic — or a library like QueryDSL for the same dynamic-query use
case with more readable syntax. I choose Criteria API/Specifications
specifically for dynamic, conditionally-built queries, and plain JPQL for
everything with a fixed, known shape."

---

## 4.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Building a search with optional filters | String-concatenates JPQL based on which filters are present | Uses Criteria API/`Specification` for type-safe, composable predicates |
| Needing a complex reporting query | Tries to force it into JPQL, fighting the abstraction | Uses native SQL, accepting the portability trade-off |
| Returning data for a read-only view | Fetches full entities, maps to DTO in application code | Uses a JPQL constructor expression (`SELECT NEW`) to project directly |
| Choosing JPQL vs native SQL | Defaults to native SQL out of familiarity with plain SQL | Defaults to JPQL, reaches for native SQL only when JPQL genuinely can't express the need |

---

## 4.6 COMMON MISTAKES

1. String-concatenating dynamic JPQL for optional-filter search
   endpoints instead of using Criteria API/Specifications.
2. Fetching full entities and manually mapping to a DTO in application
   code when a JPQL constructor expression would do it in one query.
3. Avoiding native SQL out of misplaced portability concerns for a
   system already committed to one database technology.
4. Not validating dynamic sort/filter field names against an allowlist
   when building Criteria/Specification queries from user input.
5. Overusing raw Criteria API when a `Specification` or QueryDSL would
   be significantly more readable for the same dynamic query need.

---

## 4.7 INTERVIEW QUESTION BANK — CHAPTER 4

**Basic:** 1. What's the key difference between JPQL and native SQL? 2.
What does a JPQL constructor expression (`SELECT NEW`) do?

**Intermediate:** 3. Why is Criteria API valuable specifically for
dynamic/optional-filter queries? 4. When would you choose native SQL
over JPQL?

**Senior:** 5. Design a product search endpoint with five optional
filters (category, price range, in-stock, rating, brand) — compare
implementing it with string-concatenated JPQL versus Criteria API/`Specification`.
6. Why does a JPQL constructor expression avoid the N+1 problem entirely,
tying back to Chapter 3?

**Architect:** 7. Your team is debating standardizing on Criteria API,
Spring Data `Specification`, or QueryDSL for all dynamic queries
org-wide. What factors would drive this decision?

**Scenario:** 8. A dynamic report endpoint lets users choose a sort
column via a request parameter, implemented by directly interpolating
the column name into a JPQL `ORDER BY` clause. What's the risk, and what's the fix?

**Trick:** 9. "JPQL constructor expressions can return managed entities,
just like a normal `SELECT o FROM Order o` query." True or false?

<details><summary>Key answers</summary>

- Q5: String-concatenated JPQL requires manually building the query
  string with conditional `AND` clauses and separately tracking which
  parameters to bind — error-prone and hard to maintain as filters grow.
  Criteria API/`Specification` lets each filter be an independently
  composable `Predicate`, conditionally added based on whether that
  filter was actually provided, combined via `and()`/`or()` — cleaner,
  type-safe, and much easier to extend with a sixth filter later.
- Q6: A constructor expression selects only specific scalar columns
  (via JOINs for navigated fields) directly into a DTO — it never loads
  full entities or touches any lazy association at all, so there's no
  possibility of triggering additional per-row queries; the N+1 problem
  fundamentally can't occur because entities (with their lazy
  associations) are never in the picture.
- Q7: Team familiarity and learning curve, verbosity/readability
  trade-offs (raw Criteria API is verbose, QueryDSL is more concise but
  adds a build-time code generation step, `Specification` is Spring-idiomatic
  and integrates directly with `JpaRepository`), and whether the
  additional QueryDSL dependency/tooling is worth it for the specific
  scale of dynamic-query needs across the organization — there's no
  universally correct answer, but `Specification` is often the pragmatic
  default for teams already fully invested in Spring Data.
- Q8: This is a SQL injection risk — directly interpolating a
  user-controlled column name into JPQL/SQL, even via JPQL, can allow
  injection since it's not a bound parameter (parameters bind to values,
  not identifiers, exactly as covered in Book 6 Chapter 7). Fix: validate
  the requested sort column against a fixed allowlist of legitimate,
  known-safe column names before using it in the query.
- Q9: False — constructor expressions return plain, unmanaged instances
  of the specified DTO class, not managed entities; they're not tracked
  by the persistence context, have no dirty checking applied, and
  changes to them are never persisted automatically.

</details>

---

## 4.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does a JPQL constructor expression avoid all fetch-strategy decisions entirely, unlike fetching a full entity?
- **Coding Check:** Implement a dynamic order-search method using Spring Data's `Specification` interface, supporting optional filters on status, customer ID, and date range.
- **Explanation Check:** Explain in English why Criteria API's verbosity is a real cost worth acknowledging, even while explaining why it's still valuable for dynamic queries.
- **Real-World Check:** Your team needs a report using a recursive CTE (Book 6 Chapter 1) to compute a category hierarchy with product counts. Can this be expressed in JPQL? What's your approach?
- **Senior Check:** When would you choose to fetch full entities via JPQL rather than a DTO projection, even for a read-only view?
- **Master Check:** Design a complete query strategy for an admin dashboard needing: (1) a dynamic, multi-filter order search, (2) a fixed-shape summary report using aggregates, (3) a complex recursive category report. Specify JPQL/Criteria/native SQL for each and justify.

<details><summary>Answers</summary>

- Real-World Check: JPQL doesn't support recursive CTEs — this
  requirement needs native SQL, accepting the database-portability
  trade-off (Chapter 4.3), since the recursive query is fundamentally a
  SQL-dialect feature JPQL's entity-oriented abstraction doesn't model.
- Senior Check: When the view needs to be followed by a modification —
  e.g., displaying an order for an "edit" form where the user will then
  submit changes back — fetching the actual managed entity (or at least
  enough of it) makes sense since you're not purely reading, you're about
  to write; DTO projections are optimal specifically for pure,
  non-editable read views.
- Master Check: (1) Dynamic multi-filter search → Spring Data
  `Specification` (composable, type-safe, avoids string-concatenation
  risk). (2) Fixed-shape summary report with aggregates → JPQL with a
  `SELECT NEW` constructor expression (known shape, benefits from
  single-query DTO projection, avoids N+1 and unnecessary entity
  tracking). (3) Complex recursive category report → native SQL with a
  recursive CTE (JPQL structurally cannot express this), mapped to a DTO
  via Spring's `JdbcTemplate` or a native query's result mapping.

</details>

---

## 4.9 CHEAT SHEET

| Need | Tool |
|---|---|
| Standard entity-model queries | JPQL |
| Read-only projection, avoid N+1/entity overhead entirely | JPQL constructor expression (`SELECT NEW`) |
| Database-specific functions, CTEs, window functions, hand-tuned performance | Native SQL |
| Dynamic, optional-filter queries | Criteria API / Spring Data `Specification` (or QueryDSL) |
| Dynamic sort/filter column from user input | Validate against an allowlist — never interpolate directly |
| Raw Criteria API too verbose | Consider `Specification` or QueryDSL instead |

---

*(Continues to Chapter 5 — Locking in JPA: `@Version` and Pessimistic Annotations.)*
