# CHAPTER 1 — ARRAYS & STRINGS FUNDAMENTALS

---

## 1.1 WHY THIS CHAPTER EXISTS

Arrays/Strings అనేవి DSA యొక్క **పునాది** — దాదాపు ప్రతి pattern (Two
Pointers, Sliding Window, Prefix Sum, Sorting-based Greedy) array/string
మీదే build అవుతుంది. ఈ chapter goal: array/string problems ని చూసినప్పుడు
"ఇది brute force O(n²) తో solve అవుతుంది, కానీ ఏ **structure** వాడి O(n)
కి తగ్గించగలను?" అని ఆలోచించడం నేర్పించడం.

---

## 1.2 PATTERN RECOGNITION: WHEN IS IT "JUST ARRAYS/STRINGS"?

### TELUGU EXPLANATION

ఒక problem statement చూసినప్పుడు ఇది ఈ chapter కిందకు వస్తుందా అని
గుర్తించడానికి సూచనలు:
- Input ఒక `int[]` లేదా `String`/`char[]`.
- Operations: traverse, compare adjacent elements, count frequency, in-place modify.
- "ప్రత్యేక" structure (sorted, graph, tree) ప్రస్తావన లేకపోతే.

ఇలాంటి ప్రశ్నలు వేసుకోండి: "నేను ఈ array ని ఒక్కసారి (single pass)
traverse చేస్తే సరిపోతుందా?" "నాకు కావాల్సిన సమాచారం ముందు elements
నుండి maintain చేయగలనా (running sum/max/count)?"

---

## 1.3 CORE PROBLEM 1 — MAXIMUM SUBARRAY SUM (KADANE'S ALGORITHM)

### PROBLEM
ఒక `int[]` ఇచ్చినప్పుడు, **contiguous subarray** యొక్క maximum sum
కనుక్కోండి.

### TELUGU EXPLANATION — BRUTE FORCE

అమాయకంగా ఆలోచిస్తే, ప్రతి possible subarray (i, j) sum compute చేసి,
maximum తీసుకోవచ్చు — **O(n²)** (లేదా prefix sums వాడితే కూడా O(n²)
subarrays ఉంటాయి కాబట్టి, sum compute O(1) అయినా, enumerate చేయడమే O(n²)).

```java
// Brute force — O(n²) time, O(1) space
int maxSubArrayBrute(int[] nums) {
    int maxSum = Integer.MIN_VALUE;
    for (int i = 0; i < nums.length; i++) {
        int sum = 0;
        for (int j = i; j < nums.length; j++) {
            sum += nums[j];
            maxSum = Math.max(maxSum, sum);
        }
    }
    return maxSum;
}
```

### TELUGU EXPLANATION — OPTIMIZATION (KADANE'S ALGORITHM)

**కీలక insight:** ఒక position `i` వద్ద, "ఇప్పటివరకు ఉన్న best subarray
ని ఇక్కడ కూడా కొనసాగించాలా, లేక ఇక్కడి నుండి కొత్తగా మొదలుపెట్టాలా?"
అనే **local decision** ప్రతి element దగ్గర తీసుకోవచ్చు:

- ఇప్పటివరకు వచ్చిన running sum (`currentSum`) **negative** అయిపోతే, అది
  future sum ని మాత్రమే తగ్గిస్తుంది — కాబట్టి దాన్ని **వదిలేసి, కొత్తగా
  ప్రస్తుత element నుండి మొదలుపెట్టడం** మంచిది.
- ఇది **Dynamic Programming యొక్క సరళమైన రూపం** — `dp[i] = max(nums[i],
  dp[i-1] + nums[i])` — ఇక్కడ `dp[i]` అంటే "index i తో ముగిసే best
  subarray sum".

```java
// Kadane's Algorithm — O(n) time, O(1) space
int maxSubArray(int[] nums) {
    int maxSum = nums[0];
    int currentSum = nums[0];
    for (int i = 1; i < nums.length; i++) {
        // ఇప్పటివరకు ఉన్నదాన్ని కొనసాగించాలా, కొత్తగా మొదలుపెట్టాలా?
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }
    return maxSum;
}
```

**DRY RUN:** `nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]`

| i | nums[i] | currentSum = max(nums[i], currentSum+nums[i]) | maxSum |
|---|---|---|---|
| 0 | -2 | -2 (init) | -2 |
| 1 | 1 | max(1, -2+1=-1) = 1 | 1 |
| 2 | -3 | max(-3, 1-3=-2) = -2 | 1 |
| 3 | 4 | max(4, -2+4=2) = 4 | 4 |
| 4 | -1 | max(-1, 4-1=3) = 3 | 4 |
| 5 | 2 | max(2, 3+2=5) = 5 | 5 |
| 6 | 1 | max(1, 5+1=6) = 6 | 6 |
| 7 | -5 | max(-5, 6-5=1) = 1 | 6 |
| 8 | 4 | max(4, 1+4=5) = 5 | 6 |

