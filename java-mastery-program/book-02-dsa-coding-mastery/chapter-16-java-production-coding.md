# CHAPTER 16 — JAVA PRODUCTION CODING: LRU CACHE, RATE LIMITER, PRODUCER-CONSUMER, THREAD POOL, RETRY, IDEMPOTENCY

> **Level 6 of this book.** Everything here is a real system component, not
> a puzzle — this is where DSA patterns (Chapters 1-15) and Book 1's
> concurrency chapters (9-10) fuse into what a Senior/Lead engineer is
> actually expected to build from scratch in a system-design or
> "coding round with a production bar" interview.

---

## 16.1 LRU CACHE — O(1) GET/PUT FROM SCRATCH

### TELUGU EXPLANATION

Chapter 3 (Linked List) లో ప్రస్తావించినట్టు, LRU (Least Recently Used)
Cache కి **రెండు operations O(1) లో కావాలి**: `get(key)` మరియు
`put(key, value)`, రెండూ కూడా ఆ item ని **"most recently used"** గా
mark చేయాలి, మరియు capacity దాటితే **least recently used** item ని
evict చేయాలి.

**కీలక insight — ఎందుకు రెండు structures కలపాలి:**
- **HashMap** ఇస్తుంది: O(1) key → node lookup.
- **Doubly Linked List** ఇస్తుంది: O(1) **arbitrary node removal**
  (Chapter 3 లో చూసినట్టు, `prev` pointer ఉంటేనే ఇది సాధ్యం) + O(1)
  "move to front" (most-recently-used end).

ఏ ఒక్కటీ విడిగా సరిపోదు — HashMap కి ordering లేదు, Linked List కి fast
lookup లేదు. **రెండింటినీ కలిపితేనే** రెండు operations O(1) అవుతాయి.

```java
class LRUCache {
    private final int capacity;
    private final Map<Integer, Node> cache = new HashMap<>();
    private final Node head = new Node(-1, -1); // dummy head (most recently used side)
    private final Node tail = new Node(-1, -1); // dummy tail (least recently used side)

    LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    int get(int key) {
        if (!cache.containsKey(key)) return -1;
        Node node = cache.get(key);
        moveToFront(node); // access చేసినప్పుడు "most recently used" గా mark చేయండి
        return node.value;
    }

    void put(int key, int value) {
        if (cache.containsKey(key)) {
            Node node = cache.get(key);
            node.value = value;
            moveToFront(node);
            return;
        }
        if (cache.size() == capacity) {
            Node lru = tail.prev; // dummy tail ముందు ఉన్నదే least recently used
            remove(lru);
            cache.remove(lru.key);
        }
        Node newNode = new Node(key, value);
        cache.put(key, newNode);
        addToFront(newNode);
    }

    private void moveToFront(Node node) {
        remove(node);
        addToFront(node);
    }

    private void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void addToFront(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }

    private static class Node {
        int key, value;
        Node prev, next;
        Node(int key, int value) { this.key = key; this.value = value; }
    }
}
```

**Design notes:** Dummy `head`/`tail` sentinel nodes (Chapter 3's merge-list
idiom) eliminate null-checks for the "list is empty" or "removing the first/
last real node" edge cases — every real node always has a genuine `prev`
and `next` to update. `remove()` doesn't null out the removed node's own
pointers since it's immediately re-added via `addToFront()` in the
`moveToFront` case, or discarded entirely in the eviction case.

### ENGLISH INTERVIEW ANSWER

"Neither a HashMap nor a linked list alone gives O(1) for both operations —
a HashMap has no ordering, and a plain linked list has no O(1) lookup. I
combine them: the HashMap maps keys directly to their linked-list node for
O(1) lookup, and a doubly linked list maintains recency order, with O(1)
removal from anywhere (since each node knows its own `prev`) and O(1)
insertion at the front. `get` and `put` both move the accessed node to the
front; when capacity is exceeded, the node just before the tail sentinel —
the actual least-recently-used real node — is evicted from both
structures. I use dummy head and tail sentinel nodes specifically to avoid
null-checking edge cases when the cache is empty or when removing the
first or last real element."

