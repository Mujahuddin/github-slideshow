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

- **Chapter 1** — SQL Fundamentals: Joins, Subqueries, CTEs
- **Chapter 2** — Window Functions
- **Chapter 3** — Indexes & Query Optimization
- **Chapter 4** — Normalization & Schema Design
- **Chapter 5** — ACID & Transaction Isolation Levels
- **Chapter 6** — Locks & Deadlocks
- **Chapter 7** — JDBC Fundamentals
- **Chapter 8** — Production Database Troubleshooting
- **Final Assessment, SQL/JDBC Mock Interview Round, Capstone Project**

## SCOPE NOTE

NoSQL (MongoDB, Redis) is Book 10's dedicated territory. This book is
relational-only, going deep rather than broad. JPA/Hibernate's
object-relational mapping layer — entity lifecycle, N+1, fetch
strategies — is Book 7, built directly on this book's SQL/transaction
foundation.

---

*(Chapters below are added incrementally — see each chapter file in this directory.)*
