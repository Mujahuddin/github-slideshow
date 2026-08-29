# CHAPTER 9 — RECURSION & BACKTRACKING

---

## 9.1 CONCEPT: Recursion Fundamentals — Trusting the Recursive Leap

### TELUGU EXPLANATION

Recursion అర్థం చేసుకోవడంలో అతిపెద్ద mental block: **"మొత్తం execution ని
మనసులో trace చేయాలని ప్రయత్నించడం."** బదులుగా, **"recursive leap of
faith"** అనే mindset వాడాలి: "నా function సరిగ్గా పని చేస్తుందని
అనుకుందాం **చిన్న inputs కోసం** (base case దగ్గర), ఇప్పుడు దాన్ని
వాడి, **ప్రస్తుత, పెద్ద problem** ని ఎలా solve చేయాలో మాత్రమే
ఆలోచించాలి."

ప్రతి recursive function కి రెండు తప్పనిసరి భాగాలు:
1. **Base case:** recursion ఎప్పుడు ఆగాలో (లేకపోతే infinite recursion →
   Book 1 Chapter 1 లో చూసిన `StackOverflowError`).
2. **Recursive case:** ప్రస్తుత problem ని **చిన్న sub-problem(s)** గా
   ఎలా విభజించాలో, మరియు వాటి results ని ఎలా కలపాలో.

### ENGLISH INTERVIEW ANSWER

"The mental trap with recursion is trying to trace the entire call tree by
hand. The productive mindset is the 'recursive leap of faith': assume the
function already works correctly for smaller inputs — that's what the base
case anchors — and reason only about how to combine that trusted smaller
result into a correct answer for the current, larger input. Every
recursive function needs exactly two things: a base case that terminates
the recursion, and a recursive case that reduces the problem toward that
base case. Skipping either — no base case, or a recursive case that
doesn't actually shrink the problem — is precisely what causes a
`StackOverflowError` in production, which we covered from the JVM's side
in Book 1."

---

## 9.2 CONCEPT: The Backtracking Template

### TELUGU EXPLANATION

**Backtracking** అనేది recursion యొక్క ఒక ప్రత్యేక రూపం — "**అన్ని
possible choices ని try చేయండి, ప్రతి choice తర్వాత deeper గా వెళ్ళండి,
అది dead-end అయితే వెనక్కి వచ్చి (undo చేసి) వేరే choice try చేయండి**."
దీన్ని ఒక **decision tree** గా ఊహించుకోండి — ప్రతి node ఒక "so-far"
state, ప్రతి edge ఒక choice.

**Universal template (దాదాపు అన్ని backtracking problems ఇందులోకి
ఇమిడిపోతాయి):**

```java
void backtrack(State state, List<Result> results) {
    if (isComplete(state)) {
        results.add(state.snapshot()); // ఒక valid పూర్తి solution దొరికింది
        return;
    }
    for (Choice choice : getChoices(state)) {
        if (isValid(choice, state)) {
            state.apply(choice);           // 1. Choose
            backtrack(state, results);     // 2. Explore
            state.undo(choice);            // 3. Un-choose (BACKTRACK — ఇదే కీలకమైన దశ)
        }
    }
}
```

**"Un-choose" దశ ఎందుకు అవసరం:** State ని mutate చేస్తూ recursion
వాడేటప్పుడు, ఒక branch fully explore అయిన తర్వాత, **sibling branches**
ని try చేయడానికి state ని **మునుపటి స్థితికి తిరిగి తీసుకురావాలి** —
లేకపోతే మిగతా choices తప్పుడు (polluted) state తో explore అవుతాయి.

### ENGLISH INTERVIEW ANSWER

"Backtracking is recursive exploration of a decision tree with explicit
undo. At each node, I try every valid choice, recurse deeper with that
choice applied, and — this is the step people forget — undo the choice
before trying the next sibling, so each branch starts from a clean,
correct state rather than one polluted by a previous branch's choices. The
'choose, explore, un-choose' triad is the universal skeleton; nearly every
backtracking problem — permutations, combinations, N-Queens, Sudoku
solvers — is this same template with a different `getChoices` and
`isValid` function plugged in."

---

## 9.3 CORE PROBLEM 1 — SUBSETS (POWER SET)

### PROBLEM
ఒక array ఇచ్చినప్పుడు, దాని అన్ని possible subsets (power set) generate చేయండి.

### TELUGU EXPLANATION

