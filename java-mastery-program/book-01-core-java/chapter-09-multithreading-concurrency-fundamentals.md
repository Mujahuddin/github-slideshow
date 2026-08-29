# CHAPTER 9 — MULTITHREADING AND CONCURRENCY FUNDAMENTALS

> This is where "Zero → Senior" reasoning matters most — concurrency bugs
> are the hardest production bugs to reproduce, and interviewers know it.

---

## 9.1 CONCEPT: Threads, and Why Concurrency Is Hard

### TELUGU EXPLANATION

**Thread** అనేది ఒక process లోపల ఒక independent execution path. ఒకే process
లోని multiple threads **heap memory ని share చేస్తాయి** (Chapter 1 గుర్తు
చేసుకోండి), కానీ ఒక్కొక్కరికి own stack ఉంటుంది. ఇదే concurrency bugs కి
మూల కారణం — **shared mutable state**.

రెండు threads ఒకే variable ని ఏకకాలంలో read/write చేస్తే, **race condition**
జరగవచ్చు — ఫలితం execution order మీద ఆధారపడి unpredictable గా మారుతుంది.

```java
class Counter {
    private int count = 0;
    void increment() { count++; } // ఇది ఒక్క atomic operation కాదు!
}
```

`count++` వాస్తవానికి మూడు steps: (1) `count` చదవడం, (2) 1 add చేయడం, (3)
తిరిగి write చేయడం (**Read-Modify-Write**). రెండు threads ఏకకాలంలో ఇది
చేస్తే, ఒక increment "పోగొట్టుకోవచ్చు" (lost update problem) — ఉదాహరణకి,
1000 threads x 1000 increments చేసినా, final count 1,000,000 బదులు
అంతకంటే తక్కువ రావొచ్చు.

### INDUSTRY TERMINOLOGY

`Thread`, `race condition`, `critical section`, `atomicity`,
`Read-Modify-Write`, `thread safety`, `shared mutable state`, `context switch`.

### ENGLISH INTERVIEW ANSWER

"A race condition happens when the correctness of a result depends on the
unpredictable timing/interleaving of multiple threads accessing shared
mutable state. The classic example is `count++` — it's not atomic; it's a
read, a modify, and a write, and if two threads interleave those three
steps, one thread's increment can be silently lost. The fix is always the
same in principle: protect the critical section — the code that reads and
writes shared state — so only one thread executes it at a time, using
synchronization, locks, or atomic classes, or better yet, avoid sharing
mutable state across threads in the first place wherever the design allows it."

---

## 9.2 CONCEPT: `synchronized`, Locks, and Monitor Objects

### TELUGU EXPLANATION

Java లో `synchronized` keyword ఒక **intrinsic lock (monitor)** వాడి
critical section ని protect చేస్తుంది. ఒక్క thread మాత్రమే ఒక సమయంలో ఆ
lock hold చేయగలదు.

```java
class Counter {
    private int count = 0;
    synchronized void increment() { count++; } // ఇప్పుడు atomic, thread-safe
    synchronized int get() { return count; }
}
```

**`synchronized` రకాలు:**
- **Instance method:** `this` మీద lock.
- **Static method:** `Class` object మీద lock (class-level lock).
- **Block:** ఒక specific object మీద lock, method మొత్తం కాకుండా — finer
  granularity, better performance (lock ఎక్కువసేపు hold చేయకుండా):

