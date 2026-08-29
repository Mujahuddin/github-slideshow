# CHAPTER 3 — BUILDING REST CONTROLLERS

---

## 3.1 CONCEPT: `@RestController` = `@Controller` + `@ResponseBody`

### TELUGU EXPLANATION

```java
@RestController // = @Controller + @ResponseBody (ప్రతి method మీద)
@RequestMapping("/api/orders")
class OrderController {

    private final OrderService orderService; // constructor injection — Book 3 Chapter 1 సూత్రం

    OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/{orderId}")
    OrderResponse getOrder(@PathVariable String orderId) {
        return orderService.findById(orderId); // Jackson ఇది automatic గా JSON గా serialize చేస్తుంది
    }
}
```

**`@Controller`** (view-rendering, Spring MVC యొక్క original purpose —
Thymeleaf/JSP వంటి server-rendered views కి) vs **`@RestController`**
(REST APIs కి — return value నేరుగా HTTP response body గా, JSON/XML
గా serialize అవుతుంది, `@ResponseBody` వల్ల). ఆధునిక backend APIs
దాదాపు ఎప్పుడూ `@RestController` వాడతాయి.

**Key annotations:**
- `@GetMapping`/`@PostMapping`/`@PutMapping`/`@DeleteMapping`/`@PatchMapping`
  — `@RequestMapping(method=...)` కి shortcuts.
- `@PathVariable` — URL path లో భాగం (`/orders/{orderId}`).
- `@RequestParam` — query string parameter (`?status=ACTIVE`).
- `@RequestBody` — HTTP request body ని (సాధారణంగా JSON) Java object గా deserialize.

### ENGLISH INTERVIEW ANSWER

"`@RestController` is `@Controller` plus `@ResponseBody` applied to every
handler method — instead of resolving a view name to render server-side,
the return value is serialized directly into the HTTP response body,
typically as JSON via Jackson. This is the standard choice for building
REST APIs; `@Controller` alone is for server-rendered view applications,
which is a different (and now less common for pure backend work) use
case. I keep controllers thin — constructor-injected service dependencies,
minimal logic, just translating HTTP concerns (path variables, query
params, request bodies) into calls on the service layer, which is where
actual business logic belongs."

---

## 3.2 CONCEPT: DTOs vs Entities — Why You Never Return an Entity Directly

### TELUGU EXPLANATION

**ఇది ఈ chapter యొక్క అత్యంత ముఖ్యమైన architectural rule, తరచుగా
junior developers ఉల్లంఘించేది:** JPA `@Entity` (Book 7 లో వివరంగా)
ని **నేరుగా** REST response గా return చేయకూడదు. బదులుగా, ఒక **DTO
(Data Transfer Object)** వాడాలి.

**ఎందుకు — నాలుగు నిజమైన కారణాలు:**

1. **Lazy loading exceptions:** `@Entity` లో lazy-loaded relationships
   ఉంటే, Jackson వాటిని serialize చేయడానికి ప్రయత్నించినప్పుడు
   (controller method return అయ్యాక, Hibernate session ఇప్పటికే
   closed అయ్యి ఉండొచ్చు), `LazyInitializationException` వస్తుంది —
   ఒక classic, confusing production bug (Book 7 లో వివరంగా).
2. **Over-exposure (security risk):** `Entity` లో internal fields
   (password hashes, internal flags, audit metadata) ఉండొచ్చు, అవి
   API consumers కి **ఎప్పుడూ కనిపించకూడదు** — Entity నేరుగా
   return చేస్తే, కొత్త field add చేసిన ప్రతిసారి, అది accidentally
   API లో leak అయ్యే ప్రమాదం ఉంటుంది.
3. **API-Database coupling:** Entity structure మారితే (database schema
   refactor), API response structure కూడా **automatically మారిపోతుంది**
   — ఇది API consumers ని break చేయవచ్చు. DTO ఈ రెండింటినీ **decouple**
   చేస్తుంది (Book 1 Chapter 2 DIP సూత్రం, ఇక్కడ API layer vs
   persistence layer మధ్య).
4. **Shape mismatch:** API కి అవసరమైన shape, DB entity shape తో ఎప్పుడూ
   ఒకేలా ఉండదు (ఉదా: API request కి `password` field అవసరం, కానీ
   response కి అది ఎప్పుడూ ఉండకూడదు — ఒకే class రెండు purposes కి
   సరిపోదు).

```java
// ❌ ఎప్పుడూ చేయకూడదు
@GetMapping("/{id}")
Customer getCustomer(@PathVariable Long id) { // Customer ఒక @Entity!
    return customerRepository.findById(id).orElseThrow();
}

// ✅ DTO వాడండి
record CustomerResponse(Long id, String name, String email) { } // password, internal fields ఏవీ లేవు

@GetMapping("/{id}")
CustomerResponse getCustomer(@PathVariable Long id) {
    Customer customer = customerRepository.findById(id).orElseThrow();
    return new CustomerResponse(customer.getId(), customer.getName(), customer.getEmail()); // explicit mapping
}
```

