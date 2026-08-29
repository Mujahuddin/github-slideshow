# CHAPTER 7 — JAVA 8: LAMBDAS, FUNCTIONAL INTERFACES, STREAMS, OPTIONAL

---

## 7.1 CONCEPT: Functional Interfaces and Lambdas

### TELUGU EXPLANATION

**Functional interface** అంటే **exactly ఒక్క abstract method** ఉన్న
interface (`@FunctionalInterface` annotation optional గా దీన్ని enforce
చేస్తుంది, compile-time లో). Java 8 కి ముందు, ఒక behavior ని parameter గా
pass చేయాలంటే anonymous inner class రాయాల్సి వచ్చేది — verbose:

```java
// Java 8 కి ముందు
Comparator<String> byLength = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return Integer.compare(a.length(), b.length());
    }
};
```

**Lambda expressions** దీన్ని concise గా చేస్తాయి — functional interface కి
ఒక్క implementation ఇవ్వడానికి shortcut syntax:

```java
Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());
```

**Core built-in functional interfaces** (`java.util.function` package):

| Interface | Signature | Purpose |
|---|---|---|
| `Function<T, R>` | `R apply(T t)` | Transform T → R |
| `Predicate<T>` | `boolean test(T t)` | Boolean check |
| `Consumer<T>` | `void accept(T t)` | Side-effect action, no return |
| `Supplier<T>` | `T get()` | Produce a value, no input |
| `BiFunction<T, U, R>` | `R apply(T t, U u)` | Two-arg transform |
| `UnaryOperator<T>` | `T apply(T t)` | Function where input/output type same |

### ENGLISH INTERVIEW ANSWER

"A functional interface is any interface with exactly one abstract method,
which lets a lambda expression serve as its implementation. Java 8's
`java.util.function` package standardizes the common shapes you need —
`Function` for transformation, `Predicate` for boolean tests, `Consumer` for
side-effecting actions, `Supplier` for lazy value production — so most code
doesn't need to declare custom functional interfaces at all. This is the
foundation the Streams API is built on: every intermediate and terminal
Stream operation takes one of these functional interfaces, which is what
makes stream pipelines composable and readable."

---

## 7.2 CONCEPT: The Streams API — Declarative Data Processing

### TELUGU EXPLANATION

Streams API imperative loops బదులుగా **declarative** గా collections ని
process చేయడానికి వీలు కల్పిస్తుంది — "ఏం చేయాలో" చెప్పడం, "ఎలా చేయాలో"
కాదు.

```java
// Imperative style
List<String> activeCustomerNames = new ArrayList<>();
for (Customer c : customers) {
    if (c.isActive()) {
        activeCustomerNames.add(c.getName().toUpperCase());
    }
}

// Declarative Stream style
List<String> activeCustomerNames = customers.stream()
        .filter(Customer::isActive)
        .map(c -> c.getName().toUpperCase())
        .collect(Collectors.toList());
```

**కీలక concepts:**
- **Intermediate operations** (`filter`, `map`, `sorted`, `distinct`) —
  **lazy** గా ఉంటాయి, terminal operation call చేసేవరకు execute అవ్వవు.
- **Terminal operations** (`collect`, `forEach`, `reduce`, `count`) — stream
  ని "consume" చేసి actual result produce చేస్తాయి. ఒక stream ఒకసారి
  terminal operation తర్వాత **మళ్ళీ వాడలేరు** (`IllegalStateException`).
- **Short-circuiting** operations (`findFirst`, `anyMatch`, `limit`) —
  మొత్తం collection process చేయకుండా, అవసరమైనంత మాత్రమే process చేసి
  ఆగిపోతాయి — performance benefit.
- **Parallel streams** (`.parallelStream()`) — multiple threads వాడి
  process చేస్తాయి, కానీ **జాగ్రత్త అవసరం**: small collections కి overhead
  ఎక్కువ కావొచ్చు, shared mutable state వాడితే thread-safety bugs వస్తాయి,
  మరియు order-dependent operations (e.g., `forEach` with side effects)
  non-deterministic గా మారొచ్చు.

### ENGLISH INTERVIEW ANSWER

"Streams let me express data transformations declaratively — filter, map,
reduce — as a pipeline, rather than as an imperative loop with mutable
accumulator variables. Intermediate operations are lazy; nothing actually
runs until a terminal operation triggers the pipeline. I'm careful with
parallel streams specifically: I only reach for `.parallelStream()` when the
collection is large enough that the fork-join overhead is worth it, the
operation is CPU-bound rather than I/O-bound, and — most importantly — the
lambda has no shared mutable state and no dependency on encounter order,
since parallel streams process elements out of order across threads by
design. I've seen real bugs from a `forEach` lambda mutating a shared
non-thread-safe `List` inside a parallel stream — that's a classic and
completely avoidable production bug."

### SIMPLE EXPLANATION

