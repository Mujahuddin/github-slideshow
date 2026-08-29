# CHAPTER 15 — BIT MANIPULATION & DISJOINT SET UNION (DSU)

---

## PART A — BIT MANIPULATION

## 15.1 CONCEPT: Core Bitwise Operators and What They're Actually For

### TELUGU EXPLANATION

| Operator | అర్థం | సాధారణ ఉపయోగం |
|---|---|---|
| `&` (AND) | రెండు bits 1 అయితేనే 1 | ఒక bit set అయిందో లేదో check చేయడం |
| `\|` (OR) | ఏదో ఒక bit 1 అయితే 1 | ఒక bit set చేయడం |
| `^` (XOR) | bits **వేరుగా** ఉంటేనే 1 | Toggle చేయడం, **duplicates cancel చేయడం** (15.2) |
| `~` (NOT) | bits invert చేయడం | Mask create చేయడం |
| `<<` (left shift) | bits ఎడమవైపుకి జరపడం | `x << n` = `x * 2ⁿ` (fast multiply) |
| `>>` (right shift, sign-extending) | bits కుడివైపుకి జరపడం | `x >> n` = `x / 2ⁿ` (fast divide, negative numbers కి sign preserve) |
| `>>>` (unsigned right shift) | కుడివైపుకి, **సున్నాలతో fill చేస్తూ** | Negative numbers ని unsigned గా treat చేయాల్సినప్పుడు |

**Java-specific గమనిక:** `>>` vs `>>>` తేడా ఇంటర్వ్యూలో తరచుగా అడిగేది
— `>>` sign bit ని preserve చేస్తుంది (negative numbers negative గానే
ఉంటాయి), `>>>` ఎప్పుడూ 0 తో fill చేస్తుంది (negative number ని ఒక
large positive number గా మార్చేస్తుంది) — Book 1 Chapter 1 లో చూసిన
hash spreading (`hash ^ (hash >>> 16)`) ఇదే `>>>` వాడింది, sign extension
అవాంఛనీయం కాబట్టి.

---

## 15.2 CORE PROBLEM 1 — SINGLE NUMBER (THE XOR TRICK)

### PROBLEM
ఒక array లో, ప్రతి element **రెండుసార్లు** కనిపిస్తుంది, ఒక్కటి తప్ప
— ఆ ఒక్క element ని **O(1) space** లో కనుక్కోండి.

### TELUGU EXPLANATION

**కీలక insight:** XOR యొక్క రెండు properties: `a ^ a = 0` (ఒక number
తనతో XOR చేస్తే 0), మరియు `a ^ 0 = a`. కాబట్టి, array లోని అన్ని
elements ని XOR చేస్తే, **జతలుగా ఉన్న elements అన్నీ cancel అయిపోతాయి**
(`a ^ a = 0`), మిగిలింది **ఒంటరిగా ఉన్న element మాత్రమే**.

```java
// O(n) time, O(1) space — beats a HashSet-based O(n) time/O(n) space solution
int singleNumber(int[] nums) {
    int result = 0;
    for (int num : nums) {
        result ^= num; // XOR అన్ని elements తో, order పట్టింపు లేదు (XOR commutative + associative)
    }
    return result;
}
```

### ENGLISH INTERVIEW ANSWER

"A HashSet-based approach — add on first sight, remove on second sight,
whatever remains is the answer — works but costs O(n) space. XOR gives the
same O(n) time with O(1) space, exploiting two properties: XOR-ing a
number with itself yields zero, and XOR is commutative and associative, so
order doesn't matter. XOR-ing the entire array cancels every properly
paired number down to zero, leaving only the single unpaired number. This
is a genuinely elegant example of exploiting a mathematical property to
eliminate the space cost of an otherwise-correct hash-based solution."

**Variations:** "Every element appears three times except one" — requires
tracking bit counts modulo 3 per bit position, a meaningfully harder
extension of the same underlying idea, often used to test deeper bit-level
reasoning in senior interviews.

---

## 15.3 CORE PROBLEM 2 — COUNTING BITS / POWER OF TWO CHECKS

### TELUGU EXPLANATION

**`n & (n-1)` trick:** ఇది `n` యొక్క **rightmost set bit ని క్లియర్**
చేస్తుంది. దీని రెండు ఉపయోగాలు:
1. **Power of 2 check:** ఒక number power of 2 అయితేనే, దానికి **ఒక్కటే
   set bit** ఉంటుంది — కాబట్టి `n & (n-1) == 0` (మరియు `n > 0`) అయితేనే
   `n` power of 2.