**Interviewer follow-up:** "Make this thread-safe." — Wrap `get`/`put` in
`synchronized` (simplest, correct for moderate contention), or use
`LinkedHashMap` with `accessOrder=true` and `removeEldestEntry` overridden
(Book 1 Chapter 4's built-in idiom) wrapped in external synchronization —
both are legitimate answers; a from-scratch lock-free LRU is a genuinely
hard, rarely-expected extension worth naming but not implementing live.

---

## 16.2 RATE LIMITER — TOKEN BUCKET ALGORITHM

### TELUGU EXPLANATION

**Token Bucket** అనేది అత్యంత సాధారణ rate-limiting algorithm — production
APIs లో widely వాడేది (AWS, Stripe వంటివి దీన్నే వాడతాయి). ఆలోచన: ఒక
"bucket" లో గరిష్టంగా `capacity` tokens ఉండొచ్చు. Tokens ఒక fixed rate తో
**refill** అవుతూ ఉంటాయి. ప్రతి request ఒక token "వాడుకుంటుంది" — token
లేకపోతే, request **reject** అవుతుంది (లేదా queue అవుతుంది, design బట్టి).

**ఎందుకు ఇది "bursty" traffic ని handle చేయగలదు:** బకెట్ నిండుగా ఉంటే,
ఒక్కసారిగా చాలా requests రావొచ్చు (burst) — కానీ దీర్ఘకాలంలో, average
rate refill rate ని దాటదు.

```java
class TokenBucketRateLimiter {
    private final long capacity;
    private final double refillRatePerNano; // tokens per nanosecond
    private double availableTokens;
    private long lastRefillTimestamp;

    TokenBucketRateLimiter(long capacity, double refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRatePerNano = refillRatePerSecond / 1_000_000_000.0;
        this.availableTokens = capacity; // బకెట్ నిండుగా మొదలవుతుంది
        this.lastRefillTimestamp = System.nanoTime();
    }

    // Thread-safe — production rate limiters are shared across request threads
    synchronized boolean tryAcquire() {
        refill();
        if (availableTokens >= 1) {
            availableTokens -= 1;
            return true;
        }
        return false; // token లేదు — request reject చేయండి
    }

    private void refill() {
        long now = System.nanoTime();
        long elapsed = now - lastRefillTimestamp;
        double tokensToAdd = elapsed * refillRatePerNano;
        if (tokensToAdd > 0) {
            availableTokens = Math.min(capacity, availableTokens + tokensToAdd);
            lastRefillTimestamp = now;
        }
    }
}
```

**Design notes:** Refill **lazily** (on each `tryAcquire()` call), కాదు
ఒక background thread తో — ఇది simpler, మరియు ఖచ్చితంగా correct
(background thread scheduling jitter మీద ఆధారపడదు). `synchronized`
వాడాము — Book 1 Chapter 9 సూత్రం: shared mutable state
(`availableTokens`, `lastRefillTimestamp`) కి critical section protection
అవసరం.

### ENGLISH INTERVIEW ANSWER

"Token Bucket models a bucket with a maximum capacity that refills at a
steady rate; each request consumes one token, and a request without an
available token is rejected. This naturally allows short bursts up to the
bucket's capacity while still enforcing a long-term average rate equal to
the refill rate — which is exactly the traffic-shaping behavior most APIs
actually want, as opposed to a naive fixed-window counter that can allow
2x the intended rate right at a window boundary. I compute refill lazily,
based on elapsed wall-clock time since the last check, rather than running
a background thread to add tokens on a timer — this avoids scheduling
jitter and keeps the implementation simpler and exactly correct regardless
of how often `tryAcquire` is actually called. The whole thing needs to be
synchronized since it's shared mutable state accessed by concurrent
request-handling threads, directly connecting to Book 1's concurrency material."

**Interviewer follow-up:** "What if this needs to work across multiple
service instances (distributed rate limiting)?" — A single in-process
token bucket doesn't coordinate across instances; production distributed
rate limiting typically moves the token/counter state into Redis (using
atomic `INCR`/Lua scripts, or Redis's own rate-limiting patterns) so all
instances share one source of truth.

---

## 16.3 PRODUCER-CONSUMER — `BlockingQueue`

### TELUGU EXPLANATION

Producer-Consumer అనేది ఒక classic concurrency pattern: ఒకటి (లేదా
ఎక్కువ) **producer threads** work items generate చేస్తాయి, ఒకటి (లేదా
ఎక్కువ) **consumer threads** వాటిని process చేస్తాయి, రెండింటి మధ్య ఒక
**shared queue**. Book 1 Chapter 9 లో మనం `wait()`/`notify()` వాడి ఇది
manually ఎలా implement చేయాలో నేర్చుకోవచ్చు — కానీ production Java code
లో, **`BlockingQueue`** (JDK builtin) వాడటమే సరైనది, ఇది ఇదే coordination
ని internally, correctly, tested గా handle చేస్తుంది.

```java
class ProducerConsumerExample {
    private final BlockingQueue<String> queue = new LinkedBlockingQueue<>(100); // bounded — backpressure కోసం
    private volatile boolean running = true;

    void produce() {
        while (running) {
            try {
                String item = generateWork();
                queue.put(item); // queue నిండితే, ఇది BLOCK అవుతుంది (backpressure) — busy-wait కాదు
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); // interrupt status ని restore చేయండి — Book 1 idiom
                break;
            }
        }
    }

    void consume() {
        while (running || !queue.isEmpty()) {
            try {
                String item = queue.poll(1, TimeUnit.SECONDS); // timeout తో — shutdown check చేయడానికి
                if (item != null) {
                    processWork(item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    private String generateWork() { return "work-item"; }
    private void processWork(String item) { /* ... */ }
}
```

**Design notes:** `LinkedBlockingQueue(100)` **bounded** గా వాడాము — ఇది
Book 1 Chapter 10 లో నేర్చుకున్న "unbounded queue = hidden OOM risk"
సూత్రం direct application. `queue.put()` queue నిండితే **block**
అవుతుంది (busy-wait/spin కాదు) — ఇది natural backpressure ఇస్తుంది,
producer ని consumer వేగానికి సరిపోయేలా slow చేస్తుంది. `InterruptedException`
catch చేసినప్పుడు `Thread.currentThread().interrupt()` call చేయడం —
ఇది Book 1 Chapter 6 సూత్రం (exception swallow చేయకూడదు) యొక్క
concurrency-specific రూపం: interrupt status ని restore చేయకపోతే, ఎవరో
ఈ thread ని interrupt చేయాలని అనుకున్న సమాచారం శాశ్వతంగా పోతుంది.

### ENGLISH INTERVIEW ANSWER

"I always reach for `BlockingQueue` — `LinkedBlockingQueue` or
`ArrayBlockingQueue` — for producer-consumer in Java, rather than manually
coordinating with `wait()`/`notify()`, since the JDK implementation is
correct, well-tested, and handles the subtle edge cases around spurious
wakeups that manual coordination can get wrong. I bound the queue's
capacity deliberately — this is the same principle from Book 1's
`ThreadPoolExecutor` discussion: an unbounded queue just delays an OOM
under sustained overload instead of preventing it. `put()` blocking when
full gives natural backpressure, slowing producers to match consumer
throughput, which is usually exactly the desired behavior rather than
dropping work or crashing. And critically, whenever I catch
`InterruptedException`, I restore the interrupt status via
`Thread.currentThread().interrupt()` before exiting the loop — silently
swallowing an interrupt is a real production bug, since it prevents
graceful shutdown signaling from propagating correctly."

---

## 16.4 THREAD POOL — BUILDING A MINIMAL ONE FROM SCRATCH

### TELUGU EXPLANATION

Book 1 Chapter 10 లో మనం `ThreadPoolExecutor` ని **వాడటం** నేర్చుకున్నాం.
ఇక్కడ, దాని **internals ఎలా పని చేస్తాయో** అర్థం చేసుకోవడానికి, ఒక
**సరళమైన thread pool** ని scratch నుండి build చేద్దాం — ఇది "explain
how a thread pool works internally" అనే senior interview ప్రశ్నకి
నిజమైన, working జవాబు.

```java
class SimpleThreadPool {
    private final BlockingQueue<Runnable> taskQueue;
    private final List<WorkerThread> workers = new ArrayList<>();
    private volatile boolean isShutdown = false;

    SimpleThreadPool(int numThreads, int queueCapacity) {
        taskQueue = new ArrayBlockingQueue<>(queueCapacity); // bounded — Ch. 10 principle
        for (int i = 0; i < numThreads; i++) {
            WorkerThread worker = new WorkerThread();
            worker.start();
            workers.add(worker);
        }
    }

    void submit(Runnable task) {
        if (isShutdown) {
            throw new RejectedExecutionException("Pool is shut down");
        }
        try {
            taskQueue.put(task); // queue నిండితే block అవుతుంది — backpressure
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RejectedExecutionException("Interrupted while submitting", e);
        }
    }

    void shutdown() {
        isShutdown = true;
        for (WorkerThread worker : workers) {
            worker.interrupt(); // ప్రతి worker ని wake చేసి, గమనించి exit అవ్వమని చెప్పండి
        }
    }

    private class WorkerThread extends Thread {
        @Override
        public void run() {
            while (!isShutdown || !taskQueue.isEmpty()) {
                try {
                    Runnable task = taskQueue.poll(500, TimeUnit.MILLISECONDS);
                    if (task != null) {
                        task.run(); // ఒక్కో worker, తనకు దొరికిన task ని process చేస్తుంది
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return; // shutdown signal — exit
                } catch (RuntimeException e) {
                    // ముఖ్యం: ఒక task fail అయినంత మాత్రాన, worker thread చనిపోకూడదు!
                    System.err.println("Task failed: " + e.getMessage());
                }
            }
        }
    }
}
```

**అతి ముఖ్యమైన design detail:** `task.run()` ఒక `RuntimeException` throw
చేస్తే, దాన్ని **catch చేసి, worker thread ని బతికించాలి** — లేకపోతే, ఒక్క
బగ్గీ task, ఆ worker thread ని permanently చంపేస్తుంది, pool యొక్క
effective capacity ని silently తగ్గిస్తుంది (ఒక్కొక్క failed task తో ఒక్క
thread తగ్గుతూ, చివరికి pool ఖాళీ అయిపోతుంది) — ఇది Book 1 Chapter 6
(exception handling) + Chapter 10 (thread pools) కలిపిన, నిజంగా జరిగే
production bug.

### ENGLISH INTERVIEW ANSWER

"A thread pool is fundamentally a fixed set of long-lived worker threads
pulling from a shared task queue in a loop, rather than spawning a new
thread per task. The one detail that separates a correct implementation
from a subtly broken one is exception handling inside the worker loop: if
a submitted task throws an uncaught `RuntimeException`, and I don't catch
it inside the worker's run loop, that exception propagates out and kills
the worker thread entirely — permanently shrinking the pool's effective
capacity by one for every task that ever throws, until eventually no
workers remain. So every worker's loop needs its own try/catch around task
execution specifically to keep the worker alive regardless of what the
task does. I also bound the task queue and make `submit()` block (via
`put()`) rather than grow unboundedly, for the same backpressure reasoning
as Book 1's `ThreadPoolExecutor` discussion — this from-scratch version is
deliberately a simplified teaching model of what `ThreadPoolExecutor`
already does robustly in the JDK."

---

## 16.5 RETRY MECHANISM — EXPONENTIAL BACKOFF WITH JITTER

### TELUGU EXPLANATION

**Junior thinking:** "ఒక API call fail అయితే, వెంటనే మళ్ళీ try చేద్దాం,
కొన్నిసార్లు."
**Senior thinking (ఈ chapter ప్రారంభంలో master prompt లో ఉదహరించినట్టు):**
"Retry చేయాలా? Failure type ఏమిటి? Operation **idempotent** నా? Retry
budget ఏమిటి? Retries **traffic storm** create చేయవచ్చా?"

**Exponential Backoff:** ప్రతి retry మధ్య wait time **రెట్టింపు**
అవుతుంది (1s, 2s, 4s, 8s...) — ఇది ఒక ఇబ్బందిలో ఉన్న downstream service
మీద మరింత ఒత్తిడి పెంచకుండా చూస్తుంది.

**Jitter (randomness):** అనేక clients ఒకేసారి fail అయితే, అందరూ **ఖచ్చితంగా
అదే schedule** తో retry చేస్తే (1s, 2s, 4s...), అందరూ ఏకకాలంలో మళ్ళీ hit
చేస్తారు — ఇదే **"thundering herd"** సమస్య. Jitter (కొంత random వ్యత్యాసం
add చేయడం) దీన్ని spread చేస్తుంది.

```java
class RetryWithBackoff {
    private final int maxRetries;
    private final long baseDelayMs;

    RetryWithBackoff(int maxRetries, long baseDelayMs) {
        this.maxRetries = maxRetries;
        this.baseDelayMs = baseDelayMs;
    }

    <T> T executeWithRetry(Callable<T> operation, Predicate<Exception> isRetryable) throws Exception {
        Exception lastException = null;
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return operation.call();
            } catch (Exception e) {
                lastException = e;
                if (!isRetryable.test(e) || attempt == maxRetries) {
                    throw e; // retryable కాదు, లేదా బడ్జెట్ అయిపోయింది — వెంటనే fail అవ్వండి
                }
                long delay = computeBackoffWithJitter(attempt);
                Thread.sleep(delay);
            }
        }
        throw lastException; // ఇక్కడికి చేరుకోకూడదు, కానీ compiler కి కావాలి
    }

    private long computeBackoffWithJitter(int attempt) {
        long exponentialDelay = baseDelayMs * (1L << attempt); // baseDelay * 2^attempt
        long jitter = ThreadLocalRandom.current().nextLong(exponentialDelay / 2);
        return exponentialDelay + jitter; // "full jitter" style randomization
    }
}
```

**కీలక design decisions, senior interview లో explicitly అడిగేవి:**
1. **`isRetryable` predicate తప్పనిసరి** — network timeout retry చేయవచ్చు,
   కానీ `400 Bad Request` (validation error) ని retry చేయడం **అర్థం
   లేనిది** — అది ఎప్పటికీ succeed అవదు, resources వృధా అవుతాయి.
2. **`maxRetries` (retry budget)** — infinite retry ఎప్పుడూ వద్దు.
3. **Idempotency తప్పనిసరి precondition** — section 16.6 చూడండి.

### ENGLISH INTERVIEW ANSWER

"Retrying isn't a default-on decision — it's a design decision that
requires answering several questions first, which is exactly the
junior-vs-senior distinction this whole program has emphasized: is this
failure actually transient (network timeout, 503) or permanent (400, 401,
a business rule violation)? Retrying a permanent failure just wastes
resources and delays the inevitable failure response. Is the operation
idempotent — can it be safely repeated without double effect? And what's
the retry budget — unconditional retry can turn a struggling downstream
service into a completely overwhelmed one. My implementation takes an
explicit `isRetryable` predicate rather than retrying everything, uses
exponential backoff so each retry waits longer than the last, easing
pressure on a recovering service, and adds jitter — randomizing the exact
delay — specifically to prevent many clients that failed at the same
moment from all retrying in perfect synchrony and re-creating the exact
overload that caused the original failure, known as a thundering herd."

---

## 16.6 IDEMPOTENCY — SAFE RETRIES FOR NON-IDEMPOTENT OPERATIONS

### TELUGU EXPLANATION

**Idempotent operation** అంటే, ఒకసారి చేసినా, పదిసార్లు చేసినా, **ఫలితం
ఒకటే** ఉండేది. `GET`, `PUT` (usually), `DELETE` సాధారణంగా idempotent.
**`POST` (ఉదా: "charge this credit card", "place this order") సాధారణంగా
NOT idempotent** — retry చేస్తే, **duplicate charge/order** జరగవచ్చు.

**సమస్య:** section 16.5 యొక్క retry mechanism ఒక payment API call కి
వాడితే — request నిజంగా succeed అయిపోయి, **response మాత్రమే** network
మీద పోయినా (timeout), client అది fail అయిందని అనుకుని retry చేస్తుంది
— ఫలితం: **duplicate charge**, client fault కాకుండానే.

**పరిష్కారం — Idempotency Key:** Client ప్రతి **logical operation**
కి ఒక unique key (UUID) generate చేసి, request తో పంపుతుంది. Server
ఆ key ని ఇప్పటికే process చేసి ఉంటే, **అదే మునుపటి response ని తిరిగి
ఇస్తుంది**, operation ని మళ్ళీ execute చేయదు.

```java
class IdempotentPaymentService {
    private final Map<String, PaymentResult> processedRequests = new ConcurrentHashMap<>();

    PaymentResult processPayment(String idempotencyKey, PaymentRequest request) {
        // ఇప్పటికే ప్రాసెస్ అయిన key అయితే, మళ్ళీ charge చేయకుండా, cached result తిరిగి ఇవ్వండి
        PaymentResult existing = processedRequests.get(idempotencyKey);
        if (existing != null) {
            return existing; // idempotent behavior — safe retry
        }

        // computeIfAbsent atomic గా చేస్తుంది — రెండు concurrent requests (అదే key తో)
        // ఒకేసారి వచ్చినా, actualChargeLogic ఒక్కసారి మాత్రమే run అవుతుంది
        return processedRequests.computeIfAbsent(idempotencyKey, key -> actuallyChargeCard(request));
    }

    private PaymentResult actuallyChargeCard(PaymentRequest request) {
        // వాస్తవ payment gateway call ఇక్కడ జరుగుతుంది — ఒక్కసారి మాత్రమే, per idempotency key
        return new PaymentResult("SUCCESS", UUID.randomUUID().toString());
    }
}
```

**Design note:** `computeIfAbsent` వాడటం **కేవలం style కోసం కాదు** — ఇది
Book 1 Chapter 4 (`ConcurrentHashMap`) సూత్రాన్ని నిజంగా correctness కోసం
వాడటం: `get()` + `if null, put()` అనే రెండు-దశల చెక్ **race condition**
కి గురయ్యే అవకాశం ఉంది (రెండు threads ఏకకాలంలో "not found" చూసి, రెండూ
charge చేయవచ్చు) — `computeIfAbsent` ఈ మొత్తం check-and-set ని **atomic**
గా చేస్తుంది.

### ENGLISH INTERVIEW ANSWER

"Retry mechanisms are only safe for idempotent operations — and a lot of
real operations, like charging a payment, aren't naturally idempotent.
The standard fix is an idempotency key: the client generates a unique key
per logical operation attempt and sends it with every retry of that same
logical request. The server checks whether it's already processed that
key; if so, it returns the cached result instead of re-executing the
operation, making an inherently non-idempotent operation safe to retry
from the client's perspective. The implementation detail that actually
matters for correctness under concurrency is using `computeIfAbsent` on a
`ConcurrentHashMap` rather than a manual check-then-act — a plain 'get, if
null then put' sequence has a race window where two concurrent requests
with the same idempotency key could both see 'not yet processed' and both
execute the charge, exactly the double-charge bug the idempotency key was
supposed to prevent. `computeIfAbsent` closes that race atomically."

---

## 16.7 SENIOR VS JUNIOR THINKING (CHAPTER SUMMARY)

| Component | Junior approach | Senior approach |
|---|---|---|
| LRU Cache | HashMap + manually shifting a List (O(n) operations) | HashMap + doubly linked list, true O(1) |
| Rate limiter | Fixed window counter (allows 2x burst at window edges) | Token bucket (smooth, burst-tolerant, correct average rate) |
| Producer-Consumer | Manual `wait()`/`notify()`, error-prone | `BlockingQueue`, bounded, tested JDK primitive |
| Thread pool | Doesn't consider a failing task killing the worker thread | Catches exceptions per-task inside the worker loop |
| Retry | Retries everything, unconditionally, no backoff | Retryable-only, exponential backoff + jitter, bounded budget |
| Non-idempotent retryable operations | Just retries and hopes for the best | Idempotency key + atomic check-and-store |

---

## 16.8 INTERVIEW QUESTION BANK — CHAPTER 16

**Basic:** 1. Why does LRU Cache need both a HashMap and a linked list? 2.
What problem does jitter solve in retry backoff?

**Intermediate:** 3. Why is a bounded queue important in both the
Producer-Consumer and custom Thread Pool implementations? 4. Why must a
thread pool's worker loop catch exceptions around task execution?

**Senior:** 5. Design a distributed version of the rate limiter (multiple
service instances sharing one limit) — what changes, and what new
failure modes appear (e.g., Redis being unavailable)? 6. Why is
`computeIfAbsent` on `ConcurrentHashMap` essential for correct idempotency
key handling, and what specifically goes wrong without it?

**Architect:** 7. You're designing a payment system's retry and
idempotency strategy end-to-end — client retries, server-side idempotency
keys, and downstream payment gateway calls that might themselves be slow
or ambiguous (timeout with unknown outcome). Describe the full flow and
where each safeguard sits.

**Scenario:** 8. A production LRU cache implementation using
`LinkedHashMap` with `accessOrder=true` works correctly in single-threaded
tests but corrupts under concurrent load. What's missing?

**Trick:** 9. "Adding retries to a system always improves its reliability." True or false?

<details><summary>Key answers</summary>

- Q5: Move the counter/token state to a shared store (Redis, typically,
  using atomic `INCR` + `EXPIRE` or a Lua script for atomicity), so every
  instance checks/decrements the same shared state; new failure modes
  include the shared store itself becoming a bottleneck or single point of
  failure — production systems often add a local fallback (e.g., a
  conservative local limit) for when the shared store is unreachable,
  rather than failing open or closed unconditionally.
- Q6: Without `computeIfAbsent`'s atomicity, a `get()` returning null
  followed by a separate `put()` has a race window where two threads with
  the same idempotency key can both observe "not yet processed" and both
  proceed to execute the (non-idempotent) underlying operation — exactly
  reproducing the double-charge problem idempotency keys exist to prevent.
- Q8: `LinkedHashMap` itself is not thread-safe — concurrent access needs
  external synchronization (wrapping every access in a lock) or a
  different structure entirely; this is a direct application of Book 1
  Chapter 4's "HashMap variants and thread-safety" material.
- Q9: False — retries without proper idempotency handling can actively
  *reduce* reliability (duplicate charges/orders), and retries without a
  retry budget or backoff can turn a partially-degraded downstream service
  into a fully-overwhelmed one (a retry storm), making the overall system
  less reliable, not more.

</details>

---

## 16.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Why doesn't a plain `HashMap` alone suffice for an O(1) LRU cache, even with perfect key lookups?
- **Coding Check:** Implement a "sliding window rate limiter" (as an alternative to token bucket) and compare its burst-handling behavior.
- **Explanation Check:** Explain in English, to a teammate proposing "let's just retry every failed API call three times," what questions you'd ask before agreeing.
- **Real-World Check:** Your team's payment retry logic occasionally produces duplicate charges reported by customers. Diagnose using this chapter's idempotency material and propose the fix.
- **Senior Check:** When would you choose a fixed-window rate limiter over token bucket despite its burst-allowance flaw?
- **Master Check:** Design the complete flow for a "safe checkout" feature: client-generated idempotency key, server-side idempotent payment processing, and a retry policy with backoff+jitter — describe how all three pieces from this chapter compose together into one coherent, production-safe system.

<details><summary>Answers</summary>

- Real-World Check: Root cause is almost certainly missing or improperly
  scoped idempotency keys (or a check-then-act race instead of
  `computeIfAbsent`) on the payment endpoint — fix by requiring a
  client-generated idempotency key per checkout attempt and using an
  atomic check-and-store on the server before charging.
- Senior Check: When simplicity and ease of reasoning matter more than
  precise burst control, and the 2x-at-boundary flaw is acceptable for the
  use case (e.g., a coarse, non-critical internal rate limit) — token
  bucket is strictly more correct but marginally more complex to implement and reason about.
- Master Check: Client generates a UUID idempotency key once per checkout
  attempt (not per HTTP retry — the same key is reused across retries of
  the *same* logical attempt); client's retry logic (16.5) uses exponential
  backoff+jitter and only retries genuinely transient failures (network
  timeout, 503), never a definitive decline; server's payment endpoint
  (16.6) uses `computeIfAbsent` keyed on the idempotency key to guarantee
  the actual charge logic runs at most once per logical attempt regardless
  of how many times the request is retried — the three chapter components
  compose into exactly one safe, non-duplicating checkout flow.

</details>

---

## 16.10 CHEAT SHEET

| Component | Core data structure/technique | Key correctness detail |
|---|---|---|
| LRU Cache | HashMap + doubly linked list (or `LinkedHashMap`) | Dummy head/tail sentinels avoid null-checks |
| Rate Limiter | Token Bucket | Lazy refill by elapsed time, not a background thread |
| Producer-Consumer | `BlockingQueue` (bounded) | `put()` blocks = natural backpressure |
| Thread Pool | Worker threads + shared bounded queue | MUST catch exceptions per-task inside the worker loop |
| Retry | Exponential backoff + jitter | Only retry a checked `isRetryable` predicate; bounded attempts |
| Idempotency | Idempotency key + `ConcurrentHashMap.computeIfAbsent` | Atomic check-and-store, not get-then-put |

---

## BOOK 2 — CHAPTER 16 COMPLETE

*(All 16 chapters of Book 2's chapter content are now complete. See
`final-assessment-mock-interview-project.md` in this directory for the
cumulative Final Assessment, the DSA Mock Interview round, and the Book 2
capstone Project Assignment.)*
