# CHAPTER 3 — LINKED LIST

---

## 3.1 WHY THIS CHAPTER EXISTS

Linked Lists array-based problems కంటే వేరే ఆలోచనా విధానం అడుగుతాయి —
random access (`list[i]`) లేదు, కేవలం `next` reference ద్వారానే
navigate చేయాలి. ఇక్కడ నేర్చుకునే **"fast-slow pointer" (tortoise and
hare)** technique ఇతర chapters లో కూడా (Cycle detection concepts,
duplicate detection) మళ్ళీ కనిపిస్తుంది.

```java
class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
}
```

---

## 3.2 CORE PROBLEM 1 — REVERSE A LINKED LIST

### TELUGU EXPLANATION

**కీలక insight:** ప్రతి node దగ్గర, దాని `next` pointer ని **వెనక్కి
తిప్పాలి** (point to the previous node instead of the next). దీనికి
మూడు references track చేయాలి: `prev`, `current`, `next` (temporarily
save చేయడానికి, ఎందుకంటే `current.next` మార్చే ముందు అసలు next ని
save చేయకపోతే list కోల్పోతాము).

```java
// Iterative — O(n) time, O(1) space
ListNode reverseList(ListNode head) {
    ListNode prev = null;
    ListNode current = head;
    while (current != null) {
        ListNode nextTemp = current.next; // link విరిగిపోకముందు save చేయండి
        current.next = prev;              // reverse the link
        prev = current;                   // prev ని ముందుకు జరపండి
        current = nextTemp;               // current ని ముందుకు జరపండి
    }
    return prev; // ఇదే కొత్త head
}
```

**DRY RUN:** `1 -> 2 -> 3 -> null`

| Step | prev | current | current.next (before) | action |
|---|---|---|---|---|
| 0 | null | 1 | 2 | — |
| 1 | 1 | 2 | 3 | 1.next = null; prev=1, current=2 |
| 2 | 2 | 3 | null | 2.next = 1; prev=2, current=3 |
| 3 | 3 | null | — | 3.next = 2; prev=3, current=null (loop ends) |

Result: `3 -> 2 -> 1 -> null`, return `prev` (which is `3`).

**Recursive version** (space trade-off: O(n) call stack instead of O(1)):

```java
ListNode reverseListRecursive(ListNode head) {
    if (head == null || head.next == null) return head; // base case
    ListNode newHead = reverseListRecursive(head.next);  // reverse the rest first
    head.next.next = head; // ఇప్పుడు తిరిగి వచ్చిన node ని, current వైపు point చేయండి
    head.next = null;      // current యొక్క పాత forward link తీసేయండి
    return newHead;
}
```

### ENGLISH INTERVIEW ANSWER

"Reversing a linked list iteratively means walking the list while flipping
each node's `next` pointer to point backward, which requires saving the
original `next` before overwriting it, or you lose the rest of the list.
The iterative version is O(n) time and O(1) space. The recursive version is
conceptually cleaner — reverse everything after the current node first, then
fix up the current node's link — but it costs O(n) stack space, which
matters for very long lists where it can risk a `StackOverflowError`
(directly connecting back to Book 1 Chapter 1). In an interview, I'd mention
both but default to iterative for production-quality code specifically
because of that stack depth risk."

**Interviewer follow-up:** "Reverse only nodes between position m and n."
(Same technique, applied to a sub-segment — requires carefully tracking the
node just before position m to reconnect afterward.)

---

## 3.3 CORE PROBLEM 2 — DETECT A CYCLE (FLOYD'S TORTOISE AND HARE)

### TELUGU EXPLANATION — BRUTE FORCE

`HashSet<ListNode>` వాడి, ప్రతి node ని add చేస్తూ, ఇప్పటికే set లో ఉందేమో
check చేయవచ్చు — O(n) time, **O(n) space** (Chapter 2 pattern).

### TELUGU EXPLANATION — OPTIMIZATION (O(1) SPACE)

**కీలక insight:** రెండు pointers, ఒకటి **slow** (ఒక్కో step), ఒకటి
**fast** (రెండేసి steps) — cycle ఉంటే, fast pointer ఎప్పుడో ఒకసారి slow
pointer ని **catch up** చేస్తుంది (ఒక circular track లో వేగంగా పరిగెత్తే
వ్యక్తి, నెమ్మదిగా పరిగెత్తే వ్యక్తిని catch చేయడం లాగా). Cycle లేకపోతే,
fast pointer `null` కి చేరుకుంటుంది.

