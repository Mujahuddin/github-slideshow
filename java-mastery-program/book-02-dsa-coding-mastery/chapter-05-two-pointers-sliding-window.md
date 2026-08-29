# CHAPTER 5 — TWO POINTERS & SLIDING WINDOW

---

## 5.1 WHY THIS CHAPTER EXISTS

ఈ రెండు patterns (Two Pointers, Sliding Window) కలిపి, DSA interviews లో
**అత్యంత frequently కనిపించే "O(n²) → O(n)" optimization family**. రెండూ
ఒకే core idea పంచుకుంటాయి: **redundant re-scanning ని avoid చేయడం**,
రెండు (లేదా అంతకంటే ఎక్కువ) index pointers ని ఒకే పాస్ లో intelligently
move చేయడం ద్వారా.

---

## 5.2 PATTERN RECOGNITION

### TELUGU EXPLANATION

**Two Pointers ఎప్పుడు:** Input **sorted** గా ఉన్నప్పుడు, లేదా రెండు
ends నుండి ఏకకాలంలో process చేయాల్సిన అవసరం ఉన్నప్పుడు (palindrome
check, container/area problems).

**Sliding Window ఎప్పుడు:** "**contiguous subarray/substring**" గురించి
అడిగినప్పుడు, ఒక condition (sum, distinct count, length) satisfy చేసేది
కనుక్కోవాలి అన్నప్పుడు. రెండు రకాలు:
- **Fixed-size window:** window size `k` ముందే తెలిసినప్పుడు.
- **Variable-size window:** window condition బట్టి grow/shrink అవుతుంది
  ("longest substring such that...", "smallest window such that...").

---

## 5.3 CORE PROBLEM 1 — TWO SUM ON A SORTED ARRAY (TWO POINTERS)

### TELUGU EXPLANATION

Chapter 2 లో HashMap తో Two Sum చేశాం (O(n) space). Array **already
sorted** అయితే, **O(1) space** తో కూడా O(n) time సాధించవచ్చు.

**కీలక insight:** `left` pointer array ప్రారంభంలో, `right` pointer చివర
ఉంచండి. `nums[left] + nums[right]` target కంటే **ఎక్కువ** అయితే,
`right` ని తగ్గించండి (sum తగ్గించడానికి). **తక్కువ** అయితే, `left`
పెంచండి (sum పెంచడానికి). ఇది పని చేస్తుంది ఎందుకంటే array sorted —
ప్రతి కదలిక sum ని correct దిశలో move చేస్తుందని **guarantee** ఉంది.

```java
// O(n) time, O(1) space — requires sorted input
int[] twoSumSorted(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) return new int[]{left, right};
        else if (sum < target) left++;
        else right--;
    }
    throw new IllegalArgumentException("No solution");
}
```

### ENGLISH INTERVIEW ANSWER

"Sortedness lets me replace the HashMap's O(n) space with two pointers
closing in from both ends in O(1) space. If the current sum is too small, I
know I need a bigger number, and since the array is sorted, moving the left
pointer right is the only way to get one without missing a valid pair — the
sorted order guarantees I never need to reconsider a discarded pointer
position. This is the classic case where recognizing given structure
(sortedness) changes the optimal pattern entirely, compared to the
unsorted-array Two Sum in Chapter 2."

---

## 5.4 CORE PROBLEM 2 — CONTAINER WITH MOST WATER

### PROBLEM
Heights ఇచ్చినప్పుడు, రెండు lines ఎంచుకుని, వాటి మధ్య (x-axis తో కలిపి)
ఏర్పడే container యొక్క maximum water area కనుక్కోండి. Area = `min(height[i],
height[j]) * (j - i)`.

### TELUGU EXPLANATION

