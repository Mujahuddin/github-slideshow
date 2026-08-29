# CHAPTER 4 — COLLECTIONS FRAMEWORK AND HASHMAP INTERNALS

---

## 4.1 CONCEPT: Collections Framework Overview — Choosing the Right Structure

### TELUGU EXPLANATION

Java Collections Framework ఒక **hierarchy of interfaces + implementations**.
Senior interview లో "ఏ collection వాడతారు?" అంటే, memorized list చెప్పడం
కాదు — **access pattern** ఆధారంగా reasoning చేయాలి:

| అవసరం | సరైన choice | ఎందుకు |
|---|---|---|
| Index-based fast random access | `ArrayList` | O(1) get by index (array-backed) |
| తరచుగా middle insertion/deletion | `LinkedList` | O(1) insert/delete at known node (కానీ traversal O(n)) |
| Uniqueness అవసరం, order అవసరం లేదు | `HashSet` | O(1) avg add/contains |
| Uniqueness + insertion order కావాలి | `LinkedHashSet` | O(1) avg + predictable iteration order |
| Uniqueness + sorted order కావాలి | `TreeSet` | O(log n), sorted (Red-Black tree) |
| Key-value lookup, order అవసరం లేదు | `HashMap` | O(1) avg get/put |
| Key-value + insertion order | `LinkedHashMap` | O(1) avg + order (also LRU cache base) |
| Key-value + sorted by key | `TreeMap` | O(log n), sorted |
| Thread-safe map, high concurrency | `ConcurrentHashMap` | Segmented locking, no full-map lock |
| FIFO processing | `ArrayDeque` (as Queue) | O(1) add/remove at ends, faster than `LinkedList` |
| Priority-based processing | `PriorityQueue` | O(log n) insert, O(1) peek min/max |

**Interview red flag:** "నేను ఎప్పుడూ `ArrayList` వాడతాను" అనే జవాబు —
ఇది access pattern reasoning లేకుండా ఇచ్చిన జవాబు, senior level కి సరిపోదు.

### ENGLISH INTERVIEW ANSWER

"I choose a collection based on the dominant access pattern, not habit.
If I need indexed random access, `ArrayList`. If I'm frequently inserting or
removing at a known position — especially at the ends — I consider
`ArrayDeque` or `LinkedList`, though in practice `ArrayDeque` outperforms
`LinkedList` for most queue/stack use cases due to better cache locality. For
key-based lookups, `HashMap` is the default, moving to `LinkedHashMap` when
I need predictable iteration order — which is also how I'd build an LRU
cache — or `TreeMap` when I need sorted keys. In a concurrent context, I
reach for `ConcurrentHashMap` rather than synchronizing a `HashMap`
externally, because it uses fine-grained locking instead of a single lock
across the whole map."

---

## 4.2 CONCEPT: HashMap Internals — The Interview Classic

### TELUGU EXPLANATION

`HashMap` internals అనేది Java interviews లో అత్యంత frequently అడిగే
topic. దీన్ని step by step అర్థం చేసుకుందాం:

**Structure:** `HashMap` internally ఒక `Node<K,V>[] table` (array of buckets)
కలిగి ఉంటుంది. ప్రతి bucket ఒక **linked list** (లేదా, Java 8+ లో, ఎక్కువ
entries ఉంటే ఒక **Red-Black Tree**) గా ఉంటుంది.

**`put(key, value)` ఎలా పని చేస్తుంది:**
1. `key.hashCode()` compute చేస్తారు.
2. JVM ఒక **hash spreading function** apply చేస్తుంది
   (`hash ^ (hash >>> 16)`) — ఇది higher bits ని lower bits తో XOR చేసి,
   hash distribution ని improve చేస్తుంది (ముఖ్యంగా table size తక్కువగా
   ఉన్నప్పుడు).
