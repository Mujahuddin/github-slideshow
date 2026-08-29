# CHAPTER 6 — PREFIX SUM & INTERVALS

---

## 6.1 WHY THIS CHAPTER EXISTS

**Prefix Sum** repeated range-sum queries ని O(n) నుండి O(1) per query కి
తగ్గిస్తుంది — ఒక్కసారి O(n) preprocessing చేస్తే సరిపోతుంది. **Intervals**
patterns (merge, scheduling) meetings, bookings, resource-allocation
లాంటి చాలా real-world enterprise problems కి directly map అవుతాయి.

---

## 6.2 CORE PROBLEM 1 — RANGE SUM QUERY (PREFIX SUM FUNDAMENTALS)

### PROBLEM
ఒక array మీద **multiple** `sumRange(i, j)` queries రావొచ్చు — ప్రతి
query ని fast గా answer చేయాలి.

### TELUGU EXPLANATION — BRUTE FORCE

ప్రతి query కి, `i` నుండి `j` వరకు sum చేయడం — per query O(n), `q` queries
కి total O(n·q).

### TELUGU EXPLANATION — OPTIMIZATION (PREFIX SUM ARRAY)

**కీలక insight:** ఒక్కసారి `prefix[i] = nums[0] + nums[1] + ... +
nums[i-1]` అనే array build చేస్తే (`prefix[0] = 0`), ఏ range sum అయినా
**O(1)** లో compute చేయవచ్చు: `sumRange(i, j) = prefix[j+1] - prefix[i]`.

```java
class NumArray {
    private final int[] prefix;

    NumArray(int[] nums) {
        prefix = new int[nums.length + 1]; // prefix[0] = 0 అనే sentinel తో
        for (int i = 0; i < nums.length; i++) {
            prefix[i + 1] = prefix[i] + nums[i];
        }
    }

    // O(1) per query, తర్వాత ఎన్ని queries వచ్చినా
    int sumRange(int i, int j) {
        return prefix[j + 1] - prefix[i];
    }
}
```

**ఎందుకు ఇది పని చేస్తుంది:** `prefix[j+1]` = index 0 నుండి j వరకు sum.
`prefix[i]` = index 0 నుండి i-1 వరకు sum. రెండింటి తేడా = సరిగ్గా index
i నుండి j వరకు sum — మధ్యలో overlap అయిన భాగం cancel అయిపోతుంది.

### ENGLISH INTERVIEW ANSWER

"When there are many range-sum queries against a static array, recomputing
each sum from scratch is wasteful — O(n) per query. Precomputing a prefix
sum array once, in O(n), lets every subsequent query answer in O(1) by
subtracting two prefix values. This is the single most important trade-off
insight in this chapter: pay O(n) once upfront to make every future query
O(1), which is the right call whenever queries significantly outnumber
updates — this exact trade-off, incidentally, is the reasoning behind
denormalizing/precomputing aggregates in database design too."

---

## 6.3 CORE PROBLEM 2 — SUBARRAY SUM EQUALS K (PREFIX SUM + HASHMAP)

### PROBLEM
ఒక array లో, sum గా exactly `k` వచ్చే **contiguous subarrays సంఖ్య**
కనుక్కోండి (Chapter 2 లో preview చేసిన problem, ఇక్కడ పూర్తిగా solve
చేద్దాం).

### TELUGU EXPLANATION

**కీలక insight:** `sum(i, j) = prefix[j] - prefix[i-1]`. మనకి కావాల్సింది
`prefix[j] - prefix[i-1] = k`, అంటే `prefix[i-1] = prefix[j] - k`.
కాబట్టి, array ని ఒక్కసారి traverse చేస్తూ, ప్రతి position దగ్గర
running prefix sum compute చేసి, "**ఇప్పటివరకు `prefix[j] - k` value
ఎన్నిసార్లు కనిపించింది?**" అని ఒక HashMap లో O(1) లో అడగవచ్చు — ఇది
Chapter 2 (HashMap) + ఈ chapter (Prefix Sum) కలయిక.

```java
// O(n) time, O(n) space
int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1); // ఖాళీ prefix (sum=0) ఒక్కసారి "కనిపించింది" — subarray నుండే k మొదలైతే handle చేయడానికి
    int runningSum = 0, count = 0;

    for (int num : nums) {
        runningSum += num;
        count += prefixCount.getOrDefault(runningSum - k, 0);
        prefixCount.merge(runningSum, 1, Integer::sum);
    }
    return count;
}
```

