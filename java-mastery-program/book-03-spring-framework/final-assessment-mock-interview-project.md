# BOOK 3 — FINAL ASSESSMENT, SPRING CORE MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 5 CHAPTERS)

1. Explain, without using the word "magic," how `@Autowired` field
   injection actually gets a value into a private field. *(Ch. 1, 3)*
2. A `@Service` class has a `@Transactional` public method that calls
   another `@Transactional` public method on `this`. Will the second
   method's transactional semantics apply independently? Justify. *(Ch. 4)*
3. Why is constructor injection the only injection style that makes
   circular dependencies fail fast at startup? *(Ch. 1, 3)*
4. Design a singleton `@Service` that must track "requests processed in
   the last minute" safely under concurrent load. What's wrong with a
   plain `int` field, and what's the fix? *(Ch. 3, Book 1 Ch. 9-10)*
5. Why does `@Repository` provide a real technical benefit beyond
   `@Component`, and what is it? *(Ch. 2)*
6. A bean annotated `final class PricingService` has `@Cacheable` on one
   of its methods. Does caching work? Explain precisely why or why not. *(Ch. 4)*
7. Explain the difference between `@Primary` and `@Qualifier`, and design
   a scenario requiring both simultaneously. *(Ch. 2)*
8. Why should most of your Spring application's tests NOT use
   `@SpringBootTest`? *(Ch. 5, Ch. 1)*
9. Design an event-driven flow for a "user signed up" scenario, deciding
   which listeners should be synchronous and which asynchronous, with justification. *(Ch. 5)*
10. Explain why `ApplicationContext`'s eager singleton initialization is
    described as a deliberate "fail fast" design choice rather than a
    performance quirk. *(Ch. 2)*

<details>
<summary>Answer Key</summary>

1. Spring's `AutowiredAnnotationBeanPostProcessor` (a `BeanPostProcessor`,
   Chapter 3) inspects the bean during initialization, finds fields
   annotated `@Autowired`, uses Java reflection (`Field.setAccessible(true)`
   plus `Field.set()`) to bypass normal access control and assign a
   resolved bean directly into the private field — it's reflection-based
   post-processing, not compiler-level or JVM-level magic.
2. No — this is the self-invocation problem (Chapter 4). The call via
   `this` bypasses the AOP proxy entirely, so the second method's
   `@Transactional` annotation is silently ignored; it executes as an
   ordinary method call within whatever transactional context the first
   method's proxy already established (or none, if called from
   outside any transaction).
3. Constructor injection requires all dependencies to exist before the
   object itself can be constructed — a true cycle (A needs B needs A)
   makes it impossible to construct *either* first, and Spring detects
   and fails on this immediately; field/setter injection allows
   constructing the (empty) objects first and wiring fields afterward,
   which can "succeed" at building a cycle that field injection is
   masking, not resolving.
4. A plain `int` field on a singleton is shared, unsynchronized mutable
   state across concurrent request threads — a direct race condition
   (Book 1 Chapter 9). Fix: `AtomicInteger`/`AtomicLong` for a simple
   counter, or a proper sliding-window rate-tracking structure if "last
   minute" windowing is needed precisely (Book 2 Chapter 16's rate
   limiter material).
5. `@Repository` enables Spring's automatic exception translation — it
   converts technology-specific persistence exceptions (JDBC
   `SQLException` subclasses, JPA exceptions) into Spring's unified,
   unchecked `DataAccessException` hierarchy, letting service-layer code
   catch a consistent exception type regardless of which persistence
   technology is underneath.
