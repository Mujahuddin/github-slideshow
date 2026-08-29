# CHAPTER 8 — JAVA 11/17: `var`, RECORDS, SEALED CLASSES, TEXT BLOCKS, PATTERN MATCHING

---

## 8.1 CONCEPT: `var` — Local Variable Type Inference (Java 10+)

### TELUGU EXPLANATION

`var` అనేది **type inference** — compiler right-hand side నుండి type ని
infer చేస్తుంది. ఇది **dynamic typing కాదు** — variable static గా typed
గానే ఉంటుంది, కేవలం మీరు type name ని రాయాల్సిన అవసరం లేదు:

```java
var customers = new ArrayList<Customer>(); // inferred as ArrayList<Customer>
var name = "Muzahid";                       // inferred as String
// customers = "wrong"; // ❌ compile error — customers is still ArrayList<Customer>
```

**ఎప్పుడు వాడాలి, ఎప్పుడు వాడకూడదు:** `var` readability improve చేస్తుంది
type name పొడవుగా, obvious గా ఉన్నప్పుడు (`var repo = new
CustomerJpaRepositoryImpl();`), కానీ readability **తగ్గిస్తుంది** type
right-hand side నుండి స్పష్టంగా తెలియనప్పుడు (`var result =
service.process(data);` — result ఏం type అని method చూడందే తెలియదు).
**Senior rule:** `var` వాడేటప్పుడు variable name + context నుండి type
స్పష్టంగా తెలియాలి; లేకపోతే explicit type రాయడమే మంచిది.

### ENGLISH INTERVIEW ANSWER

"`var` is compile-time local type inference, not dynamic typing — the
variable's type is fixed at compile time based on the initializer, it's just
not spelled out in source. I use it when the type is already obvious from
the right-hand side — `var list = new ArrayList<Customer>();` — where
repeating the type adds no information. I avoid it when the inferred type
isn't clear from context, such as `var result = someMethod();`, because that
actively hurts readability and code review, which defeats the point of a
statically typed language's self-documentation. It's also restricted to
local variables — you can't use it for fields, method parameters, or return
types."

---

## 8.2 CONCEPT: Records (Java 16+) — Concise Immutable Data Carriers

### TELUGU EXPLANATION

చాలా Java classes కేవలం **data ని carry చేయడానికి** మాత్రమే ఉంటాయి —
fields, constructor, getters, `equals`/`hashCode`/`toString`. Records
కి ముందు, ఇవన్నీ manually రాయాలి (లేదా Lombok లాంటి library వాడాలి):

```java
// Records కి ముందు — verbose boilerplate
public final class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int getX() { return x; }
    public int getY() { return y; }
    @Override public boolean equals(Object o) { /* ... */ }
    @Override public int hashCode() { /* ... */ }
    @Override public String toString() { /* ... */ }
}

// ✅ Record — compiler generates all of the above automatically
public record Point(int x, int y) { }
```

Record automatic గా ఇస్తుంది: `private final` fields, canonical
constructor, accessor methods (`x()`, `y()` — **not** `getX()`), correct
`equals()`/`hashCode()` (అన్ని fields based on), మరియు meaningful
`toString()`. Records **implicitly `final`** మరియు **implicitly
immutable** — ఇది Chapter 2/3 లో మనం నేర్చుకున్న "favor immutability"
principle కి language-level support.

**Compact constructors** — validation logic add చేయడానికి:

```java
public record TemperatureRange(int min, int max) {
    public TemperatureRange {  // compact constructor — no parameter list repeated
        if (min > max) {
            throw new IllegalArgumentException("min must be <= max");
        }
    }
}
```

**Records ఎప్పుడు వాడకూడదు:** Records ఒక **inheritance hierarchy** లో
extend చేయలేరు (implicitly final), మరియు అవి **pure immutable data**
కోసమే design చేయబడ్డాయి — mutable state అవసరమైతే, లేదా JPA `@Entity` లాంటి
mutable-by-framework-design classes కి records సరిపోవు (Hibernate కి proxy
generation/lazy loading కోసం mutable, no-arg-constructor classes కావాలి —
Book 7 లో వివరంగా చూద్దాం).

### ENGLISH INTERVIEW ANSWER

"Records give me a one-line, compiler-generated immutable data carrier —
fields, canonical constructor, accessors, `equals`, `hashCode`, and
`toString`, all correctly implemented based on the component list, which
eliminates a whole category of boilerplate and boilerplate bugs, like
forgetting to update `equals()` after adding a field. I use them for DTOs,
value objects, and API request/response shapes — anywhere I want an
immutable, self-contained bundle of data. I don't use them for JPA entities,
since Hibernate needs a mutable, no-arg-constructible class for proxying and
dirty-checking, and I don't use them when I need a class hierarchy, since
records are implicitly final and can't extend another class (though they
can implement interfaces)."

### REAL-TIME EXAMPLE

REST API DTOs లో records ideal:

