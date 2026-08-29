# CHAPTER 8 — HEAP & PRIORITY QUEUE

---

## 8.1 CONCEPT: Heap Internals — Why O(log n) Insert/Extract

### TELUGU EXPLANATION

**Heap** ఒక **complete binary tree** (array గా store అవుతుంది, pointer-based
tree కాదు), **heap property** తో: min-heap లో ప్రతి parent, దాని
children కంటే **చిన్నది** (లేదా సమానం) గా ఉంటుంది (max-heap లో వ్యతిరేకం).

**Array representation:** index `i` వద్ద element కి, parent index
`(i-1)/2`, children indices `2i+1` మరియు `2i+2` — ఇది **complete tree**
అయినందుకే possible (ఏ gaps లేకుండా, array indices నేరుగా tree structure
ని encode చేస్తాయి, pointer overhead లేకుండా).

**`offer()` (insert):** కొత్త element ని array చివర పెట్టి, అది తన parent
కంటే చిన్నది (min-heap) అయ్యేవరకు, **sift up** (swap with parent)
చేస్తారు — worst case, root వరకు వెళ్ళాలి, tree height `log n` కాబట్టి
**O(log n)**.

**`poll()` (extract min/max):** root ని తీసేసి, **last element** ని
root కి తీసుకువచ్చి, అది తన children కంటే పెద్దది అయ్యేవరకు **sift down**
(swap with smaller child) చేస్తారు — మళ్ళీ **O(log n)** (tree height).

**`peek()`:** root ని చూడటం మాత్రమే — **O(1)**.

### ENGLISH INTERVIEW ANSWER

"A heap is a complete binary tree stored compactly in an array, using
index arithmetic (`2i+1`, `2i+2` for children, `(i-1)/2` for parent)
instead of explicit pointers — that compactness is itself a nice, minor
memory-efficiency win. Insert appends to the end and 'sifts up,' swapping
with the parent while the heap property is violated; extract removes the
root, moves the last element there, and 'sifts down,' swapping with the
smaller (or larger) child while violated. Because a complete binary tree
with n nodes has height O(log n), both operations are O(log n), while
peeking at the min/max is O(1) since it's always the root — this O(log n)
insert/extract with O(1) peek is exactly the profile that makes heaps the
right structure for 'repeatedly need the current min/max while the set
changes' problems."

---

## 8.2 CORE PROBLEM 1 — KTH LARGEST ELEMENT

### PROBLEM
ఒక array లో, `k`-th largest element కనుక్కోండి (sorting లేకుండా, ఆదర్శంగా).

### TELUGU EXPLANATION — BRUTE FORCE

Sort చేసి, `n-k` index చూడటం — O(n log n).

### TELUGU EXPLANATION — OPTIMIZATION (MIN-HEAP OF SIZE K)

**కీలక insight:** మనకి "మొత్తం sorted order" అవసరం లేదు — కేవలం "top k
elements లో అతి చిన్నది ఏమిటి" తెలిస్తే చాలు. ఒక **min-heap size k**
maintain చేయండి: heap size `k` దాటితే, **smallest** element ని pop
చేయండి (min-heap root ఎప్పుడూ smallest). చివరికి, heap లో మిగిలిన
smallest element (root) ఇదే **k-th largest** — ఎందుకంటే heap లో ఇప్పుడు
అతిపెద్ద `k` elements మాత్రమే ఉన్నాయి, వాటిలో అతి చిన్నది root.

```java
// O(n log k) time, O(k) space — better than O(n log n) when k << n
int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(); // default: min-heap
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll(); // అతి చిన్నదాన్ని తీసేయండి — top-k లో లేనిది
        }
    }
    return minHeap.peek(); // ఇప్పుడు heap లో ఉన్న అతి చిన్నదే k-th largest
}
```

### ENGLISH INTERVIEW ANSWER

"Sorting the whole array to find one element is wasteful — O(n log n) when
we only need the k-th largest, not full order. Maintaining a min-heap
capped at size k is the key trick: every time the heap exceeds size k, I
evict the smallest element, since it can't possibly be among the k
largest once there are more than k stronger candidates. What survives at
the end is exactly the k largest elements, and the heap's root — the
smallest among them — is precisely the k-th largest. This gives O(n log k),
which is a meaningful improvement over O(n log n) whenever k is small
relative to n, a very common real-world shape (e.g., 'top 10' out of millions)."

**Interviewer follow-up:** "Why a min-heap and not a max-heap here?" — A
max-heap of the *entire* array would also work but requires O(n) space and
gets you the answer via k pops, O(n + k log n); the bounded min-heap of
size k is the superior trade-off specifically because it never holds more
than k elements at once.

---

## 8.3 CORE PROBLEM 2 — TOP K FREQUENT ELEMENTS

### PROBLEM
ఒక array లో, **frequency ప్రకారం top k** elements కనుక్కోండి.

### TELUGU EXPLANATION

ఇది Chapter 2 (HashMap frequency counting) + ఈ chapter (heap) కలయిక —
**రెండు-దశల pattern**, DSA లో చాలా frequently కనిపిస్తుంది:

1. **HashMap** తో frequency count చేయండి — O(n).
2. **Min-heap size k** (frequency ఆధారంగా ordered) వాడి, top-k frequent
   వాటిని ఎంచుకోండి — O(n log k) (8.2 సాంకేతికతే, key కేవలం "frequency" గా మారింది).

```java
// O(n log k) time, O(n) space
int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freqMap = new HashMap<>();
    for (int num : nums) {
        freqMap.merge(num, 1, Integer::sum);
    }

    // Min-heap ordered by frequency — smallest frequency at top, size capped at k
    PriorityQueue<Map.Entry<Integer, Integer>> minHeap =
            new PriorityQueue<>(Comparator.comparingInt(Map.Entry::getValue));

    for (Map.Entry<Integer, Integer> entry : freqMap.entrySet()) {
        minHeap.offer(entry);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
    }

    int[] result = new int[k];
    for (int i = k - 1; i >= 0; i--) { // heap నుండి poll చేసేటప్పుడు, తక్కువ frequency ముందు వస్తుంది
        result[i] = minHeap.poll().getKey();
    }
    return result;
}
```

### ENGLISH INTERVIEW ANSWER

"This is a two-stage pattern I recognize constantly: first reduce the
problem to 'top k by some derived score' — here, frequency, computed via a
HashMap — then apply the same bounded min-heap-of-size-k technique from Kth
Largest Element, just ordered by a custom comparator on frequency instead
of value directly. I fill the result array back-to-front because polling a
min-heap yields ascending order, so the last poll is actually the *most*
frequent element."

**Alternative (bucket sort, O(n)):** since frequency is bounded by array
length, you can use an array of buckets indexed by frequency
(`List<Integer>[] buckets = new List[n+1]`), placing each number in
`buckets[frequency]`, then scanning from the highest-frequency bucket
downward — this avoids the heap's `log k` factor entirely for O(n) overall,
worth mentioning as the "can we do even better" senior follow-up.

---

## 8.4 CONCEPT: WHEN NOT TO REACH FOR A HEAP

### TELUGU EXPLANATION

Heap అనేది **"repeatedly, dynamically మారుతున్న సమూహం నుండి min/max
కావాలి"** అనే పరిస్థితికి సరైనది. కానీ:
- Data **static** గా ఉండి, **ఒక్కసారి మాత్రమే** sort చేయాల్సి వస్తే —
  ఒక్కసారి `Arrays.sort()` చేస్తే సరిపోతుంది, heap అనవసరం (heap build
  చేయడం, repeatedly extract చేయడం కంటే direct sort సాధారణంగా faster
  constant factors తో).
- **Exact k-th element** మాత్రమే కావాలంటే (ఒక్కసారి), **Quickselect**
  (average O(n), Quicksort యొక్క partition idea వాడి) heap కంటే faster
  కావొచ్చు — heap O(n log k), quickselect average O(n). Interview లో
  రెండూ ప్రస్తావించగలగడం senior-level depth చూపిస్తుంది.

### ENGLISH INTERVIEW ANSWER

"A heap earns its keep when the set is changing dynamically and I need
repeated access to the current min/max — a streaming top-k, a scheduler
picking the next task, Dijkstra's algorithm always needing the next-closest
node. If I just need a one-time answer against static data — sort it and
index directly. And even for a one-time 'find the k-th largest,' Quickselect
is actually a better asymptotic choice than a heap on average — O(n)
average time versus the heap's O(n log k) — though it has worse worst-case
behavior and returns the single k-th element rather than the k largest as
an ordered/available set the way a heap naturally maintains them."

---

## 8.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Top K" anything | Sort everything, O(n log n) | Bounded heap of size K, O(n log k) — or bucket sort if score range is bounded |
| One-time k-th element | Reaches for a heap by habit | Considers Quickselect (average O(n)) as a faster one-shot alternative |
| Dynamic "always give me the current min" | Re-scans the whole collection each time | Maintains a heap incrementally |
| Merging K sorted sources | Concatenate + sort everything | Min-heap of size K holding current heads (Chapter 3 preview), O(n log k) |

---

## 8.6 COMMON MISTAKES

1. Using a max-heap of the entire dataset for "top k" instead of a
   bounded min-heap of size k — correct but wastes O(n) space unnecessarily.
2. Forgetting Java's `PriorityQueue` is a **min-heap by default** —
   needing a max-heap requires `new PriorityQueue<>(Comparator.reverseOrder())`.
3. Assuming `PriorityQueue.toArray()`/iteration gives sorted order — it
   does **not**; only repeated `poll()` calls yield elements in heap order.
4. Confusing "heapify" (building a heap from an existing array in O(n),
   not O(n log n) — a genuinely subtle and often-misstated complexity fact)
   with inserting elements one at a time (O(n log n) total).
5. Reaching for a heap when a simpler bucket/counting approach (bounded
   value range) would be O(n) with less code.

---

## 8.7 INTERVIEW QUESTION BANK — CHAPTER 8

**Basic:** 1. Why is heap insert/extract O(log n)? 2. Is Java's
`PriorityQueue` a min-heap or max-heap by default?

