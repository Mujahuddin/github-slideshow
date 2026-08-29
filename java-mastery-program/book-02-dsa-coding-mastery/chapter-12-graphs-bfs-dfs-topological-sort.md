# CHAPTER 12 — GRAPHS: BFS, DFS, TOPOLOGICAL SORT

---

## 12.1 CONCEPT: Graph Representations

### TELUGU EXPLANATION

Graph = nodes (vertices) + edges (connections). రెండు ప్రధాన representations:

- **Adjacency List** (`Map<Integer, List<Integer>>` లేదా `List<Integer>[]`):
  ప్రతి node కి, దాని neighbors యొక్క list. Space: O(V + E) — **sparse
  graphs** (edges తక్కువగా ఉన్నప్పుడు) కి ఇదే సరైనది, ఎందుకంటే space
  actual edges సంఖ్యకు అనుగుణంగా ఉంటుంది.
- **Adjacency Matrix** (`boolean[V][V]` లేదా `int[V][V]` weighted కి):
  Space: **O(V²)** ఎప్పుడూ, edges సంఖ్యతో సంబంధం లేకుండా. **Dense
  graphs** కి, లేదా "రెండు nodes connected అయ్యాయా?" అనే query O(1) లో
  కావాలంటే ఉపయోగపడేది.

**Senior decision rule:** చాలా real-world graphs (social networks, road
networks, dependency graphs) **sparse** గా ఉంటాయి (E << V²) — కాబట్టి
**adjacency list డిఫాల్ట్ choice**. Matrix అరుదుగా, dense graphs లేదా
frequent edge-existence queries అవసరమైనప్పుడే వాడాలి.

```java
// Adjacency list — the default choice for most problems
Map<Integer, List<Integer>> buildGraph(int[][] edges) {
    Map<Integer, List<Integer>> graph = new HashMap<>();
    for (int[] edge : edges) {
        graph.computeIfAbsent(edge[0], k -> new ArrayList<>()).add(edge[1]);
        graph.computeIfAbsent(edge[1], k -> new ArrayList<>()).add(edge[0]); // undirected అయితే రెండు వైపులా
    }
    return graph;
}
```

---

## 12.2 CORE PROBLEM 1 — BFS: SHORTEST PATH IN UNWEIGHTED GRAPH

### TELUGU EXPLANATION