```java
// O(n) time, O(1) space
boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true; // కలిశారు — cycle ఉంది
    }
    return false; // fast, null కి చేరుకుంది — cycle లేదు
}
```

**Why this is O(1) space, not O(n):** No auxiliary data structure grows
with input — just two pointer variables, regardless of list length.

**Extension — find the cycle's starting node:** once `slow == fast`, reset
one pointer to `head`, then move both one step at a time; they meet exactly
at the cycle's start. This relies on the mathematical property that the
distance from `head` to the cycle start equals the distance from the
meeting point to the cycle start (going around) — a fact worth stating in
an interview rather than treating as magic.

### ENGLISH INTERVIEW ANSWER

"The HashSet approach is a correct O(n) time, O(n) space solution — track
every visited node, and if you ever revisit one, there's a cycle. Floyd's
algorithm improves space to O(1) using two pointers moving at different
speeds: if there's a cycle, the faster pointer necessarily laps the slower
one and they meet; if there's no cycle, the faster pointer simply reaches
the end. This space-for-nothing improvement — same O(n) time, but O(1)
space instead of O(n) — is exactly the kind of trade-off interviewers want
to see you push toward once the O(n)/O(n) solution is established."

---

## 3.4 CORE PROBLEM 3 — MERGE TWO SORTED LINKED LISTS

### TELUGU EXPLANATION

**కీలక insight:** రెండు sorted lists merge చేయడం, రెండు sorted arrays
merge చేయడం లాగే (Merge Sort లో ఉండే merge step) — ఒక **dummy head** node
వాడితే, "ఇది మొదటి node నా?" అనే edge case handle చేయడం సులభం అవుతుంది
(common linked-list idiom).

```java
// O(n + m) time, O(1) extra space (reusing existing nodes, not creating new ones)
ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(-1); // placeholder, result అంతా దీని తర్వాత build అవుతుంది
    ListNode tail = dummy;

    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) {
            tail.next = l1;
            l1 = l1.next;
        } else {
            tail.next = l2;
            l2 = l2.next;
        }
        tail = tail.next;
    }
    tail.next = (l1 != null) ? l1 : l2; // మిగిలిన list ని జోడించండి
    return dummy.next; // dummy ని వదిలేసి, అసలు head తిరిగి ఇవ్వండి
}
```

### ENGLISH INTERVIEW ANSWER

"This is the merge step from merge sort, adapted to linked lists. The dummy
head node is a standard idiom that eliminates special-casing 'is this the
very first node of the result' — I always build off `dummy.next` and return
that at the end. Since both inputs are already sorted, I never need to
look back — a single pass comparing the current heads of each list and
advancing the smaller one gives O(n+m) time, and because I'm relinking
existing nodes rather than allocating new ones, this is O(1) extra space."

**Interviewer follow-up:** "Merge k sorted lists" — a direct bridge to
Chapter 8 (Heap/Priority Queue): use a min-heap of size k holding the
current head of each list, repeatedly extracting the minimum and advancing
that list, giving O(n log k) instead of a naive O(nk).

---

## 3.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Cycle detection | HashSet, O(n) space, "it works" | Floyd's algorithm, O(1) space — asks "can I avoid the extra structure entirely?" |
| Reversing a list | Recursive by default (looks elegant) | Considers stack depth risk for long lists; defaults to iterative in production code |
| Merging lists | Copy both into an array, sort, rebuild list | Recognizes sortedness is already given — direct O(n+m) merge, no sorting needed |
| Edge cases | Doesn't test empty list / single node / self-loop | Always dry-runs the empty and single-element cases explicitly |

---

## 3.6 COMMON MISTAKES

1. Losing the rest of the list by overwriting `current.next` before saving
   it — the #1 linked-list reversal bug.
2. Off-by-one in fast/slow pointer loops — forgetting to check
   `fast != null && fast.next != null` (both conditions needed to avoid a
   `NullPointerException` on `fast.next.next`).
3. Forgetting to handle `null` head or single-node list as edge cases.
4. Not using a dummy head node when building/merging lists, leading to
   awkward first-node special-casing.
5. Recursive solutions on unbounded-length production data — `StackOverflowError` risk (Book 1, Chapter 1/9).

---

## 3.7 INTERVIEW QUESTION BANK — CHAPTER 3

**Basic:** 1. How do you reverse a linked list iteratively? 2. Why use a
dummy head node?

