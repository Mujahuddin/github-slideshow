# BOOK 4 — FINAL ASSESSMENT, SPRING BOOT MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 8 CHAPTERS)

1. Explain why adding `spring-boot-starter-security` to a project can
   suddenly make previously-open endpoints require authentication —
   connect this precisely to auto-configuration mechanics. *(Ch. 1, 6)*
2. Design a `@ConfigurationProperties` class for a third-party API
   client, with validation, and explain how its values could be
   overridden in production via Kubernetes without rebuilding the JAR. *(Ch. 2)*
3. A controller returns a JPA `@Entity` directly and it works fine in
   testing but throws `LazyInitializationException` intermittently in
   production. Explain why, and the fix. *(Ch. 3)*
4. Design the complete exception-handling chain for an endpoint: Bean
   Validation failure, a custom domain exception, and an unexpected
   exception — show all three `@ExceptionHandler` methods. *(Ch. 4)*
5. Why does a checked exception thrown from a `@Transactional` method NOT
   roll back the transaction by default, and what are the two ways to fix this? *(Ch. 5)*
6. Explain the difference between 401 and 403 with a concrete example of
   each in a real API. *(Ch. 6)*
7. Your production Actuator `/actuator/env` endpoint is publicly
   reachable. What's the immediate risk, and how do you fix it? *(Ch. 7)*
