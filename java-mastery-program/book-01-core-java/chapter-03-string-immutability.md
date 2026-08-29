# CHAPTER 3 — STRING, IMMUTABILITY, AND THE STRING POOL

---

## 3.1 CONCEPT: Why String Is Immutable

### TELUGU EXPLANATION

Java లో `String` **immutable** — అంటే ఒకసారి create అయ్యాక, దాని content
ఎప్పటికీ మారదు. `s.concat("x")` లాంటి operations కొత్త `String` object create
చేస్తాయి, existing దాన్ని మార్చవు. దీనికి కారణాలు:

1. **String Pool (interning):** JVM `String` literals ని ఒక special memory
   area లో (String Pool, ఇప్పుడు Heap లోనే ఒక భాగం) cache చేస్తుంది. ఒకటే
   value ఉన్న రెండు literals ఒకే object ని share చేస్తాయి — memory ఆదా
   అవుతుంది. ఇది **mutable అయితే సాధ్యం కాదు** — ఒక reference దాన్ని మార్చితే,
   అదే object share చేస్తున్న అన్ని ఇతర references కూడా unexpectedly మారిపోతాయి.
2. **Security:** `String` లు తరచుగా file paths, network hostnames, DB
   credentials, class names (`Class.forName(name)`) పంపడానికి వాడతారు. ఒకవేళ
   mutable అయితే, ఒక method కి argument గా pass చేసిన తర్వాత, ఎవరైనా దాన్ని
   మార్చేసి security check ని bypass చేయవచ్చు (**TOCTOU** style bugs).
3. **Thread safety:** Immutable objects **inherently thread-safe** — multiple
   threads ఒకే `String` ని synchronization లేకుండా safely share చేయగలవు.
4. **HashCode caching:** `String.hashCode()` ఒకసారి compute అయ్యి cache
   అవుతుంది (`hash` field లో) — ఇది `HashMap` keys గా `String` వాడటం fast
   చేస్తుంది, ఎందుకంటే value మారదు కాబట్టి cached hash ఎప్పుడూ valid.

### INDUSTRY TERMINOLOGY

`String Pool`, `interning`, `String.intern()`, `immutability`,
`thread-safety`, `hashCode caching`, `StringBuilder`, `StringBuffer`.

### ENGLISH INTERVIEW ANSWER

"String is immutable in Java primarily for four reasons: it enables safe
string pooling/interning without aliasing bugs, it closes off a class of
security issues where a mutable string passed to a security-sensitive
API could be changed after validation but before use, it makes strings
inherently thread-safe with no synchronization needed, and it allows the hash
code to be computed once and cached, which matters a lot given how often
strings are used as `HashMap` keys. The trade-off is that naive string
concatenation in a loop creates many intermediate objects — that's why
`StringBuilder` exists for mutable, efficient string building, and why the
compiler itself rewrites simple `+` concatenation chains to use
`StringBuilder` under the hood in many cases."

### SIMPLE EXPLANATION

String never changes after creation → pool sharing safe, thread-safe,
hashCode cacheable, security-safe. Need to build strings a lot? Use
`StringBuilder` (mutable).

### REAL-TIME EXAMPLE

Production code that builds a large CSV export by concatenating strings in a
loop with `+=` for 100,000 rows creates ~100,000 intermediate `String`
objects, causing GC pressure and slow performance. Fix: use
`StringBuilder`, which mutates an internal `char[]`/`byte[]` buffer instead
of allocating a new object each time — this is a genuinely common code
review finding.

---

## 3.2 CONCEPT: String Pool Mechanics — `==` vs `.equals()`

### TELUGU EXPLANATION

```java
String a = "hello";        // literal → String Pool లో store అవుతుంది
String b = "hello";        // అదే literal → Pool లో already ఉన్న అదే object reuse
String c = new String("hello"); // "new" keyword → Heap లో కొత్త object, Pool వాడదు

System.out.println(a == b);        // true — ఇద్దరూ ఒకే Pool object ని point చేస్తున్నారు
System.out.println(a == c);        // false — c వేరే object (Heap లో), reference వేరు
System.out.println(a.equals(c));   // true — content సమానం
System.out.println(a == c.intern()); // true — intern() Pool object reference తిరిగి ఇస్తుంది
```

**సూత్రం:** `==` reference equality చెక్ చేస్తుంది (రెండూ ఒకే memory location
ని point చేస్తున్నాయా). `.equals()` content equality చెక్ చేస్తుంది. `String`
కోసం ఎప్పుడూ `.equals()` వాడాలి, `==` కాదు — ఇది Java లో అత్యంత common
beginner బగ్.