6. No, caching does not work. CGLIB-based proxying (the default when the
   class doesn't implement an interface) requires subclassing the target
   class, which is impossible for a `final` class — Spring silently skips
   proxy creation or fails depending on configuration, and in either case
   the `@Cacheable` behavior does not apply, with no obvious runtime error
   pointing directly at "the class is final."
7. `@Primary` marks one implementation as the default when multiple
   candidates exist and no more specific instruction is given;
   `@Qualifier` names an exact bean at a specific injection point.
   Scenario: three `PaymentGateway` implementations exist
   (`StripeGateway` marked `@Primary` as the default for most services,
   `PayPalGateway` and `RazorpayGateway` available but not primary) — most
   injection points get Stripe automatically via `@Primary`, while a
   specific `RegionalCheckoutService` uses
   `@Qualifier("razorpayGateway")` to explicitly request the non-default implementation.
8. Because constructor injection (Chapter 1) makes the vast majority of
   classes testable with plain `new` and mocked dependencies — no Spring
   container needed at all — reserving `@SpringBootTest`'s slow, full-context
   startup for the small number of tests that genuinely need to verify
   end-to-end wiring, not as the default for every test.
9. Synchronous, in the same transaction if possible: creating the user
   account record itself (must succeed or the whole signup fails).
   Asynchronous: sending a welcome email (slow, non-critical to signup
   completing), triggering a CRM/analytics sync (fire-and-forget,
   acceptable to occasionally lose or retry separately) — the deciding
   factor is whether the operation must complete for "signup" to be
   considered successful.
10. Because a broken bean configuration failing at application startup —
    a controlled, observable moment, typically caught in CI/deployment
    pipelines — is far preferable to the same failure surfacing
    unpredictably on a live user's first request that happens to touch
    the broken bean, hours or days after a bad deployment.

</details>

---

## PART B — MOCK INTERVIEW: SPRING CORE ROUND

**Interviewer:** "Your team is debugging a production incident: a
`@Transactional` method that's supposed to roll back on failure isn't
rolling back. Walk me through your investigation."

> *What's being tested:* whether self-invocation is the first hypothesis
> you reach for, not a random guess.

**Model answer:** "My first hypothesis, given how common this specific
failure mode is, is self-invocation — is this transactional method being
called from within the same class via `this`, bypassing the AOP proxy
entirely? I'd check the call site. Second, I'd check whether the class or
method is `final`, breaking CGLIB proxying silently. Third, I'd verify the
exception being thrown is actually an unchecked exception or explicitly
configured for rollback — `@Transactional`'s default rollback behavior
only covers unchecked exceptions unless `rollbackFor` is specified, which
is itself a very common, separate gotcha worth checking. Only after ruling
out these three well-known causes would I dig into more exotic
possibilities like propagation settings or transaction manager
misconfiguration."

**Follow-up:** "What's the fix once you've confirmed it's self-invocation?"
(Extract the transactional method into a separate bean and call it
externally through injection — the clean fix from Chapter 4, not a
workaround.)

---

**Interviewer:** "Explain dependency injection to me as if I'm a junior
developer who has only ever used `new` to create objects."

> *What's being tested:* can you teach the concept from first principles,
> not just recite a definition — this is itself the Telugu-to-English
> communication skill this whole program emphasizes.

**Model answer:** "Right now, when your class needs something, you write
`new SomethingImpl()` right inside it. That means your class is
permanently glued to that one specific implementation — if you ever want
to test your class with a fake version of that dependency, or swap in a
different implementation later, you have to go edit your class's code.
Dependency Injection flips this: instead of your class creating what it
needs, it just declares 'I need something that can do X' — usually as a
constructor parameter typed to an interface — and something else, the
Spring container, decides what actual implementation to hand it at
runtime. Your class never changes regardless of which implementation gets
plugged in. That's really the whole idea — moving the 'which concrete
thing do I use' decision out of the class and into external
configuration."

**Follow-up:** "Why is this specifically valuable for testing?" (Because
you can hand the constructor a mock/fake implementation directly in a
test, with no framework involved at all, and verify behavior in complete
isolation.)

---

**Interviewer:** "Design the bean configuration for a service that needs
three environment-specific behaviors: local development uses an in-memory
fake payment gateway, staging uses a sandboxed real gateway, and
production uses the live gateway."

**Model answer:** "I'd define a `PaymentGateway` interface, then three
implementations — `FakePaymentGateway`, `SandboxPaymentGateway`, and
`LivePaymentGateway` — each registered as a `@Bean` in a
`@Configuration` class, annotated `@Profile("local")`,
`@Profile("staging")`, and `@Profile("prod")` respectively. The consuming
service depends only on the `PaymentGateway` interface via constructor
injection and never knows or cares which implementation is active — the
active Spring profile, set via an environment variable or startup
argument per environment, decides which bean gets wired in. This means
zero code branches like `if (env.equals("local"))` anywhere in business
logic — the environment-specific decision lives entirely in configuration,
which is exactly the separation of concerns DI and profiles are meant to provide."

---

## PART C — CAPSTONE PROJECT: "NOTIFICATION ORCHESTRATION SERVICE"

**Goal:** A small Spring (no Boot yet — pure `ApplicationContext`,
reinforcing that Spring's core ideas don't require Spring Boot) console
application demonstrating every chapter of Book 3 working together.

**Requirements:**

1. Define a `Notifier` interface with `EmailNotifier`, `SmsNotifier`, and
   `ConsoleNotifier` (a fake, for local testing) implementations —
   `ConsoleNotifier` active under a `"local"` profile, the real ones under
   `"prod"` (Ch. 1, 2, 5).
2. `NotificationOrchestrator` (`@Service`, constructor-injected,
   singleton, stateless) accepts a `List<Notifier>` (Spring injects all
   matching beans automatically) and fans a message out to all active notifiers.
3. Add a `@Transactional`-annotated `recordNotificationHistory()` method
   on a separate `NotificationHistoryService` bean — deliberately call it
   both correctly (externally, through injection) and incorrectly (as a
   commented-out self-invocation example) to demonstrate you understand
   the difference (Ch. 4).
4. Publish an `NotificationSentEvent` after each successful send;
   implement one synchronous listener (updates an in-memory count,
   `AtomicInteger` — tying to Book 1's concurrency material) and one
   `@Async` listener (simulates a slow analytics call) (Ch. 3, 5).
5. Write plain JUnit tests (no Spring context) for
   `NotificationOrchestrator`'s fan-out logic using mocked `Notifier`s,
   and exactly one `AnnotationConfigApplicationContext`-based integration
   test verifying the correct `Notifier` bean is wired for each profile (Ch. 5).

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| DI style | 100% constructor injection, zero field injection |
| Profile switching | Running with `-Dspring.profiles.active=local` vs `prod` visibly changes behavior with no code changes |
| Self-invocation demonstration | The commented-out broken version and working version are both present and explained in comments |
| Event sync/async justification | README states why each listener is sync or async |
| Test pyramid | Majority of tests are Spring-context-free |

---

*(This completes BOOK 3 — SPRING FRAMEWORK. Book 4 — Spring Boot — builds
the opinionated, production-ready framework layer on top of exactly this
IoC/DI/AOP foundation.)*