**Answer: 6** (subarray `[4, -1, 2, 1]`).

### ENGLISH INTERVIEW ANSWER

"The brute force checks every subarray, which is O(n²). Kadane's algorithm
recognizes that at each position, the running sum has only two sensible
options: extend the previous subarray, or abandon it and start fresh at the
current element — and it's always correct to abandon when the running sum
has gone negative, since a negative prefix can only hurt any sum that
follows it. This turns the problem into a single O(n) pass with O(1) space,
and it's really a minimal, one-dimensional dynamic programming recurrence in
disguise, where the state is 'the best subarray sum ending exactly at
position i.'"

**Complexity:** Time O(n), Space O(1).

**Variations (interviewer follow-ups):**
- "Return the actual subarray, not just the sum" — track start/end indices
  alongside `currentSum`, resetting the start index whenever you restart.
- "What if the array is all negative?" — the algorithm still works correctly
  (it degrades to picking the single largest, least-negative element),
  since `maxSum` is initialized from `nums[0]`, not 0.
- "Circular array version (subarray can wrap around)?" — a more advanced
  variant: compute both the normal Kadane's max AND (total sum − Kadane's
  *minimum* subarray sum), take the larger, with a special case if all
  elements are negative.

---

## 1.4 CORE PROBLEM 2 — CHARACTER FREQUENCY / ANAGRAM CHECK

### PROBLEM
రెండు strings ఇచ్చినప్పుడు, అవి **anagrams** అవునా కాదా చెప్పండి (ఒకే
characters, ఒకే frequency, వేర్వేరు order).

### TELUGU EXPLANATION — BRUTE FORCE

రెండు strings ని sort చేసి compare చేయవచ్చు — **O(n log n)**.

```java
boolean isAnagramSort(String a, String b) {
    if (a.length() != b.length()) return false;
    char[] ca = a.toCharArray(), cb = b.toCharArray();
    Arrays.sort(ca);
    Arrays.sort(cb);
    return Arrays.equals(ca, cb);
}
```

### TELUGU EXPLANATION — OPTIMIZATION (FIXED-SIZE FREQUENCY ARRAY)

Sorting అవసరం లేదు — కేవలం ప్రతి character యొక్క **frequency count**
compare చేస్తే సరిపోతుంది. Alphabet తెలిసినప్పుడు (e.g., lowercase
English letters, 26 characters), ఒక **fixed-size array** (HashMap కంటే
faster, no hashing overhead) వాడొచ్చు — ఇది Chapter 2 (HashMap patterns)
కి ఒక lightweight పూర్వరూపం.

```java
// O(n) time, O(1) space (fixed 26-size array, not dependent on input size)
boolean isAnagram(String a, String b) {
    if (a.length() != b.length()) return false;
    int[] freq = new int[26];
    for (int i = 0; i < a.length(); i++) {
        freq[a.charAt(i) - 'a']++;  // 'a' నుండి increment
        freq[b.charAt(i) - 'a']--;  // 'b' నుండి decrement
    }
    for (int count : freq) {
        if (count != 0) return false; // ఏ character అయినా mismatch అయితే
    }
    return true;
}
```

**Explanation:** ఒకే array లో రెండు strings యొక్క net frequency track
చేస్తాం — `a` నుండి increment, `b` నుండి decrement. చివరికి అన్ని counts
సున్నా అయితేనే, రెండు strings identical frequency కలిగి ఉన్నాయని అర్థం.

**Complexity:** Time O(n), Space O(1) (26 is a constant, not O(n) — this
distinction matters in interviews: fixed alphabet size ≠ variable input
size).

### ENGLISH INTERVIEW ANSWER

"Sorting both strings and comparing is a valid O(n log n) solution, but
since we know the alphabet is bounded — 26 lowercase letters — a
fixed-size frequency array gets us to O(n) with true O(1) space, since the
array size doesn't grow with input length. I increment counts for the
first string and decrement for the second in the same array; if they're
anagrams, every count nets to exactly zero. This generalizes directly to a
`HashMap<Character, Integer>` the moment the alphabet is unbounded — e.g.,
Unicode — which is exactly the bridge to Chapter 2's HashMap-based
frequency counting pattern."

