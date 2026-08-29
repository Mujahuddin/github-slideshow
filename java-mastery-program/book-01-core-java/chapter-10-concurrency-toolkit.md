# CHAPTER 10 — CONCURRENCY TOOLKIT: EXECUTORSERVICE, COMPLETABLEFUTURE, ATOMICS

---

## 10.1 CONCEPT: Why Not Just Create `Thread` Objects Directly?

### TELUGU EXPLANATION

Junior code తరచుగా ఇలా కనిపిస్తుంది:

```java
for (Task task : tasks) {
    new Thread(() -> process(task)).start(); // ❌ ప్రతి task కి కొత్త OS thread!
}
```

ఇది production లో ప్రమాదకరం: 1) **Thread creation ఖరీదైనది** (OS-level
resource, memory stack allocate అవుతుంది — ఒక్కో thread ~1MB stack default
గా), 2) **unbounded thread creation** — 10,000 tasks వస్తే 10,000 threads,
ఇది `OutOfMemoryError` లేదా OS thread limit exhaustion కి దారితీస్తుంది,
3) thread lifecycle (start/stop/reuse) manual గా manage చేయాలి.

**`ExecutorService`** ఈ సమస్యలన్నింటినీ పరిష్కరిస్తుంది — ఇది ఒక **thread
pool** abstraction, fixed సంఖ్యలో threads reuse చేస్తుంది, tasks ని ఒక
queue లో పెట్టి, available threads కి assign చేస్తుంది.

```java
ExecutorService executor = Executors.newFixedThreadPool(10); // 10 threads reuse అవుతాయి
for (Task task : tasks) {
    executor.submit(() -> process(task)); // queue లో పడుతుంది, ఏదో ఒక pool thread pick చేస్తుంది
}
executor.shutdown(); // కొత్త tasks accept చేయదు, ఇప్పటికే submit అయినవి complete అవ్వనిస్తుంది
```

### INDUSTRY TERMINOLOGY

`ExecutorService`, `thread pool`, `ThreadPoolExecutor`, `Executors` factory
methods, `Future`, `CompletableFuture`, `submit()` vs `execute()`,
`shutdown()` vs `shutdownNow()`, `awaitTermination`.

### ENGLISH INTERVIEW ANSWER

"I never create raw `Thread` objects directly in service code — thread
creation is expensive, and unbounded thread creation under load is a real
production failure mode I've seen cause `OutOfMemoryError` from stack
allocation alone. `ExecutorService` gives me a managed, bounded thread pool
with a task queue, so I control concurrency explicitly instead of letting
load dictate how many OS threads get created. Even the `Executors` factory
methods deserve scrutiny, though — `Executors.newFixedThreadPool` uses an
unbounded `LinkedBlockingQueue` internally, which means under sustained
overload, the *queue* grows unbounded instead of the thread count, which
can still cause an OOM, just via a different path. For production code, I
usually construct `ThreadPoolExecutor` directly with an explicit bounded
queue and a rejection policy, so I can reason about and cap total memory use
under backpressure."

---

## 10.2 CONCEPT: Choosing the Right Executor / Thread Pool Size

### TELUGU EXPLANATION

- **`newFixedThreadPool(n)`:** fixed n threads, unbounded queue — CPU-bound
  tasks కి బాగుంటుంది, కానీ queue unbounded గా ఉండటం risk.
- **`newCachedThreadPool()`:** అవసరం మేరకు threads create చేస్తుంది, idle
  threads 60 seconds తర్వాత terminate అవుతాయి — **short-lived, bursty**
  tasks కి, కానీ **unbounded thread creation risk** (production లో
  dangerous, load spike వస్తే thousands of threads create అవ్వొచ్చు).
- **`newSingleThreadExecutor()`:** ఒకే thread, tasks sequentially process
  అవుతాయి — ordering guarantee కావాలంటే.
- **`ScheduledExecutorService`:** periodic/delayed tasks కోసం (`cron`-like).