8. Design the test strategy (which annotation, what's mocked) for testing
   that an endpoint returns 400 with field-level errors when required
   fields are missing. *(Ch. 4, 8)*
9. Why is "tests pass with H2" insufficient evidence that a native SQL
   query is production-correct? *(Ch. 8)*
10. Trace the full request lifecycle for a `POST /api/orders` call — from
    the security filter chain, through validation, through the service
    layer's transaction boundary, to the response — naming which chapter's
    concept governs each stage.

<details>
<summary>Answer Key</summary>

1. `spring-boot-starter-security` puts Spring Security classes on the
   classpath, which satisfies `@ConditionalOnClass` checks in Spring
   Security's auto-configuration classes, activating a default security
   configuration that requires authentication for all endpoints unless
   explicitly overridden with a custom `SecurityFilterChain` — a direct
   application of Chapter 1's "starters bring dependencies, conditions
   react to them" model.
2. Constructor-bound `@ConfigurationProperties` class with `@Validated`
   and constraints like `@NotBlank` on the base URL, `@Min(1)` on
   timeout/retries; production override via a Kubernetes ConfigMap
   setting environment variables, which — per Chapter 2's precedence
   order — outrank the base `application.yml` values without requiring a rebuild.
3. The entity likely has a lazy-loaded association; in testing, the
   Hibernate session may still be open when serialization happens
   (different transaction/session timing), but in production, under real
   request timing, the session closes before Jackson serializes the
   response, throwing `LazyInitializationException`. Fix: never return
   entities directly — map to a DTO within the transactional/session
   boundary (Chapter 3).
4. `@ExceptionHandler(MethodArgumentNotValidException.class)` → 400 with
   field errors; `@ExceptionHandler(YourDomainException.class)` → the
   status appropriate to that specific business failure (404, 409, etc.);
   `@ExceptionHandler(Exception.class)` → 500 with a sanitized message
   and correlation ID, full details logged server-side only (Chapter 4).
5. Spring's default rollback rule only covers unchecked exceptions.
   Fixes: (1) add `rollbackFor = YourCheckedException.class` to the
   `@Transactional` annotation, or (2) convert the exception to an
   unchecked one (favoring Book 1's unchecked-domain-exception design
   philosophy, which also sidesteps needing to remember `rollbackFor` at all).
6. 401: a request to a protected endpoint with no `Authorization` header
   at all, or an expired/invalid token — identity couldn't be established.
   403: a validly authenticated regular user attempting to hit an
   admin-only endpoint — identity is known, but that identity lacks
   permission for this specific action.
7. Immediate risk: environment variables (potentially including secrets/
   credentials) are exposed to anyone who can reach the endpoint. Fix:
   restrict Actuator endpoint exposure to only what's needed
   (`health`, `info`, `metrics`), secure remaining endpoints behind Spring
   Security, and rotate any credentials that may have been exposed.
8. `@WebMvcTest` for the specific controller, with the service layer
   mocked via `@MockitoBean` (the service isn't relevant to testing
   validation behavior), using `MockMvc` to POST an invalid JSON payload
   and asserting `status().isBadRequest()` plus the structure of the
   returned field errors (Chapter 8, exercising Chapter 4's validation).
9. H2's SQL dialect and behavior differ from production databases like
   PostgreSQL/MySQL in ways that can make an invalid or incorrect native
   query pass in tests while failing (or behaving differently) in
   production — genuine confidence requires testing against the real
   database engine, typically via Testcontainers.
10. Security filter chain (Ch. 6) authenticates/authorizes the request
    first → `@Valid` triggers Bean Validation on the request body (Ch. 4)
    → if invalid, `@RestControllerAdvice` returns 400 immediately (Ch. 4)
    → if valid, the controller (Ch. 3) delegates to the service, whose
    `@Transactional` method establishes a transaction boundary (Ch. 5) →
    on success, a DTO (not an entity, Ch. 3) is returned wrapped in a
    `ResponseEntity` with the correct status code (Ch. 3) → throughout,
    structured logs with a correlation ID are written (Ch. 7).

</details>

---

## PART B — MOCK INTERVIEW: SPRING BOOT ROUND

**Interviewer:** "A production incident: after a deploy, a
`@Transactional` service method that used to roll back correctly on
failure now commits partial data. Nothing in the transaction logic
changed — only a new checked exception type was introduced for a specific
validation failure. Diagnose this live."

> *What's being tested:* whether Chapter 5's rollback-rule gotcha is
> instantly recognized, not treated as a mysterious regression.

**Model answer:** "This pattern — a new exception type breaking rollback
behavior with no changes to the transaction logic itself — points directly
at Spring's default rollback rule: it only covers unchecked exceptions. If
the new exception is a checked exception, extending `Exception` rather
than `RuntimeException`, the transaction will commit despite it being
thrown, which matches exactly what's described. I'd verify by checking the
new exception's class hierarchy, and fix it either by adding `rollbackFor
= NewException.class` to the `@Transactional` annotation, or — my
preferred fix — converting it to an unchecked exception in line with our
broader exception-handling convention, so this entire class of bug can't
recur for future exception types either."

**Follow-up:** "Why didn't this show up in testing?" (Likely because unit
tests mocked the exception-throwing dependency and asserted the exception
was thrown, without verifying actual database state afterward — an
integration-level test asserting the transaction actually rolled back
would have caught this.)

---

**Interviewer:** "Walk me through designing a new REST endpoint from
scratch: `POST /api/customers` to register a new customer, including
validation, error handling, and testing."

**Model answer:** "I'd start with two DTOs — `CreateCustomerRequest` with
Bean Validation annotations (`@NotBlank` on name, `@Email` on email,
`@NotBlank` with a custom `@ValidPasswordStrength` validator on password
if needed), and `CustomerResponse` with only the safe-to-expose fields —
critically, no password field at all. The controller method takes
`@Valid @RequestBody CreateCustomerRequest`, delegates to
`CustomerService.register(...)`, and returns `ResponseEntity.status(CREATED)`
with a `Location` header. The service method is `@Transactional`, throwing
a domain-specific `DuplicateEmailException` (unchecked) if the email
already exists — handled by a `@RestControllerAdvice` mapping it to 409
Conflict, alongside the standard `MethodArgumentNotValidException` handler
for validation failures and a sanitized catch-all for anything
unexpected. For testing: a `@WebMvcTest` verifying the controller's
request mapping, validation, and error responses with the service mocked;
a plain unit test for `CustomerService`'s business logic with a mocked
repository; and a `@DataJpaTest` (ideally Testcontainers-backed) verifying
the actual duplicate-email database constraint behaves as expected."

**Follow-up:** "Where does password hashing happen in this flow?" (In the
service layer, using the injected `PasswordEncoder` bean from Chapter 6's
security configuration — never in the controller, and the raw password
should never be logged or persisted anywhere.)

---

**Interviewer:** "Your team's Actuator health check reports the service
as healthy, but customers are reporting failed checkouts because the
payment gateway integration is down. Why did health checks not catch
this, and how do you fix it?"

**Model answer:** "By default, Spring Boot's built-in health indicators
check things like disk space and database connectivity, but they have no
knowledge of a specific downstream dependency like a payment gateway
unless I explicitly tell them to. I'd add a custom `HealthIndicator` that
actively checks payment gateway reachability — a lightweight ping or
status call — and decide deliberately whether a payment gateway outage
should mark the whole service `DOWN` (stopping traffic entirely) or
report a degraded-but-still-serving status, depending on whether other
parts of the service can still function meaningfully without payments
working. I'd lean toward marking it `DOWN` for a service whose primary
purpose is checkout, since serving traffic that will just fail at
checkout provides little value and actively confuses monitoring."

---

## PART C — CAPSTONE PROJECT: "PRODUCTION-READY TASK MANAGEMENT API"

**Goal:** A Spring Boot REST API demonstrating every chapter of Book 4
working together as one coherent, defensible, production-minded service.

**Requirements:**

1. `Task` entity (JPA) with DTOs for create/update/response — never an
   entity returned from any controller (Ch. 3).
2. `@ConfigurationProperties`-based configuration for task limits (max
   tasks per user, default page size), validated at startup (Ch. 2).
3. Full validation on create/update requests, with a `@RestControllerAdvice`
   handling validation failures, a custom `TaskNotFoundException` (404),
   a custom `TaskLimitExceededException` (409), and a sanitized catch-all (Ch. 4).
4. `@Transactional` service methods with explicit propagation choices
   justified in code comments — including one `REQUIRES_NEW` audit-log
   write that must survive a rolled-back task creation failure (Ch. 5).
5. Spring Security securing the API: authenticated users can only
   see/modify their own tasks; an admin role can see all tasks —
   demonstrating both URL-level and `@PreAuthorize` method-level
   authorization (Ch. 6).
6. Actuator configured with minimal, secured exposure; structured
   logging with correlation IDs on every request; graceful shutdown
   configured (Ch. 7).
7. A full test suite: unit tests for business logic, `@WebMvcTest` for
   controllers, `@DataJpaTest` for repositories, and exactly one
   `@SpringBootTest` verifying end-to-end wiring (Ch. 8).

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Auto-configuration understanding | README explains which starters were added and what auto-configuration each activated |
| Config externalization | Task limits changeable via environment variable with zero code changes |
| DTO discipline | Zero entities ever appear in a controller's method signature |
| Transaction correctness | The REQUIRES_NEW audit log demonstrably survives a rollback (tested) |
| Security correctness | A non-owner, non-admin user is verified (via test) to get 403/404 on another user's task |
| Test pyramid shape | Vast majority of tests are unit/slice; exactly one full integration test |

---

*(This completes BOOK 4 — SPRING BOOT. Book 5 — REST APIs + Web
Services — goes deeper into API design itself: versioning, pagination,
OpenAPI/Swagger, OAuth2 flows, and API Gateway patterns.)*
