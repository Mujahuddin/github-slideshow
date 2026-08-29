# CHAPTER 11 — TRIE

---

## 11.1 CONCEPT: Why Trie Exists — Beyond HashMap for Strings

### TELUGU EXPLANATION

Chapter 2 లో మనం `HashSet<String>` వాడి "ఈ word ఉందా?" అని O(1) (average)
లో చెక్ చేశాం. కానీ **"X తో మొదలయ్యే words అన్నీ చూపించు"** (prefix
query, autocomplete) అనే ప్రశ్న HashSet తో **సమర్థవంతంగా చేయలేము** —
HashSet కి "prefix" అనే concept తెలియదు, మీరు ప్రతి word ని individually
check చేయాలి, O(n) (n = total words).

**Trie (Prefix Tree)** ఈ సమస్యని పరిష్కరిస్తుంది — ఇది words ని
**shared prefixes ఆధారంగా ఒక tree structure** లో store చేస్తుంది. ప్రతి
node ఒక character ని సూచిస్తుంది, root నుండి ఒక node వరకు path ఒక prefix
ని సూచిస్తుంది.

```java
class TrieNode {
    Map<Character, TrieNode> children = new HashMap<>(); // లేదా TrieNode[26] fixed alphabet కి
    boolean isEndOfWord = false; // ఈ node వద్ద ఒక పూర్తి word ముగుస్తుందా
}

class Trie {
    private final TrieNode root = new TrieNode();

    // O(L) time, L = word length
    void insert(String word) {
        TrieNode current = root;
        for (char c : word.toCharArray()) {
            current = current.children.computeIfAbsent(c, k -> new TrieNode());
        }
        current.isEndOfWord = true;
    }

    // O(L) time
    boolean search(String word) {
        TrieNode node = findNode(word);
        return node != null && node.isEndOfWord;
    }

    // O(P) time, P = prefix length — ఇదే Trie యొక్క అసలైన power
    boolean startsWith(String prefix) {
        return findNode(prefix) != null;
    }

    private TrieNode findNode(String s) {
        TrieNode current = root;
        for (char c : s.toCharArray()) {
            current = current.children.get(c);
            if (current == null) return null;
        }
        return current;
    }
}
```

**Design note:** `Map<Character, TrieNode>` (unbounded alphabet, e.g.
Unicode) vs `TrieNode[26]` (fixed lowercase English, faster array access,
no hashing overhead) — ఇది Chapter 1 లో మనం చూసిన "fixed alphabet →
array, unbounded → HashMap" decision యొక్కే మరో application.

### ENGLISH INTERVIEW ANSWER

"A HashSet answers 'does this exact word exist' in O(1), but has no notion
of prefixes — checking 'how many words start with this prefix' would mean
scanning every stored word, O(n). A Trie restructures storage around
shared prefixes: each node represents one character, and a path from the
root represents a prefix. This makes prefix queries O(L), where L is the
length of the prefix, completely independent of how many words are stored
— which is exactly the property that makes Tries the standard structure
behind autocomplete, spell-checkers, and IP routing tables (longest-prefix
matching). The trade-off is memory: a Trie can use significantly more
space than a HashSet for the same word set, since shared prefixes are
deduplicated but each unique character-path still needs its own node."

---

## 11.2 CORE PROBLEM — WORD SEARCH II (TRIE + BACKTRACKING COMBINATION)

### PROBLEM
ఒక 2D board of characters, మరియు ఒక word list ఇచ్చినప్పుడు, board లో
(adjacent cells connect చేస్తూ) కనిపించే అన్ని words కనుక్కోండి.

### TELUGU EXPLANATION — WHY NAIVE APPROACH IS SLOW

ప్రతి word కి విడిగా board మీద backtracking search (Chapter 9 pattern)
చేస్తే — `k` words, ప్రతి దానికి board search — పదే పదే **అదే board
paths** ని re-explore చేస్తాము, words shared prefixes కలిగి ఉన్నా కూడా.

### TELUGU EXPLANATION — TRIE + BACKTRACKING COMBINATION