```java
void transfer(Account from, Account to, BigDecimal amount) {
    synchronized (from) {
        synchronized (to) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

**⚠️ ఇక్కడ deadlock risk ఉంది** — section 9.4 చూడండి.

**`ReentrantLock` (java.util.concurrent.locks):** `synchronized` కంటే ఎక్కువ
control ఇస్తుంది — `tryLock()` (timeout తో lock తీసుకునే ప్రయత్నం, deadlock
avoid చేయడానికి), fairness policy, interruptible locking. Trade-off:
మీరు manual గా `lock()`/`unlock()` (`finally` block లో తప్పకుండా) చేయాలి —
`synchronized` automatic గా release చేస్తుంది exception వచ్చినా కూడా.

### ENGLISH INTERVIEW ANSWER

"`synchronized` acquires an intrinsic lock on an object's monitor — either
`this` for an instance method, the `Class` object for a static method, or an
explicit object for a synchronized block — ensuring only one thread executes
that critical section at a time. I prefer synchronized blocks over
synchronizing the entire method when only part of the method touches shared
state, to minimize how long the lock is held and maximize throughput. When I
need more control — a timeout on lock acquisition to avoid indefinite
blocking, fairness ordering, or the ability to interrupt a waiting thread —
I use `ReentrantLock` instead, accepting that I now have to manually release
it in a `finally` block, since unlike `synchronized`, the JVM won't do that
for me automatically."

---

## 9.3 CONCEPT: `volatile` and the Java Memory Model

### TELUGU EXPLANATION

Multi-core CPUs లో, ప్రతి core దాని own **cache** కలిగి ఉంటుంది. ఒక thread
ఒక variable ని update చేస్తే, ఆ update వేరే core cache కి **immediately
కనిపించకపోవచ్చు** (visibility problem) — ఇది race condition కి భిన్నమైన
సమస్య.

```java
class Flag {
    private boolean running = true;
    void stop() { running = false; }
    void run() {
        while (running) { /* work */ } // ఇంకో thread running=false చేసినా,
                                         // ఈ thread కి అది కనిపించకపోవచ్చు —
                                         // infinite loop అయిపోవచ్చు!
    }
}
```

`volatile` keyword ఈ problem ని fix చేస్తుంది — ఇది compiler/CPU కి "ఈ
variable ని ఎప్పుడూ main memory నుండి చదవండి, cache నుండి కాదు, మరియు
ప్రతి write ని వెంటనే main memory కి flush చేయండి" అని guarantee ఇస్తుంది
(**visibility guarantee**).

**ముఖ్యమైన limitation:** `volatile` **atomicity guarantee ఇవ్వదు**. `count++`
కి `volatile` add చేసినా, ఇప్పటికీ race condition ఉంటుంది — ఎందుకంటే
Read-Modify-Write ఇప్పటికీ మూడు separate steps గానే ఉంటుంది, `volatile`
కేవలం visibility ని guarantee చేస్తుంది, atomicity ని కాదు. `volatile` ఒకే
ఒక్క **single write, no dependency on previous value** (like a simple flag)
కి సరిపోతుంది — increment లాంటి compound operations కి **`AtomicInteger`
లేదా `synchronized`** అవసరం (Chapter 10 చూడండి).

### ENGLISH INTERVIEW ANSWER

"`volatile` solves the visibility problem, not the atomicity problem. It
guarantees that a write to a volatile variable is immediately visible to all
other threads, and prevents certain compiler/CPU reorderings around it — so
it's correct and sufficient for a simple flag like a shutdown signal that
one thread sets and another polls. It is not sufficient for compound
operations like increment, because `count++` is still three separate steps
even if `count` is volatile — two threads can still interleave those steps
and lose an update. This is one of the most common concurrency interview
traps: candidates say 'volatile makes it thread-safe' without distinguishing
visibility from atomicity."

---

## 9.4 CONCEPT: Deadlocks

### TELUGU EXPLANATION

**Deadlock** అంటే రెండు (లేదా ఎక్కువ) threads ఒకదానికొకటి hold చేసిన lock
కోసం ఎదురుచూస్తూ, ఎవరూ ముందుకు వెళ్ళలేని పరిస్థితి. Classic ఉదాహరణ, section
9.2 లో చూపించిన `transfer` method:

```
Thread 1: transfer(accountA, accountB, ...) → locks A, waits for B
Thread 2: transfer(accountB, accountA, ...) → locks B, waits for A
```

రెండు threads ఒకదాని కోసం మరొకటి ఎదురుచూస్తూ **శాశ్వతంగా ఆగిపోతాయి**.

**Deadlock కి నాలుగు అవసరమైన conditions (Coffman conditions):**
1. **Mutual exclusion:** resource ఒకేసారి ఒక్క thread మాత్రమే hold చేయగలదు.
2. **Hold and wait:** ఒక thread ఒక lock hold చేస్తూ, మరో lock కోసం wait చేస్తుంది.
3. **No preemption:** ఒక lock ని బలవంతంగా తీసుకోలేరు, thread స్వయంగా release చేయాలి.
4. **Circular wait:** threads ఒక circular chain లో ఒకదాని కోసం మరొకటి wait చేస్తాయి.

**Fix — Lock Ordering:** అన్ని threads locks ని **ఒకే fixed order** లో
తీసుకుంటే, circular wait సాధ్యం కాదు:

```java
void transfer(Account from, Account to, BigDecimal amount) {
    Account first = from.getId().compareTo(to.getId()) < 0 ? from : to;
    Account second = first == from ? to : from;
    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

ఇప్పుడు ఏ thread అయినా, ఎప్పుడూ **ఒకే order** లో (ID ఆధారంగా) locks
తీసుకుంటుంది — circular wait సాధ్యం కాదు.

### ENGLISH INTERVIEW ANSWER

"A deadlock occurs when threads form a circular wait on each other's locks —
classically, two threads each holding one lock and waiting for the other's.
The fix I always reach for first is consistent lock ordering: if every
thread acquires multiple locks in the same globally consistent order —
often by comparing a stable identifier like an account ID — a circular wait
becomes structurally impossible. Alternatives include using `tryLock` with
a timeout to detect and back off from a potential deadlock instead of
blocking forever, or redesigning to avoid holding multiple locks
simultaneously at all, which is usually the cleanest fix when feasible."

---

## 9.5 CODE: DIAGNOSING A DEADLOCK WITH `jstack`

**Requirement:** Show how to actually find a deadlock in production, not
just describe it.

```java
public class DeadlockDemo {
    static final Object lockA = new Object();
    static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {
                sleep(100);
                synchronized (lockB) { System.out.println("T1 done"); }
            }
        }, "Thread-1");

        Thread t2 = new Thread(() -> {
            synchronized (lockB) {
                sleep(100);
                synchronized (lockA) { System.out.println("T2 done"); }
            }
        }, "Thread-2");

        t1.start();
        t2.start();
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

**Diagnosis:** Running `jstack <pid>` against this hung process shows
exactly this in its output:

```
Found one Java-level deadlock:
=============================
"Thread-2":
  waiting to lock monitor ... (object lockA),
  which is held by "Thread-1"
"Thread-1":
  waiting to lock monitor ... (object lockB),
  which is held by "Thread-2"
```

`jstack` explicitly identifies the deadlock and names both threads and both
locks — this is the real, practical debugging step, not just theory.

**Interviewer follow-up:** "How would you fix this exact code?" — Apply
lock ordering: always lock `lockA` before `lockB` in both threads,
regardless of which thread's "logical" order was A→B or B→A.

---

## 9.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Shared counter across threads | `synchronized` on every access, no further thought | Consider `AtomicInteger` (lock-free, often faster) first |
| Two locks needed | Lock in whatever order the code naturally reads | Enforce consistent global lock ordering to prevent deadlock |
| Production hang reported | "Restart the service" | Take a thread dump (`jstack`) BEFORE restarting to preserve deadlock evidence |
| Flag shared between threads | Plain `boolean` field | `volatile boolean`, or better, `AtomicBoolean` |

---

## 9.7 COMMON MISTAKES

1. Believing `volatile` makes compound operations (like increment) thread-safe.
2. Synchronizing on a mutable/reassignable field, or on a boxed primitive/String literal (interning means you might be locking on a shared object unintentionally across unrelated code).
3. Holding a lock while calling into unknown/external code (risk of unexpected reentrancy or long lock hold times).
4. Not using consistent lock ordering when multiple locks are needed.
5. Restarting a hung production service without capturing a thread dump first — losing the only real evidence of a deadlock's cause.

---

## 9.8 INTERVIEW QUESTION BANK — CHAPTER 9

**Basic:** 1. What is a race condition? 2. What does `synchronized` do?

**Intermediate:** 3. Difference between `volatile` and `synchronized`? 4.
What are the four Coffman conditions for deadlock?

**Senior:** 5. Why doesn't `volatile` make `count++` thread-safe? 6. How
would you detect and prevent deadlocks in a production system before they
happen?

**Architect:** 7. You're designing a high-throughput service with heavy
lock contention on a shared resource. What alternatives to coarse-grained
locking would you evaluate (lock striping, lock-free structures,
partitioning the resource)?

**Scenario:** 8. A production service occasionally "hangs" completely under
load, requiring a manual restart, with no exceptions in the logs. Walk
through your investigation.

**Trick:** 9. "Using `synchronized` everywhere guarantees a bug-free
concurrent program." True or false?

<details><summary>Key answers</summary>

- Q7: Lock striping (splitting one lock into N locks over partitions of the
  data, like `ConcurrentHashMap`'s historical design), lock-free structures
  using CAS/atomics for simple state, or partitioning the workload so each
  thread/shard owns disjoint data and needs no shared lock at all — chosen
  based on how the access pattern actually distributes across the resource.
- Q8: A hang with no exceptions is a strong deadlock/livelock signal — first
  action is a thread dump (`jstack`) to look for a "Found one Java-level
  deadlock" section, not restarting immediately, since restarting destroys
  the evidence needed to find the root cause.
- Q9: False — `synchronized` prevents race conditions on the state it
  actually protects, but overuse can cause deadlocks (multiple locks,
  inconsistent ordering), doesn't prevent visibility bugs on non-synchronized
  state, and doesn't prevent logic bugs — it's a tool, not a guarantee.

</details>

---

## 9.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is `count++` not atomic even on a single CPU instruction set that seems simple?
- **Coding Check:** Fix the `DeadlockDemo` code from section 9.5 using lock ordering; verify (in reasoning, or actually run it) that it no longer deadlocks.
- **Explanation Check:** Explain in English the difference between "visibility" and "atomicity" using the flag example and the counter example side by side.
- **Real-World Check:** A production service has a shared `HashMap` cache accessed by multiple request-handling threads with no synchronization, and occasionally throws unexpected exceptions or returns corrupted data under load. Diagnose and fix (reference Chapter 4).
- **Senior Check:** When would you choose `ReentrantLock` over `synchronized` even though it requires manual unlock discipline?
- **Master Check:** Design a thread-safe bank account transfer system for N accounts (not just 2) that avoids deadlock by design, not by luck.

<details><summary>Answers</summary>

- Real-World Check: An unsynchronized `HashMap` under concurrent
  modification can corrupt its internal bucket structure (potentially an
  infinite loop in older JDKs, or lost/duplicated entries) — replace with
  `ConcurrentHashMap` (Chapter 4).
- Senior Check: When you need `tryLock` with a timeout to avoid indefinite
  blocking (e.g., to detect and back off from contention gracefully), need
  fair lock ordering, or need to interrupt a thread waiting on a lock —
  none of which `synchronized` supports.
- Master Check: Assign each account a stable, comparable ID; always acquire
  locks for a multi-account operation in ascending ID order (as shown in
  9.4), regardless of the "logical" direction of the operation — this
  generalizes cleanly to any number of accounts locked together in one
  operation.

</details>

---

## 9.10 CHEAT SHEET

| Concept | One-liner |
|---|---|
| Race condition | Unsynchronized access to shared mutable state, outcome depends on timing |
| `synchronized` | Mutual exclusion via intrinsic lock; auto-release even on exception |
| `volatile` | Visibility guarantee only, NOT atomicity |
| Deadlock | Circular wait on locks; fix with consistent lock ordering |
| `ReentrantLock` | Manual lock/unlock, but `tryLock`, fairness, interruptibility |
| First move on a production hang | `jstack` BEFORE restarting |

---

*(Continues to Chapter 10 — Concurrency Toolkit: ExecutorService, CompletableFuture, Atomics.)*
