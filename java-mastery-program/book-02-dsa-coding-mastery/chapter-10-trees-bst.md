# CHAPTER 10 — TREES & BST

---

## 10.1 CONCEPT: Tree Traversals — The Four Fundamental Orders

### TELUGU EXPLANATION

```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}
```

నాలుగు fundamental traversal orders, ప్రతి దానికి ఒక specific use case:

- **Preorder (root → left → right):** root ని ముందు process చేయడం
  ఉపయోగపడేది — ఉదా: tree ని **serialize** చేయడం (root సమాచారం ముందు
  రాస్తే, deserialize చేసేటప్పుడు సులభం).
- **Inorder (left → root → right):** **BST** కి ప్రత్యేకంగా ఉపయోగపడేది
  — BST యొక్క inorder traversal ఎప్పుడూ **sorted order** ఇస్తుంది
  (section 10.2 చూడండి).
- **Postorder (left → right → root):** root ని **చివర** process చేయడం
  ఉపయోగపడేది — ఉదా: tree ని **delete** చేయడం (పిల్లల్ని ముందు delete
  చేసి, తర్వాత parent), లేదా **subtree-dependent computations** (height,
  diameter — పిల్లల results మీద parent ఆధారపడి ఉన్నప్పుడు).
- **Level-order (BFS, Queue వాడి):** tree ని **level-by-level** process
  చేయడానికి — ఉదా: "each level యొక్క average," "rightmost node per level."

```java
// Recursive Inorder — O(n) time, O(h) space (h = tree height, from call stack)
void inorder(TreeNode node, List<Integer> result) {
    if (node == null) return;       // Base case
    inorder(node.left, result);     // Left
    result.add(node.val);           // Root
    inorder(node.right, result);    // Right
}

// Level-order (BFS) — O(n) time, O(w) space (w = max width of tree)
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new LinkedList<>(); // Ch. 4 concept: Queue for FIFO/BFS
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size(); // ఈ level లో ఎన్ని nodes ఉన్నాయో ముందే capture చేయండి
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
```

**కీలక idiom:** `levelSize = queue.size()` ని loop మొదట్లో capture
చేయడం — ఇది "ఈ level లో ఎన్ని nodes ఉన్నాయో" fix చేస్తుంది, తర్వాత
add అయిన next-level nodes తో కలవకుండా.

### ENGLISH INTERVIEW ANSWER

"I choose the traversal order based on when I need to process the root
relative to its children. Preorder processes the root first — useful for
serialization, where you want to write structural information before its
contents. Postorder processes children before the root — essential
whenever a computation genuinely depends on results from the subtrees
first, like computing height, diameter, or safely deleting nodes bottom-up.
Inorder is special for BSTs specifically, since it visits nodes in sorted
order due to the BST property. Level-order uses a queue for BFS instead of
the call stack, and needs the 'capture queue.size() before the inner loop'
idiom to correctly separate levels — without it, nodes from the next level
get mixed into the current level's processing."

---

## 10.2 CONCEPT: BST Property and Why Inorder = Sorted

### TELUGU EXPLANATION

**BST (Binary Search Tree) property:** ప్రతి node కి, దాని **left
subtree లో ఉన్న అన్ని values చిన్నవి**, **right subtree లో ఉన్న అన్ని
values పెద్దవి**. ఇది recursively ప్రతి subtree కి కూడా వర్తిస్తుంది.

**ఎందుకు inorder traversal sorted order ఇస్తుంది:** Inorder = "left
subtree మొత్తం (అన్నీ చిన్నవి) → root → right subtree మొత్తం (అన్నీ
పెద్దవి)" — ఇది **ఖచ్చితంగా sorted order యొక్క నిర్వచనమే**, recursively
apply చేస్తే.

**Search/Insert complexity:** Balanced BST లో, ప్రతి స్థాయి decision
search space ని **half** చేస్తుంది (Binary Search లాగే) — **O(log n)**.
కానీ **unbalanced** BST లో (ఉదా: sorted data ని insert order లోనే
పెడితే, ఇది ఒక **linked list** లా degenerate అవుతుంది) — **O(n)** worst
case!

### ENGLISH INTERVIEW ANSWER

