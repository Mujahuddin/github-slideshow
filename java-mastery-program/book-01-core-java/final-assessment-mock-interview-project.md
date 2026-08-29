# BOOK 1 — FINAL ASSESSMENT, MOCK INTERVIEW (CORE JAVA ROUND), AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 11 CHAPTERS)

Attempt every question before checking the answer key at the end. These are
deliberately cross-chapter — real interviews rarely stay inside one topic.

1. A `HashMap<CustomerId, Customer>` used as a cache never hits, even for
   repeated identical logical lookups. Which chapter's concept explains this,
   and what's the fix? *(Ch. 4)*
2. Why does an immutable class need defensive copying of mutable fields, and
   how does this relate to why `String` is immutable in the first place?
   *(Ch. 2, 3)*
3. Write a sealed hierarchy modeling `ApiResult<T>` as `Success<T>` or
   `Failure` (with an error code and message), and show an exhaustive
   `switch` consuming it. *(Ch. 8)*
4. A production service intermittently hangs under load with no exceptions
   logged. Name the three diagnostic tools you'd reach for, in order, and
   what each would reveal. *(Ch. 1, 9, 11)*
5. Explain why `volatile boolean running` is correct for a stop-flag but
   `volatile int count` with `count++` is not thread-safe. *(Ch. 9)*
6. Refactor this into a `CompletableFuture` pipeline with proper error
   handling: sequentially calling `fetchCustomer()`, then
   `fetchPricing(customer.getRegion())`, then combining with
   `fetchInventory()` (independent of the first two). *(Ch. 10)*
7. A code review flags `catch (Exception e) { log.error("error"); }` with no
   rethrow, no chaining, no specific type. List every problem with this line.
   *(Ch. 6)*
8. Why is `List<String>` not a subtype of `List<Object>`, and what language
   feature exists specifically to safely regain some of that flexibility?
   *(Ch. 5)*
9. Design a class hierarchy for `Employee` → `Manager`/`Engineer` that
   respects LSP, given that `Manager.approveExpense()` doesn't apply to
   `Engineer`. *(Ch. 2)*
10. Explain, using material from Chapters 1 and 11 together, why a
    Kubernetes pod can be OOM-killed even when `-Xmx` is safely below the
    pod's memory limit.

<details>
<summary>Answer Key</summary>

1. Custom key class likely overrides `equals()` without `hashCode()` (or
   vice versa), violating the HashMap contract — fix by overriding both
   consistently using the same fields (`Objects.hash(...)`).
2. Defensive copying prevents a caller from mutating internal state through
   a reference they were handed, preserving the class's invariants — the
   same underlying goal (protecting shared/referenced state from unexpected
   mutation) that motivates `String`'s immutability for pooling, security,
   and thread-safety.
3. `sealed interface ApiResult<T> permits Success, Failure {}` with
   `record Success<T>(T value) implements ApiResult<T> {}` and
   `record Failure(int code, String message) implements ApiResult<Object> {}`
   (or a non-generic `Failure` used via a shared supertype) — the exhaustive
   switch has no `default` and the compiler enforces both cases are handled.
4. `jstack` first (deadlock/thread-starvation check, no cost, no restart
   needed), GC logs / `jstat -gcutil` second (rule out GC-induced pauses),
   then JFR/CPU profiling if neither explains it (rule out a genuine
   compute bottleneck) — in that order because it's cheapest-first and
   rules out the most common causes before deeper profiling.
5. `volatile` only guarantees visibility of a single write — a boolean flag
   set once and polled is a pure single-value read/write, safe under
   `volatile` alone. `count++` is a read-modify-write of three steps;
   `volatile` makes each individual read and write visible but does nothing
   to make the three-step sequence atomic as a whole, so two threads can
   still interleave and lose an update.
6. `CompletableFuture<Customer> cf = supplyAsync(this::fetchCustomer, executor); cf.thenCompose(c -> supplyAsync(() -> fetchPricing(c.getRegion()), executor)).thenCombine(supplyAsync(this::fetchInventory, executor), this::buildResult).exceptionally(this::fallback);` — note `thenCompose` for the dependent call, `thenCombine` for the independent one, and `exceptionally` for graceful failure handling.
7. Catches the overly broad `Exception` (masking unrelated bugs), no
   exception chaining if it were to rethrow (loses root cause), generic
   unhelpful log message with no context (customer ID, operation, etc.),
   and swallows the exception entirely rather than deciding to recover,
   rethrow, or propagate — silently continuing execution as if nothing failed.
8. Generics are invariant to prevent unsound writes (e.g., inserting an
   `Integer` into what's actually a `List<String>` via an `Object`-typed
   reference) — wildcards (`List<? extends Object>`, PECS) exist
   specifically to safely regain covariant/contravariant flexibility where
   it's sound.
9. Don't put `approveExpense()` on a common `Employee` base — either give
   only `Manager` that method (client code that needs it works with
   `Manager` specifically, not the general `Employee` type), or extract an
   `ExpenseApprover` interface implemented only by `Manager`, so
   `Engineer` never has to fake or reject a method it can't honor.
10. Metaspace, thread stacks (× number of threads), JIT compiler code
    cache, and direct/native buffers all live outside the heap and thus
    outside `-Xmx`'s accounting, but still count against the container's
    total memory (RSS) that Kubernetes enforces — this is exactly why a
    JVM production checklist (Ch. 11) includes an explicit Metaspace cap
    and native memory tracking, not just a heap size.

</details>

---

## PART B — MOCK INTERVIEW: CORE JAVA ROUND

*Format: read the interviewer's question, answer out loud in English
(record yourself if possible), then compare against the model answer and
follow-ups. Do not read the model answer first.*