**Variations:** "What if the strings can contain any Unicode character, not
just lowercase English?" (Switch to `HashMap<Character, Integer>` — the
same net increment/decrement logic, just without the bounded-array
optimization.) "Find all anagram groups in a list of strings" — a preview
of `Collectors.groupingBy` from Book 1 Chapter 7, grouping by a canonical
sorted-string key.

---

## 1.5 CORE PROBLEM 3 — IN-PLACE ARRAY ROTATION

### PROBLEM
ఒక array ని `k` positions right గా rotate చేయండి, **extra array వాడకుండా**
(O(1) space).

### TELUGU EXPLANATION — BRUTE FORCE

ఒక కొత్త array create చేసి, elements ని కొత్త positions కి copy చేయవచ్చు
— O(n) time, **కానీ O(n) extra space**.

### TELUGU EXPLANATION — OPTIMIZATION (REVERSAL ALGORITHM)

**కీలక insight:** Array ని `k` positions right rotate చేయడం అంటే, మూడు
reversal operations ద్వారా achieve చేయవచ్చు, extra space లేకుండా:
1. మొత్తం array ని reverse చేయండి.
2. మొదటి `k` elements ని reverse చేయండి.
3. మిగతా `n-k` elements ని reverse చేయండి.

```java
// O(n) time, O(1) space — in-place via three reversals
void rotate(int[] nums, int k) {
    int n = nums.length;
    k = k % n; // k > n అయితే handle చేయడానికి
    reverse(nums, 0, n - 1);
    reverse(nums, 0, k - 1);
    reverse(nums, k, n - 1);
}

void reverse(int[] nums, int left, int right) {
    while (left < right) {
        int temp = nums[left];
        nums[left] = nums[right];
        nums[right] = temp;
        left++;
        right--;
    }
}
```

**DRY RUN:** `nums = [1,2,3,4,5,6,7]`, `k = 3`
- Step 1 (reverse all): `[7,6,5,4,3,2,1]`
- Step 2 (reverse first k=3): `[5,6,7,4,3,2,1]`
- Step 3 (reverse remaining n-k=4): `[5,6,7,1,2,3,4]`
- Result: `[5,6,7,1,2,3,4]` ✅ (correct right-rotation by 3)

### ENGLISH INTERVIEW ANSWER

"Rotating with a new array is trivially correct but uses O(n) extra space.
The reversal-based trick achieves the same result in-place: reversing the
whole array flips both the intended final segments into roughly the right
relative order but each segment internally reversed, so reversing each of
the two segments again fixes their internal order while preserving the
overall rotation. This is a good example of how a seemingly unrelated
operation — reversal — composes into a completely different result through
clever sequencing, and it's a common senior-level 'derive it live' question
precisely because the trick isn't obvious without working through a
concrete example first."

**Variations:** "Rotate left instead of right" (rotating left by k is
equivalent to rotating right by n-k). "What if k is much larger than n?"
(the `k = k % n` line handles this — always reduce k modulo array length first).

---

## 1.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Find X in array" | Nested loop, O(n²), ship it | Ask: is it sorted? Bounded values? Can I trade space for time (hashing) or exploit structure (two pointers, binary search)? |
| "In-place" requirement in problem statement | Ignored, allocates a new array anyway | Treats it as a hard constraint — a real interview signal the interviewer wants to see space-optimization reasoning |
| Character counting | Always reaches for `HashMap` | Considers whether alphabet is bounded (fixed array is faster, no hashing overhead) |

---

## 1.7 COMMON MISTAKES

1. Off-by-one errors in loop bounds (`<` vs `<=`) — always dry-run on a
   tiny example (empty array, single element) before trusting the loop bounds.
2. Not handling edge cases: empty array, single element, `k` larger than array length.
3. Assuming a fixed-size frequency array works for arbitrary Unicode input
   (only valid for known-bounded alphabets).
4. Modifying an array while iterating over it with an enhanced `for` loop
   in a way that skips elements (a classic bug when removing elements
   in-place) — use explicit index management or a two-pointer write cursor.
5. Reaching for sorting (O(n log n)) out of habit when an O(n) single-pass
   solution exists.

---

## 1.8 INTERVIEW QUESTION BANK — CHAPTER 1

**Basic:** 1. What's the time complexity of Kadane's algorithm and why?
2. Why is a fixed-size array sometimes preferred over a HashMap for
character counting?

**Intermediate:** 3. Modify Kadane's algorithm to also return the start and
end indices of the maximum subarray. 4. Solve the "maximum product
subarray" variant — why doesn't the same greedy insight (drop negative
running value) directly apply?

