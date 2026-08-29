# CHAPTER 2 — HASHMAP / HASHSET PATTERNS

---

## 2.1 WHY THIS CHAPTER EXISTS

DSA interviews లో అత్యంత frequently వాడే optimization technique: **"O(n²)
nested loop ని, ఒక HashMap/HashSet తో O(n) కి తగ్గించడం."** ఈ chapter ఈ
ఒక్క idea ని నాలుగు వేర్వేరు problem shapes లో practice చేయిస్తుంది —
దీని వెనుక ఉన్న Book 1 Chapter 4 (HashMap Internals) knowledge ఇక్కడ
directly apply అవుతుంది: O(1) average lookup/insert.

---

## 2.2 PATTERN RECOGNITION: WHEN DOES HASHMAP/HASHSET HELP?

### TELUGU EXPLANATION

ఈ prompt మీ మనసులో మెదలాలి: **"నేను ఇప్పటివరకు చూసిన elements గురించి ఏదో
గుర్తుపెట్టుకోవాలి, తర్వాత elements కోసం త్వరగా వెతకడానికి?"** ఇలా
అనిపిస్తే, HashMap/HashSet candidate.

సంకేతాలు:
- "ఇప్పటికే చూశామా?" (seen before?) → `HashSet`
- "ఎన్నిసార్లు కనిపించింది?" (frequency) → `HashMap<T, Integer>`
- "ఏ index/value తో పెయిర్ అవుతుంది?" (complement lookup) → `HashMap<T, Index/Value>`
- "ఒకే విధమైన వాటిని group చేయాలి" → `HashMap<Key, List<T>>`

**Trade-off ఎప్పుడూ గుర్తుంచుకోండి:** HashMap/HashSet O(n) **extra space**
తీసుకుంటుంది — ఇది "Space-Time trade-off" యొక్క classic ఉదాహరణ:
Time O(n²)→O(n) improve చేయడానికి, Space O(1)→O(n) ఇవ్వాలి.

---

## 2.3 CORE PROBLEM 1 — TWO SUM (THE CANONICAL EXAMPLE)

### PROBLEM
ఒక array, ఒక target ఇచ్చినప్పుడు, ఏ రెండు elements కలిపితే target
వస్తుందో వాటి indices కనుక్కోండి.

### TELUGU EXPLANATION — BRUTE FORCE

ప్రతి pair check చేయడం — O(n²):

```java
int[] twoSumBrute(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++) {
        for (int j = i + 1; j < nums.length; j++) {
            if (nums[i] + nums[j] == target) return new int[]{i, j};
        }
    }
    throw new IllegalArgumentException("No solution");
}
```

### TELUGU EXPLANATION — OPTIMIZATION

**కీలక insight:** `nums[i] + nums[j] = target` అంటే `nums[j] = target -
nums[i]`. ప్రతి element దగ్గర, "నేను ఇప్పటివరకు చూసిన elements లో,
`target - nums[i]` (దీన్ని **complement** అంటారు) ఉందా?" అని అడగండి —
ఇది HashMap లో O(1) లో check చేయవచ్చు, మరో loop అవసరం లేదు.

```java
// O(n) time, O(n) space — single pass
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seenValueToIndex = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (seenValueToIndex.containsKey(complement)) {
            return new int[]{seenValueToIndex.get(complement), i};
        }
        seenValueToIndex.put(nums[i], i); // ఇప్పుడే add చేయండి — తర్వాత elements వెతకడానికి
    }
    throw new IllegalArgumentException("No solution");
}
```

**DRY RUN:** `nums = [2, 7, 11, 15]`, `target = 9`

| i | nums[i] | complement | map before check | match? |
|---|---|---|---|---|
| 0 | 2 | 7 | {} | No → add {2:0} |
| 1 | 7 | 2 | {2:0} | **Yes!** → return [0, 1] |

### ENGLISH INTERVIEW ANSWER