**Thread pool sizing rule of thumb:**
- **CPU-bound tasks** (heavy computation, no I/O wait): `pool size ≈ number
  of CPU cores` (or cores + 1) — ఎక్కువ threads context-switching overhead
  మాత్రమే add చేస్తాయి.
- **I/O-bound tasks** (DB calls, HTTP calls, waiting most of the time):
  `pool size` ఎక్కువగా ఉండొచ్చు, ఎందుకంటే threads wait చేస్తూ CPU idle గా
  ఉంటుంది — formula: `threads = cores × (1 + wait_time/compute_time)`.

### ENGLISH INTERVIEW ANSWER

"Pool sizing depends entirely on whether the workload is CPU-bound or
I/O-bound. For CPU-bound work, I size the pool close to the number of
available cores — more threads than that just adds context-switching
overhead without more actual throughput, since the CPU is the bottleneck.
For I/O-bound work — waiting on a database or an external API — threads
spend most of their time blocked, not computing, so a much larger pool
makes sense; I use the standard formula of cores times (1 + wait time /
compute time) as a starting estimate, then tune based on actual measured
latency and throughput under load. I avoid `Executors.newCachedThreadPool()`
in production paths that face user-driven load, precisely because it can
spin up unbounded threads under a burst — I'd rather define an explicit
bounded pool and a considered rejection policy."

---

## 10.3 CONCEPT: `Future` and Its Limits

### TELUGU EXPLANATION

`executor.submit(callable)` ఒక `Future<T>` return చేస్తుంది — asynchronous
గా run అవుతున్న task యొక్క result కి ఒక "handle". `future.get()` result
దొరికేవరకు **block** అవుతుంది.

**`Future` limitations:** 1) Result ready అయినప్పుడు callback run చేయలేరు
(మీరు manual గా `get()` call చేసి block అవ్వాలి), 2) multiple futures ని
**compose** చేయడం (ఒకదాని result మరొక దానికి input గా వాడటం) awkward, 3)
error handling verbose.

---

## 10.4 CONCEPT: `CompletableFuture` — Composable Asynchronous Pipelines

### TELUGU EXPLANATION

`CompletableFuture` (Java 8+) `Future` యొక్క limitations ని పరిష్కరిస్తుంది
— **non-blocking composition** support చేస్తుంది:

```java
CompletableFuture<Customer> customerFuture =
        CompletableFuture.supplyAsync(() -> customerService.fetch(id), executor);

CompletableFuture<Order> resultFuture = customerFuture
        .thenApply(Customer::getPreferredRegion)                 // transform result
        .thenCompose(region -> fetchPricingAsync(region))         // chain another async call
        .thenCombine(fetchInventoryAsync(id), (pricing, inv) ->   // combine two independent futures
                buildOrder(pricing, inv))
        .exceptionally(ex -> {                                     // handle failure gracefully
            log.error("Order build failed", ex);
            return Order.fallback();
        });
```

**కీలక methods:**
- `thenApply(Function)` — result ని transform చేస్తుంది (sync, same thread usually).
- `thenApplyAsync(Function, executor)` — transform, కానీ వేరే thread pool లో.
- `thenCompose(Function<T, CompletableFuture<R>>)` — **flatMap లాంటిది** —
  ఒక async operation ఫలితం మీద ఆధారపడి మరో async operation చేయాలంటే.
- `thenCombine(other, BiFunction)` — రెండు **independent** futures ని combine చేయడానికి.
- `allOf(futures...)` / `anyOf(futures...)` — multiple futures ని wait చేయడానికి.
- `exceptionally(Function<Throwable, T>)` — exception వస్తే fallback value.
- `handle(BiFunction<T, Throwable, R>)` — success మరియు failure రెండింటినీ ఒకేచోట handle చేయడానికి.

