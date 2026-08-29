# CHAPTER 14 — DYNAMIC PROGRAMMING

> The hardest chapter in this book for most learners — not because the
> code is complex, but because *recognizing* a DP problem and *defining
> the state* correctly is a skill that only comes from deliberate practice
> of the method taught here, not from memorizing solutions.

---

## 14.1 CONCEPT: What Makes a Problem "DP" — Two Required Properties

### TELUGU EXPLANATION

Dynamic Programming వర్తించాలంటే, ఒక problem కి **రెండు properties**
ఉండాలి:

1. **Overlapping Subproblems:** Naive recursive solution, **అదే
   sub-problem ని పదే పదే** solve చేస్తుంది. (ఉదాహరణకి, `fib(5)` compute
   చేసేటప్పుడు, `fib(3)` రెండుసార్లు, `fib(2)` మూడుసార్లు compute
   అవుతుంది — ఇది waste.)
2. **Optimal Substructure:** Problem యొక్క optimal solution, దాని
   sub-problems యొక్క optimal solutions నుండి **build** చేయవచ్చు.
   (ఇది Chapter 9 backtracking/recursion తో పంచుకునే property, కానీ
   DP దీన్ని **cache చేసిన results** తో combine చేస్తుంది.)

**రెండు DP techniques:**
- **Memoization (Top-Down):** normal recursion రాయండి, కానీ results ని
  ఒక cache (`HashMap`/array) లో store చేయండి — ఇప్పటికే compute చేసిన
  sub-problem కి, మళ్ళీ recurse చేయకుండా, cache నుండి direct గా return
  చేయండి.
- **Tabulation (Bottom-Up):** ఒక table (array) build చేయండి, చిన్న
  sub-problems నుండి మొదలుపెట్టి, పెద్ద వాటి వైపు iteratively వెళ్ళండి
  — recursion అవసరం లేదు (stack overflow risk ఉండదు, Book 1 Chapter 1
  తో direct connection).

```java
// Naive recursive Fibonacci — O(2^n) time! Massive redundant recomputation
int fibNaive(int n) {
    if (n <= 1) return n;
    return fibNaive(n - 1) + fibNaive(n - 2); // fib(n-2) ఇప్పటికే fib(n-1) call లో లెక్కించబడింది
}

// Memoization (Top-Down) — O(n) time, O(n) space (cache + recursion stack)
int fibMemo(int n, Map<Integer, Integer> cache) {
    if (n <= 1) return n;
    if (cache.containsKey(n)) return cache.get(n); // ఇప్పటికే లెక్కించాం — తిరిగి వాడండి
    int result = fibMemo(n - 1, cache) + fibMemo(n - 2, cache);
    cache.put(n, result);
    return result;
}

// Tabulation (Bottom-Up) — O(n) time, O(1) space (rolling variables, no full array needed)
int fibTabulation(int n) {
    if (n <= 1) return n;
    int prev2 = 0, prev1 = 1;
    for (int i = 2; i <= n; i++) {
        int current = prev1 + prev2;
        prev2 = prev1;
        prev1 = current;
    }
    return prev1;
}
```

### ENGLISH INTERVIEW ANSWER

"A problem qualifies for DP when it has overlapping subproblems — the
naive recursive solution recomputes the same smaller problem many times —
and optimal substructure, meaning the optimal answer can genuinely be
assembled from optimal answers to subproblems. I typically start by
writing the naive recursive solution first, since it makes the recurrence
relation explicit and correct, then add memoization — caching results
keyed by the subproblem's parameters — which turns exponential time into
polynomial time for free, just by eliminating redundant recomputation.
Once the recurrence and base cases are solid, I can often convert to a
bottom-up tabulation approach, iterating from the smallest subproblems
upward, which avoids recursion's stack depth risk entirely and often
allows further space optimization, like Fibonacci's O(1)-space rolling
variables instead of a full array."

---

## 14.2 THE 5-STEP METHOD FOR SOLVING ANY DP PROBLEM

### TELUGU EXPLANATION

ఇది ఈ chapter యొక్క **అసలైన goal** — ఒక కొత్త problem చూసినప్పుడు, DP
solution ని **derive** చేసే systematic method:

