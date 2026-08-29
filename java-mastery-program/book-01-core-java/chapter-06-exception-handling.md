# CHAPTER 6 — EXCEPTION HANDLING DONE RIGHT

---

## 6.1 CONCEPT: The Exception Hierarchy — Checked vs Unchecked

### TELUGU EXPLANATION

Java exception hierarchy `Throwable` నుండి మొదలవుతుంది, రెండు major branches:

- **`Error`:** JVM-level, తీవ్రమైన సమస్యలు (`OutOfMemoryError`,
  `StackOverflowError` — Chapter 1 చూడండి). వీటిని **catch చేసి "handle"
  చేయాలని ప్రయత్నించకూడదు** — root cause fix చేయాలి.
- **`Exception`:**
  - **Checked exceptions** (`IOException`, `SQLException` వంటివి) —
    `RuntimeException` extend చేయనివి. Compiler వీటిని **తప్పకుండా handle
    చేయాలి లేదా declare (`throws`) చేయాలి** అని force చేస్తుంది. వీటిని
    **recoverable, expected** conditions కోసం design చేశారు — ఉదా: file
    దొరకకపోవడం, network timeout — వీటిని caller reasonably గా anticipate
    చేసి handle చేయగలరు.
  - **Unchecked exceptions (`RuntimeException` subclasses)** (`NullPointerException`,
    `IllegalArgumentException`, `IllegalStateException`) — compiler force
    చేయదు. ఇవి usually **programming bugs** ని సూచిస్తాయి — caller వీటిని
    "handle" చేయడం సాధారణంగా సరైనది కాదు, ఎందుకంటే bug ని catch చేసి కప్పిపుచ్చడం
    అవుతుంది; బదులుగా bug ని fix చేయాలి.

**Senior-level debate:** చెక్డ్ exceptions గురించి industry లో చాలా వాదన ఉంది.
Modern Java APIs (Streams, `Optional`, most new libraries) చెక్డ్ exceptions
ని avoid చేస్తాయి, ఎందుకంటే అవి functional-style composition ని awkward
చేస్తాయి (`Stream.map()` లో checked exception throw చేసే lambda pass
చేయలేరు direct గా). చాలా modern Spring/enterprise codebases runtime
exceptions (unchecked) వాడతారు, checked exceptions ని sparingly వాడతారు.

### INDUSTRY TERMINOLOGY

`Throwable`, `Error`, `Exception`, `checked exception`, `unchecked exception`,
`RuntimeException`, `try-with-resources`, `finally`, `exception chaining`,
`stack trace`, `custom exception`.

### ENGLISH INTERVIEW ANSWER

"I treat checked exceptions as representing conditions a well-behaved caller
can reasonably anticipate and recover from — like a file not existing or a
network call timing out — where the compiler forcing explicit handling adds
real value. I treat unchecked exceptions as signaling programmer errors or
truly exceptional, non-recoverable-at-this-layer conditions — a null where
one shouldn't be, an illegal argument, a broken invariant. In practice, for
new service code, I lean toward unchecked exceptions for most business-logic
failures too, wrapping them in a small hierarchy of domain-specific runtime
exceptions, and let a centralized handler — like Spring's
`@ControllerAdvice` — translate them into appropriate HTTP responses. This
keeps method signatures clean and avoids the common anti-pattern of
`throws Exception` scattered everywhere, which defeats the whole purpose of
checked exceptions by making every caller declare-and-ignore rather than
meaningfully handle them."

### SIMPLE EXPLANATION

Checked = compiler-enforced, for expected/recoverable conditions. Unchecked
= bugs/broken invariants, don't force handling, fix the cause instead.
`Error` = JVM-level, don't try to "handle."

### REAL-TIME EXAMPLE

ఒక REST API లో, `CustomerNotFoundException` (unchecked, extends
`RuntimeException`) throw చేస్తే, Spring's `@ControllerAdvice`
(`@ExceptionHandler(CustomerNotFoundException.class)`) దాన్ని catch చేసి
HTTP 404 గా convert చేస్తుంది — service layer లో ప్రతి caller దీన్ని
manually catch చేయాల్సిన అవసరం లేదు, centralized గా handle అవుతుంది.

---

## 6.2 CONCEPT: try-with-resources and Proper Resource Management

### TELUGU EXPLANATION

Files, DB connections, network sockets లాంటివి **`AutoCloseable`**
resources — వీటిని వాడిన తర్వాత తప్పకుండా `close()` చేయాలి, లేకపోతే **resource
leak** (production లో "connection pool exhausted" లాంటి incidents కి direct
కారణం). పాత style:

```java
// ❌ Old style — error-prone, verbose
FileInputStream fis = null;
try {
    fis = new FileInputStream("data.txt");
    // use fis
} finally {
    if (fis != null) {
        try { fis.close(); } catch (IOException e) { /* often swallowed, a bug itself */ }
    }
}
```

`try-with-resources` (Java 7+) దీన్ని safe గా, concise గా చేస్తుంది:

```java
// ✅ try-with-resources — resource automatically closed, even on exception
try (FileInputStream fis = new FileInputStream("data.txt")) {
    // use fis
} catch (IOException e) {
    // handle
}
// fis.close() ఇక్కడ automatic గా call అవుతుంది, try block exit అయ్యేటప్పుడు
```

Multiple resources ని కూడా declare చేయవచ్చు (`try (Res1 a = ...; Res2 b = ...)`)
— ఇవి **reverse order** లో close అవుతాయి (last opened, first closed).

### ENGLISH INTERVIEW ANSWER

"Any resource implementing `AutoCloseable` — streams, JDBC connections,
locks — should be managed with try-with-resources, not manual
try/finally. It guarantees `close()` is called even if an exception is
thrown mid-block, and critically, if both the try block and the implicit
close throw exceptions, the original exception is preserved as the primary
one with the close exception attached as a suppressed exception, which the
old manual pattern typically lost entirely. In production, unclosed
resources are a classic root cause of connection pool exhaustion — I've
debugged incidents where a JDBC `Connection` wasn't closed on an early-return
path in a manual try/finally, slowly leaking connections until the pool was
exhausted and every new request started timing out."

---

## 6.3 CONCEPT: Designing a Custom Exception Hierarchy

### TELUGU EXPLANATION

Enterprise applications లో, generic `RuntimeException` throw చేయడం కంటే,
ఒక **meaningful exception hierarchy** design చేయడం మంచిది — ఇది calling
code కి specific handling చేయడానికి వీలు కల్పిస్తుంది, మరియు error responses
ని (HTTP status codes వంటివి) map చేయడం సులభం చేస్తుంది.

```java
// Base for all business/domain exceptions in this service
public abstract class DomainException extends RuntimeException {
    protected DomainException(String message) { super(message); }
    protected DomainException(String message, Throwable cause) { super(message, cause); }
}

public class CustomerNotFoundException extends DomainException {
    public CustomerNotFoundException(String customerId) {
        super("Customer not found: " + customerId);
    }
}

public class InsufficientBalanceException extends DomainException {
    private final BigDecimal shortfall;
    public InsufficientBalanceException(BigDecimal shortfall) {
        super("Insufficient balance, short by: " + shortfall);
        this.shortfall = shortfall;
    }
    public BigDecimal getShortfall() { return shortfall; }
}
```

**Exception chaining:** ఒక lower-level exception ని catch చేసి, higher-level,
domain-meaningful exception గా wrap చేసేటప్పుడు, **original cause ని
తప్పకుండా preserve చేయాలి** (`new DomainException(msg, originalException)`)
— లేకపోతే production లో debugging చేసేటప్పుడు actual root cause (e.g., SQL
error) కనిపించదు, ఇది చాలా common, ఖరీదైన mistake.

```java
try {
    jdbcTemplate.update(...);
} catch (DataAccessException e) {
    // ❌ Bad — swallows the real cause
    throw new RuntimeException("Failed to save customer");

    // ✅ Good — preserves the original stack trace for debugging
    throw new CustomerPersistenceException("Failed to save customer", e);
}
```

### ENGLISH INTERVIEW ANSWER

"I design a small, purposeful exception hierarchy per service — usually a
base `DomainException` (or similar) that carries information useful for
mapping to API responses, with specific subtypes for distinct failure modes
that callers or a global handler might treat differently. The one rule I
never break is exception chaining: whenever I catch a lower-level exception
to translate it into a more meaningful domain exception, I always pass the
original as the cause. I've seen too many production incidents where someone
threw a new generic exception without chaining the original `SQLException`
or `IOException`, and the resulting stack trace in the logs pointed only to
the wrapping code — completely hiding the actual root cause and turning a
five-minute log read into an hour of guessing."

---

## 6.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Catching an exception | `catch (Exception e) {}` (swallow it) | Catch the specific exception, log with context, decide: recover, rethrow wrapped, or propagate |
| Method that might fail | `throws Exception` on everything | Specific checked exception only when truly recoverable; otherwise unchecked with a clear message |
| Validating input | Let a `NullPointerException` happen naturally | Fail fast with `Objects.requireNonNull` / `IllegalArgumentException` with a clear message, at the boundary |
| Retrying a failed operation | Retry every exception blindly | Retry only exceptions known to be transient (timeout, 503) — never retry validation errors or business rule violations |

---

## 6.5 COMMON MISTAKES