**Intermediate:** 3. Explain Floyd's cycle detection and why it works
mathematically (why must the fast pointer eventually equal the slow pointer
if a cycle exists?). 4. How would you find the middle node of a linked
list in one pass?

**Senior:** 5. Given a singly linked list, detect if it's a palindrome in
O(n) time and O(1) space (hint: combine fast/slow pointer + in-place
reversal of the second half). 6. Compare the iterative and recursive
reversal approaches — when would you actually choose recursive despite the
stack risk?

**Architect:** 7. You're designing an LRU cache (preview of Chapter 16)
that needs O(1) removal of an arbitrary node plus O(1) access. Why is a
**doubly** linked list combined with a HashMap the right structure, and
why wouldn't a singly linked list suffice?

**Scenario:** 8. A candidate's cycle-detection solution advances both slow
and fast pointers by checking `fast.next.next != null` first, then
computing `fast = fast.next.next`. What's the subtle bug?

**Trick:** 9. "Detecting a cycle always requires extra space to track visited nodes." True or false?

<details><summary>Key answers</summary>

- Q5: Find the middle with fast/slow pointers, reverse the second half
  in-place (3.2's technique), then compare the first half and reversed
  second half node-by-node — all O(n) time, O(1) space, no array copy needed.
- Q6: Recursive can be preferred for genuinely short, bounded-depth lists
  where code clarity matters more than the marginal stack usage, or in a
  language/runtime with tail-call optimization (Java's JVM does NOT
  reliably provide this, which is exactly why the stack risk is real here).
- Q7: A singly linked list only allows O(1) removal if you already have a
  reference to the *previous* node, which you often don't when removing an
  arbitrary node identified by the HashMap; a doubly linked list's `prev`
  pointer makes arbitrary-node removal O(1) without any extra traversal.
- Q8: Checking `fast.next.next != null` before advancing, but not first
  confirming `fast.next != null`, risks a `NullPointerException` when
  `fast.next` is itself `null` — the correct guard order is
  `fast != null && fast.next != null` checked *before* touching
  `fast.next.next`.
- Q9: False — Floyd's tortoise-and-hare algorithm detects cycles in O(1)
  space, no auxiliary structure needed; the HashSet approach is simpler to
  explain but not the only correct one.

</details>

---

## 3.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does the iterative reversal need a temporary variable to save `current.next`?
- **Coding Check:** Implement "remove the Nth node from the end of a list" in one pass using two pointers with a gap of N.
- **Explanation Check:** Explain in English why Floyd's algorithm is O(1) space while the HashSet approach is O(n), using the exact resource each approach allocates.
- **Real-World Check:** You're implementing a browser's back/forward navigation history. Would a singly or doubly linked list serve better, and why?
- **Senior Check:** When would a doubly linked list be worth its extra memory overhead (an extra pointer per node) over a singly linked list?
- **Master Check:** Design an algorithm to detect the *intersection point* of two singly linked lists (they merge into one list at some node) in O(n+m) time and O(1) space, without using a HashSet.

<details><summary>Answers</summary>

- Real-World Check: Doubly linked list — back/forward navigation
  fundamentally needs O(1) traversal in *both* directions from the current
  position, which a singly linked list cannot provide without re-traversing
  from the head.
- Senior Check: Whenever bidirectional traversal or O(1) arbitrary-node
  removal (given a direct node reference) is needed — LRU caches (Chapter
  16), undo/redo stacks, and the browser-history example above are all real
  cases where the extra pointer earns its cost.
- Master Check: Compute both lists' lengths, advance the pointer of the
  longer list by the length difference, then walk both pointers together
  one step at a time — they'll reach the shared intersection node
  simultaneously since both now have equal remaining distance to it. No
  extra space beyond a few pointers/counters.

</details>

---

## 3.9 CHEAT SHEET

| Problem shape | Pattern | Complexity |
|---|---|---|
| Reverse | Prev/current/next pointer walk | O(n) / O(1) iterative |
| Cycle detection | Fast/slow pointers (Floyd's) | O(n) / O(1) |
| Find middle | Fast/slow pointers | O(n) / O(1) |
| Merge sorted lists | Dummy head + two-pointer merge | O(n+m) / O(1) |
| Merge K sorted lists | Min-heap of size K (Ch. 8) | O(n log k) |
| Arbitrary O(1) removal needed | Doubly linked list + HashMap | O(1) per op |

---

*(Continues to Chapter 4 — Stack, Queue, Deque.)*