**కీలక insight:** `left=0, right=n-1` నుండి మొదలుపెట్టండి (maximum
possible width). ఏ pointer దగ్గర **తక్కువ height** ఉందో, దాన్నే
move చేయాలి — ఎందుకంటే: width ఎలాగూ తగ్గుతుంది కాబట్టి, area పెరగాలంటే
height పెరగాలి; తక్కువ height ఉన్న pointer ని అలాగే ఉంచి, పొడవైన దాన్ని
move చేస్తే, కొత్త height **ఎప్పటికీ min(height[i], height[j]) ని
పెంచదు** (ఎందుకంటే min ఇప్పటికే తక్కువ ఉన్నదాని మీద ఆధారపడి ఉంది) —
కాబట్టి తక్కువదాన్ని move చేయడం మాత్రమే improvement అవకాశం ఇస్తుంది.

```java
// O(n) time, O(1) space
int maxArea(int[] height) {
    int left = 0, right = height.length - 1;
    int maxArea = 0;
    while (left < right) {
        int width = right - left;
        int area = Math.min(height[left], height[right]) * width;
        maxArea = Math.max(maxArea, area);
        if (height[left] < height[right]) left++;
        else right--;
    }
    return maxArea;
}
```

### ENGLISH INTERVIEW ANSWER

"Starting from the widest possible container and narrowing is the key
framing. At each step, moving the pointer at the *taller* line can never
help — the area is bounded by the shorter line's height, so keeping the
taller line and shrinking width can only maintain or reduce the area. Only
moving the shorter line's pointer has any chance of finding a taller line
that increases the limiting height enough to compensate for the reduced
width. This greedy elimination argument is what justifies discarding half
the remaining search space at each step, giving O(n) instead of the O(n²)
brute force of checking every pair."

**Interviewer follow-up:** "Prove that this greedy approach never misses
the optimal answer." — This is exactly the exhaustive elimination argument
above: at every step, the pointer that moves is *provably* never part of a
better solution than what's already been considered, so nothing optimal is
ever skipped.

---

## 5.5 CORE PROBLEM 3 — LONGEST SUBSTRING WITHOUT REPEATING CHARACTERS (VARIABLE SLIDING WINDOW)

### TELUGU EXPLANATION — BRUTE FORCE

ప్రతి substring చెక్ చేయడం — O(n³) (O(n²) substrings, ఒక్కొక్కదానికి
O(n) duplicate check).

### TELUGU EXPLANATION — OPTIMIZATION (VARIABLE SLIDING WINDOW + HASHSET)

**కీలక insight:** ఒక window `[left, right]` maintain చేయండి, దీనిలో
అన్ని characters **distinct**. `right` ని ప్రతిసారి ఒక్క step ముందుకు
జరపండి — repeat character కనిపిస్తే, అది కనిపించనంత వరకు `left` ని
ముందుకు జరపండి (window shrink చేయండి). ఇది Chapter 4 యొక్క "window
expand/contract" idea తోనే సారూప్యత — కానీ ఇక్కడ condition array sum
కాదు, "అన్ని distinct" అనేది.

```java
// O(n) time, O(min(n, alphabet size)) space
int lengthOfLongestSubstring(String s) {
    Set<Character> window = new HashSet<>();
    int left = 0, maxLength = 0;

    for (int right = 0; right < s.length(); right++) {
        // duplicate దొరికినంత వరకు, window ని ఎడమవైపు నుండి shrink చేయండి
        while (window.contains(s.charAt(right))) {
            window.remove(s.charAt(left));
            left++;
        }
        window.add(s.charAt(right));
        maxLength = Math.max(maxLength, right - left + 1);
    }
    return maxLength;
}
```

**DRY RUN:** `s = "abcabcbb"`