### ENGLISH INTERVIEW ANSWER

"`==` compares references — whether two variables point to the exact same
object in memory. `.equals()` compares logical content. String literals are
interned into the String Pool automatically, so two identical literals
happen to share a reference and `==` returns true — but that's an
implementation detail you should never rely on. The moment a string is
constructed via `new String(...)`, read from user input, built via
concatenation at runtime, or comes from I/O, it's a distinct heap object even
if its content matches a pooled string, and `==` will silently return false.
This is exactly why comparing strings with `==` is a well-known bug source,
and why static analysis tools flag it."

### SIMPLE EXPLANATION

`==` → same object? `.equals()` → same content? Always use `.equals()` for
Strings.

### REAL-TIME EXAMPLE

ఒక production bug: `if (userRole == "ADMIN")` అనే code, ఒక input source
నుండి role value వచ్చినప్పుడు (e.g., DB query result, deserialized JSON)
ఎప్పుడూ `false` return అవుతుంది, role actually "ADMIN" అయినా — ఎందుకంటే
runtime లో build అయిన `String` Pool object కాదు. దీన్ని `.equals()` కి
మార్చడం fix.

---

## 3.3 CODE: STRING BUILDING PERFORMANCE

```java
// ❌ Anti-pattern: O(n²) time complexity due to repeated String allocation
String buildCsvBad(List<String> rows) {
    String result = "";
    for (String row : rows) {
        result += row + "\n"; // ప్రతి iteration లో కొత్త String object!
    }
    return result;
}

// ✅ Correct: O(n) amortized, single mutable buffer
String buildCsvGood(List<String> rows) {
    StringBuilder sb = new StringBuilder();
    for (String row : rows) {
        sb.append(row).append('\n');
    }
    return sb.toString();
}
```

**Explanation:** `buildCsvBad` creates a new `String` on every iteration
because each `+=` discards the old object and allocates a new one copying
all prior content — for `n` rows this is O(n²) total character copying.
`buildCsvGood` grows an internal buffer (doubling capacity as needed) and
only materializes the final `String` once via `toString()` — O(n) overall.

