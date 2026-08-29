# CHAPTER 8 — TESTING SPRING BOOT APPLICATIONS

---

## 8.1 CONCEPT: The Testing Layers, Concretely

### TELUGU EXPLANATION

Book 3 Chapter 5 లో మనం testing pyramid concept చూశాం. ఇక్కడ Spring
Boot **specific tools** తో దాన్ని concrete గా చేద్దాం:

| Test type | Annotation | Context loaded | ఎప్పుడు వాడాలి |
|---|---|---|---|
| **Unit test** | ఏదీ లేదు (plain JUnit) | ఏమీ లేదు | Business logic, most tests |
| **Web layer slice** | `@WebMvcTest(OrderController.class)` | Controller layer + MVC infra మాత్రమే | Controller request mapping, validation, exception handling |
| **Data layer slice** | `@DataJpaTest` | JPA/repository layer + embedded DB మాత్రమే | Repository query correctness |
| **Full integration** | `@SpringBootTest` | **మొత్తం** application context | End-to-end wiring verification |

**Senior rule (Book 3 నుండి కొనసాగింపు):** ఎక్కువ tests **plain unit
tests** గా ఉండాలి. Slice tests **మధ్యస్థ** గా — ఒక్క layer యొక్క
Spring-specific behavior verify చేయడానికి, మొత్తం context startup cost
లేకుండా. `@SpringBootTest` **అరుదుగా**, గట్టి, genuine end-to-end
confidence కోసం మాత్రమే.

---

## 8.2 CONCEPT: `@WebMvcTest` + `MockMvc` — Testing the Controller Layer in Isolation

### TELUGU EXPLANATION

