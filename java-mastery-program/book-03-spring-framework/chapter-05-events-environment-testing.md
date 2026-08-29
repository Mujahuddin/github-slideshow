# CHAPTER 5 — EVENTS, ENVIRONMENT/PROFILES, TESTING THE CONTAINER

---

## 5.1 CONCEPT: Application Events — Decoupled Communication Between Beans

### TELUGU EXPLANATION

Bean A, Bean B ని **నేరుగా call** చేయకుండా, "ఏదో జరిగింది" అని
**broadcast** చేసి, ఆసక్తి ఉన్న ఎవరైనా (Bean B, C, D...) దాన్ని **వినేలా**
చేయాలంటే — ఇది **Observer pattern** యొక్క Spring-native రూపం:
`ApplicationEvent` + `@EventListener`.

```java
// 1. Event class define చేయండి
class OrderPlacedEvent {
    private final Order order;
    OrderPlacedEvent(Order order) { this.order = order; }
    Order getOrder() { return order; }
}

// 2. Event ని publish చేయండి — publisher కి, ఎవరు వింటున్నారో తెలియదు
@Service
class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    void placeOrder(Order order) {
        // ... core order logic ...
        eventPublisher.publishEvent(new OrderPlacedEvent(order)); // decoupled — వినేవాళ్ళు ఎవరో తెలియదు
    }
}

// 3. వేర్వేరు, సంబంధం లేని beans వినొచ్చు
@Component
class EmailNotificationListener {
    @EventListener
    void onOrderPlaced(OrderPlacedEvent event) {
        // send confirmation email
    }
}

@Component
class InventoryUpdateListener {
    @EventListener
    void onOrderPlaced(OrderPlacedEvent event) {
        // decrement inventory
    }
}
```

**ఎందుకు ఇది value చేస్తుంది:** `OrderService` కి **ఎంతమంది వింటున్నారో,
వాళ్ళు ఏం చేస్తారో తెలియదు** — రేపు ఒక కొత్త `SmsNotificationListener`
add చేయాలంటే, `OrderService` code **touch చేయాల్సిన అవసరం లేదు** —
ఇది Book 1 Chapter 2 (OCP — Open/Closed Principle) యొక్క direct, real
Spring application.

**Sync vs Async:** Default గా, `@EventListener` methods **synchronously**,
**అదే thread** లో, publisher block అయ్యి run అవుతాయి. `@Async` (Book 4
లో వివరంగా చూసే `@EnableAsync` తో కలిపి) వాడితే, listener వేరే thread
లో run అవుతుంది — publisher block అవ్వదు. **Senior decision:** email
పంపడం లాంటి slow, non-critical operations కి `@Async` వాడాలి — లేకపోతే,
ఒక slow listener మొత్తం `placeOrder()` call ని నెమ్మది చేస్తుంది.

### ENGLISH INTERVIEW ANSWER

"Application events give me a decoupled, publish-subscribe communication
style between beans — the publisher has zero knowledge of who's listening
or how many listeners exist, which is a direct, practical application of
the Open/Closed Principle: I can add a new listener for an existing event
without touching the publisher's code at all. By default, listeners run
synchronously on the publishing thread, which means a slow listener
directly slows down the publisher — for something like sending a
confirmation email, which shouldn't block the checkout flow, I'd mark that
listener `@Async` so it runs on a separate thread and doesn't hold up the
response. I'm careful about this trade-off though: async listeners mean
the publisher can't know if the listener succeeded or failed without
additional coordination, so I only use async for genuinely
fire-and-forget, non-critical side effects."

---

## 5.2 CONCEPT: The `Environment` Abstraction and `@Profile`

### TELUGU EXPLANATION

**`Environment`** Spring యొక్క unified abstraction — **properties**
(config values) మరియు **profiles** (active environment identifiers,
ఉదా: "dev", "prod") రెండింటినీ handle చేస్తుంది, source ఏదైనా సరే
(properties files, environment variables, system properties, command-line args).

**`@Profile`** ఒక bean ని **నిర్దిష్ట environment(s) లో మాత్రమే**
active చేయడానికి:

```java
@Configuration
class NotifierConfig {
    @Bean
    @Profile("prod") // production లో మాత్రమే active
    Notifier realEmailNotifier() {
        return new RealSmtpNotifier();
    }

    @Bean
    @Profile("dev") // development లో మాత్రమే active
    Notifier fakeNotifier() {
        return new ConsoleLoggingNotifier(); // real emails పంపకుండా, console కి print చేస్తుంది
    }
}
```

**ఎందుకు ఇది ముఖ్యం (real production పరిస్థితి):** Developer laptop మీద
test చేసేటప్పుడు, real customers కి accidentally emails పంపకుండా —
active profile ("dev") ఆధారంగా, సరైన bean automatic గా inject
అవుతుంది, **code మార్చాల్సిన అవసరం లేకుండా**. ఇది Chapter 1 లో మనం
చూసిన "ఒకే `OrderService` ని వేర్వేరు environments లో వేర్వేరు గా
configure చేయడం" అనే సమస్యకి **direct పరిష్కారం**.

### ENGLISH INTERVIEW ANSWER

"The `Environment` abstraction unifies configuration properties and active
profiles regardless of where they come from — properties files,
environment variables, system properties, command-line arguments —
behind one consistent API. `@Profile` builds on top of it to activate
certain beans only in certain environments — the classic example being a
real SMTP-backed notifier in production and a console-logging fake in
development, so a developer running the app locally never accidentally
sends real emails to real customers. This is exactly the environment-
specific configuration problem I mentioned as one reason for DI existing
in the first place — profiles let the *same* `OrderService` code get
wired to completely different `Notifier` implementations depending purely
on which profile is active, with zero code changes."

---

## 5.3 CONCEPT: Testing Spring-Managed Code — What to Actually Load

### TELUGU EXPLANATION

**అత్యంత సాధారణ testing mistake:** ప్రతి test కి **పూర్తి Spring
context** load చేయడం — ఇది slow (context startup time), మరియు
నిజానికి చాలా tests కి **అవసరం లేదు**.

**Testing pyramid, Spring context బట్టి:**

1. **Plain unit tests (Spring context అవసరం లేదు, most common,
   fastest):** Chapter 1 లో మనం నొక్కి చెప్పినట్టు, constructor injection
   వాడితే, ఏ class నైనా `new` తో, mocked dependencies తో, **Spring
   context లేకుండానే** test చేయవచ్చు:
   ```java
   @Test
   void placeOrder_shouldNotify() {
       Notifier mockNotifier = mock(Notifier.class);
       OrderService service = new OrderService(mockNotifier); // Spring లేదు!
       service.placeOrder(someOrder);
       verify(mockNotifier).send(any(), any());
   }
   ```
2. **Slice tests (partial context, moderate speed):** Spring Boot Test
   (Book 4 లో వివరంగా) `@WebMvcTest`, `@DataJpaTest` లాంటివి ఇస్తుంది —
   మొత్తం context కాకుండా, ఒక్క layer మాత్రమే load చేయడం.
3. **Full integration tests (`@SpringBootTest`, slowest, least common,
   కానీ అవసరమైనవి):** మొత్తం application context, real (లేదా
   Testcontainers-backed) dependencies తో — end-to-end wiring correctness
   verify చేయడానికి.

**Senior rule:** **Test pyramid ని ignore** చేయకండి — ఎక్కువ
tests **plain unit tests** గా ఉండాలి (fast, focused), కొన్ని **slice
tests**, చాలా తక్కువ **full integration tests** (slow, brittle,
కానీ genuine end-to-end confidence ఇచ్చేవి).

### ENGLISH INTERVIEW ANSWER

"The single most common testing mistake I see is defaulting to
`@SpringBootTest` — a full application context — for every test, which
makes the suite slow and couples tests to the entire wiring graph when
most of them don't need it. Because I favor constructor injection, the vast
majority of my tests need zero Spring context at all: I construct the
class under test directly with `new`, pass in mocks for its dependencies,
and test behavior in plain JUnit — fast, isolated, and it doesn't even
know Spring exists. I reserve full-context integration tests for
genuinely verifying that the wiring itself is correct end-to-end — bean
configuration, profile activation, real database interaction via
Testcontainers — and I keep that layer intentionally small, following the
standard testing pyramid: many fast unit tests, fewer slice tests, very
few full integration tests."