"The brute force checks every pair, which is O(n²). The key realization is
that for each element, I don't need to search the rest of the array for its
complement — I just need to know if I've *already seen* that complement, and
a HashMap gives me that in O(1) average time. I build the map incrementally,
checking for the complement *before* inserting the current element, which
also correctly handles the edge case where the same value could pair with
itself if it appeared twice, without a value pairing with its own,
not-yet-seen instance."

**Complexity:** Time O(n), Space O(n).

**Variations (interviewer follow-ups):**
- "What if there are multiple valid pairs and you need all of them?" —
  Use `Map<Integer, List<Integer>>` to store all indices per value, or
  collect all matches instead of returning on first find.
- "What if the array is sorted?" — Then Two Pointers (Chapter 5) solves it
  in O(n) time with O(1) space instead — a strictly better trade-off when
  sortedness is available, showing why recognizing *given structure* changes the optimal pattern.
- "3Sum (three numbers summing to target)?" — Typically solved by sorting
  + fixing one element + two-pointer on the rest, O(n²) — a hybrid of two
  patterns (Chapters 5 and 7).

---

## 2.4 CORE PROBLEM 2 — GROUP ANAGRAMS

### PROBLEM
ఒక `String[]` ఇచ్చినప్పుడు, anagrams ని ఒకే group లో పెట్టండి.

### TELUGU EXPLANATION

Chapter 1 లో మనం రెండు strings anagrams అవునా కాదా చెక్ చేశాం. ఇప్పుడు
**many strings ని group చేయాలి** — దీనికి ఒక **canonical key** కావాలి, ఏ
anagram అయినా ఒకే key కి map అవ్వాలి. రెండు options:
1. **Sorted string as key:** "eat", "tea", "ate" — అన్నీ sort చేస్తే "aet" — ఒకే key.
2. **Frequency signature as key:** 26-element count array ని a string గా
   encode చేయడం (e.g., "1#0#0#...#1..." — a కి 1, e కి 1 అని) — sorting
   అవసరం లేదు, O(n) per string (Chapter 1 సాంకేతికత తో కలిపి).

```java
// Using sorted string as canonical key — O(n * k log k) where k = avg string length
List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> groups = new HashMap<>();
    for (String s : strs) {
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(groups.values());
}
```

**Design note:** `computeIfAbsent` (Book 1 Streams-adjacent idiom) avoids
the classic "check if key exists, if not create empty list, then add"
three-line dance — a single, atomic-looking, readable call.

### ENGLISH INTERVIEW ANSWER

"Grouping requires a canonical representation that all anagrams of the same
letters map to identically. Sorting each string gives that canonical form
directly at the cost of O(k log k) per string. If string lengths are large
and the alphabet is small and known, encoding the character frequency
counts into a key string avoids the sort, dropping to O(k) per string — a
real optimization worth mentioning when asked to improve on the sorted-key
approach, especially for long strings."

**Complexity:** Time O(n · k log k) with sorted keys (n strings, k = avg
length), or O(n · k) with frequency-signature keys. Space O(n · k).

---

## 2.5 CORE PROBLEM 3 — LONGEST CONSECUTIVE SEQUENCE

### PROBLEM
Unsorted `int[]` ఇచ్చినప్పుడు, longest run of consecutive integers
కనుక్కోండి (sorting వాడకుండా, O(n) target) — ఇది Chapter 1 Master Check
లో ప్రస్తావించిన problem, ఇక్కడ పూర్తిగా solve చేద్దాం.

### TELUGU EXPLANATION

**కీలక insight:** ఒక number `x` ని ఒక sequence యొక్క **start** గా
consider చేయాలి **అది `x - 1` array లో లేకపోతేనే** — ఇలా చేస్తే, ప్రతి
sequence ని దాని start నుండి మాత్రమే ఒక్కసారి count చేస్తాము, మధ్యలో
నుండి కాదు — దీనివల్ల total work అన్ని sequences కలిపి O(n) మాత్రమే
(prevents redundant re-scanning).

```java
// O(n) time, O(n) space
int longestConsecutive(int[] nums) {
    Set<Integer> numSet = new HashSet<>();
    for (int n : nums) numSet.add(n);

    int longest = 0;
    for (int num : numSet) {
        if (!numSet.contains(num - 1)) { // ఇది ఒక sequence యొక్క START మాత్రమే process చేయండి
            int length = 1;
            while (numSet.contains(num + length)) {
                length++;
            }
            longest = Math.max(longest, length);
        }
    }
    return longest;
}
```