2. **Count set bits (Brian Kernighan's Algorithm):** `n & (n-1)` ని
   repeatedly apply చేస్తూ, ప్రతిసారి ఒక set bit ని క్లియర్ చేస్తూ, ఎన్ని
   iterations పట్టిందో లెక్కిస్తే — అదే **set bits సంఖ్య**. ఇది naive
   "అన్ని 32 bits ని ఒక్కొక్కటిగా check చేయడం" కంటే fast — **only as many
   iterations as there are set bits**, not fixed 32.

```java
boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}

// O(number of set bits), not O(32) — better than checking every bit position
int countSetBits(int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1); // rightmost set bit క్లియర్ అవుతుంది
        count++;
    }
    return count;
}
```

### ENGLISH INTERVIEW ANSWER

"`n & (n-1)` clears the lowest set bit — subtracting 1 flips all trailing
zeros to ones and the lowest set bit to zero, and ANDing with the original
keeps only the higher bits unchanged while zeroing that lowest set bit
out. This gives two useful results: a power of two has exactly one set
bit, so `n & (n-1) == 0` detects that in O(1); and repeatedly applying this
to clear one set bit at a time counts the set bits in exactly as many
iterations as there are set bits, which is faster in practice than
checking all 32 bit positions individually when the number is sparse."

---

## PART B — DISJOINT SET UNION (UNION-FIND)

## 15.4 CONCEPT: Why DSU Exists — Efficient Dynamic Connectivity

### TELUGU EXPLANATION

Chapter 12 లో, connected components కనుక్కోవడానికి DFS/BFS వాడాము —
ఇది **static graph** (edges మారవు) కి బాగుంటుంది. కానీ **edges
dynamically add అవుతూ ఉంటే**, మరియు మీరు తరచుగా "ఈ రెండు nodes ఒకే
component లో ఉన్నాయా?" అని అడగాల్సి వస్తే — ప్రతిసారి DFS/BFS re-run
చేయడం wasteful. **Disjoint Set Union (Union-Find)** దీన్ని పరిష్కరిస్తుంది
— **almost O(1)** (technically O(α(n)), inverse Ackermann function, ఇది
practical గా ఏ input size కి అయినా ~5 కంటే తక్కువ, **constant గా
treat చేయవచ్చు**) లో రెండు operations ఇస్తుంది:

- **`find(x)`:** `x` ఏ component (root) కి చెందుతుందో కనుక్కోవడం.
- **`union(x, y)`:** రెండు components ని ఒకటిగా కలపడం.

**రెండు కీలక optimizations, ఇవి లేకుండా DSU O(n) కి degrade అవుతుంది:**

1. **Path Compression:** `find(x)` call చేసేటప్పుడు, path లో ఉన్న అన్ని
   nodes ని **నేరుగా root కి** point చేసేలా update చేయండి — తర్వాత
   `find` calls వేగంగా అవుతాయి.
2. **Union by Rank/Size:** రెండు trees ని union చేసేటప్పుడు, **చిన్న
   tree ని పెద్ద దాని కిందకి** attach చేయండి — దీనివల్ల tree height
   అనవసరంగా పెరగదు.

```java
class DisjointSetUnion {
    private final int[] parent;
    private final int[] rank;

    DisjointSetUnion(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i; // ప్రతి node తనకు తానే root
    }

    // Path compression తో — amortized O(α(n)), practically O(1)
    int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]); // path లో ఉన్న ప్రతి node ని నేరుగా root కి తీసుకురండి
        }
        return parent[x];
    }

    // Union by rank తో
    void union(int x, int y) {
        int rootX = find(x), rootY = find(y);
        if (rootX == rootY) return; // ఇప్పటికే ఒకే component లో ఉన్నారు

        if (rank[rootX] < rank[rootY]) {
            parent[rootX] = rootY;
        } else if (rank[rootX] > rank[rootY]) {
            parent[rootY] = rootX;
        } else {
            parent[rootY] = rootX;
            rank[rootX]++; // రెండూ సమాన rank అయితేనే rank పెరుగుతుంది
        }
    }

    boolean connected(int x, int y) {
        return find(x) == find(y);
    }
}
```

### ENGLISH INTERVIEW ANSWER

"DSU answers dynamic connectivity questions — 'are these two elements in
the same group' and 'merge these two groups' — far more efficiently than
re-running DFS/BFS from scratch on every query. Without optimization, a
naive implementation can degrade to O(n) per operation in the worst case,
forming a long chain. Two optimizations fix this: path compression, which
flattens the tree every time `find` is called by pointing every visited
node directly at the root, and union by rank, which always attaches the
smaller tree under the larger one's root to avoid unnecessarily increasing
height. Combined, these give amortized O(α(n)) per operation — the inverse
Ackermann function, which grows so slowly it's effectively a small
constant, under 5, for any input size that could realistically exist —
which is why DSU is treated as 'essentially O(1)' in practice."

---

## 15.5 CORE PROBLEM — NUMBER OF CONNECTED COMPONENTS (DSU VS DFS)

### PROBLEM
`n` nodes, edges list ఇచ్చినప్పుడు, connected components సంఖ్య
కనుక్కోండి.

### TELUGU EXPLANATION

Chapter 12 లో DFS/BFS తో ఇది solve చేశాం. DSU తో ఇదే ఇలా చేయవచ్చు —
**ఏ representation better అనేది context మీద ఆధారపడి ఉంటుంది**:

```java
// O(E · α(n)) time — effectively O(E)
int countComponents(int n, int[][] edges) {
    DisjointSetUnion dsu = new DisjointSetUnion(n);
    int components = n; // మొదట్లో ప్రతి node దాని own component

    for (int[] edge : edges) {
        if (dsu.find(edge[0]) != dsu.find(edge[1])) {
            dsu.union(edge[0], edge[1]);
            components--; // రెండు వేర్వేరు components కలిస్తే, total count తగ్గుతుంది
        }
    }
    return components;
}
```

### ENGLISH INTERVIEW ANSWER

"DFS/BFS and DSU both correctly solve static connected-components
counting in O(V+E) — for a one-time computation on a fixed graph, either
is fine, and DFS is often more intuitive to write. DSU becomes clearly
superior the moment edges arrive incrementally over time and I need
'current component count' or 'are X and Y connected' answered after each
new edge, without re-scanning the whole graph each time — that's the
scenario DSU is actually built for, like Kruskal's Minimum Spanning Tree
algorithm, or detecting the exact edge that first creates a cycle in an
incrementally-built graph (Redundant Connection), which is naturally
awkward to answer efficiently with repeated DFS/BFS re-runs."

---

## 15.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Find the single non-duplicate" | Reaches for HashSet, O(n) space | Recognizes the XOR trick, O(1) space |
| Dynamic connectivity (edges added incrementally, frequent queries) | Re-runs DFS/BFS on every query | Uses DSU with path compression + union by rank |
| Implementing DSU | Implements `find`/`union` without path compression or rank | Includes both optimizations, knows the amortized complexity bound |
| Bit tricks | Doesn't recognize `n & (n-1)` or XOR-cancellation patterns | Recognizes these as reusable, named techniques, not one-off cleverness |

---

## 15.7 COMMON MISTAKES

1. Implementing DSU without path compression AND without union by rank —
   degrades to O(n) per operation in adversarial cases.
2. Using `>>` when `>>>` is needed (or vice versa) on negative numbers,
   introducing sign-extension bugs.
3. Forgetting that XOR-based tricks require exact pairing (e.g., "every
   element appears twice except one" — a different multiplicity, like
   three times, needs a different technique, not a naive reapplication of
   the same XOR trick).
4. Off-by-one or missing checks in power-of-two detection (forgetting
   `n > 0`, since `n & (n-1) == 0` is also true for `n = 0`, which is not
   a power of two).
5. Re-running full DFS/BFS repeatedly for a dynamically-changing graph
   when DSU would be the appropriate, more efficient tool.

---

## 15.8 INTERVIEW QUESTION BANK — CHAPTER 15

**Basic:** 1. What does `n & (n-1)` do, and what two things is it useful
for? 2. What's the difference between `>>` and `>>>` in Java?

**Intermediate:** 3. Why does XOR-ing an entire array find the single
non-duplicate element? 4. Explain path compression and union by rank —
what happens to DSU's complexity without each?

**Senior:** 5. Extend the Single Number problem to "every element appears
three times except one" — why doesn't simple XOR work, and what
bit-counting approach does? 6. Design "Redundant Connection" (find the
edge that, if removed, makes a tree — i.e., the one edge causing a cycle
in an otherwise-tree-like graph) using DSU.

**Architect:** 7. You're designing a social network's "are these two
users in the same connected friend-group" feature, where friendships are
added continuously and this query happens frequently. Compare a DSU-based
approach to a graph-database/DFS-based approach for this at scale.

**Scenario:** 8. A candidate's DSU implementation is correct but noticeably
slow on a large test case with many union operations forming a long chain.
What optimization is most likely missing?

**Trick:** 9. "DSU's `find` operation is always O(1)." True or false?

<details><summary>Key answers</summary>

- Q5: With triplication, each bit position's count across all numbers is a
  multiple of 3 for paired numbers, plus whatever the single number
  contributes — simple XOR (which relies on pairs canceling via `a^a=0`)
  doesn't capture "count modulo 3"; the correct approach tracks, per bit
  position, the count of set bits across all numbers modulo 3, and
  reconstructs the answer bit by bit from whichever bits have a nonzero
  remainder.
- Q6: Process edges one at a time with DSU; for each edge, if both
  endpoints are already in the same component (`find` returns the same
  root), that edge is redundant — return it immediately, since adding it
  would create a cycle; otherwise, union the two components and continue.
- Q7: DSU is extremely efficient for exactly this incrementally-growing,
  frequently-queried connectivity pattern — amortized near-O(1) per
  friendship addition or query — while repeated DFS/BFS would re-traverse
  large portions of the graph on every query; at true internet scale,
  this often becomes a distributed graph database problem instead (sharding
  the social graph across machines), where DSU's single-machine,
  in-memory assumption stops holding and a different, distributed approach
  is needed — worth naming as the point where the DSU/DFS comparison stops
  being the right frame entirely.
- Q8: Missing path compression (or union by rank) — without at least one
  of these, DSU can degrade toward O(n) per `find` call on adversarial
  union orderings, exactly the long-chain symptom described.
- Q9: False — with both optimizations, `find` is amortized O(α(n)),
  effectively constant for realistic inputs but not a strict, unconditional
  O(1); without the optimizations, individual `find` calls can be O(n) in
  the worst case.

</details>

---

## 15.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does `a ^ a = 0` combined with XOR's commutativity make the Single Number solution correct regardless of array order?
- **Coding Check:** Implement DSU with path compression and union by rank from scratch, then use it to solve "Number of Provinces" (a matrix-based connected-components variant).
- **Explanation Check:** Explain in English why DSU's amortized complexity is described as O(α(n)) rather than a flat O(1), and why that distinction rarely matters in practice.
- **Real-World Check:** A distributed systems tool needs to detect when adding a new network link would create a redundant path (cycle) in a supposedly tree-shaped network topology, checked continuously as links are added. Map this to a chapter pattern.
- **Senior Check:** When would DFS/BFS-based connectivity checking still be preferable to DSU, despite DSU's better amortized complexity for repeated queries?
- **Master Check:** Design "Accounts Merge" (given account records where the same person may have multiple accounts sharing at least one email, merge all accounts belonging to the same person) using DSU — identify what the "nodes" and "union trigger" are in this problem, which isn't obviously graph-shaped at first glance.

<details><summary>Answers</summary>

- Real-World Check: Exactly the Redundant Connection pattern (15.5/Q6) —
  DSU processes each new link, and a link connecting two already-connected
  nodes is immediately flagged as creating a redundant path/cycle.
- Senior Check: When the graph is static and you only need connectivity
  information once (a single pass, not repeated dynamic queries) — DFS/BFS
  is simpler to write and reason about, and the DSU optimizations' benefits
  only pay off across many repeated operations.
- Master Check: Treat each account index as a DSU node; whenever two
  accounts share an email, union them; after processing all accounts,
  group accounts by their DSU root, and merge each group's emails together
  — the "graph" here is implicit, built from the shared-email relationship
  rather than an explicit edge list, which is exactly the kind of
  non-obvious DSU application that separates recognizing the *structure*
  of a problem from recognizing its surface-level framing.

</details>

---

## 15.10 CHEAT SHEET

| Need | Technique | Complexity |
|---|---|---|
| Find the single non-duplicate (pairs) | XOR entire array | O(n) / O(1) |
| Count set bits | Brian Kernighan's `n & (n-1)` loop | O(set bits) |
| Check power of two | `n > 0 && (n & (n-1)) == 0` | O(1) |
| Dynamic connectivity, frequent union/find queries | DSU (path compression + union by rank) | Amortized O(α(n)) ≈ O(1) |
| Static, one-time connected components | DFS/BFS (Ch. 12) — DSU also works, either is fine | O(V+E) |
| Detect the cycle-causing edge in an incremental graph | DSU (union returns "already connected") | O(E · α(n)) |

---

*(Continues to Chapter 16 — Java Production Coding: LRU Cache, Rate Limiter, Producer-Consumer, Thread Pool, Retry, Idempotency.)*
