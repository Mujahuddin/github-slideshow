# BOOK 6 — SQL + JDBC + DATABASES
## Relational Data at Depth (Telugu + English)

---

## COVER

**Program:** Java Backend / Microservices / Cloud / AI Engineering — Complete Mastery Series
**Book 6 of 25:** SQL, Indexing, Transactions, Locking, JDBC, Production Database Troubleshooting

## LEARNING OBJECTIVES

By the end of this book you will be able to:

- Write joins, subqueries, CTEs, and window functions confidently, and
  choose between them based on readability and performance.
- Explain how a B-tree index actually works, and design composite
  indexes with the correct column order.
- Read an execution plan (`EXPLAIN`) and use it to diagnose a slow query.
- Explain ACID properties and every standard isolation level precisely,
  including which read phenomena each one prevents.
- Diagnose and resolve database deadlocks, and choose correctly between
  pessimistic and optimistic locking.
- Use JDBC correctly and safely (PreparedStatement, connection pooling)
  without an ORM in the loop.
- Diagnose real production database problems: slow queries, connection
  pool exhaustion, lock contention.

## PREREQUISITES

Book 4 (Spring Boot) Chapter 5 (Transactions) — this book goes underneath
that material to the actual database mechanics `@Transactional` sits on
top of. Book 1 Chapter 9 (deadlocks) — the DB-level deadlock material
directly parallels and extends that.

## SKILL MAP (Master Skill Matrix, Row 9)

| Level | What you should be able to do |
|---|---|
| Beginner | Write SELECT/JOIN queries |
| Intermediate | Subqueries, indexes, basic transactions |
| Professional | Window functions, isolation levels, execution plans |
| Senior | Diagnose deadlocks, optimize slow queries, design indexes |
| Lead/Architect | Schema/sharding strategy, database technology selection |

---

## TABLE OF CONTENTS

- **Chapter 1** — SQL Fundamentals: Joins, Subqueries, CTEs ✅ `chapter-01-sql-joins-subqueries-ctes.md`
- **Chapter 2** — Window Functions ✅ `chapter-02-window-functions.md`
- **Chapter 3** — Indexes & Query Optimization ✅ `chapter-03-indexes-query-optimization.md`
- **Chapter 4** — Normalization & Schema Design ✅ `chapter-04-normalization-schema-design.md`
- **Chapter 5** — ACID & Transaction Isolation Levels ✅ `chapter-05-acid-isolation-levels.md`
- **Chapter 6** — Locks & Deadlocks ✅ `chapter-06-locks-deadlocks.md`
- **Chapter 7** — JDBC Fundamentals ✅ `chapter-07-jdbc-fundamentals.md`
- **Chapter 8** — Production Database Troubleshooting ✅ `chapter-08-production-database-troubleshooting.md`
- **Final Assessment, SQL/JDBC Mock Interview Round, Capstone Project** ✅ `final-assessment-mock-interview-project.md`

**BOOK 6 STATUS: COMPLETE.** All 8 chapters plus closing materials are
written to the full bilingual (Telugu + English) depth template. This
book repeatedly maps database mechanics onto Book 1/2 concepts already
learned — B-Tree indexes as disk-durable binary search, connection
pooling as `ThreadPoolExecutor` for connections, DB deadlock prevention
via the same consistent lock ordering as JVM-level deadlocks — so the
underlying reasoning transfers instead of being re-learned from scratch.

Next in the program: **Book 7 — JPA + Hibernate** (`java-mastery-program/book-07-jpa-hibernate/`).

## SCOPE NOTE

NoSQL (MongoDB, Redis) is Book 10's dedicated territory. This book is
relational-only, going deep rather than broad. JPA/Hibernate's
object-relational mapping layer — entity lifecycle, N+1, fetch
strategies — is Book 7, built directly on this book's SQL/transaction
foundation.

---

*(Chapters below are added incrementally — see each chapter file in this directory.)*
