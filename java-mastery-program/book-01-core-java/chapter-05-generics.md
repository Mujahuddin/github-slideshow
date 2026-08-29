# CHAPTER 5 — GENERICS AND TYPE SAFETY

---

## 5.1 CONCEPT: Why Generics Exist

### TELUGU EXPLANATION

Generics రాకముందు (Java 5 కంటే ముందు), Collections `Object` type వాడేవి:

```java
List list = new ArrayList();
list.add("hello");
list.add(42); // compile అవుతుంది, కానీ...
String s = (String) list.get(1); // Runtime లో ClassCastException!
```

ఇది **compile-time safety లేని** design — bug runtime వరకు కనిపించదు.
Generics దీన్ని fix చేస్తాయి: `List<String>` అంటే compiler కి "ఇందులో
Strings మాత్రమే ఉంటాయి" అని చెప్పడం. తప్పు type add చేయడానికి ప్రయత్నిస్తే,
**compile-time error** వస్తుంది — production runtime bug కాకుండా, developer
laptop మీదే catch అవుతుంది.

### INDUSTRY TERMINOLOGY

`Generics`, `type parameter`, `type erasure`, `bounded type`, `wildcard`,
`PECS (Producer Extends, Consumer Super)`, `raw type`,
`ClassCastException`, `heap pollution`.

### ENGLISH INTERVIEW ANSWER

"Generics move type-safety checks from runtime to compile time. Before
generics, collections held `Object`, so adding the wrong type compiled fine
and only failed with a `ClassCastException` when something later tried to
cast it back — often far from where the mistake was made. With
`List<String>`, the compiler enforces the constraint at every call site,
catching the bug immediately during development instead of in production."

---

## 5.2 CONCEPT: Type Erasure

### TELUGU EXPLANATION

Java Generics **compile-time feature మాత్రమే** — runtime లో, JVM
`List<String>` మరియు `List<Integer>` రెండింటినీ కేవలం `List` గా చూస్తుంది
(type parameter information "erased" అయిపోతుంది). దీన్నే **Type Erasure**
అంటారు. దీనికి కారణం: Java 5 కి ముందు compile అయిన pre-generics bytecode
తో **backward compatibility** maintain చేయడానికి.

**Practical consequences:**
- Runtime లో `list.getClass() == otherList.getClass()` — రెండూ ఒకే `List.class`
  అయినా, type parameter వేరు.
- మీరు `new T[10]` (generic array creation) చేయలేరు — compiler కి `T` ఏంటో
  runtime లో తెలియదు.
- `instanceof List<String>` చేయలేరు — `instanceof List<?>` మాత్రమే చేయవచ్చు.

### ENGLISH INTERVIEW ANSWER

"Generics in Java are a compile-time-only construct implemented via type
erasure — the compiler enforces type constraints and inserts the necessary
casts, but at runtime, `List<String>` and `List<Integer>` are both just
`List`. This was a deliberate design choice for backward compatibility with
pre-Java-5 bytecode. It has real, testable consequences: you can't create a
generic array (`new T[]`), you can't do `instanceof` against a parameterized
type, and reflection on a generic type's runtime class won't reveal the type
argument — this is exactly why frameworks needing that information (like
Jackson deserializing into `List<MyType>`) require extra machinery like
`TypeReference` or `ParameterizedType` to recover it."

---

## 5.3 CONCEPT: Bounded Types and Wildcards (PECS)

### TELUGU EXPLANATION

- **Bounded type parameter:** `<T extends Number>` — `T` Number లేదా దాని
  subclass మాత్రమే కావచ్చు అని restrict చేస్తుంది.
- **Wildcards:** ఒక method parameter type "ఏదో తెలియని subtype" అని express
  చేయడానికి.
  - `List<? extends Number>` — **producer** (మీరు దీని నుండి చదవగలరు, `Number`
    గా, కానీ దీనిలోకి add చేయలేరు — compiler కి actual subtype తెలియదు కాబట్టి).
  - `List<? super Integer>` — **consumer** (మీరు దీనిలోకి `Integer` add
    చేయగలరు, కానీ చదివితే `Object` గా మాత్రమే వస్తుంది).