Stream = pipeline of filter/map/etc → terminal op triggers execution once.
Parallel streams: only for large, CPU-bound, side-effect-free work.

### REAL-TIME EXAMPLE

ఒక reporting service, 500,000 transactions ని process చేసి,
region-wise totals compute చేయాల్సి వచ్చినప్పుడు, `Collectors.groupingBy()`
+ `Collectors.summingDouble()` combination వాడి, ఒక్క declarative stream
pipeline లో (imperative nested loops కంటే readable గా) చేయవచ్చు:

```java
Map<String, Double> totalsByRegion = transactions.stream()
        .collect(Collectors.groupingBy(
                Transaction::getRegion,
                Collectors.summingDouble(Transaction::getAmount)));
```

---

## 7.3 CONCEPT: Optional — Making Absence Explicit

### TELUGU EXPLANATION

`Optional<T>` ఒక **container** — value ఉండొచ్చు లేదా ఉండకపోవచ్చు అని
explicit గా type system లో express చేస్తుంది. Purpose: `null` return చేసే
methods callers కి "ఇది null అవ్వొచ్చు, check చేయండి" అని silent గా వదిలేస్తాయి
— దీనివల్ల `NullPointerException` (NPE) production లో అత్యంత common bug
అవుతుంది. `Optional` దీన్ని **compiler/API level** లో force చేస్తుంది.

```java
// ❌ Caller might forget the null check
public Customer findById(String id) { ... } // returns null if not found

// ✅ Return type itself signals "might be absent"
public Optional<Customer> findById(String id) { ... }

Optional<Customer> maybeCustomer = repository.findById("123");
String name = maybeCustomer
        .map(Customer::getName)
        .orElse("Unknown Customer"); // No null check needed, no NPE risk
```

**జాగ్రత్తలు (common misuse):**
- `Optional` ని **fields** గా వాడకూడదు (not `Serializable`, adds
  unnecessary wrapping overhead — use `null` with clear documentation, or
  better, avoid nullable fields via good design).
- `Optional` ని **method parameters** గా వాడకూడదు — method overloading
  లేదా clearer API design వాడాలి.
- `optional.get()` ని null-check లేకుండా direct గా call చేయడం `Optional`
  యొక్క purpose నే defeat చేస్తుంది — ఎప్పుడూ `map`/`orElse`/`orElseThrow`/
  `ifPresent` వాడాలి.

### ENGLISH INTERVIEW ANSWER

"`Optional` exists to make the possibility of absence part of a method's
type signature, so callers can't accidentally forget a null check the way
they can with a plain nullable return type. Its intended use is narrowly as
a *return type* for methods where 'no result' is a legitimate outcome — I
avoid it for fields and method parameters, where it just adds an unnecessary
wrapper without the same safety benefit, and Effective Java is explicit
about this too. The other discipline is not defeating its purpose by calling
`.get()` without checking presence — I always chain through `map`,
`orElse`, `orElseGet`, or `orElseThrow` so the 'what if it's absent' case is
handled at the same place the value is used, not deferred to an implicit
null check that's easy to skip."

---

## 7.4 CODE: A REALISTIC STREAM + OPTIONAL PIPELINE

**Requirement:** Given a list of orders, find the highest-value order placed
by a specific customer, or return a clear "not found" result.

```java
public Optional<Order> findHighestValueOrder(List<Order> orders, String customerId) {
    return orders.stream()
            .filter(o -> o.getCustomerId().equals(customerId))
            .max(Comparator.comparing(Order::getTotalAmount));
}

// Usage:
Optional<Order> result = findHighestValueOrder(orders, "CUST-123");
BigDecimal highestAmount = result
        .map(Order::getTotalAmount)
        .orElseThrow(() -> new CustomerNotFoundException("CUST-123"));
```

**Execution flow:** `filter` lazily marks which elements match (nothing runs
yet); `max` is a terminal operation that triggers the actual pass over the
stream, comparing elements via the given `Comparator`, and naturally
produces an `Optional<Order>` because the stream could be empty (no matching
orders) — this is exactly why `max`/`min`/`findFirst`/`findAny` in the
Streams API return `Optional`, not a raw value.

**Interviewer follow-ups:**
- "Why does `max()` return `Optional<Order>` instead of `Order`?" (Because
  the stream could be empty after filtering — there is no sensible non-null
  `Order` to return, so the API is honest about that in its return type.)
- "What's the complexity here?" (O(n) single pass for filter+max combined —
  Streams typically fuse intermediate operations into one pass where
  possible, not one pass per operation.)
- "Would you use a parallel stream here?" (Only if `orders` were very
  large and this ran on a hot path where the fork-join overhead pays off —
  for a typical bounded list, sequential is simpler and just as fast.)

---

## 7.5 COMMON MISTAKES