1. **Define the state:** "`dp[i]` (లేదా `dp[i][j]`) అంటే ఏమిటి?" —
   ఖచ్చితంగా, ఒక్క వాక్యంలో నిర్వచించండి. (ఉదా: "`dp[i]` = index i వద్ద
   ముగిసే longest increasing subsequence పొడవు.")
2. **Write the recurrence (transition):** `dp[i]` ని చిన్న sub-problems
   (`dp[i-1]`, `dp[j]` for j<i, మొదలైనవి) తో ఎలా express చేయాలో రాయండి.
3. **Identify the base case(s):** recursion/iteration ఎక్కడ మొదలవ్వాలి.
4. **Determine the order of computation:** ఏ క్రమంలో states compute
   చేయాలి, ప్రతి dependency అప్పటికే compute అయ్యిందని guarantee
   అయ్యేలా.
5. **Identify the final answer's location:** ఇది ఎప్పుడూ `dp[n-1]` కాదు
   — కొన్నిసార్లు `max(dp[])`, కొన్నిసార్లు `dp[n][capacity]` వంటిది.

**ఈ 5 steps ప్రతి DP problem కి consistently apply చేయండి** — ఇదే
"memorization కాకుండా, పద్ధతిగా ఆలోచించడం" అనేదానికి practical అర్థం.

---

## 14.3 CORE PROBLEM 1 — COIN CHANGE (WHERE GREEDY FAILED)

### PROBLEM
Chapter 13 లో greedy fail అయిన problem — coins, target amount ఇచ్చినప్పుడు,
కనీస coins సంఖ్యతో ఆ amount చేయగలమా అని కనుక్కోండి.

### TELUGU EXPLANATION — APPLYING THE 5-STEP METHOD

1. **State:** `dp[amount]` = ఆ `amount` ని చేయడానికి కావాల్సిన **కనీస
   coins సంఖ్య** (సాధ్యం కాకపోతే, infinity/-1).
2. **Recurrence:** ప్రతి coin `c` కి, `dp[amount] = min(dp[amount],
   dp[amount - c] + 1)` — "ఈ ఒక్క coin వాడితే, మిగతా `amount - c` ని
   optimal గా చేయడానికి ఎన్ని కావాలో, దానికి +1."
3. **Base case:** `dp[0] = 0` (సున్నా amount కి, సున్నా coins).
4. **Order:** `amount = 1` నుండి `target` వరకు (చిన్న amounts ముందు
   compute అవ్వాలి, ఎందుకంటే పెద్ద amounts వాటి మీద ఆధారపడతాయి).
5. **Answer:** `dp[target]` (అది infinity గా ఉంటే, సాధ్యం కాదని అర్థం).

```java
// O(amount × coins.length) time, O(amount) space
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // "infinity" గా వాడటానికి ఒక safe sentinel
    dp[0] = 0;

    for (int a = 1; a <= amount; a++) {
        for (int coin : coins) {
            if (coin <= a) {
                dp[a] = Math.min(dp[a], dp[a - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

**DRY RUN:** `coins = [1, 3, 4]`, `amount = 6`

| a | dp[a] (before) | check coin=1: dp[a-1]+1 | check coin=3: dp[a-3]+1 | check coin=4: dp[a-4]+1 | dp[a] final |
|---|---|---|---|---|---|
| 1 | ∞ | dp[0]+1=1 | — | — | 1 |
| 2 | ∞ | dp[1]+1=2 | — | — | 2 |
| 3 | ∞ | dp[2]+1=3 | dp[0]+1=1 | — | **1** |
| 4 | ∞ | dp[3]+1=2 | dp[1]+1=2 | dp[0]+1=1 | **1** |
| 5 | ∞ | dp[4]+1=2 | dp[2]+1=3 | dp[1]+1=2 | **2** |
| 6 | ∞ | dp[5]+1=3 | dp[3]+1=2 | dp[2]+1=3 | **2** |

**Answer: 2** (3+3) — **correctly** better than greedy's 3 (4+1+1)!

### ENGLISH INTERVIEW ANSWER

"Applying the 5-step method: the state `dp[amount]` is the minimum coins
needed for that exact amount. The recurrence tries every coin as 'the last
coin used' — for each coin, the rest of the amount needs
`dp[amount - coin]` coins optimally, plus this one. I compute bottom-up
from 0 to the target, since every `dp[amount]` depends only on smaller
amounts. This is exactly where DP succeeds where Chapter 13's greedy
approach failed — DP exhaustively considers every possible 'last coin'
choice at every amount, rather than committing to one greedy choice per
step, which is precisely the extra work needed to find the true optimum
for non-canonical denominations."

---

## 14.4 CORE PROBLEM 2 — 0/1 KNAPSACK (2D DP)

### PROBLEM
Items ఒక్కొక్కటికి weight, value ఉన్నాయి. Weight capacity `W` ఇచ్చినప్పుడు,
గరిష్ట value సాధించడానికి ఏ items ఎంచుకోవాలో కనుక్కోండి (ప్రతి item
ఒక్కసారి మాత్రమే — 0/1, split చేయలేరు; ఇదే Chapter 13 లో greedy fail
అయిన problem).

### TELUGU EXPLANATION — 5-STEP METHOD

1. **State:** `dp[i][w]` = మొదటి `i` items మాత్రమే వాడి, weight capacity
   `w` తో సాధించగల **గరిష్ట value**.
2. **Recurrence:** i-th item కి రెండు choices — **తీసుకోకపోవడం**
   (`dp[i-1][w]`), లేదా **తీసుకోవడం** (item బరువు `w` కంటే తక్కువ
   అయితేనే సాధ్యం: `value[i] + dp[i-1][w - weight[i]]`). రెండింటిలో
   **గరిష్టం** తీసుకోండి.
3. **Base case:** `dp[0][w] = 0` (items ఏవీ లేకపోతే value 0).
4. **Order:** `i` పెరుగుతూ (0 నుండి n వరకు), ప్రతి `i` కి `w` 0 నుండి W వరకు.
5. **Answer:** `dp[n][W]`.

```java
// O(n × W) time, O(n × W) space [optimizable to O(W) — see below]
int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1];

    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= capacity; w++) {
            dp[i][w] = dp[i - 1][w]; // Choice 1: ఈ item తీసుకోకపోవడం
            if (weights[i - 1] <= w) {
                dp[i][w] = Math.max(dp[i][w],
                        values[i - 1] + dp[i - 1][w - weights[i - 1]]); // Choice 2: తీసుకోవడం
            }
        }
    }
    return dp[n][capacity];
}
```

**Space optimization (O(n×W) → O(W)):** `dp[i][...]` కేవలం `dp[i-1][...]`
మీద మాత్రమే ఆధారపడుతుంది కాబట్టి, పూర్తి 2D table అవసరం లేదు — ఒక్క 1D
array ని **కుడి నుండి ఎడమకి** (right to left) iterate చేస్తే సరిపోతుంది
(ఎడమ నుండి కుడికి చేస్తే, ఒకే item ని పొరపాటున రెండుసార్లు వాడేసినట్టు
అవుతుంది — ఇది ఒక subtle, తరచుగా అడిగే follow-up).

### ENGLISH INTERVIEW ANSWER

"The state is 'best value achievable using only the first i items within
weight w.' The recurrence captures the fundamental 0/1 choice: for item i,
either skip it — value stays whatever it was without this item — or take
it, if it fits, adding its value to the best achievable with the remaining
capacity using only earlier items. This exhaustive consideration of both
choices at every state is exactly what a greedy ratio-based approach can't
replicate, since greedy commits to one choice without knowing how it
interacts with all future choices. I'd also mention the O(W) space
optimization — since row i only depends on row i-1, a single 1D array
suffices if updated right-to-left, which prevents an item from
accidentally being counted more than once within the same iteration."

---

## 14.5 CORE PROBLEM 3 — LONGEST COMMON SUBSEQUENCE (2D DP OVER TWO SEQUENCES)

### PROBLEM
రెండు strings ఇచ్చినప్పుడు, వాటి longest common subsequence (contiguous
అవ్వక్కర్లేదు, order మాత్రమే maintain చేయాలి) పొడవు కనుక్కోండి.

### TELUGU EXPలanaTION — 5-STEP METHOD

1. **State:** `dp[i][j]` = `text1` యొక్క మొదటి `i` characters మరియు
   `text2` యొక్క మొదటి `j` characters మధ్య LCS పొడవు.
2. **Recurrence:** `text1[i-1] == text2[j-1]` అయితే (characters
   match), `dp[i][j] = dp[i-1][j-1] + 1` (ఇద్దరూ ఈ character ని
   పంచుకుంటున్నారు, మిగతా భాగం మీద +1). Match కాకపోతే,
   `dp[i][j] = max(dp[i-1][j], dp[i][j-1])` (ఏదో ఒక string నుండి ఈ
   చివరి character ని విస్మరించండి, ఏది better ఫలితమిస్తే అది).
3. **Base case:** `dp[0][j] = dp[i][0] = 0` (ఏదైనా ఒక string ఖాళీ అయితే, LCS 0).
4. **Order:** `i`, `j` రెండూ పెరుగుతూ.
5. **Answer:** `dp[m][n]` (m, n = రెండు strings పొడవులు).

```java
// O(m × n) time, O(m × n) space
int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[m][n];
}
```

### ENGLISH INTERVIEW ANSWER

"This is the canonical two-string DP shape — the state tracks a prefix
length from *each* string, `dp[i][j]`. When the current characters match, I
extend the best solution from one character back on both strings. When
they don't match, the best I can do is the better of dropping the current
character from either string — I don't know in advance which string's
character is 'the one to skip,' so I try both and take the max. This exact
recurrence pattern — match extends diagonally, mismatch takes the best of
two adjacent cells — recurs across a whole family of two-string DP
problems: edit distance, longest common substring (with a small but
important tweak: reset to 0 on mismatch instead of taking the max), and sequence alignment."

---

## 14.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| New "optimal count/value" problem | Tries greedy first, hopes it works | Checks for overlapping subproblems + optimal substructure before choosing approach |
| Defining `dp[i]` | Jumps straight to code | Writes the state definition as an explicit sentence FIRST |
| 2D DP space usage | Always uses the full 2D table | Recognizes when only the previous row is needed, optimizes to O(n) space |
| LCS-style mismatch case | Guesses one direction (`dp[i-1][j]` only) | Correctly takes `max` of both adjacent options, since either string could be the one to skip |

---

## 14.7 COMMON MISTAKES

1. Jumping to code without first writing the state definition in words —
   this is the single biggest source of DP bugs and confusion.
2. Getting the order of computation wrong, referencing a `dp[]` cell that
   hasn't been computed yet.
3. In 0/1 Knapsack space optimization, iterating the 1D array left-to-right
   instead of right-to-left, accidentally allowing an item to be used multiple times.
4. Confusing "subsequence" (order preserved, not necessarily contiguous)
   with "substring"/"subarray" (must be contiguous) — using the wrong
   recurrence shape for the wrong variant.
5. Not recognizing that a problem is DP at all, and instead attempting a
   greedy or brute-force backtracking solution well past the point where
   the exponential blowup should have been a signal.

---

## 14.8 INTERVIEW QUESTION BANK — CHAPTER 14

**Basic:** 1. What two properties must a problem have for DP to apply? 2.
What's the difference between memoization and tabulation?

**Intermediate:** 3. Walk through the 5-step method applied to "Climbing
Stairs" (count ways to reach step n, moving 1 or 2 steps at a time). 4.
Why does 0/1 Knapsack's space optimization require iterating right-to-left?

**Senior:** 5. Design "Edit Distance" (minimum operations — insert,
delete, replace — to convert one string to another) using the LCS
recurrence shape as a starting point — what changes? 6. Explain why
Longest Increasing Subsequence has an O(n²) DP solution AND an O(n log n)
solution using binary search — what's the key insight behind the faster one?

**Architect:** 7. You're designing a route-optimization feature that must
find the minimum-cost way to visit a sequence of delivery stops with
various constraints (time windows, vehicle capacity). At what point does a
DP-based approach become computationally infeasible (state space
explosion), and what alternative approaches (heuristics, approximation
algorithms, ILP solvers) would you consider for real-world scale?

**Scenario:** 8. A candidate's Coin Change solution uses recursion with
memoization but the interviewer asks "what if `amount` is 10 million?"
What's the concern, and how does bottom-up tabulation address it?

**Trick:** 9. "Every problem with optimal substructure can be solved with
DP." True or false?

<details><summary>Key answers</summary>

- Q3: State: `dp[i]` = number of ways to reach step i. Recurrence:
  `dp[i] = dp[i-1] + dp[i-2]` (arrived via a 1-step or a 2-step move).
  Base cases: `dp[0]=1, dp[1]=1`. Order: increasing i. Answer: `dp[n]` —
  structurally identical to Fibonacci, illustrating how the same
  recurrence shape recurs across "seemingly different" problems.
- Q5: Edit Distance uses a similar `dp[i][j]` shape (prefixes of both
  strings), but the recurrence has three options on mismatch instead of
  two: insert (`dp[i][j-1]+1`), delete (`dp[i-1][j]+1`), or replace
  (`dp[i-1][j-1]+1`) — taking the minimum of all three, plus the same
  diagonal "match, no cost" case as LCS when characters are equal.
- Q6: The O(n log n) approach maintains a list representing "the smallest
  possible tail value for an increasing subsequence of each length seen so
  far," using binary search to find where each new number fits — this
  works because that list is always sorted, letting binary search replace
  the O(n) inner scan of the naive O(n²) DP, trading a more subtle
  invariant for a real asymptotic improvement.
- Q7: False — optimal substructure alone (without overlapping
  subproblems) just describes a problem solvable by plain
  divide-and-conquer or greedy in some cases; DP's actual advantage comes
  from *also* having overlapping subproblems to exploit via caching — a
  problem with optimal substructure but no repeated subproblems (like
  standard merge sort's structure) gains nothing from memoization.

</details>

---

## 14.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Apply the 5-step method (state, recurrence, base case, order, answer) to "House Robber" (maximize sum of non-adjacent array elements) — write out all 5 steps before coding.
- **Coding Check:** Implement "Longest Increasing Subsequence" using the O(n²) DP approach, then research and implement the O(n log n) binary-search-based approach.
- **Explanation Check:** Explain in English why writing the state definition as a sentence BEFORE coding prevents the most common class of DP bugs.
- **Real-World Check:** A text-diffing tool (like `git diff`) needs to find the minimal set of line insertions/deletions between two file versions. Map this to a chapter pattern.
- **Senior Check:** When would you choose memoization (top-down) over tabulation (bottom-up), or vice versa, in production code?
- **Master Check:** Design "Word Break" (given a string and a dictionary, determine if the string can be segmented into dictionary words) using DP — define the state, recurrence, and explain how this connects back to Chapter 11's Trie for an optimized dictionary-lookup implementation.

<details><summary>Answers</summary>

- Knowledge Check: State: `dp[i]` = max sum robbable from the first i
  houses. Recurrence: `dp[i] = max(dp[i-1], dp[i-2] + nums[i-1])` (skip
  house i, or rob it and add to the best from two houses back, since
  adjacent houses can't both be robbed). Base cases: `dp[0]=0, dp[1]=nums[0]`.
  Order: increasing i. Answer: `dp[n]`.
- Real-World Check: This is essentially the Longest Common Subsequence
  problem in reverse framing — the lines that DON'T need to change form
  the LCS between the two file versions, and everything else becomes an
  insertion/deletion in the diff output.
- Senior Check: Memoization (top-down) is often easier to write correctly
  first, especially when not all subproblems are actually needed for a
  given input (it only computes what's reached) and when the recursive
  structure closely mirrors the problem statement; tabulation (bottom-up)
  is preferred in production when recursion depth could risk a
  `StackOverflowError` (Book 1 Chapter 1) on large inputs, and when every
  subproblem will be needed anyway, since it also usually enables further
  space optimization.
- Master Check: State: `dp[i]` = true if the substring `s[0..i)` can be
  fully segmented into dictionary words. Recurrence: `dp[i] = true` if
  there exists some `j < i` where `dp[j]` is true AND `s[j..i)` is in the
  dictionary. Base case: `dp[0] = true` (empty prefix trivially
  segmentable). A Trie (Chapter 11) makes the "is `s[j..i)` in the
  dictionary" check efficient, especially when checking many substrings
  against a large dictionary, by walking the Trie incrementally rather
  than doing repeated HashSet lookups per substring.

</details>

---

## 14.10 CHEAT SHEET

| Problem shape | Pattern | Key recurrence idea |
|---|---|---|
| 1D sequence, decision at each step | 1D DP | `dp[i]` depends on `dp[i-1]`, `dp[i-2]`, etc. |
| Two sequences compared | 2D DP | `dp[i][j]` over prefixes of both |
| Item selection under a capacity constraint | Knapsack-style 2D DP | Take-or-skip choice per item |
| "Minimum operations to transform" | Edit-distance-style 2D DP | 3-way min (insert/delete/replace) + diagonal match |
| Naive recursion is exponential due to repeated calls | Add memoization (top-down cache) | Same recurrence, cached |
| Need to avoid recursion depth / want max speed | Convert to tabulation (bottom-up) | Iterative, often space-optimizable |
| Greedy "felt right" but failed a counter-example | DP is very likely the correct tool | Exhaustively considers all choices per state |

---

*(Continues to Chapter 15 — Bit Manipulation & Disjoint Set Union.)*