**DRY RUN:** `nums = [100, 4, 200, 1, 3, 2]`
- Set: {100, 4, 200, 1, 3, 2}
- `100`: is 99 in set? No → start. length: 100+1=101? No. length=1.
- `4`: is 3 in set? Yes → skip (not a start).
- `200`: is 199 in set? No → start. length=1.
- `1`: is 0 in set? No → start. Check 2✓,3✓,4✓,5✗ → length=4.
- `3`,`2`: are starts? 3-1=2 in set → skip. 2-1=1 in set → skip.
- **Answer: 4** (sequence `[1,2,3,4]`).

### ENGLISH INTERVIEW ANSWER

"The naive approach is to sort first, which is O(n log n). To get true
O(n), I put everything in a HashSet for O(1) lookups, then — critically — I
only start counting a sequence from a number that is genuinely a sequence
start, meaning `num - 1` is not in the set. This guarantees each number is
visited at most twice total across the whole algorithm — once as a
candidate start check, and at most once more as part of exactly one
forward scan — which is what keeps the total work linear despite the
inner while loop, a common point of confusion when first analyzing this
algorithm's complexity."

**Complexity:** Time O(n) amortized (each element visited a constant
number of times total), Space O(n).

**Interviewer follow-up:** "Why is this O(n) and not O(n²) given the nested
while loop?" — Because the while loop only executes for numbers that are
sequence *starts*, and across all starts combined, the total number of
`while` iterations equals the total number of elements that are NOT starts
(each element is consumed by exactly one sequence's forward scan) — so the
sum of all inner-loop work across the whole outer loop is bounded by n, not
n².

---

## 2.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Have I seen this before?" | Nested loop / linear search each time | `HashSet.contains()` — O(1) |
| Sees a nested loop over the same array | Ships it | Asks: "can I replace the inner loop with a hash lookup built as I go?" |
| Complexity of `longestConsecutive`'s while loop | Assumes O(n²) from nested loops | Recognizes amortized analysis: total inner-loop work is bounded across all outer iterations |

---

## 2.7 COMMON MISTAKES

1. Checking `map.get(key) != null` instead of `map.containsKey(key)` when
   the map can legitimately store `null` values or when value type is a
   primitive wrapper that could itself be a valid stored value causing confusion.
2. Forgetting that `HashMap`/`HashSet` require correct `equals`/`hashCode`
   on custom key objects (Book 1, Chapter 4) — a very common silent bug
   when using custom classes as map keys in DSA problems.
3. Not considering the space trade-off explicitly when discussing a
   solution's complexity — always state both time AND space.
4. In the Longest Consecutive Sequence problem, forgetting the `num - 1`
   check and instead scanning forward from every element — this silently
   degrades the algorithm to O(n²) in the worst case (e.g., one long run).
5. Using `Map<Integer, Integer>` where `Integer` autoboxing/caching
   behavior (`Integer` cache for -128 to 127) causes confusing `==` bugs if
   someone mistakenly compares boxed values with `==` instead of `.equals()`.

---

## 2.8 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. Why is Two Sum O(n) with a HashMap instead of O(n²)? 2. What's
the time/space trade-off being made?

**Intermediate:** 3. How would you solve Two Sum if the array were sorted
and you needed O(1) space? 4. Explain the canonical-key idea behind Group Anagrams.

**Senior:** 5. Walk through why Longest Consecutive Sequence is O(n) despite
the nested loop — defend the amortized analysis rigorously. 6. Design a
"Subarray Sum Equals K" solution using prefix sums + HashMap (bridge to
Chapter 6) — explain why a plain HashSet isn't enough here.

**Architect:** 7. You're processing a stream of millions of unique user IDs
and need to detect duplicates in near-real-time with bounded memory. A
`HashSet` grows unbounded. What alternatives would you consider (e.g.,
Bloom filters), and what's the trade-off?

**Scenario:** 8. A candidate solves Two Sum correctly but says "the space
complexity is O(1) because I'm just storing indices, which are small."
What's wrong with this reasoning?

**Trick:** 9. "Using a HashMap is always faster than a nested loop." True or false?

<details><summary>Key answers</summary>

- Q6: `Map<Integer, Integer>` mapping prefix-sum-value → count of times
  seen; at each index, check if `(currentPrefixSum - k)` has been seen
  before — a HashSet alone can't answer "how many subarrays," only "does
  one exist," and can't directly support cases with duplicate prefix sums
  representing multiple valid subarrays.
- Q7: Bloom filters trade a small, tunable false-positive rate for
  O(1)-ish, genuinely bounded memory regardless of stream size — appropriate
  when occasionally flagging a non-duplicate as "possibly seen" is
  acceptable (with a fallback check), which a `HashSet` cannot offer since
  its memory grows linearly with unique elements seen.
- Q8: Space complexity must count the HashMap's storage, which holds up to
  n entries — it's O(n), not O(1); "the indices are small" is irrelevant to
  asymptotic space analysis, which counts the *number* of stored elements
  relative to input size, not the byte size of each element.
- Q9: False — for very small inputs, the overhead of hashing and object
  allocation can make a HashMap slower in practice than a simple nested
  loop over a tiny array; asymptotic improvement matters most as n grows,
  and a senior engineer wouldn't reach for a HashMap by reflex for, say, a
  fixed 3-element array.

</details>

---

## 2.9 MASTERY CHECKPOINTS

- **Knowledge Check:** In Two Sum, why must you check for the complement *before* inserting the current element into the map?
- **Coding Check:** Solve "Contains Duplicate Within K Distance" (does the array have two equal elements within index distance ≤ k) using a HashMap/HashSet.
- **Explanation Check:** Explain in English the amortized O(n) argument for Longest Consecutive Sequence to someone who insists it "looks like O(n²)."
- **Real-World Check:** A fraud-detection service needs to check, for each incoming transaction, whether the same (customer, merchant) pair has occurred in the last 5 minutes. Map this to a HashMap-based pattern and describe the key/value shape you'd use.
- **Senior Check:** When would you prefer `LinkedHashMap` over `HashMap` for a caching/frequency-tracking DSA-style solution that later becomes production code (bridge to Book 1 Chapter 4)?
- **Master Check:** Design an O(n) solution to "find the first non-repeating character in a string" and explain why insertion order matters for this one, unlike plain frequency counting.

<details><summary>Answers</summary>

- Real-World Check: `Map<Pair<CustomerId, MerchantId>, Instant lastSeen>` —
  on each transaction, check if a recent-enough entry exists for that key
  before flagging/allowing, then update the timestamp — a direct real-world
  application of the "have I seen this before, and when" pattern.
- Senior Check: When you need to report results in insertion or
  access order — e.g., "first non-repeating character" (below) or an LRU
  cache (Chapter 16) — plain `HashMap` gives no ordering guarantee at all.
- Master Check: Use a `LinkedHashMap<Character, Integer>` (or a plain
  HashMap for counts plus a separate ordered pass) to count frequencies
  while preserving first-seen order, then scan in that order for the first
  character with count 1 — insertion order matters because "first" is
  defined by original string position, which a plain `HashMap`'s iteration
  order does not preserve.

</details>

---

## 2.10 CHEAT SHEET

| Need | Structure | Complexity |
|---|---|---|
| "Have I seen this value?" | `HashSet` | O(1) avg |
| "How many times / paired with what?" | `HashMap<K, V>` | O(1) avg |
| "Group items by a derived key" | `HashMap<Key, List<T>>` + `computeIfAbsent` | O(n) total |
| Need insertion/access order preserved | `LinkedHashMap` | O(1) avg + order |
| Complement/pair lookup (Two Sum family) | `HashMap<value, index>`, check before insert | O(n) |
| Sequence/range membership (Longest Consecutive) | `HashSet` + "is this a start?" check | O(n) amortized |

---

*(Continues to Chapter 3 — Linked List.)*
