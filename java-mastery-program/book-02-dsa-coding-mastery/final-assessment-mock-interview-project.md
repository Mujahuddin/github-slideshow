# BOOK 2 — FINAL ASSESSMENT, DSA MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 16 CHAPTERS)

Attempt every question before checking the answer key. These deliberately
mix patterns across chapters — real interviews rarely stay inside one topic,
and the goal of this book was pattern *recognition*, which this section tests directly.

1. A problem asks for "the length of the longest substring with at most 2
   distinct characters." Which chapter's pattern applies, and why? *(Ch. 5)*
2. Given an unsorted array, you need the median in better than O(n log n)
   after full sorting. What structure(s) from this book could help, and
   what's the achievable complexity? *(Ch. 8)*
3. A candidate proposes solving "minimum number of platforms needed for a
   railway station" (given arrival/departure times) with a greedy
   approach. Is greedy valid here? Map it to a specific chapter pattern. *(Ch. 6)*
4. Explain why "Word Break" is a DP problem and not a greedy one — construct
   a small counter-example showing why a greedy segmentation could fail. *(Ch. 13, 14)*
5. You need to detect whether a directed graph of package dependencies has
   a circular dependency. Name two different valid algorithms and their
   complexities. *(Ch. 12)*
6. Why does "Two Sum" have three different valid solutions across this book
   (brute force, HashMap, two-pointer-on-sorted), and when would you choose
   each? *(Ch. 2, 5)*
7. Design a solution for "find all pairs with a given difference K" — is
   this closer to a HashMap pattern or a two-pointer pattern? Does it
   depend on whether the array is sorted?
8. A backtracking solution for generating valid parentheses combinations
   (n pairs) needs pruning to avoid generating invalid sequences before
   checking validity at the end. Design the pruning condition. *(Ch. 4, 9)*
9. Explain how DSU and BFS/DFS both solve "count connected components" —
   under what changing-graph condition does DSU become clearly superior? *(Ch. 12, 15)*
10. Design the interview-round answer for "implement an LRU cache" from
    memory, including the thread-safety follow-up. *(Ch. 16)*

<details>
<summary>Answer Key</summary>

1. Variable Sliding Window (Chapter 5) with a frequency HashMap tracking
   distinct character counts within the window — grow the window while
   distinct count ≤ 2, shrink from the left when it exceeds 2.
2. A "median maintenance" structure — the two-heap approach (max-heap for
   the lower half, min-heap for the upper half, from Chapter 8's mastery
   checkpoint) gives O(log n) insert and O(1) median retrieval, better
   than repeatedly sorting.
3. Yes, greedy is valid and this is exactly the Meeting Rooms II /
   sweep-line pattern from Chapter 6 — "minimum platforms needed at any
   time" is identical in structure to "minimum meeting rooms needed."
4. Word Break requires trying multiple segmentation points and some may
   lead to dead ends while others succeed — greedily taking the first/
   longest matching dictionary word at each position can strand the
   remainder unsegmentable. Example: dictionary `{"a", "aa", "ab"}`,
   string `"aab"` — greedily matching the longest prefix "aa" first
   leaves "b" unmatched, while segmenting as "a"+"ab" succeeds; DP
   correctly explores both options via its `dp[j]` check across all `j`.
5. DFS-based three-color cycle detection (Chapter 12) or Kahn's
   topological sort failing to process all nodes (Chapter 12) — both
   O(V+E); DSU (Chapter 15) also works if dependencies are added
   incrementally and you want to detect the cycle-causing edge as it's added.
6. Brute force (O(n²), no constraints assumed) when n is tiny and
   simplicity matters most; HashMap (O(n) time, O(n) space) as the general
   default for unsorted input; two-pointer (O(n) time, O(1) space) when
   the array is sorted or sorting first is acceptable and extra space
   matters — the "right" choice depends on given structure and constraints,
   exactly the pattern-recognition method from this book's introduction.
7. HashMap-based approach works regardless of sort order — for each
   number, check if `number + K` (and/or `number - K`) exists in a
   HashSet, O(n) time; a two-pointer approach requires the array to be
   sorted first, making it O(n log n) overall unless already sorted, though
   it uses less extra space.
8. Track `open` and `close` counts used so far; only add an opening
   paren if `open < n`, only add a closing paren if `close < open` (never
   more closing than opening so far) — this pruning generates only ever-valid
   prefixes, guaranteeing every complete sequence is valid without any
   post-hoc validity check.
9. Both are O(V+E) for a one-time, static computation. DSU becomes clearly
   superior when edges are added incrementally over time and you need
   "current component count" or "are X and Y connected" answered
   repeatedly after each addition — DSU updates incrementally in near-O(1)
   amortized per edge, while re-running DFS/BFS after every edge addition
   would be far more expensive overall.
10. HashMap (key → doubly-linked-list node) + doubly linked list with
    dummy head/tail sentinels for O(1) get/put with recency tracking
    (Chapter 16); thread-safety follow-up: wrap operations in
    `synchronized`, or use `LinkedHashMap` with `accessOrder=true` plus an
    overridden `removeEldestEntry`, itself externally synchronized since
    `LinkedHashMap` is not thread-safe on its own.

</details>

---

## PART B — MOCK INTERVIEW: DSA ROUND

*Format: read the interviewer's prompt, answer out loud in English before
reading the model answer.*

**Interviewer:** "Given an array of integers, find the length of the
longest subarray with an equal number of 0s and 1s. Walk me through your
approach from brute force to optimal."

> *What's being tested:* whether you can independently apply the
> prefix-sum + HashMap combination pattern to a problem that doesn't
> obviously look like "Subarray Sum Equals K" at first glance.