"A BST's defining invariant is that every node's left subtree contains
only smaller values and its right subtree only larger ones, recursively.
This is exactly why inorder traversal yields sorted order — you're always
visiting 'everything smaller,' then the node, then 'everything larger.'
The practical catch is that a BST's O(log n) search/insert/delete
guarantee only holds if the tree is *balanced*. Insert already-sorted data
into a plain BST and it degenerates into what's structurally a linked
list, giving O(n) operations — this is exactly the motivation for
self-balancing variants like AVL trees and Red-Black trees, which
constrain the tree's shape during insertion/deletion to guarantee O(log n)
even in adversarial insertion orders. Java's `TreeMap`/`TreeSet` are
backed by a Red-Black tree specifically for this reason."

---

## 10.3 CORE PROBLEM 1 — VALIDATE BINARY SEARCH TREE

### PROBLEM
ఒక binary tree ఇచ్చినప్పుడు, అది valid BST అవునా కాదా check చేయండి.

### TELUGU EXPLANATION — COMMON WRONG APPROACH

అమాయకంగా, "ప్రతి node కి, `node.left.val < node.val < node.right.val`"
అని మాత్రమే check చేస్తే **సరిపోదు** — ఇది కేవలం **immediate children**
తోనే compare చేస్తుంది, కానీ BST property **మొత్తం subtree** కి
వర్తించాలి (ఒక deep-nested right-left grandchild, root కంటే చిన్నదైతే
కూడా అది invalid).

### TELUGU EXPLANATION — CORRECT APPROACH (RANGE PROPAGATION)

**కీలక insight:** ప్రతి node కి ఒక **valid range** (`min`, `max`) pass
చేయండి, recursion లో — ఎడమవైపు వెళ్ళినప్పుడు `max` ని current node value
కి update చేయండి, కుడివైపు వెళ్ళినప్పుడు `min` ని update చేయండి.

```java
// O(n) time, O(h) space
boolean isValidBST(TreeNode root) {
    return validate(root, null, null);
}

boolean validate(TreeNode node, Long min, Long max) {
    if (node == null) return true; // ఖాళీ tree valid గానే count అవుతుంది
    if ((min != null && node.val <= min) || (max != null && node.val >= max)) {
        return false; // range violation
    }
    return validate(node.left, min, (long) node.val)   // ఎడమవైపు: ఇప్పటి node value కొత్త max
        && validate(node.right, (long) node.val, max); // కుడివైపు: ఇప్పటి node value కొత్త min
}
```

**Design note:** `Long` (boxed, nullable) వాడాము `int` బదులు — ఎందుకంటే
`Integer.MIN_VALUE`/`MAX_VALUE` values కూడా tree లో valid node values
కావొచ్చు, కాబట్టి "no bound yet" ని సూచించడానికి `null` కావాలి, ఒక
sentinel int value సరిపోదు.

### ENGLISH INTERVIEW ANSWER

"The common mistake is checking only immediate parent-child relationships,
which misses violations several levels down — a right child's left
grandchild could still be smaller than the original root, silently
breaking the BST property while passing a naive local check. The correct
approach propagates a valid (min, max) range down through recursion: going
left tightens the upper bound to the current node's value, going right
tightens the lower bound. I use boxed `Long` rather than primitive `int`
specifically so `null` can represent 'no bound yet,' since `Integer.MIN_VALUE`
or `MAX_VALUE` might be legitimate node values and can't double as sentinels."

**Interviewer follow-up:** "Alternative approach?" — Do an inorder
traversal and check that the resulting sequence is strictly increasing —
directly leveraging the "inorder = sorted" property from 10.2, arguably
more intuitive to explain, same O(n) complexity.

---

## 10.4 CORE PROBLEM 2 — LOWEST COMMON ANCESTOR (LCA)

### PROBLEM
ఒక binary tree లో, రెండు nodes `p` మరియు `q` యొక్క lowest common ancestor కనుక్కోండి.

### TELUGU EXPLANATION

**General Binary Tree (BST కాదు) కోసం:** **Postorder-style recursion**
వాడండి — ప్రతి subtree నుండి, "ఇందులో p దొరికిందా, q దొరికిందా, లేక
రెండూ దొరికాయా" అనే సమాచారం **తిరిగి పైకి పంపండి** (bubble up).

