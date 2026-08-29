# CHAPTER 7 — BINARY SEARCH & SORTING

---

## 7.1 WHY THIS CHAPTER EXISTS

Binary Search అనేది "sorted structure మీద O(log n) లో search చేయడం"
అనే మామూలు నిర్వచనానికి మించి, **"Binary Search on the Answer"** అనే చాలా
శక్తివంతమైన meta-pattern కి పునాది — ఇది "ఒక condition true/false గా
ఉండే **monotonic** search space ఏదైనా ఉంటే, direct array కాకపోయినా,
binary search వర్తించవచ్చు" అనే idea.

---

## 7.2 CORE PROBLEM 1 — CLASSIC BINARY SEARCH (AND ITS OFF-BY-ONE TRAPS)

### TELUGU EXPLANATION

```java
// O(log n) time, O(1) space
int binarySearch(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {                 // <= ముఖ్యం — == అయినప్పుడు కూడా check చేయాలి
        int mid = left + (right - left) / 2; // (left+right)/2 కాదు — integer overflow avoid చేయడానికి
        if (nums[mid] == target) return mid;
        else if (nums[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

**రెండు classic bugs, ఇంటర్వ్యూలో explicitly అడిగేవి:**
1. `int mid = (left + right) / 2;` — `left` మరియు `right` రెండూ చాలా
   పెద్దవి అయితే (`Integer.MAX_VALUE` దగ్గర), వాటి sum **overflow**
   అవుతుంది. `left + (right - left) / 2` దీన్ని avoid చేస్తుంది.
2. `while (left < right)` vs `while (left <= right)` — ఇది exact target
   వెతుకుతున్నప్పుడు `<=` కావాలి (`left == right` అయినప్పుడు కూడా ఆ
   ఒక్క element check చేయాలి); కానీ "**boundary**" వెతుకుతున్నప్పుడు
   (e.g., "first element ≥ target") pattern మారుతుంది (7.4 చూడండి).

### ENGLISH INTERVIEW ANSWER

"The core idea is halving the search space each iteration by comparing the
middle element to the target, giving O(log n). Two details separate a
correct implementation from a buggy one that still 'looks right' for most
test cases: computing mid as `left + (right - left) / 2` instead of
`(left + right) / 2` avoids integer overflow when both bounds are large,
and using `left <= right` as the loop condition, not `left < right`,
ensures the single-element case is actually checked rather than skipped."

---

## 7.3 CORE PROBLEM 2 — SEARCH IN ROTATED SORTED ARRAY

### PROBLEM
ఒక sorted array ని ఒక unknown pivot వద్ద rotate చేశారు (ఉదా:
`[4,5,6,7,0,1,2]`). Target ని O(log n) లో వెతకండి.

### TELUGU EXPLANATION

**కీలక insight:** Array మొత్తం sorted కాకపోయినా, **ఏదో ఒక half ఎప్పుడూ
sorted గా ఉంటుంది**. ప్రతి step లో, ఏ half sorted గా ఉందో గుర్తించి,
target ఆ sorted half లో ఉందా అని check చేయండి (normal binary search
range check వాడి), లేకపోతే మరో half లో వెతకండి.

```java
// O(log n) time, O(1) space
int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return mid;

        if (nums[left] <= nums[mid]) { // ఎడమ half sorted గా ఉంది
            if (nums[left] <= target && target < nums[mid]) {
                right = mid - 1; // target ఎడమ half లో ఉంది
            } else {
                left = mid + 1;
            }
        } else { // కుడి half sorted గా ఉంది
            if (nums[mid] < target && target <= nums[right]) {
                left = mid + 1; // target కుడి half లో ఉంది
            } else {
                right = mid - 1;
            }
        }
    }
    return -1;
}
```

### ENGLISH INTERVIEW ANSWER

"A rotated sorted array isn't globally sorted, but at every midpoint, at
least one of the two halves is guaranteed to be properly sorted — I check
`nums[left] <= nums[mid]` to determine which. Once I know which half is
sorted, a simple range check tells me whether the target could be in that
sorted half; if it's in range, I search there, otherwise I know it must be
in the other (possibly still-rotated) half. This preserves the O(log n)
halving property despite the rotation, because I always discard exactly
half the remaining space each iteration, just with an extra decision step
about which half to trust."

---

## 7.4 CORE PROBLEM 3 — BINARY SEARCH ON THE ANSWER (THE META-PATTERN)

### PROBLEM
"Koko Eating Bananas" style: Koko కి `n` banana piles ఉన్నాయి, `h` గంటల్లో
అన్నీ తినేయాలి. ప్రతి గంటకి ఒక pile ఎంచుకుని, గంటకి `k` bananas తినగలదు
(pile అంతకంటే తక్కువ ఉంటే, ఆ pile మొత్తం తినేసి ఆ గంట waste అవుతుంది).
**కనీస `k`** (bananas per hour) ఏమిటో కనుక్కోండి, h గంటల్లో అన్నీ
అయిపోయేలా.

### TELUGU EXPLANATION

**కీలక insight:** ఇది array మీద binary search **కాదు** — ఇది **possible
answers (k values) మీద binary search**. ఎందుకు ఇది పని చేస్తుంది?
ఎందుకంటే "k speed తో h గంటల్లో పూర్తి చేయగలదా?" అనే function **monotonic**
— `k` పెరిగేకొద్దీ, అవసరమైన గంటలు తగ్గుతూనే ఉంటాయి (ఎప్పుడూ పెరగవు).
ఈ monotonicity ఉన్నప్పుడు, "సాధ్యమయ్యే కనిష్ట/గరిష్ట విలువ" కోసం binary
search వర్తిస్తుంది — ఇది Array indices మీద కాకుండా, **answer space**
మీద binary search చేసే **meta-pattern**.

```java
// O(n log m) time, where n = number of piles, m = max pile size (search space size)
int minEatingSpeed(int[] piles, int h) {
    int left = 1, right = Arrays.stream(piles).max().getAsInt();

    while (left < right) { // "boundary" search — first k that works
        int mid = left + (right - left) / 2;
        if (canFinish(piles, mid, h)) {
            right = mid; // mid పని చేస్తే, ఇంకా చిన్న k పని చేస్తుందేమో చూడండి
        } else {
            left = mid + 1; // mid సరిపోకపోతే, పెద్ద k కావాలి
        }
    }
    return left; // left == right, ఇదే కనీస పని చేసే speed
}