**PECS సూత్రం (Producer Extends, Consumer Super):** ఒక structure నుండి
మీరు **చదివితేనే** (`produces` values for you) → `extends` వాడండి. ఒక
structure లోకి మీరు **రాస్తేనే** (`consumes` values from you) → `super`
వాడండి.

```java
// Producer: copies FROM src (reads) — src produces values
static void copy(List<? extends Number> src, List<? super Number> dest) {
    for (Number n : src) {  // ✅ reading as Number is safe
        dest.add(n);         // ✅ writing Number into a "? super Number" is safe
    }
    // src.add(5); // ❌ compile error — compiler doesn't know actual subtype
}
```

### ENGLISH INTERVIEW ANSWER

"I use PECS to decide which wildcard to reach for: if a parameter only
produces values I read out — like a source list I'm copying from — I use
`? extends T`, which is safe for reading as `T` but unsafe for writing,
since the compiler can't guarantee what concrete subtype it actually holds.
If a parameter only consumes values I write in — like a destination
collection — I use `? super T`, which is safe to write `T` into, but reading
only gives you `Object` back safely. `Collections.copy(List<? super T> dest,
List<? extends T> src)` in the JDK is the textbook example of both used
together correctly."

---

## 5.4 CODE: A GENERIC REPOSITORY PATTERN

**Requirement:** A reusable, type-safe base repository for any entity type
with an ID — a real pattern used across enterprise Spring Data-style code.

```java
public interface Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    T save(T entity);
    void deleteById(ID id);
}

public abstract class InMemoryRepository<T, ID> implements Repository<T, ID> {
    protected final Map<ID, T> store = new ConcurrentHashMap<>();

    protected abstract ID extractId(T entity);

    @Override
    public Optional<T> findById(ID id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<T> findAll() {
        return List.copyOf(store.values()); // defensive, immutable snapshot
    }

    @Override
    public T save(T entity) {
        store.put(extractId(entity), entity);
        return entity;
    }

    @Override
    public void deleteById(ID id) {
        store.remove(id);
    }
}

// Concrete usage:
class Customer {
    private final String id;
    private final String name;
    Customer(String id, String name) { this.id = id; this.name = name; }
    String getId() { return id; }
}

class CustomerRepository extends InMemoryRepository<Customer, String> {
    @Override
    protected String extractId(Customer entity) { return entity.getId(); }
}
```

**Explanation:** `Repository<T, ID>` is generic over both the entity type
and its ID type — this is exactly the shape Spring Data JPA's
`JpaRepository<T, ID>` follows (Book 7). `findAll()` returns
`List.copyOf(...)`, an immutable snapshot, so callers can't mutate the
internal store through the returned list — encapsulation from Chapter 2
combined with generics here.

**Interviewer follow-ups:**
- "Why is `store` a `ConcurrentHashMap` and not a plain `HashMap`?" (Thread
  safety for a shared repository accessed by multiple request threads —
  Chapter 4/9 material.)