**కీలక insight:** అన్ని words ని ఒక్క **Trie** లో insert చేయండి. తర్వాత,
board మీద **ఒక్కసారి మాత్రమే** backtracking DFS చేయండి — ప్రతి దశలో,
"ఇప్పటివరకు ఉన్న path, Trie లో ఒక valid prefix నా?" అని check చేయండి.
prefix invalid అయితే, **వెంటనే ఆ dirction ని abandon చేయండి** (pruning
— Chapter 9 సూత్రం) — words shared prefixes కలిగి ఉంటే, వాటి common
part ఒక్కసారే explore అవుతుంది, అన్ని words కి విడివిడిగా కాదు.

```java
List<String> findWords(char[][] board, String[] words) {
    Trie trie = new Trie();
    for (String word : words) trie.insert(word);

    Set<String> result = new HashSet<>();
    for (int r = 0; r < board.length; r++) {
        for (int c = 0; c < board[0].length; c++) {
            dfs(board, r, c, trie.root, new StringBuilder(), result);
        }
    }
    return new ArrayList<>(result);
}

void dfs(char[][] board, int r, int c, TrieNode node, StringBuilder path, Set<String> result) {
    if (r < 0 || r >= board.length || c < 0 || c >= board[0].length || board[r][c] == '#') return;

    char ch = board[r][c];
    TrieNode next = node.children.get(ch);
    if (next == null) return; // PRUNING — ఇది Trie లో valid prefix కాదు, ఇక్కడితో ఆపేయండి

    path.append(ch);
    if (next.isEndOfWord) {
        result.add(path.toString());
    }

    board[r][c] = '#'; // Choose — ఈ cell ని temporarily "visited" గా mark చేయండి
    int[][] directions = {{0,1},{0,-1},{1,0},{-1,0}};
    for (int[] dir : directions) {
        dfs(board, r + dir[0], c + dir[1], next, path, result);
    }
    board[r][c] = ch; // Un-choose — Chapter 9 backtracking template
    path.deleteCharAt(path.length() - 1);
}
```

### ENGLISH INTERVIEW ANSWER

"Searching for each word independently wastes work whenever words share
prefixes — you'd re-explore the same board paths repeatedly. Building a
single Trie from all target words lets me do exactly one backtracking DFS
pass over the board, checking at each cell whether the path so far is
still a valid prefix in the Trie; the instant it isn't, I prune that
direction entirely — no more searching down a path that can't possibly
complete into any target word. This is a direct combination of two earlier
chapters: the Trie's prefix-checking (Chapter 11) driving the pruning
decision inside a backtracking template (Chapter 9) — a very common
'combine two patterns' senior-level problem shape."

---

## 11.3 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Does any word start with this prefix?" | Scans all stored words, O(n · L) | Trie lookup, O(L) regardless of word count |
| Searching for multiple target words in a shared search space | Runs a separate search per word | Builds one Trie, does one combined pruned search |
| Choosing Trie node storage | Always uses `Map<Character, TrieNode>` | Uses a fixed-size array for known small alphabets (faster, no hashing) |

---

## 11.4 COMMON MISTAKES

1. Using a Trie when only exact-match lookups are needed — a `HashSet` is
   simpler and sufficient; a Trie's benefit is specifically prefix operations.
2. Forgetting to mark `isEndOfWord` correctly, causing a stored word's
   prefix to be indistinguishable from the full word itself.
3. Not pruning early in combined Trie+backtracking searches, losing most
   of the efficiency benefit.
4. Using a fixed-size array (`TrieNode[26]`) for an unbounded/Unicode
   alphabet, silently failing or wasting massive memory per node.

---

## 11.5 INTERVIEW QUESTION BANK — CHAPTER 11

**Basic:** 1. What problem does a Trie solve that a HashSet cannot
efficiently? 2. What does a node in a Trie represent?

**Intermediate:** 3. Why is `startsWith()` O(prefix length) regardless of
how many words are stored? 4. When would you use `TrieNode[26]` instead of
`Map<Character, TrieNode>`?

**Senior:** 5. Design an autocomplete feature that returns the top 3 most
frequent words matching a prefix — how would you extend the basic Trie
structure to support this efficiently? 6. Compare the memory trade-offs of
a Trie versus a HashSet for a large dictionary with many shared prefixes
versus one with almost no shared prefixes.

**Architect:** 7. You're designing IP routing table lookups (longest-prefix
match) for a network device. How does a Trie (specifically, a bit-level
Trie over IP address bits) apply here, and why is this a genuinely
different use case from string autocomplete despite using the same
underlying structure?

