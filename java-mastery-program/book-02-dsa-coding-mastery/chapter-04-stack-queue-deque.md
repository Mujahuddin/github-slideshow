# CHAPTER 4 — STACK, QUEUE, DEQUE

---

## 4.1 WHY THIS CHAPTER EXISTS

Stack (LIFO — Last In, First Out) మరియు Queue (FIFO — First In, First Out)
రెండూ **order-sensitive processing** కోసం. ఈ chapter యొక్క star pattern:
**Monotonic Stack** — ఇది "next greater/smaller element" family problems
అన్నింటినీ O(n²) brute force నుండి O(n) కి తీసుకువెళ్తుంది, మరియు ఇది
DSA interviews లో అండర్‌రేటెడ్ కానీ చాలా frequently అడిగే pattern.

---

## 4.2 CORE PROBLEM 1 — VALID PARENTHESES

### TELUGU EXPLANATION

`(`, `)`, `{`, `}`, `[`, `]` కలిగిన string valid గా balanced గా ఉందా అని
check చేయాలి. **కీలక insight:** ప్రతి closing bracket, **most recently
opened** bracket ని match చేయాలి — ఇది సరిగ్గా **Stack యొక్క LIFO
స్వభావం**.

```java
// O(n) time, O(n) space
boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>(); // Java లో Stack కి ArrayDeque వాడటం recommended
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');

    for (char c : s.toCharArray()) {
        if (pairs.containsValue(c)) {
            stack.push(c); // opening bracket — push
        } else if (pairs.containsKey(c)) {
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                return false; // mismatch లేదా unmatched closing bracket
            }
        }
    }
    return stack.isEmpty(); // అన్ని brackets close అయ్యాయా అని చివరి check
}
```

**Interview note:** `java.util.Stack` (legacy, `Vector`-based, synchronized
— unnecessary overhead) బదులు, **`ArrayDeque` వాడటం Java లో idiomatic**
— ఇది Book 1 Chapter 4 లో నేర్చుకున్న "legacy synchronized collections
avoid చేయండి" సూత్రానికి direct application.

### ENGLISH INTERVIEW ANSWER

"Every closing bracket must match the most recently opened, unclosed
bracket — that 'most recent' requirement is exactly what a stack's LIFO
property gives me for free. I push opening brackets, and on a closing
bracket, I check that the top of the stack is its matching opener; any
mismatch, or a closing bracket with nothing to match, means invalid. At the
end, the stack must be empty — otherwise there are unclosed openers left
over. I use `ArrayDeque` instead of the legacy `Stack` class here, since
`Stack` extends `Vector` and carries synchronization overhead that serves
no purpose in single-threaded logic."

---

## 4.3 CORE PROBLEM 2 — MONOTONIC STACK: NEXT GREATER ELEMENT

### PROBLEM
ప్రతి element కి, దాని **కుడివైపు మొదటి greater element** ఏదో కనుక్కోండి
(లేకపోతే -1).

### TELUGU EXPLANATION — BRUTE FORCE

ప్రతి element కోసం, దాని కుడివైపు అంతా scan చేయడం — O(n²).

### TELUGU EXPLANATION — OPTIMIZATION (MONOTONIC STACK)

**కీలక insight:** ఒక **decreasing stack** maintain చేయండి (stack లో
elements ఎప్పుడూ top నుండి bottom వరకు decreasing order లో ఉండాలి,
indices గా store చేస్తూ). ప్రతి కొత్త element వచ్చినప్పుడు:
- stack top element, కొత్త element కంటే **చిన్నది** అయితే — అది "దాని next
  greater element ని కనుక్కుంది" అని అర్థం — pop చేసి, answer record
  చేయండి. ఇలా stack top కంటే చిన్న elements అన్నీ pop అయ్యేవరకు repeat చేయండి.
- చివరికి కొత్త element ని push చేయండి.

ఈ trick వెనుక ఉన్న గణితం: **prతి element stack లోకి ఒక్కసారి push
అవుతుంది, ఒక్కసారి pop అవుతుంది** — total work O(n), individual లుక్‌అప్
కాదు O(n²).

```java
// O(n) time, O(n) space
int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>(); // indices ని store చేస్తుంది, decreasing value order లో

    for (int i = 0; i < n; i++) {
        // stack top యొక్క value, ప్రస్తుత element కంటే చిన్నది అయినంత వరకు, resolve చేయండి
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            result[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return result;
}
```

**DRY RUN:** `nums = [2, 1, 2, 4, 3]`