**`thenApply` vs `thenCompose` — తరచుగా అడిగే confusion:**
`thenApply` వాడితే మీ function ఒక **plain value** return చేస్తుంది.
`thenCompose` వాడితే మీ function మరో **`CompletableFuture`** return
చేస్తుంది (nested `CompletableFuture<CompletableFuture<T>>` బదులు, ఇది
flatten చేస్తుంది). ఇది Chapter 7 లో మనం చూసిన `map` vs `flatMap` కి direct
parallel.

### ENGLISH INTERVIEW ANSWER

"`CompletableFuture` lets me build non-blocking, composable asynchronous
pipelines instead of calling `.get()` and blocking a thread while waiting.
`thenApply` is for transforming a result with a plain function — the
`map` equivalent — while `thenCompose` is for chaining to another
asynchronous operation that itself returns a `CompletableFuture` — the
`flatMap` equivalent — avoiding a nested future. `thenCombine` joins two
independent async operations once both complete. I always pass an explicit
executor to the `Async` variants in production code, rather than relying on
the default `ForkJoinPool.commonPool()`, because that pool is shared
JVM-wide — if I'm doing blocking I/O inside a `CompletableFuture` stage
without a dedicated executor, I risk starving the common pool for unrelated
parallel streams or other code that also defaults to it."

### REAL-TIME EXAMPLE

ఒక e-commerce order page, customer info, pricing, inventory — మూడు
వేర్వేరు (potentially slow) services నుండి fetch చేయాల్సి వస్తే, వీటిని
**sequentially** (ఒకదాని తర్వాత మరొకటి) call చేస్తే total latency = sum of
all three. `CompletableFuture` తో వీటిని **parallel** గా kick off చేసి,
`allOf`/`thenCombine` తో combine చేస్తే, total latency ≈ **max** of the
three, sum కాదు — ఇది real production latency optimization.

---

## 10.5 CONCEPT: Atomic Classes — Lock-Free Thread Safety

### TELUGU EXPLANATION

`AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference` — ఇవి
**CAS (Compare-And-Swap)** అనే CPU-level instruction వాడి, **lock లేకుండా**
atomic operations అందిస్తాయి.

**CAS ఎలా పని చేస్తుంది:** "ఈ memory location లో ఇప్పటికీ expected value X
ఉంటే మాత్రమే, దాన్ని కొత్త value Y తో replace చేయి — ఇది ఒక్క atomic CPU
instruction" — ఇంకో thread మధ్యలో value మార్చేస్తే, CAS fail అవుతుంది,
మరియు thread retry చేస్తుంది (busy-loop, కానీ lock వల్ల block అవ్వదు).

```java
class Counter {
    private final AtomicInteger count = new AtomicInteger(0);
    void increment() { count.incrementAndGet(); } // atomic, lock-free
    int get() { return count.get(); }
}
```

**ఎప్పుడు `AtomicInteger` వాడాలి, ఎప్పుడు `synchronized`:** ఒక్క variable
మీద simple operations (increment, compare-and-set) కి `AtomicInteger`
usually **faster** (no lock contention, no context switching for blocked
threads). Multiple variables ని ఒకేసారి atomically మార్చాలంటే (multi-field
invariant maintain చేయాలంటే), `synchronized`/`ReentrantLock` అవసరం —
atomics ఒక్క variable కి మాత్రమే పరిమితం.

### ENGLISH INTERVIEW ANSWER

"Atomic classes use compare-and-swap, a hardware-level instruction, to
perform lock-free atomic updates — the CPU attempts to swap in a new value
only if the current value still matches what was expected, and retries on
conflict rather than blocking. For a single counter or flag, this is
typically faster than `synchronized` because there's no lock acquisition,
no context switching for blocked threads, just a tight retry loop under
contention. The limitation is scope — atomics protect exactly one variable.
The moment I need to update two related fields together as one atomic unit
to preserve an invariant, I need real locking, because CAS on two separate
atomics doesn't give me a combined atomic guarantee across both."

---

## 10.6 CODE: A BOUNDED THREAD POOL WITH BACKPRESSURE