**Decision tree framing:** ప్రతి element దగ్గర, రెండు choices — "**ఈ
element ని subset లో చేర్చాలా, వద్దా?**" — `n` elements కి, `2ⁿ` total
subsets (ప్రతి element కి binary choice).

```java
// Time O(n · 2^n) [2^n subsets, each up to O(n) to copy], Space O(n) recursion depth
List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

void backtrack(int[] nums, int start, List<Integer> current, List<List<Integer>> result) {
    result.add(new ArrayList<>(current)); // ప్రతి state ఒక valid subset — ఇక్కడ isComplete ఎప్పుడూ true

    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);                          // Choose
        backtrack(nums, i + 1, current, result);        // Explore
        current.remove(current.size() - 1);             // Un-choose
    }
}
```

**Design note:** `result.add(new ArrayList<>(current))` — **తప్పకుండా
కొత్త copy చేయాలి**, `current` reference ని direct గా add చేయకూడదు —
ఎందుకంటే `current` mutable గా ఉంటుంది, తర్వాత మారుతూనే ఉంటుంది; direct
reference add చేస్తే, అన్ని results ఒకే (చివరికి empty అయిన) list ని
point చేస్తాయి — ఇది **అత్యంత సాధారణ backtracking bug**.

### ENGLISH INTERVIEW ANSWER

"I frame this as a decision tree where every recursive call already
represents a valid subset — the current partial list, at any point in the
recursion, is itself one of the answers. So I add a defensive copy of
`current` at the top of every call, then continue trying to extend it with
each remaining element. The `start` index prevents revisiting earlier
elements, which is what avoids generating duplicate subsets in different
orders. The single most common bug here is adding `current` directly
instead of a copy — since `current` is mutated throughout the recursion,
every entry in the result list would end up referencing the same
underlying list, which becomes empty by the time recursion finishes."

---

## 9.4 CORE PROBLEM 2 — PERMUTATIONS

### PROBLEM
ఒక array యొక్క అన్ని permutations generate చేయండి.

### TELUGU EXPLANATION

Subsets కి భిన్నంగా, ఇక్కడ **order ముఖ్యం**, మరియు ప్రతి permutation
లో **అన్ని** elements ఉండాలి. కీలక వ్యత్యాసం: ఏ element ని ఎప్పుడైనా
ఎంచుకోవచ్చు (`start` index వాడం), కానీ ఇప్పటికే వాడిన elements ని
**మళ్ళీ వాడకూడదు** — దీనికి ఒక `used[]` boolean array (లేదా
`Set<Integer>`) track చేయాలి.

```java
// Time O(n · n!), Space O(n) recursion depth + O(n) for used[]
List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, new ArrayList<>(), new boolean[nums.length], result);
    return result;
}

void backtrack(int[] nums, List<Integer> current, boolean[] used, List<List<Integer>> result) {
    if (current.size() == nums.length) {
        result.add(new ArrayList<>(current)); // ఇక్కడే isComplete
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue; // ఇప్పటికే వాడేసిన element — skip
        used[i] = true;
        current.add(nums[i]);              // Choose
        backtrack(nums, current, used, result); // Explore
        current.remove(current.size() - 1);
        used[i] = false;                   // Un-choose — ఇది లేకపోతే bug!
    }
}
```

### ENGLISH INTERVIEW ANSWER

"Permutations differ from subsets in two ways: the base case only triggers
when the current arrangement includes every element, and instead of a
`start` index to prevent revisiting, I track *which specific elements*
have already been used in this branch with a `used[]` array, since any
remaining element can be chosen next regardless of position. The undo step
here is doubly important — both removing from `current` and resetting
`used[i] = false` — forgetting the second one is a very common bug that
silently makes later branches think an element is permanently used up."

**Interviewer follow-up:** "Handle duplicate elements without duplicate
permutations in the output" — sort the array first, then skip a choice at
the same recursion depth if it equals the previous sibling choice AND that
previous choice was already fully explored (`!used[i-1]`) — a classic,
frequently-tested extension of this exact template.

---

## 9.5 CORE PROBLEM 3 — N-QUEENS (CONSTRAINT SATISFACTION)

### PROBLEM
`n×n` chessboard మీద `n` queens ని ఏ రెండు queens ఒకదానికొకటి attack
చేయకుండా (అదే row, column, diagonal లో లేకుండా) పెట్టండి.

### TELUGU EXPLANATION

