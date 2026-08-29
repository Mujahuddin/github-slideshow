# BOOK 3 — SPRING FRAMEWORK
## IoC, Dependency Injection, and AOP from First Principles (Telugu + English)

---

## COVER

**Program:** Java Backend / Microservices / Cloud / AI Engineering — Complete Mastery Series
**Book 3 of 25:** Spring Core — IoC Container, Dependency Injection, Bean Lifecycle, AOP

## LEARNING OBJECTIVES

By the end of this book you will be able to:

- Explain *why* Spring's IoC container exists as a solution to a real
  coupling problem, not just define "IoC" and "DI" as buzzwords.
- Choose correctly between constructor, setter, and field injection, and
  defend the choice in a review.
- Explain the bean lifecycle end-to-end and correctly place logic in
  `@PostConstruct`/`@PreDestroy`/`BeanPostProcessor` hooks.
- Diagnose and fix a circular dependency between beans.
- Explain how Spring AOP proxies actually work (JDK dynamic proxy vs
  CGLIB) and predict when a `@Transactional`/`@Cacheable` annotation will
  silently NOT apply (the self-invocation problem) — a very common,
  very costly production gotcha.
- Configure beans via Java Config and component scanning idiomatically
  (modern Spring), while understanding XML configuration well enough to
  read legacy enterprise codebases.

## PREREQUISITES

Book 1 (Core Java), especially Chapter 2 (SOLID/DIP), Chapter 9 (Concurrency
basics for singleton thread-safety).

## SKILL MAP (Master Skill Matrix, Row 4)

| Level | What you should be able to do |
|---|---|
| Beginner | Understand what a "bean" is |
| Intermediate | Configure beans, understand scopes |
| Professional | Use AOP for cross-cutting concerns correctly |
| Senior | Diagnose container-level bugs (circular deps, proxy issues) |
| Lead/Architect | Extend the container (custom `BeanPostProcessor`), design DI-friendly module boundaries |

---

## TABLE OF CONTENTS

- **Chapter 1** — IoC Container & Dependency Injection ✅ `chapter-01-ioc-dependency-injection.md`
- **Chapter 2** — Bean Configuration Styles & the Container ✅ `chapter-02-bean-configuration-styles.md`
- **Chapter 3** — Bean Scopes & Lifecycle ✅ `chapter-03-bean-scopes-lifecycle.md`
- **Chapter 4** — Aspect-Oriented Programming (AOP) ✅ `chapter-04-aspect-oriented-programming.md`
- **Chapter 5** — Events, Environment/Profiles, Testing the Container ✅ `chapter-05-events-environment-testing.md`
- **Final Assessment, Spring Core Mock Interview Round, Capstone Project** ✅ `final-assessment-mock-interview-project.md`

**BOOK 3 STATUS: COMPLETE.** All 5 chapters plus closing materials are
written to the full bilingual (Telugu + English) depth template — each
chapter includes worked code, common mistakes, a full interview question
bank with answers, mastery checkpoints, and a cheat sheet. Special
emphasis throughout on the two costliest real-world Spring gotchas: the
self-invocation AOP proxy bypass (Chapter 4) and singleton-plus-mutable-
state race conditions (Chapter 3), both tied back to Book 1's concurrency
material.

Next in the program: **Book 4 — Spring Boot** (`java-mastery-program/book-04-spring-boot/`).

## WHY THIS BOOK IS SHORTER THAN BOOKS 1-2

Spring Core is a narrower, more focused topic than either Core Java or
DSA — its ideas (IoC, DI, AOP, bean lifecycle) are fewer in number but
each is deep. This book goes deep on those five chapters rather than
padding with breadth; Book 4 (Spring Boot) then builds the practical,
opinionated framework on top of exactly this foundation.

---

*(Chapters below are added incrementally — see each chapter file in this directory.)*
