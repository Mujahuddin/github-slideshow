# CHAPTER 13 — GREEDY ALGORITHMS

---

## 13.1 CONCEPT: What Makes an Algorithm "Greedy," and Why Correctness Isn't Automatic

### TELUGU EXPLANATION

**Greedy algorithm** అంటే, ప్రతి step లో, **local గా అత్యుత్తమంగా
కనిపించే choice** ఎంచుకోవడం, "మొత్తం మీద ఇది optimal అవుతుందా?" అని
ఆలోచించకుండా ప్రతిసారీ. ఇది Dynamic Programming (Chapter 14) కంటే
**సరళమైనది, వేగమైనది** (సాధారణంగా O(n log n) లేదా O(n)) — **కానీ ఎప్పుడూ
correct అవదు.**

**అత్యంత ముఖ్యమైన senior-level insight:** గ్రీడీ approach ఒక problem కి
సరిపోతుందో లేదో తెలుసుకోవడానికి, మీరు **"greedy choice property"** మరియు
**"optimal substructure"** ఉన్నాయో లేదో నిరూపించాలి (లేదా కనీసం strongly
argue చేయాలి) — "ఇది పని చేస్తుందని అనిపిస్తోంది" అని ఊహించడం సరిపోదు.
ఒక classic counter-example: **Coin Change** (arbitrary denominations,
ఉదా: `[1, 3, 4]`, target 6) — greedy (ఎప్పుడూ అతిపెద్ద coin ఎంచుకోవడం)
`4+1+1 = 3 coins` ఇస్తుంది, కానీ optimal `3+3 = 2 coins`. **ఇక్కడ greedy
fail అవుతుంది** — ఇది Dynamic Programming (Chapter 14) అవసరమైన
classic ఉదాహరణ.

### ENGLISH INTERVIEW ANSWER

"A greedy algorithm makes the locally best choice at each step without
reconsidering it later, which makes it fast and simple when it works — but
it doesn't always produce the globally optimal answer. The senior-level
discipline here is: before committing to a greedy solution, I try to
either prove the greedy choice property — that a locally optimal choice is
always part of *some* globally optimal solution — or at least construct a
convincing argument via exchange argument or contradiction. The classic
counter-example that keeps engineers honest is coin change with arbitrary
denominations: greedily picking the largest coin first fails for
denominations like `[1, 3, 4]` targeting 6 — greedy gives 3 coins
(4+1+1), but the optimal answer is 2 (3+3). This is exactly the kind of
problem where Dynamic Programming's exhaustive-but-memoized approach is
necessary, because no greedy rule reliably gets the right answer."

---

## 13.2 CORE PROBLEM 1 — ACTIVITY SELECTION / NON-OVERLAPPING INTERVALS

### PROBLEM
Intervals ఇచ్చినప్పుడు, **overlap లేకుండా ఉంచడానికి తీసేయాల్సిన కనీస
intervals సంఖ్య** కనుక్కోండి (లేదా, సమానంగా, గరిష్ట సంఖ్యలో
non-overlapping intervals ఎంచుకోండి).

### TELUGU EXPLANATION — THE GREEDY INSIGHT AND ITS PROOF

**Greedy rule:** Intervals ని **end time ప్రకారం sort** చేయండి. ప్రతిసారి,
ఇప్పటివరకు ఎంచుకున్న చివరి interval తో overlap కాని, **అతి త్వరగా
ముగిసే** interval ని ఎంచుకోండి.