ఇది backtracking యొక్క "**pruning**" (early exit) power ని బాగా చూపిస్తుంది.
**కీలక insight:** ఒక్కో row కి **ఒక్కటే queen** ఉండాలి కాబట్టి, ప్రతి
row కి "ఏ column లో పెట్టాలో" మాత్రమే decide చేయాలి — దీనివల్ల problem
size drastically తగ్గుతుంది (row-by-row placement, column/diagonal
conflicts check చేస్తూ).

```java
List<List<String>> solveNQueens(int n) {
    List<List<String>> result = new ArrayList<>();
    int[] queenColumns = new int[n]; // queenColumns[row] = ఆ row లో queen ఉన్న column
    backtrack(0, n, queenColumns, result);
    return result;
}

void backtrack(int row, int n, int[] queenColumns, List<List<String>> result) {
    if (row == n) {
        result.add(buildBoard(queenColumns, n)); // అన్ని rows కి queen దొరికింది
        return;
    }
    for (int col = 0; col < n; col++) {
        if (isValidPlacement(queenColumns, row, col)) { // ముఖ్యమైన PRUNING దశ
            queenColumns[row] = col;         // Choose
            backtrack(row + 1, n, queenColumns, result); // Explore
            // Un-choose అవసరం లేదు — తర్వాత iteration queenColumns[row] ని overwrite చేస్తుంది
        }
    }
}

boolean isValidPlacement(int[] queenColumns, int row, int col) {
    for (int prevRow = 0; prevRow < row; prevRow++) {
        int prevCol = queenColumns[prevRow];
        if (prevCol == col) return false;                          // అదే column
        if (Math.abs(prevCol - col) == Math.abs(prevRow - row)) return false; // అదే diagonal
    }
    return true;
}
```

### ENGLISH INTERVIEW ANSWER

"N-Queens is the textbook example of backtracking's real power: pruning.
Instead of trying all `n^n` row/column combinations blindly, I place one
queen per row and, before recursing into the next row, immediately check
whether the placement conflicts with any already-placed queen — column
match or diagonal match. This check prunes entire invalid subtrees before
ever exploring them, which is the difference between a naive exponential
blowup and a still-exponential-but-practically-fast solution for
reasonable board sizes. I also point out that here, unlike Subsets and
Permutations, there's no need to explicitly undo `queenColumns[row]` since
the very next loop iteration simply overwrites it before it's read again —
a small but real detail that shows attentiveness to what 'undo' actually requires."

---

## 9.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Generating all combinations/subsets/permutations | Writes nested loops for small fixed sizes, doesn't generalize | Reaches for the backtracking template immediately, parametrized by `getChoices`/`isValid` |
| Adding a partial result to the output list | `results.add(current)` | `results.add(new ArrayList<>(current))` — defensive copy |
| N-Queens / constraint problems | Generates all placements, then filters valid ones | Prunes during generation — checks validity before recursing deeper |
| Estimating recursion feasibility | Doesn't consider recursion depth vs. Book 1's stack size warnings | Connects backtracking depth to `StackOverflowError` risk for pathological inputs |

---

## 9.7 COMMON MISTAKES

1. Adding a mutable reference to the results list instead of a defensive copy.
2. Forgetting the "undo" step, silently corrupting sibling branches' starting state.
3. Missing or incorrect base case, causing infinite recursion.
4. Not pruning early when a partial state is already invalid — exploring
   the full subtree before checking validity wastes enormous amounts of
   work in problems like N-Queens or Sudoku.
5. Confusing "subsets" (order doesn't matter, `start` index prevents
   revisiting) with "permutations" (order matters, `used[]` prevents reuse)
   — using the wrong template shape for the wrong problem.

---

## 9.8 INTERVIEW QUESTION BANK — CHAPTER 9

**Basic:** 1. What are the two required parts of any recursive function?
2. What does "backtrack" mean in the choose/explore/un-choose template?

**Intermediate:** 3. Why must you defensive-copy a partial result before
adding it to the results list? 4. Explain the structural difference
between the Subsets and Permutations backtracking templates.

**Senior:** 5. In N-Queens, explain why checking validity *before*
recursing (pruning) is asymptotically important, not just a style choice.
6. Design "Combination Sum" (find all unique combinations summing to a
target, elements reusable) — how does allowing reuse change the recursive
call compared to Subsets?

**Architect:** 7. You're designing a constraint-solving feature (e.g., a
scheduling system that must assign shifts to employees under complex
availability constraints) — how does the backtracking-with-pruning
template generalize to this real business problem, and where would you
add heuristics (like trying the most-constrained variable first) to make
it practical at real scale?