```java
public record CreateOrderRequest(String customerId, List<String> productIds, String couponCode) { }
public record OrderResponse(String orderId, BigDecimal totalAmount, String status) { }
```

ఇవి Jackson తో directly (de)serialize అవుతాయి, boilerplate లేకుండా.

---

## 8.3 CONCEPT: Sealed Classes (Java 17) — Restricted Hierarchies

### TELUGU EXPLANATION

Normal గా, ఏ class అయినా ఎవరైనా extend చేయవచ్చు (unless `final`). Sometimes
మీకు **"ఈ hierarchy లో ఇవే subtypes ఉండాలి, ఇంకేవీ కాదు"** అని control
కావాలి — ఉదాహరణకి, ఒక payment result "Success, Failure, Pending" — ఈ మూడే
ఉండాలి, ఎవరూ కొత్త, unexpected subtype add చేయకూడదు. **Sealed classes**
దీన్ని enable చేస్తాయి:

```java
public sealed interface PaymentResult
        permits PaymentSuccess, PaymentFailure, PaymentPending {
}

public record PaymentSuccess(String transactionId) implements PaymentResult { }
public record PaymentFailure(String reason) implements PaymentResult { }
public record PaymentPending(String referenceId) implements PaymentResult { }
```

దీని అతిపెద్ద benefit — **exhaustive pattern matching** (Java 21 `switch`
expressions తో, Java 17 లో `instanceof` pattern matching తో preview):
compiler మీకు అన్ని subtypes handle చేశారా లేదా అని **compile-time** లో
చెప్పగలదు — `default` case అవసరం లేకుండా.

```java
// Java 21 style exhaustive switch (conceptually applicable, shown for completeness)
String message = switch (result) {
    case PaymentSuccess s -> "Paid: " + s.transactionId();
    case PaymentFailure f -> "Failed: " + f.reason();
    case PaymentPending p -> "Pending: " + p.referenceId();
    // no default needed — compiler knows these are ALL permitted subtypes
};
```

### ENGLISH INTERVIEW ANSWER

"Sealed classes let me declare a closed, known set of subtypes for a
hierarchy — the compiler enforces that only the permitted types can
implement or extend it. This is powerful combined with records and pattern
matching: I can model something like a payment result as a sealed hierarchy
of `PaymentSuccess`/`PaymentFailure`/`PaymentPending` records, and then get
compile-time exhaustiveness checking on a switch over that type — if someone
later adds a new permitted subtype and forgets to handle it in a switch
expression, the code won't compile, instead of silently falling through a
`default` branch at runtime. This is essentially bringing algebraic data
type / sum type modeling, common in functional languages, into Java."

---

## 8.4 CONCEPT: Text Blocks (Java 15+)

### TELUGU EXPLANATION

Multi-line strings (SQL queries, JSON payloads) రాయడానికి, పాత style లో
`\n` + string concatenation వాడాల్సి వచ్చేది — messy. **Text blocks**
(`"""`) దీన్ని clean చేస్తాయి:

```java
String query = """
        SELECT id, name, email
        FROM customers
        WHERE status = 'ACTIVE'
        ORDER BY created_at DESC
        """;
```

Indentation ని compiler automatic గా strip చేస్తుంది (common leading
whitespace based on the closing `"""` position).

### ENGLISH INTERVIEW ANSWER

"Text blocks remove the need for manual `\n` and string concatenation when
embedding multi-line content like SQL, JSON, or HTML directly in Java source
— common in test fixtures and native queries. The compiler strips common
leading whitespace automatically, so the block can be indented naturally
with the surrounding code without extra characters leaking into the string
content."

---

## 8.5 CODE: PUTTING IT TOGETHER — A MODERN JAVA DOMAIN MODEL

```java
public sealed interface OrderStatus permits Placed, Shipped, Delivered, Cancelled { }
public record Placed(Instant at) implements OrderStatus { }
public record Shipped(Instant at, String trackingNumber) implements OrderStatus { }
public record Delivered(Instant at) implements OrderStatus { }
public record Cancelled(Instant at, String reason) implements OrderStatus { }

public record Order(String id, String customerId, BigDecimal total, OrderStatus status) {
    public Order {
        Objects.requireNonNull(id);
        Objects.requireNonNull(customerId);
        if (total.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("total cannot be negative");
        }
    }

    public String describeStatus() {
        return switch (status) {
            case Placed p -> "Placed at " + p.at();
            case Shipped s -> "Shipped (tracking: " + s.trackingNumber() + ")";
            case Delivered d -> "Delivered at " + d.at();
            case Cancelled c -> "Cancelled: " + c.reason();
        };
    }
}
```

**Design notes:** This models the classic "order status" domain problem —
previously often done with an error-prone `enum` + nullable extra fields
(e.g., `trackingNumber` only meaningful for `SHIPPED`, `reason` only for
`CANCELLED`, both `null` otherwise). The sealed-interface-of-records
approach makes illegal states unrepresentable: you cannot construct a
`Shipped` without a tracking number, and the exhaustive `switch` guarantees
every status is handled — a direct, practical upgrade over the old
enum-plus-nullable-fields anti-pattern.

