# BOOK 7 — JPA + HIBERNATE
## The Object-Relational Mapping Layer, Done Correctly (Telugu + English)

---

## COVER

**Program:** Java Backend / Microservices / Cloud / AI Engineering — Complete Mastery Series
**Book 7 of 25:** Entity Lifecycle, Relationships, Fetching, N+1, Locking, Caching, Production Hibernate

## LEARNING OBJECTIVES

By the end of this book you will be able to:

- Explain the entity lifecycle (Transient/Managed/Detached/Removed) and
  exactly when Hibernate flushes changes to the database.
- Map relationships correctly, understanding the owning side and cascade
  behavior precisely enough to avoid data-loss bugs.
- Diagnose and fix the N+1 query problem — the single most common
  Hibernate performance bug — using the right tool for each situation.
- Choose between JPQL, native SQL, and the Criteria API deliberately.
- Apply optimistic (`@Version`) and pessimistic (`@Lock`) locking
  correctly, building directly on Book 6's locking chapter.
- Diagnose `LazyInitializationException` precisely and know every valid
  fix, including the trade-offs of Open Session In View.
- Explain why entity `equals`/`hashCode` is uniquely tricky in JPA, and
  implement it correctly.

## PREREQUISITES

Book 6 (SQL + JDBC + Databases) — this book is the ORM layer sitting on
top of that book's SQL, transaction, and locking foundation. Book 4
Chapter 3's DTO-vs-entity rule and Chapter 5's `@Transactional` material
are assumed and extended here.

## SKILL MAP (Master Skill Matrix, Row 13)

| Level | What you should be able to do |
|---|---|
| Beginner | Define entities, basic CRUD via `JpaRepository` |
| Intermediate | Map relationships, write JPQL |
| Professional | Diagnose and fix N+1, choose fetch strategies deliberately |
| Senior | Debug `LazyInitializationException`, design locking strategy |
| Lead/Architect | Set ORM usage standards, decide when to bypass the ORM entirely |

---

## TABLE OF CONTENTS

- **Chapter 1** — Entity Lifecycle & Persistence Context
- **Chapter 2** — Relationships & Mapping
- **Chapter 3** — Fetching Strategies & The N+1 Problem
- **Chapter 4** — JPQL, Native Queries, and Criteria API
- **Chapter 5** — Locking in JPA: `@Version` and Pessimistic Annotations
- **Chapter 6** — Second-Level Cache & Performance Optimization
- **Chapter 7** — Common Hibernate Pitfalls & Production Patterns
- **Final Assessment, JPA/Hibernate Mock Interview Round, Capstone Project**

---

*(Chapters below are added incrementally — see each chapter file in this directory.)*