**`StringBuilder` vs `StringBuffer`:** `StringBuffer` is the legacy
synchronized (thread-safe) version; `StringBuilder` is unsynchronized and
faster. Use `StringBuilder` unless you specifically need one buffer shared
and mutated across multiple threads (rare — usually you'd redesign instead).

**Interviewer follow-up:** "Does the compiler optimize simple `+`
concatenation automatically?" — Yes, for a fixed, simple expression like
`String s = a + b + c;`, javac typically compiles this to use
`StringBuilder`/`invokedynamic` (`StringConcatFactory` since Java 9) under
the hood. The anti-pattern specifically is concatenation **inside a loop**,
where each iteration creates a fresh `StringBuilder` unless you explicitly
hoist one outside the loop.

---

## 3.4 IMMUTABLE CLASSES — DESIGNING YOUR OWN

### TELUGU EXPLANATION

`String` లాగే, మీరు మీ own domain classes ని immutable గా design చేయవచ్చు —
ఇది concurrency bugs తగ్గిస్తుంది, defensive copying అవసరాన్ని తగ్గిస్తుంది.

**Rules for an immutable class:**
1. Class ని `final` చేయండి (subclass mutable behavior add చేయకుండా).
2. అన్ని fields `private final` గా ఉండాలి.
3. No setters.
4. Mutable fields (e.g., `List`, `Date`) ని constructor లో **defensively
   copy** చేయండి, మరియు getter లో కూడా copy/unmodifiable view return చేయండి —
   లేకపోతే caller internal state ని modify చేయగలరు (encapsulation break).

```java
public final class Money {
    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        this.amount = Objects.requireNonNull(amount);
        this.currency = Objects.requireNonNull(currency);
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency); // new object; original unchanged
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money m = (Money) o;
        return amount.equals(m.amount) && currency.equals(m.currency);
    }

    @Override
    public int hashCode() { return Objects.hash(amount, currency); }
}
```

### ENGLISH INTERVIEW ANSWER

"I favor immutable value objects for anything representing a value rather
than an identity — money, dates, coordinates, configuration snapshots.
Immutability eliminates an entire category of concurrency bugs because
there's no shared mutable state to synchronize, it makes objects safe to use
as `HashMap` keys or in `Set`s without fear of their hash code changing
after insertion, and it makes reasoning about code far easier since a
`Money` object I hold a reference to can never change under me. The main
cost is allocation — every 'change' creates a new object — which is
negligible for small value types and a real consideration only for
frequently-mutated large objects."

---

## 3.5 COMMON MISTAKES

1. Using `==` to compare `String` content.
2. String concatenation with `+`/`+=` inside loops.
3. "Immutable" classes that still expose a mutable internal `List`/`Map` via
   a getter without copying or wrapping in `Collections.unmodifiableList()`.
4. Forgetting to override both `equals()` and `hashCode()` together — this
   breaks `HashMap`/`HashSet` behavior (see Chapter 4).
5. Calling `.intern()` indiscriminately on runtime strings, thinking it's
   "free" — it has its own memory/performance cost and should be used
   deliberately (e.g., de-duplicating a huge set of repeated runtime strings
   from a data feed), not as a habit.

---

## 3.6 INTERVIEW QUESTION BANK — CHAPTER 3

**Basic:** 1. Why is String immutable? 2. Difference between `String`,
`StringBuilder`, `StringBuffer`?

**Intermediate:** 3. Explain `==` vs `.equals()` for Strings with an example.
4. What does `String.intern()` do?

**Senior:** 5. Design an immutable class with a mutable field (e.g., a
`List<String>` of tags) — show the defensive copy in both constructor and getter.
6. Why must `equals()` and `hashCode()` be overridden together?

**Architect:** 7. You're processing a 50GB data feed with millions of
repeated string values (e.g., country codes). How would you use interning
or a custom approach to reduce memory footprint, and what are the trade-offs?

**Scenario:** 8. A code review shows `if (status == "ACTIVE")` in a
service that reads `status` from a JSON payload. What's the bug, and what's
the fix?

**Trick:** 9. "`new String("hello") == "hello"` is always false." — is this
guaranteed by the language spec or just typical behavior? (It's guaranteed —
`new String(...)` explicitly creates a new heap object outside the pool.)

<details>
<summary>Key answers</summary>

- Q7: Use `String.intern()` selectively on the small set of genuinely
  repeated values (like country codes, a bounded set), not on high-cardinality
  free text — interning arbitrary high-cardinality strings can itself bloat
  the pool. An alternative is a custom deduplication map you control and can
  bound/evict, giving more control than the JVM's string pool.
- Q8: Reference comparison on a non-literal string (JSON-deserialized) will
  essentially always be false even when content matches — fix by using
  `"ACTIVE".equals(status)` (also null-safe against a null `status`).

</details>

---

## 3.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does caching `hashCode()` require immutability?
- **Coding Check:** Write an immutable `Address` class with a `List<String>` of `addressLines`, correctly defensive-copied.
- **Explanation Check:** Explain in English why `StringBuilder` is not thread-safe by design, and when that's actually fine.
- **Real-World Check:** A report-generation service concatenates 200,000 lines of text using `+=` and is timing out. Diagnose and fix.
- **Senior Check:** Would you make a JPA `@Entity` immutable? What tension does this create with ORMs like Hibernate?
- **Master Check:** Design a value class `TemperatureRange` (min, max) that enforces `min <= max` as an invariant, immutable, with a method `contains(Temperature t)`. Where does the invariant get enforced, and why there?

<details><summary>Answers</summary>

- Real-World Check: Root cause is O(n²) concatenation; fix with `StringBuilder`.
- Senior Check: Immutable entities clash with Hibernate's dirty-checking and
  proxy-based lazy loading, which expect mutable fields and a no-arg
  constructor; a common compromise is treating entities as mutable for
  persistence but exposing immutable DTOs/records to the rest of the
  application (see Book 7).
- Master Check: Enforce in the constructor (fail-fast on construction, so an
  invalid `TemperatureRange` can never exist) rather than in `contains()` —
  this is the "make illegal states unrepresentable" principle.

</details>

---

## 3.8 CHEAT SHEET

| Rule | Why |
|---|---|
| Never `==` for String content | Pool sharing is an implementation detail |
| `StringBuilder` in loops | Avoids O(n²) allocation |
| Immutable classes: `final` class, `private final` fields, no setters, defensive copies | Thread-safety, invariant safety, safe hashing |
| Override `equals()` + `hashCode()` together | Required for correct `HashMap`/`HashSet` behavior |

---

*(Continues to Chapter 4 — Collections Framework and HashMap Internals.)*