1. Calling `Optional.get()` without checking `isPresent()`/using `map`/`orElse` — reintroduces the exact NPE risk `Optional` was meant to prevent.
2. Using `Optional` as a field type or method parameter.
3. Mutating shared state inside a stream's lambda (especially parallel streams).
4. Overusing streams for simple loops where a plain `for` loop is more readable — streams aren't always the "senior" choice; readability matters more than looking modern.
5. Reusing a stream after a terminal operation (`IllegalStateException: stream has already been operated upon or closed`).
6. Ignoring that `Collectors.toList()` doesn't guarantee a mutable or specific `List` implementation — use `Collectors.toCollection(ArrayList::new)` if mutability/type matters.

---

## 7.6 INTERVIEW QUESTION BANK — CHAPTER 7

**Basic:** 1. What is a functional interface? 2. Difference between `map` and `filter`?

**Intermediate:** 3. Explain lazy evaluation of intermediate stream operations with an example. 4. When should you NOT use `Optional`?

**Senior:** 5. When would you avoid parallel streams even for a large collection? 6. Explain `flatMap` with a real use case (e.g., flattening a `List<List<Order>>`).

**Architect:** 7. You're processing a 10-million-row dataset with a complex multi-stage transformation. Would you use Java Streams, parallel streams, or a dedicated data-processing framework (e.g., Spark)? Justify based on scale and operational concerns.

**Scenario:** 8. A parallel stream `forEach` that appends to a shared `ArrayList` occasionally loses elements or throws `ArrayIndexOutOfBoundsException` in production, but only under load. Diagnose.

**Trick:** 9. "Streams are always faster than loops." True or false?

<details><summary>Key answers</summary>

- Q7: For 10 million rows in a single JVM, plain Streams (possibly parallel,
  carefully tuned) can work if it fits comfortably in memory and runs on one
  machine; beyond that scale, or when distributed processing / fault
  tolerance / disk-spilling is needed, a framework like Spark is the right
  tool — Streams aren't a distributed processing engine and don't provide
  those guarantees.
- Q8: Classic parallel-stream bug — `ArrayList` isn't thread-safe, and
  multiple threads calling `.add()` concurrently corrupt its internal state.
  Fix: use a thread-safe collector (`Collectors.toList()` inside `.collect()`
  on the parallel stream itself, which handles this correctly) instead of
  manually mutating a shared list from `forEach`.
- Q9: False — for small collections or simple operations, a plain loop can
  be faster due to stream pipeline setup overhead (boxing, lambda
  invocation overhead); streams win on *readability* and, for large
  parallelizable CPU-bound work, on *throughput* — not universally on raw
  speed.

</details>

---

## 7.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Name three built-in functional interfaces and their exact method signatures.
- **Coding Check:** Given `List<Order>`, produce a `Map<CustomerId, List<Order>>` grouping using streams.
- **Explanation Check:** Explain in English why `Optional.get()` without a check is considered an anti-pattern, using the exact failure mode it reintroduces.
- **Real-World Check:** A service method returns `null` when a configuration key is missing, and three different callers handle that inconsistently (one NPEs, one silently proceeds with wrong behavior, one checks correctly). Redesign the method signature to make this impossible.
- **Senior Check:** When is a plain `for` loop the better choice over a stream pipeline, even for an experienced engineer?
- **Master Check:** Design a stream pipeline that reads a `List<Transaction>`, filters to only successful ones, groups by month, and finds the top-3 highest-value transactions per month — using the Collectors API correctly.

<details><summary>Answers</summary>

- Real-World Check: Change the return type to `Optional<ConfigValue>`, forcing every caller to explicitly handle absence via `orElse`/`orElseThrow`/`map` — eliminating the possibility of an inconsistent, ad-hoc null-handling convention across call sites.
- Senior Check: When the logic involves complex branching, early returns, or mutable accumulation that would make a stream pipeline convoluted or need ugly workarounds (like mutable arrays captured in lambdas) — forcing stream style onto inherently imperative logic hurts readability, which is the opposite of the goal.
- Master Check: `transactions.stream().filter(Transaction::isSuccessful).collect(Collectors.groupingBy(t -> t.getDate().getMonth(), Collectors.collectingAndThen(Collectors.toList(), list -> list.stream().sorted(Comparator.comparing(Transaction::getAmount).reversed()).limit(3).collect(Collectors.toList()))))` — grouping by month, then within each group's downstream collector, sorting descending and limiting to 3.

</details>

---

## 7.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Functional interface | Exactly one abstract method |
| Intermediate ops | Lazy, return a new Stream (`filter`, `map`, `sorted`) |
| Terminal ops | Trigger execution, consume the stream (`collect`, `forEach`, `reduce`) |
| Parallel streams | Only for large, CPU-bound, side-effect-free, order-independent work |
| `Optional` | Return type only; never field/parameter; never bare `.get()` |
| `flatMap` | Flattens nested structures (`Stream<Stream<T>>` → `Stream<T>`) |

---

*(Continues to Chapter 8 — Java 11/17: `var`, Records, Sealed Classes, Text Blocks.)*
