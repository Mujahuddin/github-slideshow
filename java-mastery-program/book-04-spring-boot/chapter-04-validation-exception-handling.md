# CHAPTER 4 — VALIDATION & EXCEPTION HANDLING

---

## 4.1 CONCEPT: Bean Validation — Failing Fast on Bad Input

### TELUGU EXPLANATION

**Bean Validation** (JSR 380, `jakarta.validation` annotations) DTO
fields మీద **declarative** గా constraints define చేయడానికి వీలు
కల్పిస్తుంది:

```java
record CreateOrderRequest(
    @NotBlank(message = "customerId is required") String customerId,
    @NotEmpty(message = "items cannot be empty") List<@Valid OrderItemRequest> items,
    @Email(message = "must be a valid email") String contactEmail,
    @Min(value = 1, message = "quantity must be at least 1") int quantity
) { }
```

```java
@PostMapping
ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
    // @Valid ఇక్కడ ఉంటే, Spring request ని validate చేస్తుంది,
    // constraints violate అయితే, controller method body execute కూడా అవ్వదు —
    // Spring ఒక MethodArgumentNotValidException throw చేస్తుంది
    return ResponseEntity.status(HttpStatus.CREATED).body(orderService.create(request));
}
```

**కీలక insight:** `@Valid` లేకపోతే, ఈ annotations **ఏమీ చేయవు** —
Spring వాటిని అమలు చేయాలంటే `@Valid` (controller parameter మీద) తప్పకుండా
ఉండాలి. ఇది Book 1 Chapter 6 "fail fast at the boundary" సూత్రానికి
direct application — invalid data business logic లోకి **ఎప్పుడూ
చేరుకోకూడదు**.

### ENGLISH INTERVIEW ANSWER

"Bean Validation lets me declare constraints directly on the DTO —
`@NotBlank`, `@Email`, `@Min`, nested `@Valid` for collections of nested
objects. The critical detail is that `@Valid` on the controller parameter
is what actually activates enforcement — the annotations on the DTO alone
do nothing without it. When validation fails, Spring throws a
`MethodArgumentNotValidException` before the controller method body ever
executes, which is exactly the 'fail fast at the boundary' principle from
Book 1's exception handling chapter — invalid input never has a chance to
reach business logic."

---

## 4.2 CONCEPT: Centralized Exception Handling with `@ControllerAdvice`

### TELUGU EXPLANATION

Every controller లో individually try/catch రాయడం బదులు (repetitive,
Chapter 4/Book 3 లో మనం చూసిన cross-cutting concern సమస్యే), Spring
**`@ControllerAdvice`** (Book 3's AOP-adjacent, controller-scoped
mechanism) వాడి, **అన్ని controllers కి centralized గా** exception
handling apply చేయవచ్చు:

```java
@RestControllerAdvice // = @ControllerAdvice + @ResponseBody
class GlobalExceptionHandler {

    @ExceptionHandler(CustomerNotFoundException.class) // Book 1 Chapter 6 custom exception hierarchy
    ResponseEntity<ErrorResponse> handleNotFound(CustomerNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(InsufficientBalanceException.class)
    ResponseEntity<ErrorResponse> handleInsufficientBalance(InsufficientBalanceException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(new ErrorResponse("INSUFFICIENT_BALANCE", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class) // Bean Validation failures
    ResponseEntity<ValidationErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
                .toList(); // Book 1 Chapter 7 Streams
        return ResponseEntity.badRequest()
                .body(new ValidationErrorResponse("VALIDATION_FAILED", fieldErrors));
    }

    @ExceptionHandler(Exception.class) // catch-all — SANITIZED, never leaks internals
    ResponseEntity<ErrorResponse> handleUnexpected(Exception ex) {
        String correlationId = UUID.randomUUID().toString();
        log.error("Unhandled exception [correlationId={}]", correlationId, ex); // full details logged internally
        return ResponseEntity.internalServerError()
                .body(new ErrorResponse("INTERNAL_ERROR",
                        "An unexpected error occurred. Reference: " + correlationId)); // sanitized externally
    }
}

record ErrorResponse(String code, String message) { }
record FieldError(String field, String message) { }
record ValidationErrorResponse(String code, List<FieldError> errors) { }
```