| right | char | window before | action | window after | maxLength |
|---|---|---|---|---|---|
| 0 | a | {} | add | {a} | 1 |
| 1 | b | {a} | add | {a,b} | 2 |
| 2 | c | {a,b} | add | {a,b,c} | 3 |
| 3 | a | {a,b,c} | duplicate! remove s[left=0]='a', left=1 | {b,c,a} | 3 |
| 4 | b | {b,c,a} | duplicate! remove s[left=1]='b', left=2 | {c,a,b} | 3 |
| 5 | c | {c,a,b} | duplicate! remove s[left=2]='c', left=3 | {a,b,c} | 3 |
| 6 | b | {a,b,c} | duplicate! remove s[left=3]='a',left=4; still has b? remove s[4]='b',left=5 | {c,b} | 2 |
| 7 | b | {c,b} | duplicate! remove s[5]='c',left=6; still dup? remove s[6]='b',left=7 | {b} | 1 |

**Answer: 3** (`"abc"`).

### ENGLISH INTERVIEW ANSWER

"The brute force checks every substring for duplicates independently,
O(n³). The sliding window insight is that the window only ever needs to
grow from the right and shrink from the left — once a duplicate appears, I
don't need to restart the search; I only need to shrink from the left until
the duplicate is removed, and everything I already knew about the rest of
the window remains valid. Each character is added to the window set and
removed from it at most once across the whole algorithm, which is the same
amortized-O(n) argument as the monotonic stack and monotonic deque patterns
from Chapter 4 — a recurring proof technique across nearly every pattern in this book."

---

## 5.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Sorted array, pair-sum problem | Reaches for HashMap out of habit | Recognizes sortedness enables O(1)-space two pointers |
| "Longest/smallest substring satisfying X" | Nested loop checking every substring | Recognizes variable sliding window; asks "can I maintain window validity incrementally?" |
| Container/area maximization | Tries all pairs O(n²) | Looks for a greedy elimination argument to discard half the search space per step |
| Amortized complexity of window/pointer algorithms | Assumes worst case per step multiplies out | Proves total work via "each element touched O(1) times overall" |

---

## 5.7 COMMON MISTAKES

1. Applying two-pointer convergence to an **unsorted** array, silently
   producing wrong answers (two pointers requires the sorted-order
   guarantee to be valid).
2. In sliding window, forgetting to shrink the window fully (using `if`
   instead of `while` when the shrink condition can require multiple steps).
3. Off-by-one in window length calculation (`right - left + 1` vs `right - left`).
4. Not clearing/updating the tracking structure (HashSet/HashMap/frequency
   array) correctly when shrinking the window from the left.
5. Assuming every "two pointers" problem starts from both ends — some
   (like the fast/slow pattern from Chapter 3, or same-direction two
   pointers for in-place array partitioning) move both pointers from the
   same side instead.

---

## 5.8 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. When is two pointers applicable but a HashMap-based approach
isn't necessary? 2. What's the difference between fixed-size and
variable-size sliding windows?

**Intermediate:** 3. Solve "minimum size subarray sum ≥ target" using a
variable sliding window — why does shrinking work here without missing a
better answer? 4. Explain why the Container With Most Water greedy
pointer-move never misses the optimal solution.

**Senior:** 5. Solve "Minimum Window Substring" (find the smallest window
in s containing all characters of t, with frequency) — this combines
variable sliding window with a HashMap frequency-matching condition. Walk
through the design. 6. When would a variable sliding window NOT work for a
"subarray satisfying condition X" problem? (Hint: think about what
property the condition needs — monotonicity.)

**Architect:** 7. You're designing a real-time system to detect "the
longest streak of transactions without a fraud flag" in a live transaction
stream, where old transactions eventually age out (bounded window in
time, not count). How does the sliding window pattern adapt to a
time-bounded, streaming context instead of a fixed in-memory array?

**Scenario:** 8. A candidate's sliding-window solution shrinks the window
using `if (condition) { left++; }` instead of a `while` loop. Under what
input does this produce a wrong answer, and why?

**Trick:** 9. "Sliding window always requires the window to only grow, never shrink, to be O(n)." True or false?

<details><summary>Key answers</summary>