```java
// O(n) time, O(h) space
TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root; // base case

    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);

    if (left != null && right != null) return root; // p, q రెండూ వేర్వేరు వైపుల దొరికాయి — root ఇక్కడే LCA
    return (left != null) ? left : right; // ఒకటే వైపు దొరికితే, అదే propagate చేయండి పైకి
}
```

**BST అయితే (ఒక shortcut ఉంది):** BST property వాడి, O(h) లో, extra
space లేకుండా చేయవచ్చు — `p` మరియు `q` రెండూ root కంటే చిన్నవైతే ఎడమవైపు
వెళ్ళండి, రెండూ పెద్దవైతే కుడివైపు వెళ్ళండి, split అయితే (ఒకటి చిన్నది,
ఒకటి పెద్దది, లేదా root అయితే) — **అదే LCA**.

### ENGLISH INTERVIEW ANSWER

"For a general binary tree, I use a postorder-style search: recurse into
both subtrees, and if one call finds `p` and the other finds `q`, the
current node must be their split point — the LCA. If only one side finds
something, I propagate that result upward, since the LCA must be higher up
in that direction. This is O(n) since in the worst case I visit every node
once. If I'm told it's specifically a BST, I can do meaningfully better —
O(h) with no extra space — by using the ordering property directly: if
both targets are smaller than the current node, the LCA must be in the
left subtree; if both are larger, the right subtree; the moment they split
across (or one equals the current node), I've found the LCA without
needing to explore both subtrees at all."

---

## 10.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| BST validation | Checks only immediate parent-child relationship | Propagates valid range through recursion, or uses inorder-sorted-check |
| LCA on a BST specifically | Uses the general O(n) binary-tree algorithm | Recognizes the BST property enables an O(h) shortcut without visiting both subtrees |
| Choosing a traversal | Picks one arbitrarily | Matches the traversal to the dependency direction (top-down → preorder/BFS, bottom-up → postorder) |
| Assuming a BST guarantees O(log n) | Yes, unconditionally | Only if balanced — flags degenerate/adversarial insertion order risk |

---

## 10.6 COMMON MISTAKES

1. Validating a BST by only checking immediate children, missing deeper violations.
2. Using `int` sentinels for "no bound" in range-propagation problems
   instead of nullable boxed types, breaking on edge-case values like `Integer.MIN_VALUE`.
3. Forgetting the `levelSize = queue.size()` idiom in BFS/level-order,
   mixing levels together.
4. Assuming BST operations are always O(log n) without considering balance.
5. Confusing preorder/inorder/postorder use cases — e.g., trying to compute
   subtree-dependent values (height, diameter) with preorder instead of postorder.

---

## 10.7 INTERVIEW QUESTION BANK — CHAPTER 10

**Basic:** 1. What does inorder traversal of a BST always produce? 2.
What's the time complexity difference between balanced and unbalanced BST operations?

**Intermediate:** 3. Why does the "check immediate children only" BST
validation approach fail? 4. Explain the `queue.size()` idiom in
level-order traversal.

**Senior:** 5. Design an algorithm to compute the diameter of a binary
tree (longest path between any two nodes, not necessarily through the
root) — why must this use a postorder-style computation? 6. Why does
Java's `TreeMap` use a Red-Black tree instead of a plain BST?

**Architect:** 7. You're designing an in-memory index that must support
fast range queries (`findAll(min, max)`) on frequently-updated data. Would
you use a plain BST, a self-balancing tree, or a different structure
entirely (e.g., a skip list or B-tree)? Justify considering both update
frequency and query patterns.

**Scenario:** 8. A candidate's BST validation passes all "normal" test
cases but fails on a tree containing `Integer.MIN_VALUE` as a node value.
What's the likely bug?

**Trick:** 9. "Every binary search tree guarantees O(log n) search." True or false?

<details><summary>Key answers</summary>

- Q5: Diameter requires knowing each subtree's height to compute the
  path length "through" the current node, and the recursive height
  computation for a subtree must complete before it's useful at the parent
  — a bottom-up (postorder) traversal naturally provides this, computing
  height AND updating a running max-diameter as a side effect, in a single O(n) pass.