**Scenario:** 8. A candidate's permutation generator produces correct
individual permutations, but the final results list ends up containing
several identical (and wrong) entries. What's the most likely bug?

**Trick:** 9. "Backtracking always has exponential time complexity, so
it's never suitable for production code." True or false?

<details><summary>Key answers</summary>

- Q6: In Combination Sum, since elements can be reused, the recursive call
  for "include this element" passes the *same* index forward (not `i+1`)
  so the same element can be chosen again in a later step, while "skip this
  element" still advances to `i+1` — the key structural change from
  Subsets is that "choose" doesn't necessarily consume the choice.
- Q7: Real scheduling/CSP systems add heuristics on top of the same
  choose/explore/un-choose skeleton: ordering variable selection by "most
  constrained first" (fail fast on the hardest-to-satisfy assignments) and
  value ordering heuristics, plus constraint propagation (forward checking)
  to prune even more aggressively than a pure "check after full choice"
  approach — the backtracking template is the foundation, but production
  CSP solvers layer significant additional pruning/ordering logic on top.
- Q8: The classic bug — forgetting `used[i] = false` in the un-choose step
  (or adding `current` by reference instead of copying) — either causes
  elements to appear permanently "used" across sibling branches or causes
  all result entries to reference the same, eventually-emptied list.
- Q9: False — backtracking's worst-case complexity is often exponential,
  but effective pruning can make it entirely practical for real input
  sizes (N-Queens for reasonable n, Sudoku solvers, real constraint-solving
  systems in production scheduling/planning tools) — "exponential worst
  case" and "unusable in practice" are not the same thing when pruning is
  aggressive and typical inputs are far from worst case.

</details>

---

## 9.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is adding `new ArrayList<>(current)` instead of `current` itself critical in backtracking solutions?
- **Coding Check:** Implement "Combination Sum" (elements reusable, find all combinations summing to target) using the backtracking template, explicitly noting the reuse-enabling change from Subsets.
- **Explanation Check:** Explain in English, using N-Queens, why "prune before recursing" is asymptotically better than "generate everything, then filter."
- **Real-World Check:** A form-validation wizard needs to explore different valid combinations of optional add-ons a customer can select within a budget constraint. Map this to the backtracking template — what are the choices, and what's the pruning condition?
- **Senior Check:** When would you choose an iterative/dynamic-programming reformulation over backtracking for a problem that backtracking could technically solve?
- **Master Check:** Design a Sudoku solver using backtracking — describe the choices, the validity check, and where pruning happens, and estimate (in general terms) why this remains practical despite a theoretically enormous search space.

<details><summary>Answers</summary>

- Real-World Check: Choices = "include this add-on or not" (Subsets-style
  template); pruning condition = "if the running total already exceeds
  budget, stop exploring this branch further" — an early-exit pruning check
  applied the same way N-Queens prunes on column/diagonal conflicts.
- Senior Check: When the problem has overlapping subproblems and optimal
  substructure (the hallmarks of DP, Chapter 14) — e.g., "count the number
  of ways" or "find the minimum/maximum" over choices, rather than
  "enumerate all valid configurations" — backtracking enumerates
  everything, which is wasteful when a DP table can compute an aggregate
  answer without ever materializing every individual configuration.
- Master Check: Choices = which digit (1-9) to place in the next empty
  cell; validity check = digit not already present in the same row,
  column, or 3x3 box; pruning happens at every single cell placement
  (checked before recursing to the next empty cell), which in practice
  eliminates the overwhelming majority of the theoretical search space
  almost immediately — real Sudoku puzzles are solvable in milliseconds
  despite a nominally astronomical total configuration count, precisely
  because constraint violations are caught extremely early and often.

</details>

---

## 9.10 CHEAT SHEET

| Problem shape | Pattern | Key structural detail |
|---|---|---|
| All subsets | Backtracking + `start` index | Order doesn't matter, no reuse |
| All permutations | Backtracking + `used[]` | Order matters, no reuse |
| Combinations with reuse allowed | Backtracking, same index passed forward | Order doesn't matter, reuse allowed |
| Constraint satisfaction (N-Queens, Sudoku) | Backtracking + prune-before-recurse | Validity check BEFORE recursing, not after |
| Adding partial state to results | `new ArrayList<>(current)` | Defensive copy, always |

---

*(Continues to Chapter 10 — Trees & BST.)*