boolean canFinish(int[] piles, int speed, int h) {
    int hoursNeeded = 0;
    for (int pile : piles) {
        hoursNeeded += (pile + speed - 1) / speed; // ceiling division
    }
    return hoursNeeded <= h;
}
```

**Note on the loop pattern:** ఇక్కడ `while (left < right)` (కాదు `<=`) —
ఎందుకంటే ఇది "**first value that satisfies the condition**" వెతుకుతోంది
(a boundary), exact match కాదు — ఇది section 7.2 లో ప్రస్తావించిన రెండో
pattern.

### ENGLISH INTERVIEW ANSWER

"The insight that unlocks this whole category of problems is recognizing
that you're not searching an array at all — you're searching the space of
*possible answers* for the smallest (or largest) one satisfying a
condition, and that condition needs to be monotonic in the answer variable
for binary search to be valid. Here, 'can Koko finish within h hours at
speed k' becomes strictly easier as k increases, so I binary search over
possible k values, using a helper function to check feasibility at each
candidate speed. This 'binary search on the answer' pattern shows up
constantly once you know to look for it — capacity-to-ship-packages,
minimum-max-distance problems, and many optimization problems with a
monotonic feasibility check all reduce to this same template."

---

## 7.5 SORTING — WHAT A SENIOR ENGINEER ACTUALLY NEEDS TO KNOW

### TELUGU EXPLANATION

Interview లో sorting algorithms ని **implement** చేయమని అడగరు (production
లో ఎప్పుడూ `Arrays.sort()`/`Collections.sort()` వాడతారు) — కానీ
**వాటి guarantees ఎందుకు ముఖ్యమో** అర్థం చేసుకోవాలి:

| Algorithm | Time (avg) | Time (worst) | Space | Stable? | Java use |
|---|---|---|---|---|---|
| Quicksort | O(n log n) | O(n²) | O(log n) | No | `Arrays.sort()` on **primitives** (dual-pivot quicksort) |
| Mergesort | O(n log n) | O(n log n) | O(n) | Yes | `Arrays.sort()`/`Collections.sort()` on **objects** (TimSort, a merge-sort variant) |
| Heapsort | O(n log n) | O(n log n) | O(1) | No | Rarely used directly; heap itself is used constantly (Ch. 8) |

**ఎందుకు Java రెండు వేర్వేరు algorithms వాడుతుంది** (primitives కి
quicksort-based, objects కి mergesort-based)? **Stability** — objects
sort చేసేటప్పుడు, "equal" గా compare అయిన elements వాటి **original
relative order** ని maintain చేయాలి (ఉదా: employees ని department ప్రకారం
sort చేశాక, ఇంకా name ప్రకారం sort చేస్తే, ఒకే department వాళ్ళు ఇంకా
name-sorted గానే ఉండాలి — **multi-key sorting** కి stability అవసరం).
Primitives కి "identity" అనే concept లేదు (రెండు `5`లు ఎప్పుడూ
"identical"), కాబట్టి stability అవసరం లేదు — Java non-stable కానీ faster
dual-pivot quicksort వాడుతుంది.

### ENGLISH INTERVIEW ANSWER

"I rarely implement sorting from scratch in production — `Arrays.sort()`
and `Collections.sort()` are well-tested and highly optimized. What matters
is knowing *why* Java uses different algorithms for primitives versus
objects: primitives are sorted with a dual-pivot quicksort, which is fast
but not stable, and that's fine because primitive values have no identity
to preserve. Objects are sorted with a mergesort variant, TimSort, which is
stable — critical for multi-key sorting, like sorting by department and
then, within each department, needing the previous name-based order to
survive. This is exactly the kind of 'why does the API work this way'
question that separates someone who's memorized `Arrays.sort()` from
someone who understands it."

**Custom comparators (a real, frequently-tested skill):**

```java
// Sort employees by department, then by salary descending within department
employees.sort(
    Comparator.comparing(Employee::getDepartment)
              .thenComparing(Employee::getSalary, Comparator.reverseOrder())
);
```

---

## 7.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Sees "minimum/maximum X such that condition holds" | Tries to brute-force every X | Checks if the condition is monotonic in X — if so, binary search on the answer |
| Computing `mid` | `(left + right) / 2` | `left + (right - left) / 2` (overflow-safe) |
| Multi-key sorting | Sorts by one key, then re-sorts by another (loses first ordering) | Uses `Comparator.thenComparing()`, relies on stable sort semantics |
| Binary search loop bound | Always uses `<=` regardless of what's being searched | Distinguishes "exact match" (`<=`) from "boundary/first-true" (`<`) search templates |

---

## 7.7 COMMON MISTAKES

1. Integer overflow from `(left + right) / 2` on large inputs.
2. Mixing up the "exact match" and "find boundary" binary search loop
   templates, causing infinite loops or missed answers.
3. Not verifying that a "binary search on the answer" candidate problem
   actually has a **monotonic** feasibility condition before applying the
   pattern — applying it to a non-monotonic condition silently gives wrong answers.
4. Assuming a custom `Comparator` needs to return exactly -1/0/1 — any
   negative/zero/positive value is valid, but consistency (a well-defined
   total order) matters, or `Collections.sort` can throw
   `IllegalArgumentException: Comparison method violates its general contract!`
5. Forgetting that `Arrays.sort()` on an `Object[]`/generic array uses a
   different (stable) algorithm than `Arrays.sort()` on a primitive array.

---

## 7.8 INTERVIEW QUESTION BANK — CHAPTER 7

**Basic:** 1. What's the time complexity of binary search, and why? 2. Why
is `Arrays.sort()` stable for objects but not for primitives?

**Intermediate:** 3. Explain how binary search still works on a rotated
sorted array despite it not being fully sorted. 4. What makes a problem a
candidate for "binary search on the answer"?

**Senior:** 5. Given a function `isFeasible(x)` that's monotonic (false...false,
true...true, with one crossover point), design the exact binary search
template to find the crossover — walk through the loop invariant. 6. Why
would `Comparison method violates its general contract!` occur, and how do
you fix a custom comparator that triggers it?

**Architect:** 7. You're designing a capacity-planning tool that needs to
answer "what's the minimum number of servers needed to handle this
workload within an SLA" — recognize this as binary-search-on-the-answer
and describe the feasibility check function's shape and complexity budget.

**Scenario:** 8. A candidate's rotated-array binary search doesn't handle
duplicate values correctly (e.g., `[1,1,1,0,1]`), infinite-looping or
giving wrong answers. What's the root cause, and what extra check is needed?

**Trick:** 9. "Binary search only works on arrays." True or false?

<details><summary>Key answers</summary>

- Q6: The comparator isn't defining a valid total order — often from
  comparing via subtraction (`a.getValue() - b.getValue()`) which can
  overflow for large values, or from an inconsistent multi-field comparison
  that isn't transitive; fix by using `Integer.compare()`/`Comparator.comparing()`
  chains instead of manual subtraction, and ensuring transitivity holds.
- Q8: When `nums[left] == nums[mid]` (and possibly `nums[mid] == nums[right]`
  too), you can no longer reliably determine which half is sorted using the
  simple `nums[left] <= nums[mid]` check — the fix is an extra branch: when
  `nums[left] == nums[mid]`, shrink from the left by one (`left++`) instead
  of trying to determine sortedness, since that one element didn't
  eliminate any useful information anyway. This is also why this variant
  degrades to O(n) worst case with many duplicates, worth stating explicitly.
- Q9: False — the array framing is the classic teaching example, but
  binary search applies to any monotonic, ordered/comparable search space —
  "binary search on the answer" over a numeric range (as in Koko Eating
  Bananas) is the clearest proof this isn't array-specific at all.

</details>

---

## 7.9 MASTERY CHECKPOINTS

- **Knowledge Check:** What property must a condition have for "binary search on the answer" to be valid?
- **Coding Check:** Implement "Capacity To Ship Packages Within D Days" using the binary-search-on-the-answer template from section 7.4.
- **Explanation Check:** Explain in English, to someone who only knows classic array binary search, what changes (and what stays the same) when applying binary search to an answer space instead.
- **Real-World Check:** A video streaming service needs to find the minimum bitrate that keeps buffering under a threshold for a given network condition, where higher bitrate monotonically increases buffering risk. Map this to this chapter's meta-pattern.
- **Senior Check:** When would you NOT use binary search on the answer even though a numeric answer range exists — what has to be true about the feasibility check's cost for this to still be worth it?
- **Master Check:** Design a solution for "Split Array Largest Sum" (split an array into m subarrays to minimize the largest subarray sum) using binary search on the answer — specify exactly what the search space and feasibility check are.

<details><summary>Answers</summary>

- Real-World Check: Binary search over candidate bitrates — the
  feasibility check is "does this bitrate keep buffering under the
  threshold," which is monotonic (lower bitrate is always safer), so the
  minimum viable bitrate can be found in O(log(range)) feasibility checks
  instead of testing every bitrate linearly.
- Senior Check: If each feasibility check itself is expensive (e.g., O(n²)
  or requires an expensive simulation), the O(log(range)) multiplier can
  still be worthwhile, but if a direct O(n) or O(n log n) closed-form/greedy
  solution exists without needing to search a range at all, that's simpler
  and should be preferred — binary search on the answer is a fallback
  pattern for when no direct formula is apparent, not an automatic first choice.
- Master Check: Search space is `[max(nums), sum(nums)]` (the largest
  subarray sum must be at least the single biggest element, and at most the
  total sum); feasibility check: "can this array be split into ≤ m
  subarrays such that no subarray sum exceeds this candidate value?" —
  computed greedily in O(n) by accumulating a running sum and starting a
  new subarray whenever adding the next element would exceed the candidate.

</details>

---

## 7.10 CHEAT SHEET

| Problem shape | Pattern | Complexity |
|---|---|---|
| Exact target in sorted array | Classic binary search (`<=` loop) | O(log n) |
| Sorted array, unknown rotation | Determine sorted half each step | O(log n) |
| "Minimum/maximum X such that condition(X)" | Binary search on the answer (`<` loop, boundary search) | O(log(range) × feasibility cost) |
| Multi-key sort, order-preservation needed | Stable sort + `Comparator.thenComparing` | O(n log n) |
| `mid` computation | `left + (right - left) / 2` | Avoids overflow |

---

*(Continues to Chapter 8 — Heap & Priority Queue.)*