1. **Swallowing exceptions** (`catch (Exception e) {}` with no logging) —
   the single worst exception anti-pattern; it hides bugs that surface as
   mysterious symptoms far downstream.
2. Catching `Exception` (or worse, `Throwable`) generically instead of
   specific types, hiding programming errors alongside genuine recoverable
   conditions.
3. Losing the original cause when wrapping exceptions (no chaining).
4. Using exceptions for normal control flow (e.g., using an exception to
   break out of a loop instead of a `break`) — exceptions are expensive
   (stack trace capture) and hurt readability.
5. Returning `null` instead of throwing/using `Optional` for "not found"
   cases, pushing a `NullPointerException` risk onto every caller.

---

## 6.6 INTERVIEW QUESTION BANK — CHAPTER 6

**Basic:** 1. Difference between checked and unchecked exceptions? 2. What
does `finally` guarantee?

**Intermediate:** 3. Why is try-with-resources preferred over manual
try/finally? 4. What is exception chaining and why does it matter?

**Senior:** 5. Design a custom exception hierarchy for an e-commerce
checkout flow (out of stock, payment declined, invalid coupon). How would a
global exception handler map these to HTTP responses? 6. Why do many modern
Java APIs (Streams, functional interfaces) avoid checked exceptions?

**Architect:** 7. In a microservices architecture, how do you decide which
failures should be retried, which should trigger a circuit breaker, and
which should fail fast and propagate to the caller immediately?

**Scenario:** 8. A production log shows thousands of generic "Something
went wrong" messages with no stack trace, and the team can't diagnose an
intermittent failure. What exception-handling anti-pattern is likely at
fault, and how do you fix the codebase going forward?

**Trick:** 9. "You should always catch exceptions as close as possible to
where they occur." True or false — explain the nuance.

<details><summary>Key answers</summary>

- Q5: `OutOfStockException`, `PaymentDeclinedException`,
  `InvalidCouponException` extending a common `CheckoutException`; a
  `@ControllerAdvice` maps `OutOfStockException`→409 Conflict,
  `PaymentDeclinedException`→402 Payment Required (or a custom code),
  `InvalidCouponException`→400 Bad Request — each with a clear,
  client-actionable message.
- Q9: False as a blanket rule — catch where you can *meaningfully act* on
  the exception (retry, fall back, translate to a domain exception), not
  merely where it's syntactically closest; catching too early and
  swallowing/logging-and-continuing often just delays and obscures the real
  failure. The right layer is "where you have enough context to decide what
  to do," which is often higher up the call stack, not the lowest point.

</details>

---

## 6.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Name two guarantees try-with-resources gives you over manual try/finally.
- **Coding Check:** Refactor a method with `catch (Exception e) { e.printStackTrace(); }` into proper, chained, logged exception handling.
- **Explanation Check:** Explain in English why swallowing exceptions is worse than letting them propagate uncaught.
- **Real-World Check:** A batch job processing 10,000 records throws on record #4,521 and the whole batch fails, losing all successful work. Redesign the error handling.
- **Senior Check:** When is it correct to catch `Exception` broadly (not a specific subtype)?
- **Master Check:** Design the exception-to-HTTP-status mapping strategy for a public API used by third parties, considering that leaking internal exception messages/stack traces to external clients is a security risk.

<details><summary>Answers</summary>

- Real-World Check: Process records individually with per-record try/catch,
  collect failures into a report (record ID + reason) instead of aborting
  the whole batch, and continue processing remaining records — a common
  senior-level batch-processing pattern.
- Senior Check: At a top-level boundary (like a global exception handler or
  a `main` method) specifically to prevent an unexpected exception from
  crashing the process or leaking to a user ungracefully — but even then,
  log full details internally and re-throw/convert rather than silently
  swallowing.
- Master Check: Never return raw exception messages or stack traces to
  external clients; map internal exceptions to a small set of
  well-defined, sanitized error codes/messages, log full details
  (including stack trace) server-side with a correlation ID that IS
  returned to the client for support/debugging purposes.

</details>

---

## 6.8 CHEAT SHEET

| Rule | Why |
|---|---|
| Never swallow exceptions silently | Hides bugs, makes production debugging nearly impossible |
| Always chain the original cause when wrapping | Preserves root cause for debugging |
| Use try-with-resources for anything `AutoCloseable` | Prevents resource leaks, preserves suppressed exceptions |
| Unchecked for bugs/broken invariants, checked sparingly for truly recoverable conditions | Matches modern Java/Spring conventions |
| Never expose raw exceptions to external API clients | Security — avoid leaking internals |

---

*(Continues to Chapter 7 — Java 8: Lambdas, Functional Interfaces, Streams, Optional.)*