3. `index = (table.length - 1) & hash` — bucket index compute చేస్తారు
   (table length ఎప్పుడూ power of 2 గా ఉంటుంది కాబట్టి, ఇది modulo కంటే
   fast bitwise operation).
4. ఆ bucket లో ఇప్పటికే entries ఉంటే (**collision**):
   - ప్రతి existing entry తో `key.equals()` check చేస్తారు — match దొరికితే
     value replace.
   - Match దొరకకపోతే, కొత్త entry ని bucket చివర (linked list గా) add
     చేస్తారు.
   - ఒక bucket లో entries count **8** దాటితే **మరియు** table size కనీసం
     **64** ఉంటే, ఆ bucket linked list నుండి **Red-Black Tree** గా convert
     అవుతుంది (**treeification**) — worst-case lookup O(n) నుండి
     O(log n) కి improve అవుతుంది (Java 8+ లో ప్రవేశపెట్టిన optimization,
     hash-flooding attacks నుండి కూడా రక్షణ కోసం).
5. Total entries, **load factor** (default 0.75) × current capacity ని దాటితే,
   **resize** (rehash) జరుగుతుంది — table size double అవుతుంది, అన్ని
   entries కొత్త buckets కి redistribute అవుతాయి (ఇది expensive operation,
   O(n)).

**`get(key)` ఎలా పని చేస్తుంది:** అదే hash → index compute చేసి, ఆ bucket లో
`equals()` ద్వారా matching key కోసం వెతుకుతారు.

**ఎందుకు `equals()` మరియు `hashCode()` రెండూ override చేయాలి?**
`HashMap` contract: రెండు objects `.equals()` ప్రకారం equal అయితే, వాటి
`hashCode()` **తప్పకుండా** సమానంగా ఉండాలి (reverse తప్పనిసరి కాదు — different
objects కి same hashCode ఉండొచ్చు, అది collision అంతే). ఈ contract break
చేస్తే (ఉదా: `equals()` override చేసి `hashCode()` override చేయకపోతే), రెండు
"equal" objects వేర్వేరు buckets లోకి వెళ్ళిపోయి, `map.get(key)` దాన్ని
find చేయలేకపోతుంది — ఇది classic, hard-to-debug production bug.

### INDUSTRY TERMINOLOGY

`bucket`, `collision`, `load factor`, `treeification`, `Red-Black Tree`,
`resize/rehash`, `hash spreading`, `equals-hashCode contract`,
`hash-flooding attack`.

### ENGLISH INTERVIEW ANSWER

"`HashMap` is backed by an array of buckets, each holding a linked list (or,
since Java 8, a red-black tree once a bucket exceeds 8 entries and the table
is at least 64 buckets — this treeification also hardens against
hash-flooding denial-of-service attacks that deliberately cause massive
collisions). On `put`, the key's hash code is spread using
`hash ^ (hash >>> 16)` to improve distribution, then masked against
`table.length - 1` to get the bucket index. On collision, keys are compared
with `equals()` to decide whether to replace a value or append a new entry.
When the number of entries exceeds capacity times the load factor — 0.75 by
default — the table resizes, doubling in size and rehashing every entry,
which is an O(n) operation you want to avoid triggering repeatedly, hence
sizing the initial capacity when you know the expected entry count in
advance. The reason `equals()` and `hashCode()` must be overridden together
is the `HashMap` contract: equal objects must produce equal hash codes, or
they'll land in different buckets and `get()` will silently fail to find an
entry that logically exists — one of the most common real bugs I've seen
with custom key classes."

### SIMPLE EXPLANATION

Array of buckets → hash decides bucket → collisions form a list (or tree if
long) → resize doubles capacity when too full. equals+hashCode must agree,
or lookups silently break.

### REAL-TIME EXAMPLE