**Design note:** Book 1 Chapter 8 లో నేర్చుకున్న **Records** DTOs కి
ఖచ్చితంగా సరిపోతాయి — immutable, boilerplate-free, request/response
shapes కి ideal.

### ENGLISH INTERVIEW ANSWER

"I never return JPA entities directly from a controller, and this is one
of the first things I check in a code review. Four concrete reasons: lazy-
loaded associations can throw `LazyInitializationException` during
serialization if the Hibernate session has already closed by the time
Jackson processes the response; entities often carry internal fields that
should never be exposed externally, and returning them directly makes
accidental leakage a when-not-if problem as the entity evolves; it
tightly couples your public API shape to your database schema, so a
schema refactor becomes a breaking API change; and the shape needed for a
request often genuinely differs from the shape needed for a response —
one entity class can't cleanly serve both. I use dedicated DTOs — Records
are ideal for this, being immutable and boilerplate-free — with explicit
mapping between entity and DTO, which keeps the API contract stable and
intentional regardless of how the persistence layer evolves."

---

## 3.3 CONCEPT: `ResponseEntity` — Controlling Status Codes and Headers

### TELUGU EXPLANATION

Simple గా object return చేస్తే, Spring default గా **200 OK** పంపుతుంది.
నిజమైన REST APIs కి **status code మీద explicit control** అవసరం:

```java
@PostMapping
ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
    OrderResponse created = orderService.create(request);
    return ResponseEntity
            .status(HttpStatus.CREATED) // 201, 200 కాదు — "కొత్త resource create అయ్యింది"
            .header("Location", "/api/orders/" + created.id())
            .body(created);
}

@GetMapping("/{id}")
ResponseEntity<OrderResponse> getOrder(@PathVariable String id) {
    return orderService.findById(id)
            .map(ResponseEntity::ok)           // దొరికితే 200
            .orElseGet(() -> ResponseEntity.notFound().build()); // దొరకకపోతే 404
}
```

**Senior rule:** REST API design లో **correct HTTP status codes వాడటం**
అనేది cosmetic కాదు — ఇది API consumers కి (మరియు API gateways,
monitoring tools కి) meaningful సమాచారం ఇస్తుంది. `200` అన్నింటికీ
వాడటం (ఒక error అయినా) **anti-pattern**.

### ENGLISH INTERVIEW ANSWER

"`ResponseEntity` gives explicit control over status code, headers, and
body together, which matters because status codes are part of the API
contract, not cosmetic detail. A `POST` that creates a resource should
return `201 Created` with a `Location` header pointing to the new
resource, not a blanket `200 OK` — API gateways, monitoring, and clients
all reasonably expect this distinction. I chain `Optional`-returning
service methods (Book 1 Chapter 7) directly into `ResponseEntity` via
`.map(ResponseEntity::ok).orElseGet(() -> ResponseEntity.notFound().build())`,
which reads cleanly and correctly represents 'found → 200' vs 'not found →
404' without an explicit if/else."

---

## 3.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Returning data from a controller | Returns the `@Entity` directly | Maps to a dedicated DTO/Record |
| Successful resource creation | Returns `200 OK` for everything | Returns `201 Created` with a `Location` header |
| "Not found" case | Lets a `NoSuchElementException` bubble up as a 500 | Returns `404` explicitly, or via a centralized handler (Chapter 4) |
| API shape vs DB shape | Assumes they should always match 1:1 | Treats them as independently evolvable, connected by explicit mapping |

---

## 3.5 COMMON MISTAKES

1. Returning `@Entity` objects directly from controllers.
2. Using `200 OK` for every response regardless of what actually happened.
3. Putting business logic directly in the controller instead of
   delegating to a service layer.
4. Forgetting that request DTOs and response DTOs often need to be
   *different* classes (e.g., a create-request DTO with a password field
   that a response DTO must never include).
5. Not setting the `Location` header on `201 Created` responses.

---

## 3.6 INTERVIEW QUESTION BANK — CHAPTER 3

**Basic:** 1. What does `@RestController` add over `@Controller`? 2. What
is a DTO and why use one instead of an entity?

**Intermediate:** 3. Explain `LazyInitializationException` as a reason to
avoid returning entities directly (preview of Book 7). 4. When would you
use `@RequestParam` vs `@PathVariable`?

**Senior:** 5. Design the DTO structure for an endpoint that creates a
user (needs a password) and returns the created user (must never include
the password) — show both DTOs. 6. Why is decoupling the API shape from
the database schema an application of the Dependency Inversion Principle?

**Architect:** 7. Your API has been stable for two years, but the
database schema needs a significant refactor (splitting one table into
two). How does the DTO-based design from this chapter make this refactor
safe for existing API consumers?