**Intermediate:** 3. Why is a bounded min-heap of size k better than a
full max-heap for "top k" problems? 4. Explain the two-stage HashMap +
heap pattern used in Top K Frequent Elements.

**Senior:** 5. Compare Quickselect and heap-based approaches for
finding the k-th largest element — time complexity, space, and when
you'd choose one over the other. 6. Why is heapify O(n), not O(n log n),
when building a heap from an existing array?

**Architect:** 7. Design a system that must always serve "the current top
10 trending topics" from a continuous stream of topic mentions, where
counts change constantly and old data should decay/expire. How does the
bounded-heap pattern extend (or fail to extend) to this streaming,
decaying-count scenario?

**Scenario:** 8. A candidate implements Top K Frequent Elements by sorting
all `(element, frequency)` pairs by frequency descending, then taking the
first k. What's the complexity compared to the heap approach, and when
might the simpler sort-based approach actually be preferred?

**Trick:** 9. "A heap gives you a fully sorted sequence if you keep
calling `poll()`." True or false?

<details><summary>Key answers</summary>

- Q6: Heapify processes nodes bottom-up, sifting each down as needed; most
  nodes are near the bottom of the tree and require very little sift-down
  work (a node at height h does at most O(h) work, and there are
  exponentially more nodes at small heights than large ones) — summing this
  geometric-series-like total work across all nodes yields O(n) overall,
  not the O(n log n) you'd get from naively inserting n elements one at a
  time into an empty heap.
- Q7: The static bounded-heap pattern needs adaptation for decay — a naive
  heap doesn't handle "this count decreases over time" well since heap
  order would go stale; real systems typically use time-windowed counting
  (e.g., counts bucketed per time window, old windows expired/decayed) combined
  with periodic re-aggregation into a fresh top-k heap, rather than trying to
  keep one heap perpetually consistent under decay.
- Q8: Sorting all pairs is O(n log n) — worse than the heap's O(n log k)
  when k is much smaller than n, but simpler to implement/reason about, and
  perfectly reasonable to prefer when n is small, k is close to n anyway
  (so the complexity gap is negligible), or code simplicity outweighs the
  marginal performance difference.
- Q9: True — repeatedly polling a heap yields elements in fully sorted
  order (ascending for a min-heap), which is exactly the mechanism behind
  heapsort; the heap doesn't store elements in sorted order internally, but
  the *sequence of poll results* is sorted.

</details>

---

## 8.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does a min-heap of size k, not a max-heap, correctly solve "find the k largest elements"?
- **Coding Check:** Implement "Merge K Sorted Lists" (Chapter 3 preview) using a min-heap of size k holding each list's current head.
- **Explanation Check:** Explain in English why heapify is O(n) and not O(n log n), in terms a junior engineer who assumes "n inserts = n log n" would find convincing.
- **Real-World Check:** A logging/observability platform needs to show "the 20 slowest API endpoints right now" from continuously arriving latency measurements, refreshed every few seconds. Design the approach using this chapter's patterns.
- **Senior Check:** When would you pick Quickselect over a heap-based solution for a one-time k-th-element query, and what's the risk you're accepting?
- **Master Check:** Design a "Median from a Data Stream" solution (numbers arrive one at a time; report the median at any point) using two heaps — explain what invariant the two heaps maintain and why that gives O(log n) insert and O(1) median retrieval.

<details><summary>Answers</summary>

- Real-World Check: Periodically (every few seconds) recompute a bounded
  min-heap of size 20 over the current window's aggregated per-endpoint
  latency data (using the Top-K-Frequent-style two-stage pattern: aggregate
  first, then bounded heap) — a fresh recompute per window is simpler and
  more correct than trying to maintain one perpetually-updated heap against
  a sliding time window.
- Senior Check: When you need only the single k-th value once, not an
  ordered/maintained top-k set, and average-case performance matters more
  than worst-case guarantees — the risk accepted is Quickselect's O(n²)
  worst case (mitigated in practice with randomized pivot selection, similar
  to quicksort's own worst-case mitigation).
- Master Check: A max-heap holds the smaller half of numbers seen so far,
  a min-heap holds the larger half, kept balanced in size (differing by at
  most 1); the median is either the max-heap's root (odd total count) or
  the average of both roots (even count) — O(1) to retrieve. Insertion
  decides which heap a new number belongs to, then rebalances sizes with at
  most one element moved between heaps, each heap operation O(log n).

</details>

---

## 8.9 CHEAT SHEET

| Problem shape | Pattern | Complexity |
|---|---|---|
| Top K largest/smallest | Bounded heap of size K | O(n log k) |
| Top K by a derived score (frequency, etc.) | HashMap (compute score) + bounded heap | O(n log k) |
| One-time k-th element only | Quickselect (average case) | O(n) average |
| Merge K sorted sources | Min-heap of size K (current heads) | O(n log k) |
| Running median from a stream | Two heaps (max-heap of lower half, min-heap of upper half) | O(log n) insert, O(1) median |
| Java `PriorityQueue` default | Min-heap; use `Comparator.reverseOrder()` for max-heap | — |

---

*(Continues to Chapter 9 — Recursion & Backtracking.)*