ఒక production bug: `Map<OrderKey, Order> cache` వాడుతున్నారు, `OrderKey`
class `equals()` override చేసింది కానీ `hashCode()` override చేయలేదు (default
`Object.hashCode()` — identity-based). Result: content ప్రకారం సమానమైన రెండు
`OrderKey` instances వేర్వేరు hash codes ఇస్తాయి, వేర్వేరు buckets లోకి
వెళ్తాయి — `cache.get(sameLogicalKey)` ఎప్పుడూ `null` return చేస్తుంది,
cache "never hits," silently degrading performance without throwing any error.

---

## 4.3 CODE: A CORRECT CUSTOM KEY CLASS

```java
public final class OrderKey {
    private final String customerId;
    private final String orderId;

    public OrderKey(String customerId, String orderId) {
        this.customerId = Objects.requireNonNull(customerId);
        this.orderId = Objects.requireNonNull(orderId);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof OrderKey)) return false;
        OrderKey that = (OrderKey) o;
        return customerId.equals(that.customerId) && orderId.equals(that.orderId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(customerId, orderId); // combines both fields consistently with equals()
    }
}
```

**Design notes:** Both fields used in `equals()` are also used in
`hashCode()` — this is the non-negotiable rule. `Objects.hash(...)` is a
convenient, correct way to combine multiple fields into one hash code.
The class is also immutable (Chapter 3), which is doubly important for map
keys: if a key's fields could change after insertion, its hash code would
change too, and the entry would become unreachable in its original bucket —
a subtle, real bug class.

**Interviewer follow-ups:**
- "What happens if you mutate a mutable object after using it as a HashMap
  key?" (You risk permanently 'losing' that entry — `get()` recomputes the
  hash on the *current* state, so it looks in the wrong bucket.)
- "Why is `Objects.equals()` preferred over `==` inside `equals()` for
  fields?" (Null-safety — `Objects.equals(a, b)` handles either being null
  without a `NullPointerException`.)

---

## 4.4 CONCURRENTHASHMAP VS SYNCHRONIZED HASHMAP

### TELUGU EXPLANATION

`Collections.synchronizedMap(new HashMap<>())` మొత్తం map మీద ఒకే lock
వాడుతుంది — ప్రతి `get`/`put` ఆ ఒక్క lock కోసం wait చేయాలి, high-concurrency
లో ఇది **bottleneck**. `ConcurrentHashMap` దీనికి భిన్నంగా, **fine-grained
locking** (bucket-level, CAS operations) వాడుతుంది — multiple threads వేర్వేరు
buckets మీద ఏకకాలంలో పని చేయగలవు, `size()`/read operations చాలా వరకు lock-free.
Java 8+ లో `ConcurrentHashMap` internal గా HashMap లాంటి tree-based buckets +
CAS (Compare-And-Swap) + minimal synchronized blocks వాడుతుంది.

### ENGLISH INTERVIEW ANSWER

"`Collections.synchronizedMap` wraps every method call with a single lock on
the whole map, so under contention it serializes all access regardless of
which keys are touched. `ConcurrentHashMap` uses much finer-grained
concurrency control — historically per-segment locking, and since Java 8,
CAS operations combined with locking only at the individual bin level for
writes — so threads operating on different buckets don't block each other.
I default to `ConcurrentHashMap` for any shared mutable map in a multithreaded
service; I'd only reach for a synchronized wrapper in legacy code I'm not
free to change, or when I need to synchronize compound operations across
multiple map calls atomically using the map's own lock."

---

## 4.5 COMMON MISTAKES

1. Overriding `equals()` without `hashCode()` (or vice versa).
2. Using a mutable object as a `HashMap`/`HashSet` key.
3. Iterating and modifying a collection at the same time without an
   `Iterator.remove()` (causes `ConcurrentModificationException`).
4. Not setting an initial capacity when the expected size is known, causing
   avoidable resizes.
5. Using `Vector`/`Hashtable` (legacy, fully synchronized, slower) in new
   code instead of `ArrayList`/`ConcurrentHashMap`.