`@WebMvcTest` **controller layer మాత్రమే** load చేస్తుంది (service
layer beans **కాదు**) — service dependencies ని **mock** చేయాలి
(`@MockBean` — లేదా Spring Boot 3.4+ లో `@MockitoBean`).

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc; // real HTTP server లేకుండా, HTTP-level requests simulate చేస్తుంది

    @MockitoBean
    private OrderService orderService; // real service కాదు — mock

    @Test
    void getOrder_whenFound_returns200() throws Exception {
        when(orderService.findById("123"))
                .thenReturn(Optional.of(new OrderResponse("123", BigDecimal.TEN, "PLACED")));

        mockMvc.perform(get("/api/orders/123"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value("123"))
                .andExpect(jsonPath("$.totalAmount").value(10));
    }

    @Test
    void getOrder_whenNotFound_returns404() throws Exception {
        when(orderService.findById("999")).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/orders/999"))
                .andExpect(status().isNotFound());
    }

    @Test
    void createOrder_withInvalidRequest_returns400WithFieldErrors() throws Exception {
        String invalidJson = """
                {"customerId": "", "items": []}
                """; // Chapter 4 Bean Validation ని test చేస్తోంది

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidJson))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors").isArray());
    }
}
```

**ఇది ఎందుకు valuable:** ఇది Chapter 3 (`ResponseEntity`, status codes),
Chapter 4 (`@Valid`, `@RestControllerAdvice`) రెండింటినీ ఒకేసారి,
**real HTTP semantics తో** (కానీ real server startup లేకుండా, fast గా)
test చేస్తుంది. `MockMvc` నిజంగా Spring MVC dispatch mechanism ద్వారా
వెళ్తుంది — కేవలం method ని direct గా call చేయడం కాదు — కాబట్టి
`@Valid`, `@ExceptionHandler` వంటివి **నిజంగా exercise అవుతాయి**.

### ENGLISH INTERVIEW ANSWER

"`@WebMvcTest` loads only the web layer — the specific controller plus
MVC infrastructure like `@RestControllerAdvice` — not the whole
application context, so it starts much faster than a full
`@SpringBootTest`. I mock the service layer with `@MockitoBean` since
that's a boundary I don't want this test crossing. What makes this
genuinely valuable over a plain unit test calling the controller method
directly is that `MockMvc` routes through Spring's actual dispatch
mechanism — so `@Valid` validation, content negotiation, and my centralized
exception handling from Chapter 4 are all really exercised, not bypassed.
This is exactly the right layer to verify 'does a malformed request
correctly produce a 400 with field-level errors' — a plain unit test
calling the controller method with a Java object wouldn't even exercise
JSON deserialization or validation at all."

---

## 8.3 CONCEPT: `@DataJpaTest` — Testing the Repository Layer

### TELUGU EXPLANATION

`@DataJpaTest` **JPA-related beans మాత్రమే** load చేస్తుంది (repositories,
entity manager) — default గా, ఒక **embedded, in-memory database** (H2)
వాడుతుంది, ప్రతి test method **transaction లో run అయ్యి, చివర
rollback** అవుతుంది (test isolation, ఒక test యొక్క data మరో test ని
ప్రభావితం చేయకుండా).

```java
@DataJpaTest
class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager entityManager; // setup data కోసం, repository కంటే direct control

    @Test
    void findByCustomerIdAndStatus_returnsMatchingOrders() {
        Order order = new Order("CUST-1", OrderStatus.PLACED);
        entityManager.persistAndFlush(order); // Chapter 5 లో మనం చూసిన derived query method ని test చేస్తోంది

        List<Order> results = orderRepository.findByCustomerIdAndStatus("CUST-1", OrderStatus.PLACED);

        assertThat(results).hasSize(1);
    }
}
```

**⚠️ అత్యంత ముఖ్యమైన senior-level gotcha:** H2 (in-memory, test-only
database) **production database (PostgreSQL/MySQL/Oracle) తో identical
గా ప్రవర్తించదు** — SQL dialect differences, constraint enforcement
differences, JSON/array column support differences ఉన్నాయి. **"Tests
pass with H2" ≠ "code production DB తో సరిగ్గా పని చేస్తుంది"** అనేది
ఒక genuine, తరచుగా తప్పుగా అర్థం చేసుకునే విషయం.

**పరిష్కారం: Testcontainers** (Book 7లో వివరంగా చూస్తాం) — ఇది
**నిజమైన PostgreSQL** ని (Docker container గా) test సమయంలో spin up
చేస్తుంది, H2 బదులుగా — ఇది "test గా ఉంటూనే, production-realistic"
behavior ఇస్తుంది.

### ENGLISH INTERVIEW ANSWER

"`@DataJpaTest` loads just the JPA layer against an embedded database by
default, with each test running in a transaction that rolls back
afterward for isolation. The gotcha every senior engineer should know
here: the default embedded H2 database does NOT behave identically to
your real production database — different SQL dialect nuances, different
constraint enforcement, different support for database-specific features.
Tests passing against H2 is not proof your queries work correctly against
PostgreSQL or MySQL in production — I've seen native queries pass locally
against H2 and then fail in production due to a syntax difference H2
happened to tolerate. This is exactly why Testcontainers — spinning up a
real PostgreSQL instance in Docker for the test's duration — has become
the standard, more trustworthy alternative for any test that needs
genuine confidence in database behavior, which we'll build on fully in
Book 7."

---

## 8.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Testing a controller | Calls the method directly with `new SomeDto()` | Uses `@WebMvcTest` + `MockMvc` to exercise real HTTP/validation/exception-handling behavior |
| Testing a repository | Trusts that "tests pass" means production-correct | Knows H2 vs real-DB behavioral differences, considers Testcontainers |
| Test suite speed | Every test uses `@SpringBootTest` | Reserves full context tests for genuine end-to-end concerns; most tests need none or a slice |
| Mocking in slice tests | Confused about how to provide a "fake" service | Uses `@MockitoBean`, precisely scoped to what the slice test needs |

---

## 8.5 COMMON MISTAKES

1. Defaulting to `@SpringBootTest` for every test, slowing the suite
   dramatically as the application grows.
2. Assuming a test passing against H2 guarantees correctness against the
   real production database.
3. Testing controllers by calling the Java method directly instead of
   through `MockMvc`, missing real HTTP/validation/serialization behavior.
4. Not understanding `@DataJpaTest`'s automatic rollback-per-test
   behavior, leading to confusion about "missing" data between tests that
   were never meant to share state.
5. Over-mocking in integration tests — mocking so much that the test no
   longer verifies anything meaningful about how components actually integrate.

---

## 8.6 INTERVIEW QUESTION BANK — CHAPTER 8

**Basic:** 1. What's the difference between `@WebMvcTest` and
`@SpringBootTest`? 2. Why does `MockMvc` give more confidence than
calling a controller method directly?

**Intermediate:** 3. Why does `@DataJpaTest`'s default rollback behavior
matter for test isolation? 4. What's the key risk of relying solely on
H2 for repository tests?

**Senior:** 5. Design the full test strategy (unit/slice/integration)
for a `PaymentController` → `PaymentService` → `PaymentRepository` +
external gateway client stack. 6. When would you choose Testcontainers
over H2, and what's the trade-off (speed vs. fidelity)?

**Architect:** 7. Your CI pipeline takes 45 minutes to run the full test
suite, and most of that time is `@SpringBootTest` full-context startups.
How would you restructure the test suite to dramatically cut CI time
without losing meaningful coverage?

**Scenario:** 8. A repository test passes locally (H2) but the same query
fails in production (PostgreSQL) with a SQL syntax error. Explain how
this happened and how it could have been caught earlier.

**Trick:** 9. "Slice tests like `@WebMvcTest` provide the same confidence
as a full `@SpringBootTest` since they still use real Spring
infrastructure." True or false?

<details><summary>Key answers</summary>

- Q5: Unit tests (no Spring context) for `PaymentService`'s business
  logic with a mocked repository and mocked gateway client — the bulk of
  the test suite. `@WebMvcTest` for `PaymentController`'s request
  mapping/validation/exception handling with `PaymentService` mocked.
  `@DataJpaTest` (ideally with Testcontainers) for `PaymentRepository`'s
  actual query correctness. A small number of full `@SpringBootTest`
  integration tests verifying the entire stack wires together correctly
  end-to-end, possibly including a mocked or sandboxed external gateway.
- Q6: Testcontainers gives production-fidelity at the cost of slower test
  startup (spinning up a real Docker container) versus H2's near-instant
  in-memory startup; the trade-off is worth it specifically for tests
  where SQL dialect correctness or database-specific behavior genuinely
  matters — for very simple, dialect-agnostic queries, H2's speed
  advantage may still be an acceptable trade-off, but any native/complex
  SQL warrants Testcontainers.
- Q7: Audit which tests actually need full context — many are likely
  slice-test-sufficient or don't need Spring at all; convert unnecessary
  `@SpringBootTest` usages down to `@WebMvcTest`/`@DataJpaTest`/plain unit
  tests; keep only a small, deliberately curated set of true end-to-end
  `@SpringBootTest`s; consider parallelizing test execution and caching
  Spring context between tests that can legitimately share one
  (Spring's test context caching already helps here if configurations are
  consistent across test classes).
- Q8: The query likely used H2-tolerated syntax that isn't valid/behaves
  differently in PostgreSQL's SQL dialect — this could have been caught
  earlier by testing against a real PostgreSQL instance via
  Testcontainers instead of relying on H2's approximate compatibility.
- Q9: False — `@WebMvcTest` uses real Spring MVC infrastructure for the
  web layer specifically, but everything below the controller (service,
  repository) is mocked, so it provides confidence about the web layer's
  behavior only, not about how the full stack integrates together — that
  broader confidence still requires either targeted slice tests for other
  layers or a (sparing) full integration test.

</details>

---

## 8.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does `@WebMvcTest` load faster than `@SpringBootTest`, and what's the trade-off in what it can verify?
- **Coding Check:** Write a complete `@WebMvcTest` test class for a controller with create (201), get-by-id (200/404), and validation-failure (400) scenarios, following section 8.2's pattern.
- **Explanation Check:** Explain in English, to a teammate who says "our tests all pass, so we're fine," why H2-based repository tests alone don't guarantee production database correctness.
- **Real-World Check:** Your team's `@DataJpaTest` suite is fast but has twice caught bugs in local development that then still broke in production. Diagnose and propose the fix.
- **Senior Check:** When is it acceptable to skip Testcontainers and rely on H2 for a repository test?
- **Master Check:** Design the complete CI test strategy for a service with 200 unit tests, 30 slice tests, and 5 full integration tests — describe how you'd structure test execution stages (fast feedback first) and what would block a merge vs. what might run only on a nightly schedule.

<details><summary>Answers</summary>

- Real-World Check: This is the H2-vs-production-database fidelity gap
  from section 8.3 — migrate the repository test suite (or at least the
  tests exercising non-trivial/native queries) to Testcontainers with a
  real PostgreSQL instance, accepting slightly slower test execution in
  exchange for catching exactly this class of bug before production.
- Senior Check: For simple, standard JPQL queries using widely-portable
  SQL constructs with no database-specific features or syntax involved,
  where the risk of dialect-specific behavior differences is genuinely
  low — native queries or anything using database-specific functions
  should always prefer Testcontainers instead.
- Master Check: Fast feedback stage (runs on every commit/PR, blocks
  merge): the 200 unit tests plus the 30 slice tests — fast, no Docker
  needed for most of them, gives quick signal. Slower stage (runs on
  merge to main, or on-demand): the 5 full integration tests, potentially
  Testcontainers-backed, since these are slower but still valuable enough
  to run before deployment. Optionally, a nightly/scheduled run for any
  even-slower, broader end-to-end or performance tests that don't need
  to block every single merge but should still run regularly to catch
  regressions.

</details>

---

## 8.8 CHEAT SHEET

| Test type | Annotation | What's loaded | Speed |
|---|---|---|---|
| Unit test | None (plain JUnit + Mockito) | Nothing Spring-related | Fastest |
| Web slice | `@WebMvcTest` | Controller + MVC infra, service mocked | Fast |
| Data slice | `@DataJpaTest` | JPA layer + embedded DB (or Testcontainers) | Moderate |
| Full integration | `@SpringBootTest` | Entire application context | Slowest — use sparingly |
| H2 vs real DB | H2 ≠ production dialect behavior | Testcontainers for genuine fidelity |
| Mocking beans in slice tests | `@MockitoBean` | Scoped precisely to what the slice needs |

---

## BOOK 4 — CHAPTER 8 COMPLETE

*(All 8 chapters of Book 4's chapter content are now complete. See
`final-assessment-mock-interview-project.md` in this directory for the
cumulative Final Assessment, the Spring Boot Mock Interview round, and
the Book 4 capstone Project Assignment.)*