**Senior:** 5. Given an array of daily stock prices, find the maximum
profit from one buy and one sell (this is Kadane's algorithm in disguise —
explain the mapping). 6. Design an O(n) algorithm to find if any
contiguous subarray sums to exactly a target value K (hint: prefix sum +
hash set, previewing Chapters 2 and 6).

**Architect:** 7. You need to compute "maximum subarray sum" on a
continuously streaming array (numbers keep arriving, old numbers never
leave) with O(1) additional memory beyond the running state. Does Kadane's
algorithm adapt directly to a streaming context? What has to change, if
anything?

**Scenario:** 8. A candidate proposes sorting the array first to solve
"maximum subarray sum." What's wrong with this approach for this
particular problem (as opposed to the anagram problem, where sorting is a
valid approach)?

**Trick:** 9. "Kadane's algorithm always includes at least one element
in the answer, even if all elements are negative." True or false, and why does this matter for correctness?

<details><summary>Key answers</summary>

- Q4: Product subarrays don't have Kadane's "drop negative prefix" property
  because a large *negative* running product can become the maximum
  *positive* product if multiplied by another negative number — so you must
  track both the running max AND running min product at each step (the min
  can flip to become the new max on encountering a negative number).
- Q5: "Buy low, sell high once" maps directly to "maximum subarray sum" on
  the array of day-to-day price *differences* — the maximum subarray sum of
  differences equals the maximum achievable profit.
- Q7: Yes, it adapts naturally — Kadane's is already an O(1)-state
  streaming-friendly algorithm (`currentSum` and `maxSum` are the entire
  state); each new streamed number is processed exactly like the next
  array element, with no need to revisit earlier data.
- Q8: Sorting destroys the "contiguous" requirement — sum of a subarray
  depends on the original order/adjacency of elements, which sorting
  discards entirely; sorting is valid for the anagram problem because
  anagram-equality only depends on the *multiset* of characters, not their
  order.
- Q9: True — this matters because the problem (as classically stated)
  requires a non-empty subarray; initializing `maxSum`/`currentSum` from
  `nums[0]` rather than 0 ensures at least one element is always included,
  correctly handling the all-negative-numbers edge case.

</details>

---

## 1.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is Kadane's algorithm O(n) time and O(1) space, precisely?
- **Coding Check:** Implement the "maximum product subarray" variant, tracking both running max and running min.
- **Explanation Check:** Explain in English, using the array-rotation reversal trick, why "in-place" and "extra memory" are different constraints an interviewer might test separately.
- **Real-World Check:** A monitoring dashboard needs to show "the best 5-minute revenue window in the last 24 hours" from a stream of per-minute revenue figures. Map this to a pattern from this chapter.
- **Senior Check:** When would you deliberately choose the O(n log n) sort-based anagram check over the O(n) frequency-array check?
- **Master Check:** Given an unsorted array, design an O(n) algorithm to find the longest run of consecutive integers (e.g., `[100,4,200,1,3,2]` → longest run is `[1,2,3,4]`, length 4) without sorting. What data structure from an upcoming chapter would you reach for, and why can't a fixed-size array work here in general?

<details><summary>Answers</summary>

- Real-World Check: This is a sliding-window variant of the "maximum
  subarray" family — a fixed-size (5-minute) sliding window sum over the
  stream, tracking the maximum window sum seen (Chapter 5 builds on this directly).
- Senior Check: When the alphabet is unbounded/large (e.g., full Unicode)
  and building a frequency structure isn't clearly cheaper than sorting, or
  when code simplicity/readability matters more than the (in practice,
  often negligible for small strings) performance difference.
- Master Check: A `HashSet<Integer>` (Chapter 2 preview) — for each number,
  check if it's the *start* of a sequence (i.e., `num - 1` isn't in the
  set), then count forward while `num+1` is present; this is O(n) because
  each number is visited as part of a "count forward" scan at most once
  overall. A fixed-size array doesn't generalize because the value range
  here is unbounded/unknown in advance (unlike the 26-letter alphabet case).

</details>

---

## 1.10 CHEAT SHEET

| Problem shape | Pattern | Complexity |
|---|---|---|
| Best contiguous sum | Kadane's algorithm | O(n) / O(1) |
| Character frequency, bounded alphabet | Fixed-size array | O(n) / O(1) |
| Character frequency, unbounded alphabet | HashMap (Ch. 2) | O(n) / O(n) |
| In-place rotation | Three reversals | O(n) / O(1) |
| "Sorted order doesn't matter, only counts do" | Sort-based comparison OK | O(n log n) |
| "Order/adjacency matters" | Sorting destroys the answer — don't sort | — |

---

*(Continues to Chapter 2 — HashMap / HashSet Patterns.)*