- Q6: A plain BST has no balance guarantee and can degrade to O(n) under
  adversarial (e.g., sorted) insertion order; a Red-Black tree enforces
  balance invariants during insertion/deletion (via rotations and
  recoloring) to guarantee O(log n) worst-case operations regardless of
  insertion order — essential for a general-purpose library class where
  callers' insertion patterns can't be controlled or assumed.
- Q7: For frequent updates with range queries, a self-balancing tree
  (Red-Black/AVL, or in Java, `TreeMap`) is usually the right in-memory
  choice for guaranteed O(log n) on all operations; a B-tree (wider nodes,
  fewer levels) becomes preferable when data doesn't fit in memory and
  disk/page I/O cost dominates (this is why databases use B-trees/B+trees
  for indexes, not binary BSTs) — the deciding factor is where the data
  lives and what dominates cost: comparisons in memory, or page reads on disk.
- Q8: Using `int` sentinel bounds (like `Integer.MIN_VALUE`/`MAX_VALUE`)
  to represent "no bound yet" collides with a legitimate node value equal
  to that sentinel — the fix is nullable boxed `Long`/`Integer` bounds,
  exactly as shown in section 10.3.
- Q9: False — only a *balanced* BST guarantees O(log n); an unbalanced
  BST (e.g., built by inserting already-sorted data) can degrade to O(n),
  behaving like a linked list.

</details>

---

## 10.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does postorder traversal suit computations where a parent depends on its children's results, while preorder does not?
- **Coding Check:** Implement "Maximum Depth of Binary Tree" using both a recursive (postorder-style) and an iterative (BFS level-counting) approach.
- **Explanation Check:** Explain in English why a BST built from already-sorted input degenerates to O(n) operations, using the tree-shape reasoning from this chapter.
- **Real-World Check:** A file-system explorer needs to compute the total size of a directory tree (each directory's size = sum of its files' sizes + all subdirectories' sizes). Which traversal order does this require, and why?
- **Senior Check:** When would you choose a Trie (Chapter 11 preview) over a BST for a set of strings, even though a BST could technically store strings too?
- **Master Check:** Design "Serialize and Deserialize a Binary Tree" (convert a tree to a string and back) — which traversal order do you use for serialization, and what marker do you need for `null` children to make deserialization unambiguous?

<details><summary>Answers</summary>

- Real-World Check: Postorder — a directory's total size genuinely depends
  on first computing all of its children's (files' and subdirectories')
  sizes, so children must be fully processed before the parent's aggregate
  can be computed, exactly the "bottom-up dependency" pattern from section 10.1.
- Senior Check: When the primary operations are prefix-based (autocomplete,
  "all words starting with X") — a Trie makes prefix queries O(length of
  prefix) directly via structure, while a BST would require string
  comparisons at each node and doesn't naturally expose the prefix-sharing
  structure a Trie is built around.
- Master Check: Preorder is the natural choice (root written first makes
  reconstructing structure straightforward), with an explicit marker
  (e.g., `"#"` or `"null"`) written for every `null` child reference —
  without that marker, the serialized string is ambiguous about where one
  subtree ends and another begins; with it, deserialization can
  reconstruct the exact original tree shape by consuming tokens in the
  same preorder sequence.

</details>

---

## 10.9 CHEAT SHEET

| Need | Traversal | Complexity |
|---|---|---|
| Sorted order from a BST | Inorder | O(n) |
| Serialize a tree | Preorder (with null markers) | O(n) |
| Subtree-dependent computation (height, diameter) | Postorder | O(n) |
| Level-by-level processing | BFS (Queue), capture `size()` per level | O(n) |
| BST validation | Range propagation (nullable bounds) OR inorder-sorted check | O(n) |
| LCA, general binary tree | Postorder-style bubble-up | O(n) |
| LCA, BST specifically | Ordering-based direct descent | O(h) |
| Balance matters for BST guarantees | Self-balancing (Red-Black/AVL); `TreeMap`/`TreeSet` in Java | O(log n) guaranteed |

---

*(Continues to Chapter 11 — Trie.)*