**ఎందుకు ఇది correct (exchange argument):** ఏదైనా optimal solution
consider చేయండి. ఆ optimal solution యొక్క మొదటి ఎంచుకున్న interval,
greedy ఎంచుకున్న దాని కంటే **ఆలస్యంగా ముగుస్తుంది** అనుకుందాం. అప్పుడు,
greedy యొక్క ఎంపిక తో దాన్ని **replace చేసినా**, remaining intervals తో
ఎలాంటి కొత్త conflict రాదు (ఎందుకంటే greedy ముందుగా ముగుస్తుంది కాబట్టి
తర్వాత వచ్చే వాటికి **ఎక్కువ అవకాశం** ఇస్తుంది, తక్కువ కాదు) — కాబట్టి
greedy ఎంపిక తో replace చేసిన solution కూడా అంతే optimal. ఇదే **greedy
choice ఎప్పుడూ ఏదో ఒక optimal solution లో భాగం గా ఉండగలదు** అని
నిరూపించడం.

```java
// O(n log n) time (dominated by sort), O(1) extra space
int eraseOverlapIntervals(int[][] intervals) {
    if (intervals.length == 0) return 0;
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[1], b[1])); // END TIME ప్రకారం sort — కీలకం

    int count = 0; // తీసేయాల్సిన intervals సంఖ్య
    int lastEnd = intervals[0][1];

    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] < lastEnd) {
            count++; // overlap — ఈ interval ని "తీసేయాలి" (ఎంచుకోకండి)
            // lastEnd ని మార్చవద్దు — ఇప్పటికే ఎంచుకున్నదే (త్వరగా ముగిసేది) ఉంచండి
        } else {
            lastEnd = intervals[i][1]; // overlap లేదు — దీన్ని ఎంచుకోండి, lastEnd update చేయండి
        }
    }
    return count;
}
```

**Design note:** Sort **end time** ప్రకారం చేయాలి, **start time** ప్రకారం
కాదు — ఇది Chapter 6 (Merge Intervals) తో పోలిస్తే **ఒక ముఖ్యమైన
వ్యత్యాసం**: అక్కడ overlapping వాటిని merge చేయాలి (start time sort
సరిపోతుంది), ఇక్కడ "ఎన్ని conflict-free గా ఎంచుకోవచ్చు" (greedy, end time
sort అవసరం — ఎందుకంటే త్వరగా ముగిసేది future కి ఎక్కువ freedom ఇస్తుంది).

### ENGLISH INTERVIEW ANSWER

"The greedy rule is to always prefer the interval that finishes earliest
among the remaining candidates, which requires sorting by end time — not
start time, which is a deliberate and important distinction from Chapter
6's Merge Intervals problem. The correctness argument is an exchange
argument: if some optimal solution's first pick ends later than greedy's
first pick, swapping in greedy's earlier-ending choice can only free up
more room for subsequent picks, never less — so the swap can't make the
solution worse, meaning greedy's choice is always at least as good. This
kind of exchange-argument reasoning is exactly what I'd want to articulate
in an interview to demonstrate the greedy choice isn't just 'the answer I
guessed' but a provably safe one."

---

## 13.3 CORE PROBLEM 2 — JUMP GAME (GREEDY REACHABILITY)

### PROBLEM
ఒక array ఇచ్చినప్పుడు, ప్రతి element ఆ position నుండి **గరిష్టంగా ఎన్ని
steps ముందుకు jump చేయవచ్చో** సూచిస్తుంది. చివరి index ని చేరుకోగలమా అని
చెప్పండి.

### TELUGU EXPLANATION