**Scenario:** 8. A teammate's endpoint occasionally throws
`LazyInitializationException` in production but never in local testing.
Explain why this environment difference might occur and how a DTO-based
approach would have prevented it entirely.

**Trick:** 9. "Using a DTO always means writing more code than just
returning the entity, so it's a pure trade-off of safety for effort."
True or false?

<details><summary>Key answers</summary>

- Q5: `record CreateUserRequest(String username, String email, String
  password) {}` for the request (password required to create the
  account), and `record UserResponse(Long id, String username, String
  email) {}` for the response (no password field at all — not just
  hidden, structurally absent) — this is the "shape mismatch" reason from
  section 3.2 made concrete.
- Q6: The API layer (high-level, consumer-facing) no longer depends
  directly on the persistence layer's concrete structure (low-level,
  internal, subject to change) — both now depend on an explicit,
  independently-defined DTO "contract," which is exactly DIP's shape:
  depend on abstractions/contracts, not on a concrete, volatile implementation detail.
- Q7: Since the API's DTOs are independent of the entity structure, the
  refactor only requires updating the *mapping* code (entity-to-DTO
  translation) to pull data from the new two-table structure — the DTO
  shape, and therefore the public API contract, doesn't need to change at
  all, so existing consumers see zero difference.
- Q8: Local testing might use eager fetching, a single-threaded/simple
  transaction scope, or simply never happen to hit the code path where the
  session closes before serialization — in production, under real load
  and with lazy-loaded associations, the timing/session lifecycle
  difference surfaces the exception. Returning DTOs mapped explicitly
  within the transactional/session boundary (where lazy associations can
  still be safely accessed) avoids ever needing serialization to trigger
  lazy loading at all.
- Q9: False, or at least incomplete — while there is more code
  (explicit mapping), the "safety" gained isn't just abstract risk
  reduction; it directly prevents concrete bug classes (lazy-loading
  exceptions, accidental field leakage) and enables real architectural
  flexibility (independent schema evolution, per Q7) that would otherwise
  require much larger, riskier changes later — the extra code is a small,
  known cost paid to avoid larger, unknown future costs.

</details>

---

## 3.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Name the four concrete reasons (not just "best practice") for never returning a JPA entity directly from a controller.
- **Coding Check:** Build a small `ProductController` with `create` (201 + Location header), `getById` (200/404 via Optional mapping), and `list` (200) endpoints, using Record DTOs throughout.
- **Explanation Check:** Explain in English why "200 OK for everything" is a real API design mistake, not just a stylistic preference.
- **Real-World Check:** Your team's mobile app team asks for a `PATCH` endpoint that only updates a subset of fields on a resource. Design the request DTO and explain how it differs from your `PUT`/create DTOs.
- **Senior Check:** When, if ever, might returning something very close to an entity structure be acceptable (hint: think about internal-only, trusted service-to-service APIs vs public APIs)?
- **Master Check:** Design the full DTO strategy for an order-management API supporting create, full update (PUT), partial update (PATCH), and read — specify each DTO's fields and justify any fields that intentionally differ between them.

<details><summary>Answers</summary>

- Real-World Check: A `PATCH` DTO typically uses nullable/`Optional`-
  wrapped fields for everything (e.g., `record UpdateProductRequest(String
  name, BigDecimal price)` where `null` means "don't change this field"),
  fundamentally different from a `PUT`/create DTO where all required
  fields are mandatory and non-null — conflating the two leads to
  ambiguity about whether a null field means "clear this value" or "leave
  it unchanged."
- Senior Check: For genuinely internal, trusted service-to-service
  communication where both sides are maintained by the same team and
  evolve together (not a public or even cross-team API contract), the
  strict DTO boundary can be relaxed somewhat — though even then, many
  teams still maintain the separation for the lazy-loading and
  future-flexibility reasons alone, independent of the security/exposure concern.
- Master Check: `CreateOrderRequest` (all required fields, e.g.
  customerId, items, shippingAddress); `UpdateOrderRequest`/PUT (same full
  shape, represents the complete new state); `PatchOrderRequest` (all
  optional/nullable fields, only non-null ones applied);
  `OrderResponse` (id, status, computed totals, timestamps — fields that
  only exist after creation and are never part of any request DTO) — the
  intentional differences (response-only computed/generated fields,
  request-only fields that shouldn't be echoed back) directly demonstrate
  the "shape mismatch" principle from section 3.2.

</details>

---

## 3.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| `@RestController` | `@Controller` + `@ResponseBody` — return values become the HTTP body |
| Never return | JPA `@Entity` objects directly — always map to a DTO |
| DTOs | Use Records; request and response DTOs are often different shapes |
| `201 Created` | For successful resource creation, with a `Location` header |
| `404 Not Found` | `Optional`-returning services map cleanly via `.map(ResponseEntity::ok).orElseGet(...)` |
| Controller responsibility | Thin — HTTP translation only, business logic belongs in the service layer |

---

*(Continues to Chapter 4 — Validation & Exception Handling.)*