**DRY RUN:** `nums = [1, 1, 1]`, `k = 2`

| num | runningSum | look for (sum-k) | found count | total count | map after |
|---|---|---|---|---|---|
| 1 | 1 | 1-2=-1 | 0 | 0 | {0:1, 1:1} |
| 1 | 2 | 2-2=0 | 1 (from initial {0:1}) | 1 | {0:1,1:1,2:1} |
| 1 | 3 | 3-2=1 | 1 (from {1:1}) | 2 | {0:1,1:1,2:1,3:1} |

**Answer: 2** (subarrays `[1,1]` at index 0-1, and `[1,1]` at index 1-2).

### ENGLISH INTERVIEW ANSWER

"This combines two chapters directly: the prefix sum insight that any
subarray sum is a difference of two prefix sums, and the HashMap
complement-lookup pattern from Chapter 2. Instead of precomputing a full
prefix array and checking all pairs — still O(n²) — I maintain a running
sum and, at each step, ask how many earlier prefix sums equal
`currentSum - k`, which is exactly the count of subarrays ending here that
sum to k. Seeding the map with `{0: 1}` is essential — it correctly counts
subarrays that start from index 0 itself, representing 'the empty prefix'
as having occurred once before any elements were processed."

---

## 6.4 CORE PROBLEM 3 — MERGE INTERVALS

### PROBLEM
Intervals list ఇచ్చినప్పుడు, overlapping ones ని merge చేయండి.

### TELUGU EXPLANATION

**కీలక insight:** Intervals ని **start time ప్రకారం sort** చేస్తే,
overlapping intervals ఎప్పుడూ **adjacent** గా వస్తాయి — దీనివల్ల ఒక్క
linear scan లో merge చేయవచ్చు. Two intervals `[a,b]` మరియు `[c,d]`
(sorted, so `a <= c`) overlap అవుతాయి **అప్పుడే** `c <= b` అయినప్పుడు.

```java
// O(n log n) time (dominated by sort), O(n) space
int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0])); // start time ప్రకారం sort

    List<int[]> merged = new ArrayList<>();
    for (int[] interval : intervals) {
        if (merged.isEmpty() || merged.get(merged.size() - 1)[1] < interval[0]) {
            merged.add(interval); // overlap లేదు — కొత్త interval గా add చేయండి
        } else {
            // overlap ఉంది — చివరి merged interval యొక్క end ని extend చేయండి
            merged.get(merged.size() - 1)[1] =
                    Math.max(merged.get(merged.size() - 1)[1], interval[1]);
        }
    }
    return merged.toArray(new int[merged.size()][]);
}
```

### ENGLISH INTERVIEW ANSWER

"Sorting by start time is what makes this a simple linear scan instead of
an all-pairs comparison. Once sorted, I only ever need to compare the
current interval against the *last merged* interval — if it starts before
the last merged interval ends, they overlap and I extend the end (taking
the max, since the current interval's end might be entirely contained
within the previous one); otherwise it's a genuinely new, non-overlapping
interval. The overall complexity is dominated by the sort, O(n log n) —
the merge pass itself is a single O(n) scan."

**Interviewer follow-up:** "Insert a new interval into an already-sorted,
non-overlapping list" — a variant that can skip the sort entirely since the
list is already ordered, achieving O(n) instead of O(n log n).

---

## 6.5 CORE PROBLEM 4 — MEETING ROOMS II (MINIMUM ROOMS NEEDED)

### PROBLEM
Meeting intervals ఇచ్చినప్పుడు, ఏకకాలంలో అన్ని meetings accommodate
చేయడానికి **కనీసం ఎన్ని rooms** కావాలో కనుక్కోండి.

### TELUGU EXPLANATION

**కీలక insight:** ఇది "**ఏ సమయంలో అయినా, గరిష్టంగా ఎన్ని meetings
ఏకకాలంలో overlap అవుతున్నాయి?**" అనే ప్రశ్న. దీన్ని **start times మరియు
end times ని వేర్వేరుగా sort** చేసి, రెండు pointers తో answer చేయవచ్చు —
ఇది Two Pointers (Chapter 5) యొక్క మరో application:

```java
// O(n log n) time, O(n) space
int minMeetingRooms(int[][] intervals) {
    int n = intervals.length;
    int[] starts = new int[n], ends = new int[n];
    for (int i = 0; i < n; i++) {
        starts[i] = intervals[i][0];
        ends[i] = intervals[i][1];
    }
    Arrays.sort(starts);
    Arrays.sort(ends);

    int rooms = 0, maxRooms = 0;
    int startPtr = 0, endPtr = 0;
    while (startPtr < n) {
        if (starts[startPtr] < ends[endPtr]) {
            rooms++;        // ఒక meeting మొదలైంది, ఇంకా ముగియలేదు
            startPtr++;
        } else {
            rooms--;        // ఒక meeting ముగిసింది
            endPtr++;
        }
        maxRooms = Math.max(maxRooms, rooms);
    }
    return maxRooms;
}
```

**Alternative (heap-based, also common):** sort by start time, use a
min-heap of end times — for each new meeting, if the earliest-ending
meeting in the heap has already ended, reuse that room (pop it); otherwise
allocate a new room (heap size grows) — heap size at the end (or its max
during the process) is the answer. This is a direct bridge to Chapter 8.

### ENGLISH INTERVIEW ANSWER

"The question is really 'what's the maximum number of meetings active at
the same instant,' which is a classic interval-overlap-counting problem. My
two-pointer approach separates all start times and all end times into their
own sorted arrays, then sweeps through time: whenever the next event is a
start, room usage goes up; whenever it's an end, room usage goes down —
tracking the running maximum gives the answer. I'd also mention the
equivalent min-heap formulation, which is arguably more intuitive to
explain out loud: track currently-occupied rooms by their end times in a
min-heap, and reuse a room the moment its meeting has ended by the time the
next one starts."

---

## 6.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Many range-sum queries on a static array | Recompute sum per query, O(n) each | Precompute prefix sums once, O(1) per query |
| "Number of subarrays summing to K" | Brute-force all subarrays, O(n²)/O(n³) | Prefix sum + HashMap complement lookup, O(n) |
| Overlapping intervals | Compare every pair, O(n²) | Sort by start time first — overlaps become adjacent, O(n log n) |
| "Max concurrent X at any time" | Simulate minute-by-minute | Sweep-line over sorted start/end events |

---

## 6.7 COMMON MISTAKES

1. Forgetting the `prefix[0] = 0` sentinel, causing off-by-one errors in
   range-sum formulas.
2. In Subarray Sum Equals K, forgetting to seed the map with `{0: 1}`,
   silently undercounting subarrays that start at index 0.
3. Merging intervals without sorting first — the "compare only to the last
   merged interval" trick is *only* valid after sorting by start time.
4. In Meeting Rooms, comparing `starts[i] <= ends[j]` instead of `<` (or
   vice versa) — getting the boundary condition (does a meeting ending at
   time T free the room in time for one starting at T?) wrong for the
   specific problem's stated semantics.
5. Not recognizing when prefix sum does NOT apply — updates to the array
   interleaved with range queries need a different structure entirely
   (a Fenwick Tree / Binary Indexed Tree, or a Segment Tree — advanced/
   specialized topics beyond this book's core scope, but worth naming in a
   senior interview as "the right tool once updates are frequent").

---

## 6.8 INTERVIEW QUESTION BANK — CHAPTER 6

**Basic:** 1. Why does a prefix sum array turn O(n) range queries into
O(1)? 2. Why must intervals be sorted before merging?

**Intermediate:** 3. Explain the seeded `{0: 1}` entry in Subarray Sum
Equals K — what would break without it? 4. Walk through the sweep-line
two-pointer solution for Meeting Rooms II with a concrete example.

**Senior:** 5. Your array now supports both range-sum queries AND
point updates (`update(i, newValue)`). Why does the simple prefix sum
array break down, and what structure would you reach for instead? 6.
Design "Employee Free Time" (given each employee's busy intervals, find
common free time across all employees) using the interval-merging pattern.

**Architect:** 7. You're designing a resource-booking system (conference
rooms, cloud instances) that must answer "is this time slot available"
queries at high volume, with bookings added/removed frequently. How does
this differ from the static-array prefix sum / interval-merge problems in
this chapter, and what production data structure/approach would you
actually use (interval trees, database range queries with indexes)?

**Scenario:** 8. A candidate solves Subarray Sum Equals K by building a
full prefix sum array first, then checking all O(n²) pairs of prefix
values. What's the complexity, and how does the HashMap approach improve it?

**Trick:** 9. "Prefix sums only work for sum queries, not for other
aggregations like min/max." True or false, and why?

<details><summary>Key answers</summary>