- Q3: Since all numbers are positive (a typical constraint for this
  problem), adding elements only increases the sum and removing only
  decreases it — this monotonic relationship is exactly what makes
  shrink-while-still-valid safe without missing a shorter valid window;
  this monotonicity requirement is the deeper answer to Q6.
- Q6: Sliding window requires that expanding the window moves the tracked
  condition monotonically in one direction and shrinking moves it back —
  if elements can be negative (breaking the sum-monotonicity assumption,
  for example), a straightforward sliding window can silently give wrong
  answers, and a different technique (like prefix sums, Chapter 6) is needed instead.
- Q7: Replace "index-based window boundaries" with "timestamp-based
  eviction" — instead of `right - k`, evict from the front while
  `currentTime - front.timestamp > windowDuration`; the core two-pointer
  expand/shrink logic is unchanged, just re-anchored to time instead of array position.
- Q8: If a single new element requires shrinking the window by more than
  one position to restore validity (e.g., multiple duplicates need
  removing before the window is valid again), an `if` only shrinks once,
  leaving the window still invalid — a `while` loop is required to fully
  restore the invariant before continuing.
- Q9: False — many sliding window problems (like this chapter's "longest
  substring without repeating characters") both grow AND shrink the
  window; the O(n) guarantee comes from each pointer moving monotonically
  forward and each element being touched a constant number of times
  overall, not from the window only ever growing.

</details>

---

## 5.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does the two-pointer technique for sorted-array Two Sum require the array to be sorted — what breaks if it isn't?
- **Coding Check:** Implement "longest substring with at most K distinct characters" using a variable sliding window + frequency HashMap.
- **Explanation Check:** Explain in English why Container With Most Water's greedy pointer move is provably correct, not just "usually works."
- **Real-World Check:** An API gateway needs to detect "the longest period in the last hour during which request latency stayed under 200ms" from a stream of per-second latency readings. Map this to a sliding window pattern and identify the monotonicity property it relies on.
- **Senior Check:** When would you choose a fixed-size sliding window's simpler two-pointer-with-constant-gap approach over a variable-size window, even if both could theoretically solve a problem?
- **Master Check:** Design an O(n) solution for "minimum window substring" (Q5 above) fully — specify the exact data structures for tracking required vs. current character counts, and the exact conditions for growing vs. shrinking.

<details><summary>Answers</summary>

- Real-World Check: This is a variable sliding window over the latency
  stream — grow the window while all readings stay under 200ms, shrink
  (reset) when a reading exceeds it; the monotonicity property relied on is
  that a single bad reading immediately invalidates continuing the current
  streak, so there's no need to "look back" once a violation occurs.
- Senior Check: When the window size is genuinely fixed and known
  upfront (like Chapter 1's 5-minute revenue window) — a fixed-gap
  two-pointer approach is simpler to reason about and implement correctly
  than a general variable-window template, even though the variable
  template could be parameterized to behave the same way.
- Master Check: Two frequency maps — `need` (fixed, built from `t`) and
  `window` (current window's counts) — plus a `formed` counter tracking how
  many distinct required characters currently meet their needed count.
  Grow `right` until `formed == need.size()` (window is valid), then
  greedily shrink `left` while still valid to find the minimum length,
  updating the best answer each time the window is valid, before resuming growth.

</details>

---

## 5.10 CHEAT SHEET

| Problem shape | Pattern | Complexity |
|---|---|---|
| Sorted array, pair sum | Two pointers, converging | O(n) / O(1) |
| Maximize area/container between two elements | Two pointers, move the limiting side | O(n) / O(1) |
| Longest/shortest substring/subarray with a condition | Variable sliding window | O(n) / O(1) or O(k) |
| Fixed window size K | Fixed sliding window | O(n) / O(1) |
| Condition isn't monotonic (e.g., negative numbers break sum growth) | Sliding window does NOT apply — consider prefix sum (Ch. 6) | — |

---

*(Continues to Chapter 6 — Prefix Sum & Intervals.)*