| i | nums[i] | stack before (indices) | action | result so far |
|---|---|---|---|---|
| 0 | 2 | [] | push 0 | [-1,-1,-1,-1,-1] |
| 1 | 1 | [0] | nums[0]=2 not < 1 → push 1 | stack=[0,1] |
| 2 | 2 | [0,1] | nums[1]=1<2 → pop 1, result[1]=2; nums[0]=2 not<2 → push 2 | result[1]=2, stack=[0,2] |
| 3 | 4 | [0,2] | nums[2]=2<4→pop2,result[2]=4; nums[0]=2<4→pop0,result[0]=4; push 3 | result=[4,2,4,-1,-1], stack=[3] |
| 4 | 3 | [3] | nums[3]=4 not<3 → push 4 | stack=[3,4] |

Final: `result = [4, 2, 4, -1, -1]` ✅

### ENGLISH INTERVIEW ANSWER

"The brute force is O(n²) — for each element, scan right until you find
something bigger. The monotonic stack flips the direction of thinking: instead
of each element searching forward, I maintain a stack of indices whose
values are decreasing, and whenever a new, bigger element arrives, it
resolves the answer for every smaller element still waiting on the stack,
popping them off. Each index is pushed exactly once and popped at most
once across the entire algorithm, so total work is O(n) even though there's
a nested-looking while loop — this is the same amortized-analysis argument
as Chapter 2's Longest Consecutive Sequence, applied to a stack instead of
a hash set."

**Variations:** "Daily Temperatures" (same pattern, answer is the *distance*
to next greater, not the value itself). "Next Smaller Element" (flip the
comparison, maintain an increasing stack instead).

---

## 4.4 CORE PROBLEM 3 — SLIDING WINDOW MAXIMUM (MONOTONIC DEQUE)

### PROBLEM
ఒక array మరియు window size `k` ఇచ్చినప్పుడు, ప్రతి window యొక్క maximum
కనుక్కోండి.

### TELUGU EXPLANATION

ప్రతి window కి max compute చేయడం brute force O(n·k). **Deque (Double-Ended
Queue)** వాడి, ఇది O(n) కి తగ్గించవచ్చు — ఒక **decreasing deque** (front
నుండి back వరకు, values decreasing order లో) maintain చేయాలి, front
element ఎప్పుడూ current window యొక్క maximum గా ఉంటుంది.

```java
// O(n) time, O(k) space
int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> deque = new ArrayDeque<>(); // indices, decreasing value order
    int[] result = new int[nums.length - k + 1];

    for (int i = 0; i < nums.length; i++) {
        // window నుండి బయటపడిన indices ని front నుండి తొలగించండి
        if (!deque.isEmpty() && deque.peekFirst() <= i - k) {
            deque.pollFirst();
        }
        // ప్రస్తుత element కంటే చిన్న values ని back నుండి తొలగించండి — అవి ఇకపై ఉపయోగపడవు
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        deque.offerLast(i);

        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()]; // front ఎప్పుడూ current max
        }
    }
    return result;
}
```

**Why the front is always the max:** any element that's smaller than a
later element within the current window range is removed immediately (it
can never be the max while a bigger, more-recent element remains in the
window) — so the deque only ever retains a decreasing sequence of
"still-possibly-relevant" candidates, with the true max always at the front.

### ENGLISH INTERVIEW ANSWER

"A naive sliding window max recomputes the maximum for every window, which
is O(n·k). The monotonic deque keeps only the indices that could still
possibly be the maximum for some future window — I evict from the back any
element smaller than the one just arriving, since it can never be the max
again once a bigger element is in the same or a later window, and I evict
from the front once an index falls outside the current window's left
boundary. What remains is always decreasing front-to-back, so the front is
always the current window's maximum — giving O(n) total time since each
index is added and removed from the deque at most once."

---

## 4.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Next greater/smaller element" family | Nested loop, O(n²) | Recognizes the monotonic stack pattern immediately |
| Sliding window max/min | Recompute per window | Monotonic deque, O(n) |
| Choosing a stack implementation in Java | `java.util.Stack` | `ArrayDeque` (faster, not synchronized) |
| Queue via two stacks (classic problem) | Doesn't see why this is asked | Understands it tests amortized analysis reasoning, common in interview design questions |

---

## 4.6 COMMON MISTAKES

1. Using legacy `java.util.Stack`/`java.util.Vector` in new Java code
   instead of `ArrayDeque`.
2. In monotonic stack problems, storing *values* instead of *indices* when
   the answer requires positional information (like distance, as in Daily Temperatures).
3. Forgetting to evict expired indices from the front of a sliding-window deque.
4. Assuming a nested while loop inside a for loop is automatically O(n²)
   without doing the amortized analysis (each element pushed/popped at most
   once, in both the monotonic stack and monotonic deque patterns).
5. Off-by-one in window boundary checks (`i - k` vs `i - k + 1`).

---

## 4.7 INTERVIEW QUESTION BANK — CHAPTER 4

**Basic:** 1. Why is a stack the natural structure for the Valid
Parentheses problem? 2. What does "monotonic stack" mean?