**Scenario:** 8. A candidate implements a Trie-based autocomplete but it
returns results in an arbitrary, non-useful order for a large word list.
What would you suggest to make results more useful (ranked by relevance/frequency)?

**Trick:** 9. "A Trie is always more memory-efficient than a HashSet for storing a set of strings." True or false?

<details><summary>Key answers</summary>

- Q5: Store a frequency/popularity value at each `isEndOfWord` node (or
  maintain a small top-k cache at each Trie node summarizing the most
  frequent completions reachable from that point), updated on
  insert/usage — trading some extra memory and insert-time bookkeeping for
  O(prefix length) retrieval of ranked suggestions instead of collecting
  all matches and sorting them at query time.
- Q6: With heavily shared prefixes (e.g., a dictionary of English words), a
  Trie can be considerably more memory-efficient since common prefixes are
  stored once; with little to no shared structure (e.g., random UUID
  strings), a Trie can use *more* memory than a HashSet, since every
  character still needs its own node with per-node overhead (map/array)
  and there's no prefix-sharing benefit to offset it.
- Q7: IP routing tries operate over the *bits* of an IP address rather than
  characters, and the query is "longest matching prefix" (find the most
  specific matching route) rather than "does this exact/prefix string
  exist" — structurally the same prefix-tree idea, but the matching
  semantics (longest-prefix-wins) and the fixed, small "alphabet" (0/1 bits)
  make it a meaningfully different specialization of the same core idea.
- Q8: Augment the Trie with per-word frequency/popularity data (as in Q5)
  and rank results by that, rather than returning results in raw
  traversal/insertion order, which has no relationship to actual usefulness.
- Q9: False — as explained in Q6, when strings share little to no prefix
  structure, a Trie's per-character node overhead can make it less
  memory-efficient than a HashSet storing the same strings directly.

</details>

---

## 11.6 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is `startsWith()` O(prefix length) but a HashSet-based prefix check would be O(total words)?
- **Coding Check:** Implement a Trie supporting wildcard search (`.` matches any single character) using backtracking over the Trie structure.
- **Explanation Check:** Explain in English when a Trie is NOT the right choice, using the shared-prefix-vs-no-shared-prefix distinction from this chapter.
- **Real-World Check:** A search bar needs to suggest matching product names as the user types, from a catalog of 2 million products. Justify using a Trie (or an argument against it) considering both memory and query-latency requirements.
- **Senior Check:** When would you choose a HashSet + sorted-list-based prefix search (binary search for the prefix boundary) over a full Trie implementation?
- **Master Check:** Design "Replace Words" (given a dictionary of root words and a sentence, replace each word in the sentence with its shortest matching root from the dictionary, if any) using a Trie — explain why a Trie naturally finds the *shortest* matching prefix efficiently.

<details><summary>Answers</summary>

- Real-World Check: A Trie is a strong fit here — product names/searches
  typically share significant prefix structure (common brand names, common
  words), and O(prefix length) lookup latency, independent of the 2
  million-item catalog size, directly serves the low-latency-as-you-type
  requirement; the memory cost is a reasonable trade for that latency guarantee.
- Senior Check: When the word set is relatively static, memory is more
  constrained than latency requirements demand, and prefix queries are
  infrequent enough that O(log n) binary search over a sorted list (finding
  the prefix's range boundary) is an acceptable latency trade for
  significantly simpler code and lower memory overhead than a full Trie.
- Master Check: Insert all root words into a Trie; for each sentence word,
  walk the Trie character by character, and the *moment* you hit a node
  marked `isEndOfWord`, stop and use that root — since you're walking from
  the shortest prefix outward, the first `isEndOfWord` hit is guaranteed to
  be the shortest matching root, with no need to compare multiple candidate
  roots against each other afterward.

</details>

---

## 11.7 CHEAT SHEET

| Need | Structure | Complexity |
|---|---|---|
| Exact word existence only | HashSet | O(1) avg |
| Prefix existence/count queries | Trie | O(prefix length) |
| Autocomplete / ranked suggestions | Trie + frequency augmentation | O(prefix length) + ranking |
| Multi-word search over a shared space (Word Search II) | Trie + backtracking (pruned DFS) | Much faster than per-word search |
| Strings with little/no shared prefix structure | HashSet may beat a Trie on memory | — |

---

*(Continues to Chapter 12 — Graphs: BFS, DFS, Topological Sort.)*