**Interviewer:** "Walk me through what happens when you run `java
MyApp.jar` — from the command down to your `main` method executing."

> *What's being tested:* whether you actually understand the runtime, not
> just "it runs my code." *How to think:* trace it layer by layer — process
> start → JVM launch → classloading → execution.

**Model answer:** "The `java` launcher starts a new JVM process. The JVM's
bootstrap, platform, and application classloaders load `MyApp`'s class
following the parent-delegation model — bootstrap first, for core JDK
classes, down to the application classloader for my own code. The class
goes through linking — bytecode verification, static field preparation, and
resolution — then initialization, running any static initializers. Once
`MyApp` is initialized, the JVM invokes the `main` method on the main
thread, and from there my code executes, either interpreted or JIT-compiled
as hot paths are detected."

**Common weak answer:** "It just runs the Java code" — no mention of
classloading, linking, or the JVM's role at all.

**Follow-ups:** "What would make this fail with `NoClassDefFoundError`
instead of running?" / "Where does the bytecode verifier fit into security?"

---

**Interviewer:** "You inherited a service with a suspected memory leak.
Walk me through your investigation, live, right now."

> *What's being tested:* real production instinct, not textbook GC theory.

**Model answer:** "First, I'd confirm it's actually a leak — check whether
heap usage returns to a stable baseline after GC over multiple cycles, or
keeps climbing release over release; a high-but-stable usage isn't a leak.
If it's climbing, I'd enable — or use existing — GC logs to see Old
Generation trend, then take a heap dump at an elevated point, ideally with
`-XX:+HeapDumpOnOutOfMemoryError` already configured so I get one
automatically at the moment of failure rather than guessing when to trigger
one manually. I'd load that dump into Eclipse MAT, look at the dominator
tree for the largest retained-size objects, and trace their GC roots — most
often this points to something like an unbounded static cache, a listener
never deregistered, or a `ThreadLocal` not cleared in a pooled-thread
context. I'd correlate the timeline with recent deploys before concluding
it's a long-standing issue versus a recent regression."

**Follow-up:** "The dump shows a huge `HashMap` retained by a `static`
field. What are the next three things you check?" (What's it caching, is
there any eviction, and did a recent change remove or break the eviction logic.)

---

**Interviewer:** "Two threads deadlock in production. How do you find out,
and how do you prevent it from recurring?"

**Model answer:** "I take a thread dump with `jstack` before doing anything
else, including before restarting — modern JVMs explicitly report 'Found
one Java-level deadlock' along with the exact threads and locks involved,
so I don't need to guess. To prevent recurrence, the standard fix is
consistent lock ordering — every code path that needs multiple locks
acquires them in the same globally agreed order, usually based on a stable
identifier, which makes circular wait structurally impossible rather than
just less likely."

**Follow-up:** "What if the deadlock is intermittent and hasn't happened
since the restart — do you close the incident?" (No — same reasoning as the
OOM scenario in Chapter 1: absence of recurrence isn't evidence of a fix
without an identified and corrected root cause.)

---

## PART C — CAPSTONE PROJECT ASSIGNMENT: "IN-MEMORY ORDER PROCESSING ENGINE"

**Goal:** Combine every chapter of Book 1 into one small but real system —
you should be able to explain every design decision in this project in an
interview as if you built it for production.

**Requirements:**

1. Model `Order`, `OrderStatus` (sealed interface + records: `Placed`,
   `Paid`, `Shipped`, `Cancelled` — Ch. 8), and `Money` (immutable value
   class — Ch. 2, 3).
2. Build an `OrderRepository<T, ID>` generic interface with an in-memory
   `ConcurrentHashMap`-backed implementation (Ch. 4, 5, 9).
3. Implement an `OrderProcessingException` hierarchy with proper chaining
   for failure cases (insufficient stock, invalid transition) (Ch. 6).
4. Use Streams to implement a `report()` method: total revenue by status,
   using `Collectors.groupingBy` + `Collectors.summingDouble` (Ch. 7).
5. Simulate concurrent order submissions from multiple threads via a
   bounded `ThreadPoolExecutor`, and use `CompletableFuture` to fan out
   simulated "payment verification" + "inventory check" calls in parallel
   per order before finalizing it (Ch. 9, 10).
6. Add a deliberately reproducible race condition (e.g., an unsynchronized
   order counter), demonstrate the bug under load, then fix it with
   `AtomicLong` — document both the bug and the fix (Ch. 9, 10).
7. Run your simulation with GC logging enabled
   (`-Xlog:gc*:file=order-engine-gc.log`) under two different heap sizes,
   and write a short comparison of the resulting pause behavior (Ch. 1, 11).

**Deliverable:** working Java source, a short README explaining every
design decision with a one-line justification referencing the relevant
Book 1 chapter/principle, and the two GC log excerpts with your comparison.

**Self-assessment rubric (use this to grade your own submission honestly):**

| Criterion | Signal of mastery |
|---|---|
| Immutability | `Money`/`OrderStatus` records genuinely can't be mutated after construction |
| Exception design | Every custom exception chains its cause; no bare `catch (Exception e) {}` |
| Concurrency correctness | The race-condition-then-fix section actually reproduces the bug before fixing it |
| Generics usage | `OrderRepository<T, ID>` compiles and is reused for `Order` without casting |
| Streams | `report()` uses no manual loops |
| JVM reasoning | GC log comparison draws a real conclusion, not just "logs looked different" |

---

*(This completes BOOK 1 — CORE JAVA + JAVA 8/11/17. Book 2 — DSA + Coding
Mastery — begins the next phase of the program.)*