- Q5: Point updates invalidate every prefix value from the updated index
  onward, so a naive prefix array requires O(n) to fix up after each
  update, defeating the purpose; a Fenwick Tree (Binary Indexed Tree) or
  Segment Tree supports both point updates and range queries in O(log n)
  each — worth naming even without full implementation depth in an interview.
- Q6: Merge all employees' busy intervals together (Merge Intervals
  pattern), sort, then the gaps *between* consecutive merged intervals
  (within the overall time bounds) are the common free time slots.
- Q7: Real systems favor database-level range queries with proper indexing
  (e.g., a range/interval index, or an interval tree in memory for
  high-frequency in-process checks) specifically because bookings are
  highly dynamic (frequent inserts/deletes), unlike this chapter's
  static-array assumptions — a full prefix-sum/sorted-merge recompute per
  booking change would be far too slow at scale.
- Q8: Building the full array is O(n), but checking all O(n²) pairs of
  prefix values for a difference of exactly k is O(n²) — the HashMap
  approach folds the "have I seen `prefix[j] - k` before" check into the
  same single O(n) pass that computes the running sum, eliminating the
  need for a second nested loop entirely.
- Q9: False in principle for min/max specifically — a "prefix min/max"
  array works fine for range min/max queries over a **prefix from index
  0**, but general **arbitrary-range** min/max queries (not anchored at
  index 0) can't be derived by simple subtraction the way sums can (min/max
  isn't invertible), which is exactly why range-min/max query problems
  typically use a Sparse Table or Segment Tree instead of a plain prefix array.

</details>

---

## 6.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Derive the formula `sumRange(i, j) = prefix[j+1] - prefix[i]` from first principles — what does each term represent?
- **Coding Check:** Implement "Insert Interval" into an already-sorted, non-overlapping interval list in O(n) without re-sorting.
- **Explanation Check:** Explain in English why prefix sums don't directly generalize to arbitrary-range min/max queries the way they do for sum queries.
- **Real-World Check:** A billing system needs to answer "total usage between any two dates" for a customer's daily usage log, queried very frequently but updated only once per day (a daily batch append). Design the data structure using this chapter's pattern.
- **Senior Check:** When would you use the two-pointer sweep-line approach for Meeting Rooms II versus the min-heap approach — is there a meaningful difference, or is it purely style?
- **Master Check:** Design "Range Sum Query — Mutable" (supports both `update(i, val)` and `sumRange(i, j)` efficiently) using a Fenwick Tree or Segment Tree conceptually — you don't need to implement the full tree, but explain why O(log n) is achievable for both operations where a plain prefix sum array cannot achieve better than O(n) for updates.

<details><summary>Answers</summary>

- Real-World Check: A prefix-sum array over the daily usage log, rebuilt
  incrementally (append the new day's cumulative sum to the existing
  array) rather than recomputed from scratch each day — since updates are
  append-only and infrequent (once daily) while queries are frequent, this
  matches the "pay for preprocessing once, query O(1) many times"
  trade-off perfectly.
- Senior Check: Largely equivalent in complexity (both O(n log n)); the
  two-pointer sweep is slightly more memory-lean (two primitive int
  arrays vs. a heap of objects/boxed integers) and can be marginally
  faster in practice, while the min-heap version is often considered more
  intuitive to explain and extend (e.g., to also report *which* room each
  meeting uses) — a legitimate "either is correct, pick based on what you
  need to extend it to" senior answer.
- Master Check: A Segment Tree (or Fenwick Tree for sums specifically)
  represents the array as a tree where each node aggregates a range;
  updating one leaf only requires updating O(log n) ancestor nodes (not
  the whole array), and a range query only visits O(log n) nodes that
  together cover the requested range — trading the plain array's O(1)
  query / O(n) update for a balanced O(log n) / O(log n), which is the
  right trade-off once updates are frequent enough that O(n) per update is
  unacceptable.

</details>

---

## 6.10 CHEAT SHEET

| Problem shape | Pattern | Complexity |
|---|---|---|
| Many range-sum queries, static array | Prefix sum array | O(n) build, O(1) per query |
| "Subarray summing to K" (count/existence) | Prefix sum + HashMap | O(n) |
| Overlapping intervals to merge | Sort by start, linear merge | O(n log n) |
| Max concurrent intervals at any time | Sweep-line (sorted starts/ends) or min-heap of end times | O(n log n) |
| Frequent updates + range queries needed | Fenwick Tree / Segment Tree (advanced) | O(log n) per op |

---

*(Continues to Chapter 7 — Binary Search & Sorting.)*