**Requirement:** Build a production-grade executor for processing incoming
tasks with an explicit bound on both threads AND the queue, so the service
degrades predictably (rejects new work) instead of running out of memory
under overload.

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        10,                                  // core pool size
        20,                                  // max pool size
        60L, TimeUnit.SECONDS,               // idle thread keep-alive
        new ArrayBlockingQueue<>(500),        // BOUNDED queue — the key safety mechanism
        new ThreadPoolExecutor.CallerRunsPolicy() // backpressure: caller thread runs the task itself
);

try {
    executor.execute(() -> processTask(task));
} catch (RejectedExecutionException e) {
    // Only reachable if using AbortPolicy instead of CallerRunsPolicy
    log.warn("Task rejected, system overloaded");
}
```

**Design explanation:** Unlike `Executors.newFixedThreadPool()`, this uses
an explicit `ArrayBlockingQueue(500)` — bounded — so under sustained
overload, the queue fills up rather than growing unbounded toward an OOM.
`CallerRunsPolicy` is a deliberate backpressure choice: when the pool and
queue are both full, the *submitting* thread runs the task itself, which
naturally slows down the rate of new submissions (the caller is now busy)
instead of silently dropping work or crashing.

**Interviewer follow-ups:**
- "What are the other rejection policies, and when would you use each?"
  (`AbortPolicy` — throw, fail fast, let the caller decide; `DiscardPolicy`
  — silently drop, rarely appropriate; `DiscardOldestPolicy` — drop the
  oldest queued task to make room, useful when only the newest data matters,
  e.g., latest sensor readings.)
- "Why not just make the queue unbounded?" (Exactly the anti-pattern this
  design avoids — an unbounded queue under sustained overload just delays
  the OOM instead of preventing it, and hides the overload condition from
  monitoring until it's catastrophic.)

---

## 10.7 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Running background tasks | `new Thread(...).start()` per task | Bounded `ThreadPoolExecutor` sized for the workload type |
| Combining 3 independent async calls | Call each sequentially, sum the latency | `CompletableFuture` fan-out + `allOf`/`thenCombine`, latency ≈ max not sum |
| Simple shared counter | `synchronized` on every access | `AtomicInteger`, unless multi-field invariants are involved |
| Overload under load spike | Let the queue grow unbounded ("it'll catch up") | Bounded queue + explicit rejection/backpressure policy |

---

## 10.8 COMMON MISTAKES

1. Using `Executors.newCachedThreadPool()`/`newFixedThreadPool()` in
   production without understanding their unbounded queue or unbounded
   thread creation risk.
2. Forgetting to call `future.get()`'s exception handling — an exception
   thrown inside a task submitted via `submit()` is silently swallowed
   unless you call `.get()` (which rethrows it wrapped in
   `ExecutionException`) or use `CompletableFuture`'s `exceptionally`/`handle`.
3. Using the default `ForkJoinPool.commonPool()` for blocking I/O inside
   `CompletableFuture` async stages, starving it for unrelated parallel work.
4. Using `AtomicInteger` for a compound multi-field invariant instead of
   proper locking.
5. Never calling `executor.shutdown()` — leaking thread pools, especially
   in short-lived contexts (tests, batch jobs) that create a new executor
   repeatedly.

---

## 10.9 INTERVIEW QUESTION BANK — CHAPTER 10

**Basic:** 1. Why avoid creating raw `Thread` objects for application tasks?
2. What does `AtomicInteger` guarantee that a plain `int` doesn't?

**Intermediate:** 3. Difference between `thenApply` and `thenCompose`?
4. What's the risk of `Executors.newFixedThreadPool()`'s default queue?

**Senior:** 5. How do you size a thread pool for a mixed CPU-bound and
I/O-bound workload? 6. Why might `CallerRunsPolicy` be a better backpressure
strategy than simply increasing the queue size?

**Architect:** 7. Design the concurrency model for a service that fans out
to 5 downstream microservices per request, aggregates their results, and
must respond within a 200ms SLA even if one downstream is slow. What
`CompletableFuture` composition and timeout strategy would you use?

**Scenario:** 8. A batch job submits 1 million tasks to a
`newCachedThreadPool()` and the JVM crashes with `OutOfMemoryError:
unable to create new native thread`. Diagnose and redesign.

**Trick:** 9. "`AtomicInteger` uses a lock internally, just a very fast
one." True or false?

<details><summary>Key answers</summary>

- Q7: Kick off all 5 calls as `CompletableFuture.supplyAsync` on a
  dedicated executor immediately (parallel fan-out), combine with
  `CompletableFuture.allOf`, and apply a timeout (`orTimeout` /
  `completeOnTimeout` in modern Java, or a scheduled timeout task) per
  individual future so one slow downstream can't blow the whole request's
  latency — degrade gracefully with a partial/fallback result for whichever
  future didn't complete in time rather than waiting on all 5 unconditionally.
- Q8: `newCachedThreadPool()` creates a new thread per task with no upper
  bound, so 1 million tasks attempted essentially all at once tries to
  create up to 1 million OS threads, exhausting native memory/OS thread
  limits; redesign with a bounded `ThreadPoolExecutor` (fixed reasonable
  pool size, bounded queue, backpressure policy) sized for the actual
  achievable throughput, not the input volume.
- Q9: False — atomics are lock-*free*, using CAS retry loops, not locking
  at all; this is precisely why they can outperform `synchronized` under
  contention for single-variable updates.

</details>

---

## 10.10 MASTERY CHECKPOINTS

- **Knowledge Check:** What does CAS stand for, and what does it do at the CPU level?
- **Coding Check:** Rewrite a sequential three-API-call aggregation into a parallel `CompletableFuture` pipeline with proper exception handling.
- **Explanation Check:** Explain in English why an unbounded task queue is a hidden `OutOfMemoryError` waiting to happen under sustained load.
- **Real-World Check:** Your service's thread pool metrics show queue size steadily climbing during peak hours, then draining overnight. What does this indicate, and what are your options?
- **Senior Check:** When would you deliberately choose `synchronized`/`ReentrantLock` over an equivalent-looking atomic-based lock-free design?
- **Master Check:** Design (in words) a rate limiter using `AtomicLong` and `ScheduledExecutorService` that resets a request counter every second — identify exactly where atomicity matters and where it doesn't.

<details><summary>Answers</summary>

- Real-World Check: Sustained queue growth during peak hours indicates the
  pool is undersized for peak load (arrival rate > processing rate) —
  options: increase pool size if more CPU/I/O headroom exists, add more
  service instances (horizontal scaling), or add explicit backpressure/load
  shedding if the downstream truly can't keep up regardless of pool size.
- Senior Check: When multiple related fields must change together
  atomically to preserve an invariant (e.g., debit one account and credit
  another as one atomic unit) — no combination of separate atomics gives
  you a joint atomic guarantee across both.
- Master Check: An `AtomicLong requestCount` incremented via
  `incrementAndGet()` per request (atomicity matters here — concurrent
  requests must not lose increments); a `ScheduledExecutorService` task
  runs every second calling `requestCount.getAndSet(0)` (atomic reset+read)
  to check against the limit and reset — the scheduling itself doesn't need
  atomicity (only one scheduled task runs the reset), only the
  increment/reset operations on the shared counter do.

</details>

---

## 10.11 CHEAT SHEET

| Tool | Use for |
|---|---|
| `ThreadPoolExecutor` (explicit) | Production task execution, bounded queue + rejection policy |
| `CompletableFuture` | Composable, non-blocking async pipelines |
| `thenApply` | Transform a result (map) |
| `thenCompose` | Chain to another async call (flatMap) |
| `thenCombine` | Join two independent futures |
| `AtomicInteger`/`AtomicLong`/`AtomicReference` | Lock-free single-variable atomic updates |
| `synchronized`/`ReentrantLock` | Multi-field invariants, compound critical sections |

---

*(Continues to Chapter 11 — JVM Performance and Tuning.)*