- "Could `extractId` be avoided?" (Yes, with a functional interface passed
  into the constructor instead of subclassing and overriding — a preview
  of Chapter 7's move toward composition-over-inheritance with lambdas.)

---

## 5.5 COMMON MISTAKES

1. Using raw types (`List` instead of `List<String>`) in new code — loses
   all compile-time safety and generates "unchecked" warnings.
2. Trying to create a generic array (`new T[10]`) — doesn't compile; use
   `ArrayList<T>` or `(T[]) new Object[10]` with an explicit, documented
   unchecked cast only when unavoidable (e.g., in library internals).
3. Confusing `<? extends T>` and `<? super T>` — remember PECS.
4. Assuming generic type information is available via reflection at
   runtime without special handling (it's erased).
5. Overusing generics for one-off code that never needs multiple type
   parameters — adds ceremony without payoff (a real senior judgment call).

---

## 5.6 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. Why were generics introduced? 2. What is a raw type, and why avoid it?

**Intermediate:** 3. What is type erasure, and what practical limitations
does it impose? 4. Explain PECS with an example.

**Senior:** 5. Why can't you create `new T[]` in a generic class? What
workarounds exist? 6. How does Jackson deserialize into a
`List<MyType>` given type erasure?

**Architect:** 7. Design a generic, reusable `Result<T, E>` type (like
Rust's `Result`) for representing success/failure without exceptions in a
Java codebase. What are the trade-offs vs. checked exceptions?

**Scenario:** 8. A teammate writes `public <T> T process(Object input) {
return (T) input; }` and calls it everywhere with different type arguments.
What's wrong, and when does it blow up?

**Trick:** 9. "Because of type erasure, generics provide zero runtime
benefit." True or false?

<details><summary>Key answers</summary>

- Q6: Jackson requires an explicit `TypeReference<List<MyType>>` (or
  `JavaType` built via `TypeFactory`) specifically because the raw
  `Class<List>` object at runtime can't reveal that it was parameterized
  with `MyType` — this is a direct, practical consequence of erasure.
- Q8: This compiles with an "unchecked cast" warning and defers the actual
  type check to wherever the caller assigns the result — it can silently
  accept the wrong type and blow up with a `ClassCastException` far from
  the actual bug, exactly the problem generics were meant to prevent; this
  pattern effectively reintroduces the pre-generics failure mode.
- Q9: False — while there's no runtime type-checking benefit (erasure
  removes that), the *compile-time* benefit is the entire point: catching
  type mistakes before the code ever runs, which is a very real benefit.

</details>

---

## 5.7 MASTERY CHECKPOINTS

- **Knowledge Check:** What does type erasure erase, specifically?
- **Coding Check:** Write a generic `Pair<A, B>` class with `equals`/`hashCode`.
- **Explanation Check:** Explain PECS in English using a real `copyAll(List<? extends T> src, Collection<? super T> dest)` style method you write yourself.
- **Real-World Check:** A generic caching utility `Cache<K, V>` needs to support both reading and writing. Design its method signatures using wildcards correctly, or explain why wildcards don't apply here.
- **Senior Check:** When would you choose NOT to make a class generic, even though it technically could be?
- **Master Check:** Explain, from first principles, why `List<Object>` and `List<String>` are NOT related by subtyping (you can't pass a `List<String>` where `List<Object>` is expected) even though `String` is a subtype of `Object`.

<details><summary>Answers</summary>

- Real-World Check: A `Cache<K, V>` is both a producer (reads return `V`) and
  a consumer (writes take `V`) for the *same* type parameter on the *same*
  object, so wildcards (which are for one-directional producer/consumer
  relationships on a parameter) don't apply — you just use the invariant
  type parameter `V` directly for both `get` and `put`. PECS applies to
  method parameters describing a relationship, not to a type's own
  read/write pair.
- Senior Check: When the class only ever handles one concrete type in
  practice and genericizing it would only add ceremony with no real reuse
  benefit — YAGNI applies to generics too.
- Master Check: If `List<String>` were a subtype of `List<Object>`, you
  could do `List<Object> l = new ArrayList<String>(); l.add(42);` — adding
  an `Integer` into what's actually a `List<String>` at runtime, violating
  type safety. Generics are deliberately invariant to prevent exactly this;
  wildcards (`? extends`/`? super`) are the controlled, safe way to get
  flexibility back where it's actually sound.

</details>

---

## 5.8 CHEAT SHEET

| Concept | Rule of thumb |
|---|---|
| Raw type | Never use in new code |
| `<? extends T>` | Read-only source (Producer) |
| `<? super T>` | Write-only destination (Consumer) |
| Type erasure | No generic type info at runtime — no `new T[]`, no `instanceof List<String>` |
| Bounded type `<T extends X>` | Restricts T to X or its subtypes, gives access to X's methods |

---

*(Continues to Chapter 6 — Exception Handling Done Right.)*