**Intermediate:** 3. Solve "Daily Temperatures" (days until a warmer
temperature) using a monotonic stack — how does it differ from Next
Greater Element? 4. Implement a queue using two stacks — explain the
amortized O(1) analysis for dequeue.

**Senior:** 5. Prove, in words, why the monotonic stack / monotonic deque
approaches are O(n) despite the inner while loop. 6. Design a solution for
"largest rectangle in a histogram" using a monotonic stack (a well-known
hard variant of this pattern).

**Architect:** 7. You're building a real-time analytics system that must
report the maximum value in the last N events at all times, with events
arriving continuously. How does the sliding window maximum pattern apply,
and what changes for an unbounded/streaming context vs. a fixed array?

**Scenario:** 8. A candidate implements sliding window maximum by
recomputing the max via `Collections.max()` on a sub-list for every window.
What's the complexity, and how would you guide them toward the deque
optimization?

**Trick:** 9. "A monotonic stack must always be strictly decreasing — using
`<=` instead of `<` in the popping condition is always a bug." True or false?

<details><summary>Key answers</summary>

- Q4: Two stacks, `inStack` for enqueue (push directly) and `outStack` for
  dequeue; when `outStack` is empty, pour all of `inStack` into it
  (reversing order so the oldest is now on top), then pop. Each element is
  moved from `inStack` to `outStack` at most once in its lifetime, so total
  work across n operations is O(n), giving amortized O(1) per dequeue even
  though a single dequeue call can occasionally be O(n).
- Q7: The core idea (evict smaller elements from the back, evict
  out-of-window elements from the front) applies directly; for a truly
  unbounded stream, "the front" naturally represents "last N events" if you
  evict by timestamp/sequence number instead of array index — the pattern
  doesn't fundamentally change, just what "expired" means.
- Q8: `Collections.max()` per window is O(k) per window, O(n·k) total —
  guide them by asking "do you need to recompute the whole max from
  scratch, or can you reuse work from the previous window?" leading toward
  the observation that only newly smaller-than-incoming elements can ever
  be safely discarded.
- Q9: False as a blanket rule — whether to use `<` or `<=` (strictly
  decreasing vs non-increasing) depends on whether the problem needs to
  treat equal elements specially (e.g., "next greater OR equal element"
  requires `<=` in the eviction condition); it's a real, problem-specific
  decision, not an arbitrary choice.

</details>

---

## 4.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does each element get pushed and popped at most once in a monotonic stack, guaranteeing O(n) total work?
- **Coding Check:** Implement "Daily Temperatures" using the Next Greater Element monotonic stack pattern, returning distances instead of values.
- **Explanation Check:** Explain in English why `ArrayDeque` is preferred over `java.util.Stack` in modern Java code.
- **Real-World Check:** A monitoring system needs to detect, for each minute, how many minutes until latency exceeds the current minute's latency (a "next greater" style question over a live stream). Map this to the chapter's pattern and note what changes for a live stream vs. a fixed array.
- **Senior Check:** When would you choose a plain stack/queue over a deque, even though a deque can do everything both can?
- **Master Check:** Using a monotonic stack, design an O(n) solution for "largest rectangle in a histogram" — explain what the stack tracks and what triggers a pop.

<details><summary>Answers</summary>

- Real-World Check: Same monotonic-stack logic, storing indices/timestamps;
  for a live stream, you can't "look ahead" the way a fixed array allows —
  answers can only be resolved retroactively, when a later value happens to
  exceed an earlier pending one, which is exactly what the monotonic stack
  already does incrementally (it never assumed lookahead in the first place).
- Senior Check: When code clarity benefits from expressing intent narrowly
  — a method that only ever needs FIFO or only ever needs LIFO reads more
  clearly with `Queue`/`Deque`-as-stack typed narrowly, communicating to
  future readers exactly which operations are actually used, even though
  `ArrayDeque` implements both interfaces.
- Master Check: The stack holds indices of bars in increasing height order;
  when a shorter bar arrives, pop taller bars off, and for each popped bar,
  compute the rectangle area using that bar's height and a width spanning
  from the new stack top (exclusive) to the current index (exclusive) — the
  monotonic stack here tracks "bars still tall enough to potentially form a
  larger rectangle."

</details>

---

## 4.9 CHEAT SHEET

| Problem shape | Pattern | Complexity |
|---|---|---|
| Matching/nesting (brackets, tags) | Stack (LIFO) | O(n) |
| "Next greater/smaller element" family | Monotonic stack | O(n) |
| Sliding window min/max | Monotonic deque | O(n) |
| FIFO processing (BFS, task queues) | Queue | O(1) per op |
| Queue via two stacks | Amortized O(1) dequeue | O(n) total for n ops |
| Java stack/queue implementation | `ArrayDeque`, not `java.util.Stack` | — |

---

*(Continues to Chapter 5 — Two Pointers & Sliding Window.)*