**ఇది ఎన్ని Book 1 సూత్రాలని ఒకచోట కలుపుతుందో గమనించండి:**
- **Custom exception hierarchy** (Book 1 Chapter 6) → ఒక్కో exception
  type కి ఒక్కో specific HTTP status.
- **"Never expose raw exceptions/stack traces externally"** (Book 1
  Chapter 6, security) → catch-all handler internal details ని log
  చేస్తుంది, **sanitized** message + correlation ID మాత్రమే client కి
  ఇస్తుంది.
- **Streams** (Book 1 Chapter 7) → field errors ని collect చేయడానికి.

### ENGLISH INTERVIEW ANSWER

"`@RestControllerAdvice` is Spring's mechanism for centralizing exception-
to-HTTP-response mapping across every controller — a direct application
of the cross-cutting concern idea from AOP, applied specifically to
exception handling. I map each domain exception from my custom hierarchy
(Book 1's `DomainException` subtypes) to the appropriate HTTP status —
not-found to 404, a business rule violation to 409 Conflict, and so on —
and I handle `MethodArgumentNotValidException` specifically to turn Bean
Validation failures into a structured, field-level error response the
client can actually act on. The one handler I'm most careful about is the
catch-all `Exception` handler: it must never leak a raw exception message
or stack trace to an external client — that's a real security exposure —
so it logs full details internally with a correlation ID, and returns
only a sanitized message plus that correlation ID externally, which
support/debugging can later cross-reference against the full server-side
logs."

---

## 4.3 CONCEPT: Custom Validators — Beyond Built-in Constraints

### TELUGU EXPLANATION

Built-in constraints (`@NotBlank`, `@Min`, మొదలైనవి) సరిపోని,
**business-specific** validation logic కి, ఒక **custom constraint
annotation** create చేయవచ్చు:

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ValidCouponCodeValidator.class)
public @interface ValidCouponCode { // custom annotation
    String message() default "Invalid coupon code format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

class ValidCouponCodeValidator implements ConstraintValidator<ValidCouponCode, String> {
    @Override
    public boolean isValid(String code, ConstraintValidatorContext context) {
        return code == null || code.matches("^[A-Z0-9]{6,10}$"); // business-specific format rule
    }
}

record CreateOrderRequest(@ValidCouponCode String couponCode, /* ... */) { }
```

**Senior రూల్:** Field-level validation (format, presence) కోసం custom
validators బాగుంటాయి. **Cross-field** validation (ఉదా: "startDate <
endDate") లేదా **database-dependent** validation (ఉదా: "ఈ email
ఇప్పటికే exists") — ఇవి తరచుగా **service layer** లో handle చేయడం
మంచిది, DTO-level constraint annotations లో కాదు, ఎందుకంటే అవి
external state (DB) మీద ఆధారపడతాయి, ఇది Bean Validation యొక్క
lightweight, stateless purpose కి సరిపోదు.

### ENGLISH INTERVIEW ANSWER

"Built-in constraints cover generic shape validation, but business-
specific rules — like a coupon code format — need a custom
`ConstraintValidator`. I use these for stateless, field-level checks. For
validation that requires database state — 'is this email already
registered' — or cross-field logic — 'start date must be before end date'
— I move that into the service layer instead of forcing it into a Bean
Validation annotation, since Bean Validation is designed to be lightweight
and side-effect-free; involving a database call in a `ConstraintValidator`
mixes concerns and makes testing/reasoning about validation much harder."

---

## 4.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Handling exceptions per controller | try/catch in every method, repeated | Centralized `@RestControllerAdvice` |
| Unexpected exception in production | Lets the default error page/raw stack trace leak to the client | Sanitized message + correlation ID externally, full details logged internally |
| Validation failure response | Generic "Bad Request" with no detail | Structured, field-level error response the client can act on |
| Business-specific input rule | Hand-checks it manually inside the controller/service | Custom `ConstraintValidator` for stateless field-level rules |

---

## 4.5 COMMON MISTAKES

1. Forgetting `@Valid` on the controller parameter, silently disabling
   all Bean Validation annotations on that DTO.
2. Letting a catch-all exception handler leak raw exception messages or
   stack traces to external API clients.
3. Duplicating exception-to-status mapping logic in every controller
   instead of centralizing it in one `@RestControllerAdvice`.
4. Putting database-dependent validation logic inside a
   `ConstraintValidator`, mixing concerns and complicating testing.
5. Returning an unstructured, generic error message for validation
   failures instead of field-level detail the client needs to fix the request.

---

## 4.6 INTERVIEW QUESTION BANK — CHAPTER 4

**Basic:** 1. What does `@Valid` actually do, and what happens without it?
2. What is `@RestControllerAdvice` for?

**Intermediate:** 3. How do you turn a `MethodArgumentNotValidException`
into a useful, field-level error response? 4. Why shouldn't a catch-all
exception handler return the raw exception message to the client?

**Senior:** 5. Design the full exception-to-HTTP-status mapping strategy
for an e-commerce checkout API (out of stock, payment declined, invalid
coupon, unexpected error) — tie this back to Book 1's custom exception
hierarchy. 6. When would you write a custom `ConstraintValidator`, and
when would you push validation into the service layer instead?

**Architect:** 7. You're designing error response standards across 50
microservices consumed by external partners. What would you standardize
(status codes, error body shape, correlation IDs) to make debugging
consistent across teams, and would you adopt a standard like RFC 7807
Problem Details?

**Scenario:** 8. A production incident report shows a client received a
500 error with a full Java stack trace, including internal package names
and a database connection string fragment, in the response body. What
went wrong, and what's the fix?

**Trick:** 9. "Once you add `@RestControllerAdvice`, you no longer need
try/catch anywhere in your service layer." True or false?

<details><summary>Key answers</summary>

- Q5: `OutOfStockException` → 409 Conflict; `PaymentDeclinedException` →
  402 Payment Required (or a custom application-level code, since 402
  isn't universally used); `InvalidCouponException` → 400 Bad Request;
  unexpected exceptions → 500 with sanitized message + correlation ID —
  each domain exception type maps to exactly one `@ExceptionHandler` in
  the centralized advice, directly extending Book 1 Chapter 6's exception
  hierarchy design into the HTTP layer.
- Q6: Custom `ConstraintValidator` for stateless, field-shape rules
  (format, range, pattern matching) with no external dependencies; service
  layer for anything needing database state, cross-field logic, or
  business rules that could change based on other entities' current state
  — the deciding factor is whether the validation needs anything beyond
  the field's own value.
- Q7: Standardize on a consistent error body shape (ideally adopting RFC
  7807 Problem Details — `type`, `title`, `status`, `detail`, `instance`
  fields — since it's a real, documented standard rather than an
  ad-hoc format each team invents independently), mandate a correlation/
  trace ID in every error response for cross-service debugging (bridging
  to Book 8's distributed tracing material), and standardize which HTTP
  status codes mean what across all services so partner integrations
  don't need per-service special-casing.
- Q8: The catch-all exception handler either didn't exist, was
  misconfigured, or explicitly returned `ex.getMessage()`/the exception
  object itself instead of a sanitized response — a real security and
  information-disclosure issue (leaking a connection string fragment is
  genuinely serious). Fix: ensure a `@ExceptionHandler(Exception.class)`
  always returns a sanitized message and logs full details only
  server-side, exactly as shown in section 4.2.
- Q9: False — `@RestControllerAdvice` handles translating exceptions into
  HTTP responses at the controller boundary, but service-layer code still
  needs try/catch wherever it needs to *recover*, *retry*, or *translate*
  a lower-level exception into a meaningful domain exception (Book 1
  Chapter 6's exception chaining) — centralized handling and
  service-layer exception handling solve different problems and both are needed.

</details>

---

## 4.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does forgetting `@Valid` silently disable all Bean Validation annotations on a DTO, rather than throwing an error pointing this out?
- **Coding Check:** Build a `@RestControllerAdvice` handling three custom domain exceptions plus `MethodArgumentNotValidException` plus a sanitized catch-all, following section 4.2's pattern exactly.
- **Explanation Check:** Explain in English why leaking a raw exception message to an external API client is a security issue, not just an aesthetic one.
- **Real-World Check:** Your team's API returns `400 Bad Request` for both "malformed JSON" and "valid JSON but a business rule was violated." A partner integration team complains they can't distinguish the two cases programmatically. Redesign the error responses to fix this.
- **Senior Check:** When would you choose NOT to centralize exception handling for a specific controller, handling an exception locally instead?
- **Master Check:** Design the complete validation and error-handling strategy for a multi-step checkout API (cart validation → payment → inventory reservation), where a failure at any step needs to roll back prior steps and return a client-actionable error indicating exactly which step failed and why.

<details><summary>Answers</summary>

- Real-World Check: Use distinct error codes in the response body (not
  just relying on the HTTP status) — e.g., `{"code":
  "MALFORMED_REQUEST", ...}` for JSON parsing failures (typically a
  `HttpMessageNotReadableException`, handled separately) vs `{"code":
  "VALIDATION_FAILED", "errors": [...]}` for Bean Validation failures vs
  `{"code": "COUPON_INVALID", ...}` for business rule violations — the
  HTTP status can remain 400 for the first two, but the structured `code`
  field lets partners branch programmatically without needing to
  string-match a human-readable message.
- Senior Check: When a specific controller has an unusual, highly
  specific exception-handling need that doesn't generalize (e.g., a
  legacy endpoint needing to preserve outdated response formatting for a
  specific old client that can't be changed) — a local
  `@ExceptionHandler` method within that one controller class overrides
  the global advice for that controller, which is the correct escape
  hatch for a genuine one-off need.
- Master Check: Each step (cart validation, payment, inventory
  reservation) throws its own specific exception type on failure
  (`CartValidationException`, `PaymentDeclinedException`,
  `InventoryUnavailableException`), all part of one `CheckoutException`
  hierarchy; the orchestrating service catches failures at each step and
  performs compensating actions for already-completed steps (e.g., if
  inventory reservation fails after payment succeeded, refund the
  payment) — a preview of Book 8's Saga pattern for distributed
  transactions; the centralized `@RestControllerAdvice` maps each specific
  exception to a response indicating exactly which step failed
  (`{"code": "PAYMENT_DECLINED", "step": "payment", "message": "..."}`),
  giving the client precise, actionable information rather than a generic checkout failure.

</details>

---

## 4.8 CHEAT SHEET

| Need | Mechanism |
|---|---|
| Declarative field validation | Bean Validation annotations (`@NotBlank`, `@Email`, `@Min`, etc.) |
| Actually enforce those annotations | `@Valid` on the controller parameter — required, easy to forget |
| Centralized exception-to-HTTP mapping | `@RestControllerAdvice` + `@ExceptionHandler` |
| Field-level validation error detail | Handle `MethodArgumentNotValidException` explicitly |
| Unexpected/unhandled exceptions | Sanitized message + correlation ID externally; full detail logged internally |
| Business-specific stateless field rule | Custom `ConstraintValidator` |
| Cross-field / DB-dependent validation | Service layer, not Bean Validation |

---

*(Continues to Chapter 5 — Spring Data Basics & Transactions Deep Dive.)*