---

## 5.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Beans need to react to something happening elsewhere | Directly calls each interested bean from the source | Publishes an `ApplicationEvent`, lets listeners subscribe independently |
| Sending a slow notification after a core operation | Does it synchronously, blocking the main flow | Uses `@Async` on the listener specifically for non-critical, slow side effects |
| Different config per environment | Hardcodes `if (isDev) {...} else {...}` in business logic | Uses `@Profile` to swap bean implementations declaratively |
| Writing a test for a service class | Spins up `@SpringBootTest` by habit | Constructs the class directly with mocks; reserves full context tests for genuine integration concerns |

---

## 5.5 COMMON MISTAKES

1. Using events for tightly-coupled, must-happen-in-order logic that's
   actually core business logic — events are for genuinely independent,
   decoupled side effects, not a substitute for direct method calls where
   coupling is actually appropriate.
2. Making a critical, must-succeed operation `@Async` without any
   failure-handling strategy, silently losing failures.
3. Hardcoding environment-specific branches in business logic instead of
   using `@Profile` to swap bean implementations.
4. Defaulting every test to `@SpringBootTest`, producing a slow test suite
   that doesn't reflect real unit-test isolation.
5. Forgetting that a plain `@EventListener` exception can silently
   propagate back to the publisher (since it's synchronous, same-thread by
   default) — an unhandled exception in a listener can actually break the
   publishing code path if not anticipated.

---

## 5.6 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. What problem do Spring events solve? 2. What is `@Profile`
used for?

**Intermediate:** 3. What's the default execution mode (sync/async) for
`@EventListener`, and what's the risk of not knowing this? 4. Why does
constructor injection make plain unit testing easier?

**Senior:** 5. Design an event-driven order-processing flow (order
placed → inventory updated, email sent, analytics recorded) — which parts
should be synchronous events, which async, and why? 6. Explain the
testing pyramid as it applies specifically to a Spring application (unit
vs slice vs full integration tests).