**కీలక insight:** BFS **level-by-level** గా explore చేస్తుంది (Chapter
10's level-order tree traversal, graphs కి extend చేయబడింది) — దీనివల్ల,
BFS ఒక node ని **మొదటిసారి** చేరుకున్నప్పుడు, అదే **shortest path**
(unweighted edges అయితే, ప్రతి edge "cost" 1 గా అనుకుంటే).

```java
// O(V + E) time, O(V) space
int shortestPath(Map<Integer, List<Integer>> graph, int start, int target) {
    Queue<Integer> queue = new LinkedList<>();
    Set<Integer> visited = new HashSet<>();
    queue.offer(start);
    visited.add(start);
    int distance = 0;

    while (!queue.isEmpty()) {
        int levelSize = queue.size(); // Chapter 10 idiom, level-by-level
        for (int i = 0; i < levelSize; i++) {
            int node = queue.poll();
            if (node == target) return distance;
            for (int neighbor : graph.getOrDefault(node, List.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor); // ఇక్కడే mark చేయడం ముఖ్యం — queue లో పెట్టేటప్పుడే
                    queue.offer(neighbor);
                }
            }
        }
        distance++;
    }
    return -1; // చేరుకోలేకపోయాం
}
```

**కీలక bug నివారణ:** `visited` ని node ని **queue నుండి poll చేసేటప్పుడు**
కాకుండా, **queue లో offer చేసేటప్పుడే** mark చేయాలి — లేకపోతే, ఒకే node
multiple సార్లు queue లో add అయ్యే అవకాశం ఉంది (వేరే వేరే neighbors
నుండి), duplicate work మరియు incorrect distance లకు దారితీస్తుంది.

### ENGLISH INTERVIEW ANSWER

"BFS explores level by level, which is exactly why the first time it
reaches any node, that's guaranteed to be via the shortest path, when
every edge has equal weight — this doesn't hold for weighted graphs,
where Dijkstra's algorithm is needed instead. A subtle but important
correctness detail: I mark a node as visited the moment I enqueue it, not
when I dequeue it — marking on dequeue allows the same node to be added to
the queue multiple times via different neighbors before it's first
processed, which wastes work and can be a genuine correctness bug in some
variants of BFS-based problems."

---

## 12.3 CORE PROBLEM 2 — DFS: CONNECTED COMPONENTS AND CYCLE DETECTION

### TELUGU EXPLANATION

**DFS (Depth-First Search)** ఒక direction లో వీలైనంత లోతుగా వెళ్ళి,
dead-end అయితేనే వెనక్కి వస్తుంది (Chapter 9 backtracking తో సారూప్యత
ఉంది — నిజానికి DFS అనేది గ్రాఫ్ మీద backtracking యొక్క ఒక ప్రత్యేక
రూపమే).

**Connected Components:** unvisited node ఎక్కడ దొరికితే అక్కడ నుండి DFS
మొదలుపెట్టండి, అది చేరుకోగల అన్ని nodes ని ఒకే component గా mark
చేయండి. మొత్తం ఎన్నిసార్లు కొత్తగా DFS మొదలుపెట్టాల్సి వచ్చిందో అదే
components సంఖ్య.

**Cycle Detection (Directed Graph):** ఇక్కడ ఒక subtlety ఉంది — కేవలం
"visited" మాత్రమే track చేస్తే సరిపోదు (ఇది undirected గ్రాఫ్ కి
సరిపోతుంది, కానీ directed కి కాదు). Directed graph లో cycle కనుక్కోవడానికి
**మూడు states** track చేయాలి: **WHITE** (unvisited), **GRAY** (ప్రస్తుత
DFS path లో ఉంది — "in progress"), **BLACK** (పూర్తిగా process అయ్యింది).
ఒక **GRAY node** ని మళ్ళీ చేరుకుంటే — అదే **back edge**, అంటే **cycle
ఉంది** అని అర్థం.

```java
// O(V + E) time, O(V) space
boolean hasCycleDirected(Map<Integer, List<Integer>> graph, int numNodes) {
    int[] state = new int[numNodes]; // 0=WHITE, 1=GRAY, 2=BLACK
    for (int node = 0; node < numNodes; node++) {
        if (state[node] == 0 && dfs(graph, node, state)) return true;
    }
    return false;
}

boolean dfs(Map<Integer, List<Integer>> graph, int node, int[] state) {
    state[node] = 1; // GRAY — ప్రస్తుత path లో
    for (int neighbor : graph.getOrDefault(node, List.of())) {
        if (state[neighbor] == 1) return true;               // GRAY కి తిరిగి వెళ్ళాం — CYCLE!
        if (state[neighbor] == 0 && dfs(graph, neighbor, state)) return true;
    }
    state[node] = 2; // BLACK — పూర్తిగా అయిపోయింది
    return false;
}
```

### ENGLISH INTERVIEW ANSWER

"For an undirected graph, a simple visited set is enough to detect a
cycle — revisiting any already-visited node (other than the immediate
parent you came from) means a cycle. For a *directed* graph, that's not
sufficient — you need three states, not two: white (unvisited), gray
(currently on the active DFS path/recursion stack), and black (fully
processed, done). A cycle exists specifically when you encounter a gray
node — that means you've found a back edge to something still on your
current path, i.e., a real cycle — whereas reaching an already-black node
is perfectly fine, it's just a previously-explored, unrelated path merging
in, not a cycle. This exact three-color technique is what powers cycle
detection for problems like Course Schedule, where a cycle in course
prerequisites means the schedule is impossible."

---

## 12.4 CORE PROBLEM 3 — TOPOLOGICAL SORT (KAHN'S ALGORITHM)

### PROBLEM
"Course Schedule": courses మరియు prerequisites ఇచ్చినప్పుడు, అన్ని
courses ని ఒక valid order లో (prerequisites ముందు) complete చేయగలమా,
మరియు ఆ order ఏమిటి?

### TELUGU EXPLANATION

**కీలక insight:** ఇది ఒక **Directed Acyclic Graph (DAG)** మీద
**Topological Sort** — ప్రతి node కి, దాని అన్ని prerequisites (incoming
edges) ముందు వచ్చేలా ఒక linear order కనుక్కోవడం. **Kahn's Algorithm**
(BFS-based) దీన్ని ఇలా చేస్తుంది:

1. ప్రతి node యొక్క **in-degree** (ఎన్ని incoming edges ఉన్నాయో) compute
   చేయండి.
2. In-degree **0** ఉన్న nodes అన్నింటినీ (ఏ prerequisite అవసరం లేనివి) ఒక
   queue లో పెట్టండి.
3. Queue నుండి ఒక node తీసుకుని, దాన్ని result లో add చేయండి, దాని
   neighbors యొక్క in-degree ని 1 తగ్గించండి — ఏదైనా neighbor in-degree
   0 కి చేరితే, దాన్ని queue లో పెట్టండి.
4. చివరికి, result లో **అన్ని nodes రాకపోతే** — అంటే **cycle ఉంది** అని
   అర్థం (ఏవో nodes ఎప్పుడూ in-degree 0 కి చేరుకోలేదు, ఎందుకంటే అవి
   ఒకదానికొకటి ఆధారపడి ఉన్నాయి).

```java
// O(V + E) time, O(V) space
int[] findOrder(int numCourses, int[][] prerequisites) {
    Map<Integer, List<Integer>> graph = new HashMap<>();
    int[] inDegree = new int[numCourses];

    for (int[] pre : prerequisites) {
        graph.computeIfAbsent(pre[1], k -> new ArrayList<>()).add(pre[0]); // pre[1] -> pre[0]
        inDegree[pre[0]]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (inDegree[i] == 0) queue.offer(i);
    }

    int[] order = new int[numCourses];
    int index = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        order[index++] = course;
        for (int next : graph.getOrDefault(course, List.of())) {
            if (--inDegree[next] == 0) {
                queue.offer(next);
            }
        }
    }

    return (index == numCourses) ? order : new int[0]; // అన్నీ process కాకపోతే cycle ఉంది
}
```

### ENGLISH INTERVIEW ANSWER

"Kahn's algorithm is BFS applied to in-degrees: courses with zero
prerequisites can be taken immediately, so I seed the queue with those.
Whenever a course is 'completed,' I decrement the in-degree of everything
that depended on it, and anything that just hit zero becomes available
next. If I process every node this way, I've got a valid topological
order. If I finish and some nodes were never processed — their in-degree
never reached zero — that means those courses are stuck in a cycle of
mutual dependency, which is precisely how this algorithm doubles as cycle
detection for free, without needing the separate three-color DFS approach
from the previous problem. Either technique — Kahn's BFS-based approach or
DFS with post-order reversal — is a valid, commonly accepted answer; I'd
mention I know both."

---

## 12.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Choosing graph representation | Defaults to adjacency matrix (feels more "visual") | Defaults to adjacency list for sparse graphs, matrix only when justified |
| Marking visited in BFS | Marks on dequeue | Marks on enqueue — avoids duplicate queue entries |
| Cycle detection in directed graph | Uses a simple visited set (works for undirected only) | Uses three-color (white/gray/black) state tracking |
| "Order these dependent tasks" | Doesn't recognize this as a graph problem | Immediately recognizes topological sort / Kahn's algorithm |

---

## 12.6 COMMON MISTAKES

1. Using a simple visited-set for cycle detection on a **directed** graph
   — this is only correct for undirected graphs.
2. Marking visited on dequeue instead of enqueue in BFS, causing duplicate
   processing.
3. Forgetting that Kahn's algorithm implicitly detects cycles — not
   checking `index == numCourses` at the end and thus missing the
   impossible-schedule case.
4. Using DFS/BFS interchangeably without considering that only BFS
   guarantees shortest path in unweighted graphs — DFS finds *a* path, not
   necessarily the shortest.
5. Choosing an adjacency matrix by default for large sparse graphs, wasting
   O(V²) space when O(V+E) would suffice.

---

## 12.7 INTERVIEW QUESTION BANK — CHAPTER 12

**Basic:** 1. When is adjacency list preferred over adjacency matrix? 2.
Why does BFS (not DFS) guarantee shortest path in an unweighted graph?

**Intermediate:** 3. Why does directed-graph cycle detection need three
states instead of a simple visited set? 4. Explain Kahn's algorithm's
cycle-detection side effect.

**Senior:** 5. Design "Number of Islands" (count connected groups of land
cells in a grid) using DFS or BFS — explain the grid-as-graph framing. 6.
Compare Kahn's algorithm (BFS-based) and DFS-with-postorder-reversal for
topological sort — are there situations favoring one over the other?

**Architect:** 7. You're designing a build system that must determine
build order for modules with dependencies, detect circular dependencies,
and report a clear error including which modules form the cycle. How
would you extend Kahn's algorithm (which only tells you "a cycle exists")
to also identify and report the specific cycle?

**Scenario:** 8. A candidate's BFS shortest-path solution occasionally
returns a value larger than the true shortest path on graphs with many
alternate routes. What's the likely bug (tie back to section 12.2)?

**Trick:** 9. "DFS and BFS always visit nodes in the same order for a
given graph and starting node." True or false?

<details><summary>Key answers</summary>

- Q5: Treat each grid cell as a graph node, with edges to its (up to 4)
  adjacent land cells; DFS or BFS from each unvisited land cell marks its
  entire connected component as visited, and the number of times you have
  to start a fresh search from an unvisited land cell is the island count
  — directly analogous to the graph Connected Components problem, just
  with implicit grid-adjacency edges instead of an explicit edge list.
- Q6: Functionally equivalent for correctness; Kahn's algorithm is often
  preferred when you also want cycle detection as a natural side effect
  and want to avoid deep recursion (BFS/queue-based, no recursion stack
  depth concern for very large/deep graphs — tying back to Book 1's
  `StackOverflowError` risk for recursive DFS on pathologically deep graphs).
- Q7: Track, for each node still stuck with nonzero in-degree at the end,
  which of its dependencies are also stuck — then run a DFS restricted to
  just the stuck subgraph to walk and report an actual cycle path, rather
  than just declaring "a cycle exists somewhere among these N modules."
- Q8: Almost certainly marking `visited` on dequeue instead of enqueue —
  this allows a node to be added to the queue multiple times through
  different paths before being processed once, and if it's processed via a
  longer path before its shorter-path entry is dequeued, incorrect
  (inflated) distances can result.
- Q9: False — DFS and BFS fundamentally differ in traversal order (depth
  vs. breadth first) and will generally visit nodes in different
  sequences, even from the same starting node on the same graph, except in
  trivial cases (like a graph that's just a simple path).

</details>

---

## 12.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is marking a node visited at enqueue time (not dequeue time) essential for BFS correctness in graphs with multiple paths to the same node?
- **Coding Check:** Implement "Number of Islands" using BFS instead of DFS, and note any complexity/practical differences.
- **Explanation Check:** Explain in English why undirected-graph cycle detection needs only a visited set, while directed-graph cycle detection needs three states.
- **Real-World Check:** A microservices dependency-mapping tool needs to detect circular service dependencies (Service A calls B calls C calls A) before allowing a deployment. Map this directly to a chapter pattern and name the algorithm you'd use.
- **Senior Check:** When would you choose DFS over BFS even for a shortest-path-style question, accepting that DFS alone doesn't guarantee shortest path?
- **Master Check:** Design "Course Schedule III" or a similar weighted-dependency variant — how does the presence of edge weights change which algorithm (BFS/Kahn's vs. Dijkstra) is appropriate, and why does Kahn's/BFS's shortest-path guarantee break down with weights?

<details><summary>Answers</summary>

- Real-World Check: This is directly the directed-graph cycle detection
  problem (section 12.3) — three-color DFS (or equivalently, Kahn's
  algorithm failing to process all services) applied to the service call
  graph, flagging a deployment as unsafe if a cycle is found.
- Senior Check: When memory is a tighter constraint than the shortest-path
  guarantee matters (DFS can use less memory than BFS's queue holding an
  entire frontier level, particularly on very wide, shallow graphs), or
  when you specifically need to enumerate *all* paths (e.g., for
  backtracking-style problems) rather than just the shortest one — DFS is
  the natural fit there regardless of shortest-path concerns.
- Master Check: BFS's shortest-path guarantee relies on every edge costing
  exactly 1 — the moment edges have different weights, a path with more
  hops can still be "shorter" in total weight than a path with fewer hops,
  which BFS's level-by-level assumption can't account for. Dijkstra's
  algorithm (a priority-queue-driven generalization, always expanding the
  currently-cheapest-known node next, directly connecting back to Chapter
  8's heap material) is the correct tool once edges carry weights.

</details>

---

## 12.9 CHEAT SHEET

| Problem shape | Pattern | Complexity |
|---|---|---|
| Shortest path, unweighted graph | BFS (mark visited on enqueue) | O(V+E) |
| Explore all reachable paths / backtracking-style graph search | DFS | O(V+E) |
| Connected components | DFS or BFS from each unvisited node | O(V+E) |
| Cycle detection, undirected | Visited set (skip immediate parent) | O(V+E) |
| Cycle detection, directed | Three-color (white/gray/black) DFS | O(V+E) |
| Dependency ordering (build order, course schedule) | Topological sort (Kahn's BFS or DFS+reverse postorder) | O(V+E) |
| Shortest path, weighted graph | Dijkstra's algorithm (heap-driven, Ch. 8 bridge) | O((V+E) log V) |
| Sparse graph representation | Adjacency list | O(V+E) space |
| Dense graph / O(1) edge-existence needed | Adjacency matrix | O(V²) space |

---

*(Continues to Chapter 13 — Greedy Algorithms.)*