6. Assuming `HashMap` iteration order is stable — it isn't; use
   `LinkedHashMap` if order matters.

---

## 4.6 INTERVIEW QUESTION BANK — CHAPTER 4

**Basic:** 1. Difference between `ArrayList` and `LinkedList`? 2. What is a
load factor?

**Intermediate:** 3. Walk through what happens internally on `HashMap.put()`
with a collision. 4. Why does `HashMap` resize, and what's the cost?

**Senior:** 5. Explain treeification — why was it introduced, and what
security concern does it partially address? 6. Why is `ConcurrentHashMap`
faster than a synchronized `HashMap` under high concurrency?

**Architect:** 7. You're building a cache layer used by 500 threads doing a
mix of 90% reads / 10% writes. Would you use `ConcurrentHashMap`, a
`ReadWriteLock`-guarded `HashMap`, or something else (e.g., Caffeine)? Justify.

**Scenario:** 8. A `Map<CustomerId, Customer> cache` "never hits" in
production despite obviously repeated lookups for the same customer. What do
you check first?

**Trick:** 9. "Two unequal objects can never have the same `hashCode()`."
True or false?

<details><summary>Key answers</summary>

- Q7: For a 90% read-heavy workload, `ConcurrentHashMap` is usually the
  right default — its read path is largely lock-free. A specialized cache
  library like Caffeine is worth it if you also need eviction (size/TTL),
  which raw `ConcurrentHashMap` doesn't provide.
- Q8: Check the key class's `equals()`/`hashCode()` implementation first —
  this is the single most common cause of a "silently never hits" cache.
- Q9: False — collisions are expected and allowed; only the reverse
  (equal objects must have equal hash codes) is required by the contract.

</details>

---

## 4.7 MASTERY CHECKPOINTS

- **Knowledge Check:** What two conditions must both be true before a bucket treeifies?
- **Coding Check:** Implement a simple LRU cache using `LinkedHashMap` by overriding `removeEldestEntry`.
- **Explanation Check:** Explain in English why `Vector` is now considered legacy despite being thread-safe.
- **Real-World Check:** Your `HashMap<String, BigOrderObject>` is initialized with default capacity but will hold ~1,000,000 entries. What's the perf issue and the fix?
- **Senior Check:** When would you choose `TreeMap` over `HashMap` despite the O(log n) cost?
- **Master Check:** Design a thread-safe, bounded, LRU cache from scratch without a third-party library — what data structures and synchronization would you use, and why?

<details><summary>Answers</summary>

- Real-World Check: Repeated resizes (each O(n) rehash) as the map grows
  from default capacity (16) to ~1M entries; fix by constructing with an
  appropriate initial capacity (accounting for load factor) to avoid
  resize churn.
- Senior Check: When you need sorted-key iteration or range queries
  (`headMap`/`tailMap`/`ceilingKey`), which `HashMap` cannot provide at any cost.
- Master Check: A `LinkedHashMap` in access-order mode with
  `removeEldestEntry` override gives LRU eviction naturally; wrap all access
  in `synchronized` blocks (or a `ReentrantLock`) since `LinkedHashMap`
  itself isn't thread-safe — this is essentially what libraries like
  Caffeine do internally, with far more optimization.

</details>

---

## 4.8 CHEAT SHEET

| Structure | Avg lookup | Ordered? | Thread-safe? |
|---|---|---|---|
| `ArrayList` | O(1) index / O(n) search | Insertion | No |
| `LinkedList` | O(n) | Insertion | No |
| `HashMap` | O(1) | No | No |
| `LinkedHashMap` | O(1) | Insertion/access | No |
| `TreeMap` | O(log n) | Sorted | No |
| `ConcurrentHashMap` | O(1) | No | Yes |
| `HashSet`/`TreeSet` | O(1)/O(log n) | No/Sorted | No |

---

*(Continues to Chapter 5 — Generics and Type Safety.)*