**Architect:** 7. Your monolith is being decomposed into microservices,
and several of these in-process `ApplicationEvent`s need to become
cross-service events (Book 9's Kafka/RabbitMQ territory). What changes
conceptually, and what stays the same in the publish-subscribe mental model?

**Scenario:** 8. A synchronous `@EventListener` throws an exception, and
the team is surprised that the original `placeOrder()` call itself failed
and rolled back. Explain why this happened.

**Trick:** 9. "Using `@Profile` is the only way to have environment-
specific bean configuration in Spring." True or false?

<details><summary>Key answers</summary>

- Q5: Inventory update might need to be synchronous if overselling must be
  prevented immediately (correctness-critical, should ideally happen in
  the same transaction as the order, arguably not even via an event at
  all); email sending should be `@Async` (slow, non-critical to the
  immediate response); analytics recording is a good `@Async` candidate
  too (fire-and-forget, acceptable to lose occasionally without breaking
  the core flow) — the deciding factor each time is "does this need to
  succeed for the operation to be considered complete, and can it tolerate
  being slow/occasionally lost."
- Q6: Unit tests (no Spring context, plain `new` + mocks) should be the
  large base of the pyramid — fast, numerous, testing business logic in
  isolation; slice tests (`@WebMvcTest`, `@DataJpaTest`) form a smaller
  middle layer, testing one architectural layer's Spring-specific wiring
  (e.g., a controller's request mapping) without the full context; full
  `@SpringBootTest` integration tests form the smallest top layer,
  verifying genuine end-to-end wiring and behavior, used sparingly due to
  their cost.
- Q7: The core mental model — publisher doesn't know or care who
  subscribes — carries over directly to Kafka/RabbitMQ; what changes is
  durability (in-process events are lost if the JVM crashes mid-processing;
  message brokers persist them), delivery guarantees (at-least-once,
  exactly-once semantics become explicit concerns), and the fact that
  consumers are now separate processes/services entirely, not just
  separate beans in the same JVM — introducing network failure modes that
  in-process events never had to consider.
- Q8: Synchronous `@EventListener`s run on the same thread, within the
  same call stack, as the publisher — if the listener throws and it's
  within the same transactional boundary (or the exception simply
  propagates back up through `publishEvent()` uncaught), it can cause the
  original transaction to roll back or the original method call to fail,
  exactly as if the listener's code had been inlined directly into
  `placeOrder()`. This is precisely why async, or careful exception
  handling within listeners, matters for genuinely independent side effects.
- Q9: False — `@Profile` is the idiomatic Spring mechanism, but
  environment-specific behavior can also be achieved via
  `@ConditionalOnProperty` and other `@Conditional`-based mechanisms (Book
  4's Spring Boot material), or even plain `Environment.getProperty()`
  checks inside `@Bean` methods — `@Profile` is the cleanest, most common
  choice, not the only one.

</details>

---

## 5.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does a synchronous `@EventListener`'s exception potentially affect the publisher's own success/failure outcome?
- **Coding Check:** Convert a direct method call chain (`OrderService` calling `EmailService.send()` and `AnalyticsService.record()` directly) into an event-driven design with appropriate sync/async choices.
- **Explanation Check:** Explain in English why "most of my tests need zero Spring context" is a sign of good, not lazy, test design.
- **Real-World Check:** Your team ships to three environments (local, staging, production) with different database connection behavior needed in each. Design the `@Profile`-based bean configuration.
- **Senior Check:** When would you choose direct method calls over events, even though events offer more decoupling?
- **Master Check:** Design the test strategy for a `PaymentService` that has: pure calculation logic, a call to an external payment gateway, and a database write. Specify exactly which parts get plain unit tests, which get slice/integration tests, and why.

<details><summary>Answers</summary>

- Real-World Check: Three `@Bean` methods for the datasource/connection
  configuration, each annotated `@Profile("local")`, `@Profile("staging")`,
  `@Profile("prod")` respectively — e.g., local uses an in-memory or
  local Docker database, staging/prod use real managed database
  connections with different credentials/pooling settings, all without
  the consuming code (`@Repository` classes) knowing or caring which is active.
- Senior Check: When the coupling is actually appropriate and desired —
  if step B absolutely must happen as a direct, traceable consequence of
  step A, with clear error propagation and no ambiguity about ordering or
  whether it ran, a direct method call is more honest about that
  dependency than hiding it behind an event, which can make critical
  execution paths harder to trace through the codebase.
- Master Check: Pure calculation logic (fee computation, validation
  rules) gets plain unit tests, no Spring context, no mocks needed beyond
  simple value objects; the external payment gateway call gets tested
  with a mocked gateway client in a unit test (verifying retry/error
  handling logic) AND optionally a slice/contract test against a sandboxed
  real gateway; the database write gets a slice test (`@DataJpaTest`-style,
  Book 4/7) or a full integration test with Testcontainers to verify the
  actual persistence behavior, since mocking a database interaction tests
  much less than actually exercising it against something real.

</details>

---

## 5.8 CHEAT SHEET

| Need | Mechanism |
|---|---|
| Decoupled reaction to something happening | `ApplicationEvent` + `@EventListener` |
| Non-blocking side effect | `@EventListener` + `@Async` |
| Environment-specific bean implementation | `@Profile` |
| Unified config/profile access | `Environment` abstraction |
| Fast, isolated test of business logic | Plain `new` + mocks, no Spring context |
| Test one architectural layer's Spring wiring | Slice tests (`@WebMvcTest`, `@DataJpaTest` — Book 4) |
| Full end-to-end wiring verification | `@SpringBootTest` — use sparingly |

---

## BOOK 3 — CHAPTER 5 COMPLETE

*(All 5 chapters of Book 3's chapter content are now complete. See
`final-assessment-mock-interview-project.md` in this directory for the
cumulative Final Assessment, the Spring Core Mock Interview round, and
the Book 3 capstone Project Assignment.)*