**Model answer:** "Brute force checks every subarray and counts 0s and 1s,
O(n²) or O(n³) depending on how counting is done. The key transformation:
treat every 0 as -1. Now the problem becomes 'find the longest subarray
summing to exactly 0' — which is exactly Chapter 6's Subarray Sum Equals K
pattern, with K=0. I compute a running prefix sum; if the same prefix sum
value has occurred before at an earlier index, the subarray between those
two indices sums to zero, meaning equal 0s and 1s. I track the *first*
occurrence of each prefix sum in a HashMap — for 'longest,' I want the
earliest index with that prefix sum to maximize the subarray length,
unlike Subarray-Sum-Equals-K's 'count' variant, which tracks frequency
instead. This is O(n) time and O(n) space."

**Follow-up:** "Why do you store the *first* occurrence instead of every
occurrence?" (Because for maximizing length, only the earliest matching
prefix index can ever produce the longest subarray to the current
position — later occurrences would only produce shorter subarrays.)

---

**Interviewer:** "Design a data structure that supports `insert`,
`remove`, and `getRandom` — all in average O(1) time."

> *What's being tested:* combining an array (Chapter 1) and a HashMap
> (Chapter 2) creatively — this doesn't map to one named pattern, testing
> synthesis rather than recall.

**Model answer:** "A HashSet alone gives O(1) insert/remove but can't do
O(1) random access — sets don't support indexing. An ArrayList alone gives
O(1) random access by index but O(n) remove (shifting elements). I combine
both: an ArrayList holding the actual values, plus a HashMap mapping each
value to its current index in the list. Insert appends to the list and
records its index in the map — O(1). `getRandom` picks a random valid
index and returns that list element — O(1). The clever part is remove:
instead of shifting elements after removing from the middle, I swap the
element to remove with the *last* element in the list, update the swapped
element's index in the map, then remove the last element — O(1) instead of
O(n), because removing from the end of an ArrayList is O(1) and swapping
avoids ever shifting the rest of the array."

**Follow-up:** "What if duplicates are allowed?" (The HashMap needs to map
each value to a *set* of indices instead of a single index, and remove
picks any one of those indices to swap-and-pop — a meaningful but
tractable extension.)

---

**Interviewer:** "You have a stream of integers arriving continuously.
At any point, someone can ask for the k-th largest element seen so far.
Design this."

> *What's being tested:* recognizing the bounded min-heap pattern (Chapter
> 8) applies even when "the array" isn't static or even fully known in advance.

**Model answer:** "This is the streaming version of Chapter 8's Kth
Largest Element pattern. I maintain a min-heap capped at size k throughout
the stream's lifetime — every new number is offered to the heap, and if
the heap exceeds size k, I evict the smallest. At any point, the heap's
root is the answer, in O(1), and each insertion is O(log k). This is
strictly better than re-sorting everything seen so far on each query,
which would be increasingly expensive as the stream grows — the bounded
heap's cost per insertion stays constant relative to total stream length,
depending only on k."

**Follow-up:** "What if k itself can change between queries?" (The bounded
min-heap approach breaks down cleanly for this — you'd need to either
maintain the full sorted history, e.g., via a balanced structure supporting
order-statistics queries, or accept a slower approach; this is a good
moment to say "here's the trade-off I'd need to discuss with the team" rather than guessing.)

---

## PART C — CAPSTONE PROJECT: "MULTI-PATTERN CODING CHALLENGE PLATFORM"

**Goal:** Build a small Java command-line tool that exercises pattern
recognition across the whole book, and includes one real production
component from Chapter 16 — the way a senior engineer would actually
combine DSA thinking with a system component in a single project.

**Requirements:**

1. Implement a `PatternSolver` interface with at least 8 methods, one per
   major pattern family covered (Two Pointers, Sliding Window, HashMap,
   Stack/Monotonic, Binary Search, Backtracking, Graph BFS/DFS, DP) — each
   backed by a genuinely different problem from this book (not the same
   problem restated 8 times).
2. For each method, include a comment stating: the pattern used, the time/
   space complexity, and one sentence on why that pattern (not another)
   fits this problem — this is the "explain your reasoning" discipline
   from section 1.1 of Book 2's README, applied to your own code.
3. Build an in-memory `RateLimitedSubmissionQueue` (Chapter 16) that wraps
   your `PatternSolver`: incoming "submissions" (simulated as tasks on
   separate threads) must pass through a Token Bucket rate limiter before
   being processed by a small custom thread pool (also Chapter 16),
   demonstrating that Book 2's algorithmic content and Book 1's
   concurrency content compose into one working system.
4. Include a `benchmark()` method that runs your DP-based Coin Change
   (Chapter 14) against a naive recursive (non-memoized) version on
   increasing input sizes, and prints the point at which the naive version
   becomes impractically slow — making the "why DP matters" argument with
   real numbers, not just theory.
5. Write a README explaining, for each of the 8 `PatternSolver` methods,
   the "5-question pattern recognition method" from the Book 2 README
   applied to that specific problem — this proves you can apply the
   method to new problems, not just recite worked examples.

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Pattern diversity | All 8 methods use genuinely different patterns, not variations of one |
| Reasoning documented | Every method's comment correctly names why its pattern fits |
| Concurrency integration | Rate limiter + thread pool correctly bound requests without unbounded queues |
| DP vs naive benchmark | Actually demonstrates the exponential blowup, not just claims it |
| Pattern-recognition README | Applies the 5-question method to problems NOT worked as examples in the book |

---

*(This completes BOOK 2 — DSA + CODING MASTERY. Book 3 — Spring Framework —
begins the transition from algorithmic foundations to enterprise Java
framework mastery.)*