**Interviewer follow-up:** "How would you have modeled this before Java
17?" — Typically an `enum OrderStatus { PLACED, SHIPPED, DELIVERED,
CANCELLED }` plus nullable `trackingNumber`/`cancellationReason` fields on
`Order` itself, which allows invalid combinations to compile (e.g., a
`PLACED` order with a non-null `trackingNumber`) — precisely the class of
bug sealed types + records eliminate at compile time.

---

## 8.6 COMMON MISTAKES

1. Overusing `var` where the inferred type isn't obvious, hurting readability.
2. Trying to use a Record for a JPA entity or anything needing mutability/inheritance.
3. Forgetting that sealed types require every permitted subtype to be
   `final`, `sealed`, or `non-sealed` — an unqualified subtype won't compile.
4. Using nullable fields to represent state-dependent data (old
   enum+nullable-fields anti-pattern) instead of sealed types + records
   when targeting Java 17+.

---

## 8.7 INTERVIEW QUESTION BANK — CHAPTER 8

**Basic:** 1. Is `var` dynamic typing? 2. What does a Record generate automatically?

**Intermediate:** 3. Why can't Records be extended? 4. What problem do sealed classes solve that a plain `abstract class` doesn't?

**Senior:** 5. Redesign an enum-with-nullable-fields anti-pattern you've seen (or can imagine) using sealed interfaces + records.

**Architect:** 6. Your team is migrating a Java 8 codebase to Java 17. Where would you introduce Records and sealed classes first for maximum benefit with minimum risk?

**Scenario:** 7. A reviewer flags `var response = restTemplate.getForObject(url, Object.class);` in a PR. What's the issue?

**Trick:** 8. "Records are just syntactic sugar with no behavioral difference from a normal class." True or false?

<details><summary>Key answers</summary>

- Q7: `var` here infers `Object`, hiding the actual useful type and making
  every subsequent line reliant on unsafe casting or `instanceof` checks —
  exactly the case where `var` obscures rather than clarifies; the fix is
  either an explicit type argument (`getForObject(url, MyDto.class)`) or an
  explicit variable type.
- Q8: False — Records also enforce immutability (fields are implicitly
  `final`), generate correct `equals`/`hashCode`/`toString` based on all
  components automatically, and are implicitly `final` themselves (no
  subclassing) — these are real semantic guarantees, not just shorthand syntax.

</details>

---

## 8.8 MASTERY CHECKPOINTS

- **Knowledge Check:** What three things does a compact constructor let you do that a normal constructor also does, and why prefer it for records?
- **Coding Check:** Model a `Shape` sealed hierarchy (`Circle`, `Rectangle`, `Triangle`) as records with an exhaustive `area()` computation via switch.
- **Explanation Check:** Explain to a Java 8 developer, in English, why sealed types + records + exhaustive switch is safer than enum + nullable fields.
- **Real-World Check:** Your team has an `enum NotificationType { EMAIL, SMS, PUSH }` with three nullable fields (`emailAddress`, `phoneNumber`, `deviceToken`) on the containing class, only one ever populated. Migrate this to Java 17 idioms.
- **Senior Check:** Would you introduce records into a codebase still targeting Java 8 (via a backport or Lombok)? What's the trade-off?
- **Master Check:** Explain why `permits` combined with exhaustive `switch` is a compile-time substitute for the Visitor design pattern in many cases.

<details><summary>Answers</summary>

- Real-World Check: `sealed interface Notification permits EmailNotification, SmsNotification, PushNotification`, each a record carrying only its relevant field — eliminates the always-two-null-fields problem entirely.
- Senior Check: On Java 8, Lombok's `@Value` gives similar boilerplate reduction but without compiler-enforced immutability guarantees or pattern-matching integration — reasonable as a stopgap, but migrating to real records once on Java 17+ is still worth doing for the stronger guarantees.
- Master Check: The Visitor pattern exists specifically to add new operations over a closed set of types without modifying each type — sealed types + exhaustive switch achieve the same goal (compiler-verified handling of every case) with far less ceremony (no `accept`/`visit` method scaffolding), at the cost of needing to modify the switch (not the types) when a new operation is added — similar trade-off, more concise mechanism.

</details>

---

## 8.9 CHEAT SHEET

| Feature | Version | One-line use |
|---|---|---|
| `var` | 10+ | Local type inference when RHS type is obvious |
| Records | 16+ | Immutable data carriers, auto equals/hashCode/toString |
| Sealed classes | 17 | Closed hierarchies, exhaustive pattern matching |
| Text blocks | 15+ | Clean multi-line strings (SQL, JSON) |

---

*(Continues to Chapter 9 — Multithreading and Concurrency Fundamentals.)*