**కీలక insight:** ప్రతి position వద్ద ఏ jump ఎంచుకోవాలో వెతకాల్సిన
అవసరం లేదు — కేవలం **"ఇప్పటివరకు చేరుకోగల గరిష్ట index (farthest
reach)"** ఏమిటో track చేయండి. ఏ position వద్ద అయినా `farthest reach <
current position` అయితే, ముందుకు వెళ్ళలేమని అర్థం.

```java
// O(n) time, O(1) space
boolean canJump(int[] nums) {
    int farthestReach = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > farthestReach) return false; // ఈ position కి చేరుకోలేకపోయాం
        farthestReach = Math.max(farthestReach, i + nums[i]);
    }
    return true;
}
```

### ENGLISH INTERVIEW ANSWER

"Rather than exploring all possible jump sequences — which would be
exponential — the greedy insight is that I only need to track the single
best number I've seen: the farthest index reachable so far from anything
already visited. If I ever reach a position beyond that farthest reach
before updating it, the array is unreachable from there. This collapses
what looks like a combinatorial search problem into a single O(n) linear
scan, because 'can I reach position i' only ever depends on the single
running maximum, not on which specific path got me there."

---

## 13.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Find the optimal choice" problem | Assumes greedy works because a rule "feels right" | Tests the greedy rule against a hand-constructed counter-example before committing |
| Interval scheduling (max non-overlapping) | Sorts by start time (copies the Merge Intervals pattern) | Recognizes this needs sorting by END time specifically |
| Coin change | Applies greedy (largest coin first) unconditionally | Knows greedy only works for "canonical" coin systems (like US currency) and fails for arbitrary denominations — reaches for DP instead |
| Explaining why a greedy solution is correct | "I tried it and it passed the test cases" | Articulates an exchange argument or exhaustive local-choice justification |

---

## 13.5 COMMON MISTAKES

1. Applying a greedy strategy without any correctness justification, then
   being surprised when an edge case fails.
2. Sorting by the wrong key (start vs. end time) for interval-greedy problems.
3. Assuming greedy always beats DP on speed without checking that greedy
   is even *correct* for the specific problem first — a fast wrong answer
   is worse than a correct one.
4. Not recognizing "canonical" vs. "non-canonical" coin/denomination
   systems — greedy works for well-behaved denominations (US coins) but
   not arbitrary ones.
5. Confusing "greedy" with "brute force with early exit" — true greedy
   never reconsiders a prior choice; if your solution backtracks or
   revisits decisions, it's not actually greedy.

---

## 13.6 INTERVIEW QUESTION BANK — CHAPTER 13

**Basic:** 1. What makes an algorithm "greedy"? 2. Why doesn't greedy
always produce the optimal answer?

**Intermediate:** 3. Walk through the exchange argument proving the
Activity Selection greedy rule (sort by end time) is correct. 4. Why does
Jump Game's greedy "farthest reach" approach avoid exploring every
possible jump path?

**Senior:** 5. Construct a coin denomination set where the greedy
"largest coin first" approach fails, and explain precisely why. 6.
Design "Gas Station" (given gas and cost arrays for a circular route, find
the starting station that allows completing the full circuit, if one
exists) using a greedy approach — what's the key insight that avoids
checking every possible starting point individually?

**Architect:** 7. You're designing a meeting-scheduling optimizer for a
company that wants to maximize the number of meetings held in shared
rooms. How does the Activity Selection greedy pattern extend when there
are multiple identical rooms available instead of just one resource?

**Scenario:** 8. A candidate proposes a greedy solution for "0/1 Knapsack"
(each item can be taken or not, maximize value under a weight limit) by
always taking the item with the highest value-to-weight ratio first. Does
this work? Why or why not?

**Trick:** 9. "If a greedy algorithm passes all provided test cases, it's
proven correct." True or false?

<details><summary>Key answers</summary>

- Q5: `[1, 3, 4]` targeting 6 — greedy gives 4+1+1 (3 coins), optimal is
  3+3 (2 coins); the failure occurs because greedily maximizing progress
  toward the target at each step doesn't account for how remaining
  denominations combine — this is precisely why Coin Change is a canonical
  Dynamic Programming problem (Chapter 14), not a greedy one, whenever
  denominations aren't guaranteed "canonical" (like the US coin system,
  where greedy happens to always be optimal).
- Q6: The key insight is that if the total gas across the whole circuit is
  at least the total cost, a valid starting point is guaranteed to exist,
  and it can be found in one O(n) pass: track a running tank balance
  starting from candidate station 0; whenever the balance goes negative,
  none of the stations from the last candidate start through the current
  one could have been a valid start either (they'd have run out even
  sooner), so reset the candidate start to the next station and the
  running balance to 0 — a single linear scan, no need to simulate every
  possible starting point independently.
- Q7: With multiple identical rooms, this becomes an extension: greedily
  process activities sorted by start time, and assign each to any
  currently-free room (tracked via a min-heap of room end-times, checking
  if the earliest-freeing room is free by the new activity's start time) —
  a direct bridge back to the Meeting Rooms II pattern from Chapter 6,
  since "minimum rooms needed" and "can we schedule all activities with K
  rooms" are the same underlying computation.
- Q8: No — value-to-weight ratio greedy is correct for the *fractional*
  knapsack problem (where items can be split), but fails for 0/1 knapsack
  because taking a high-ratio item might use up capacity that would have
  allowed two other, individually lower-ratio items whose combined value
  exceeds it; 0/1 Knapsack requires Dynamic Programming (Chapter 14)
  precisely because greedy's local ratio-based choice isn't provably safe
  under the all-or-nothing constraint.
- Q9: False — passing given test cases only shows the algorithm is
  correct on those specific inputs; without a genuine correctness argument
  (exchange argument, greedy-choice-property proof, or exhaustive case
  analysis), an untested edge case can still break it, exactly as the coin
  denomination counter-example demonstrates for a greedy approach that
  "looks obviously right."

</details>

---

## 13.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why must Activity Selection sort by end time specifically, not start time?
- **Coding Check:** Implement "Gas Station" (Q6 above) using the single-pass greedy reset technique.
- **Explanation Check:** Explain in English, using the coin-denomination counter-example, why "it passed my test cases" is not proof a greedy algorithm is correct.
- **Real-World Check:** A video-editing tool needs to select the maximum number of non-overlapping clips from a timeline to fit into a montage. Map this directly to a chapter pattern.
- **Senior Check:** When would you deliberately choose a greedy approximation over an exact DP solution, even knowing greedy might not be globally optimal?
- **Master Check:** Design "Minimum Number of Arrows to Burst Balloons" (find the minimum number of arrows, each fired straight up, needed to pop all balloons represented as intervals) using a greedy approach — identify exactly which sort key and greedy rule apply, and how this problem relates to (but differs from) Activity Selection.

<details><summary>Answers</summary>

- Real-World Check: Directly the Activity Selection pattern — sort clips
  by end time, greedily select the earliest-ending non-conflicting clip
  repeatedly, maximizing the count of clips included.
- Senior Check: When the problem is large-scale (DP's polynomial time
  might still be too slow, e.g., NP-hard variants where DP itself doesn't
  scale), when an approximate/good-enough answer is acceptable business-wise
  (e.g., a real-time recommendation or resource-allocation heuristic where
  speed matters more than perfect optimality), or when a proven
  approximation-ratio guarantee exists for the greedy approach even without full optimality.
- Master Check: Sort balloons by end coordinate; greedily fire an arrow at
  the end of the first (earliest-ending) balloon, then skip all balloons
  that overlap with that arrow's position, and fire the next arrow at the
  next remaining balloon's end — this is structurally identical to
  Activity Selection's greedy rule (sort by end, greedily pick), just
  reframed as "count arrows needed" instead of "count intervals kept,"
  showing how the same underlying greedy template solves a differently-worded problem.

</details>

---

## 13.8 CHEAT SHEET

| Problem shape | Pattern | Correctness note |
|---|---|---|
| Max non-overlapping intervals | Greedy, sort by END time | Exchange argument proof |
| Reachability with variable jump length | Greedy "farthest reach so far" | No need to explore every path |
| Coin change, arbitrary denominations | Greedy FAILS — use DP (Ch. 14) | Classic counter-example: `[1,3,4]`, target 6 |
| 0/1 Knapsack | Greedy (ratio-based) FAILS — use DP (Ch. 14) | Works only for fractional knapsack |
| Circular resource feasibility (Gas Station) | Greedy single-pass with reset on negative balance | O(n), no per-start simulation needed |

---

*(Continues to Chapter 14 — Dynamic Programming.)*
