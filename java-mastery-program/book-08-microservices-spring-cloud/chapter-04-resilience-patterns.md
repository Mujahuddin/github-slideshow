# CHAPTER 4 — RESILIENCE PATTERNS: CIRCUIT BREAKER, RETRY, TIMEOUT, BULKHEAD

> This chapter is the direct payoff of Chapter 3's cascading-failure
> setup, and it's where this program's opening example of junior-vs-senior
> reasoning ("let's add retry" vs. asking about idempotency, failure
> type, retry budget, traffic storms) becomes fully concrete.

---

## 4.1 CONCEPT: Circuit Breaker — Stop Calling What's Already Broken

### TELUGU EXPLANATION

Chapter 3 లో మనం చూసిన cascading failure సమస్యకి, **Circuit Breaker**
ప్రధాన పరిష్కారం. ఆలోచన: ఒక downstream service **repeatedly fail**
అవుతుంటే, దాన్ని **మళ్ళీ మళ్ళీ call చేస్తూ ఉండటం అర్థరహితం** — ఇది
ఆ failing service మీద మరింత load పెంచుతుంది (అది recover అవ్వకుండా
ఆపొచ్చు), మరియు calling service యొక్క threads ని వృధాగా block
చేస్తుంది. Circuit Breaker ఒక **electrical circuit breaker** లాగే
పని చేస్తుంది — repeated faults గుర్తించి, **circuit ని "trip" చేసి**,
తర్వాతి calls ని **వెంటనే fail చేస్తుంది** (downstream ని call
చేయకుండానే), కొంత సమయం తర్వాత "recover అయ్యిందేమో" అని జాగ్రత్తగా
మళ్ళీ try చేస్తుంది.

**మూడు states (Resilience4j — Spring Boot యొక్క standard choice,
Netflix Hystrix ని replace చేసింది):**

```
CLOSED (normal) --failure rate > threshold--> OPEN (fail-fast)
                                                    |
                                          (wait duration తర్వాత)
                                                    ↓
                                              HALF_OPEN (కొన్ని test calls అనుమతిస్తుంది)
                                              /              \
                                    (success rate good)   (still failing)
                                         ↓                      ↓
                                      CLOSED                   OPEN
```

- **CLOSED:** Normal operation — calls downstream కి వెళ్తాయి, failures
  track అవుతాయి (sliding window లో — ఉదా: చివరి 100 calls, ఎన్ని fail
  అయ్యాయో).
- **OPEN:** Failure rate ఒక threshold (ఉదా: 50%) దాటితే — circuit
  "trips" — తర్వాతి calls **వెంటనే fail** అవుతాయి (`CallNotPermittedException`),
  downstream ని **అసలు call చేయకుండానే** — ఇది downstream ని rest
  ఇవ్వడానికి, calling service threads ని waste చేయకుండా ఉండటానికి.
- **HALF_OPEN:** ఒక wait duration తర్వాత, పరిమిత సంఖ్యలో "test" calls
  అనుమతిస్తుంది — అవి succeed అయితే CLOSED కి తిరిగి వెళ్తుంది, ఇంకా
  fail అయితే OPEN కి తిరిగి వెళ్తుంది.

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "chargeFallback")
PaymentResponse charge(ChargeRequest request) {
    return paymentClient.charge(request);
}

PaymentResponse chargeFallback(ChargeRequest request, Exception e) {
    // Circuit open అయినప్పుడు, లేదా actual failure అయినప్పుడు run అవుతుంది
    return PaymentResponse.pending("Payment service unavailable, queued for retry");
}
```

### ENGLISH INTERVIEW ANSWER

"A circuit breaker prevents the exact cascading failure from Chapter 3 —
if a downstream service is genuinely failing repeatedly, continuing to
call it wastes the caller's threads and adds load to a service that's
already struggling to recover. It works in three states: Closed is
normal operation, tracking failures over a sliding window; once the
failure rate crosses a threshold, it trips to Open, where subsequent
calls fail immediately without even attempting the downstream call,
giving both sides relief; after a wait duration, it moves to Half-Open,
allowing a small number of test calls through to check whether the
dependency has recovered, transitioning back to Closed on success or
back to Open on continued failure. I always pair a circuit breaker with
a fallback method — a sensible degraded response, like queuing a payment
for later processing, rather than just propagating the failure raw to
the end user."

---

## 4.2 CONCEPT: Retry Revisited — And Its Dangerous Interaction with Circuit Breakers

### TELUGU EXPLANATION

Book 2 Chapter 16 లో మనం retry సూత్రాలు (idempotency, retryable
exceptions మాత్రమే, exponential backoff + jitter) నేర్చుకున్నాం.
Resilience4j ఇది **annotation గా** ఇస్తుంది:

```java
@Retry(name = "inventoryService")
@CircuitBreaker(name = "inventoryService", fallbackMethod = "checkStockFallback")
StockResponse checkStock(String productId) {
    return inventoryClient.checkStock(productId);
}
```

**⚠️ అత్యంత ముఖ్యమైన senior-level trap — Retry + Circuit Breaker
ordering:** ఈ రెండు annotations యొక్క **order ముఖ్యం**. `@Retry`
`@CircuitBreaker` కి **లోపల** (inner) ఉంటే, ప్రతి retry attempt,
circuit breaker యొక్క failure count లోకి **విడివిడిగా** count
అవుతుంది — ఇది circuit ని **త్వరగా trip** చేయవచ్చు (ఒక్క logical
call, 3 retries తో, circuit కి 3 failures లా కనిపిస్తుంది).

**ఇంకా ప్రమాదకరమైనది — "Retry Storm":** ఒక downstream service
degrade అవుతున్నప్పుడు (పూర్తిగా down కాదు, కానీ slow/intermittent),
**అనేక callers ఏకకాలంలో retry చేస్తే** — ఇది ఆ downstream మీద
**ఆకస్మికంగా ఎక్కువ load** పెంచుతుంది (retries యొక్క combined
volume), ఇది **partial degradation ని పూర్తి outage గా మార్చవచ్చు**
— ఇదే master prompt లో మనం మొదట్లో చూసిన సూత్రం: **"Retries could
create a traffic storm?"** అనేది ఒక theoretical concern కాదు, ఇది
ఒక **నిజమైన, well-documented production failure mode**.

**Senior mitigations:**
1. **Exponential backoff + jitter** (Book 2 Chapter 16) — retries ని
   time లో spread చేయడం.
2. **Retry budget** — ఒక service, total requests లో **గరిష్టంగా
   ఎంత శాతం** retries గా ఉండాలో limit చేయడం (ఉదా: 10% కంటే ఎక్కువ
   కాదు) — ఇది "ప్రతి failed request ని retry చేయడం" అనే unconditional
   policy కంటే safer.
3. **Circuit breaker ని retry కి బయట ఉంచడం** (`@CircuitBreaker` outer,
   `@Retry` inner) — circuit open అయిన తర్వాత, **retries కూడా ఆగిపోతాయి**
   (circuit fail-fast చేస్తుంది, retry కి అవకాశమే రాదు) — ఇది retry
   storm ని పరిమితం చేస్తుంది.

### ENGLISH INTERVIEW ANSWER

"Resilience4j gives retry as a clean annotation, but I'm careful about
one specific danger: combining retry and circuit breaker without
thinking about their interaction. If retry sits inside the circuit
breaker, each individual retry attempt counts separately toward the
circuit's failure threshold, which can trip the circuit faster than
intended — one logical failed call with 3 retries looks like 3 failures
to the breaker. The bigger, genuinely production-documented danger is a
retry storm: when a downstream service is degraded but not fully down,
many callers retrying simultaneously multiplies load on exactly the
service that's already struggling, potentially turning partial
degradation into a full outage. This is precisely the concern this whole
program opened with — a senior engineer doesn't just add retry, they ask
whether retries could create a traffic storm. My mitigations: exponential
backoff with jitter to spread retries out in time, a retry budget capping
what fraction of total traffic can be retries, and structuring the
circuit breaker to wrap the retry — once the circuit opens, retry attempts
stop entirely too, since the call fails fast before ever reaching the
retry logic, which directly caps how much retry-driven load a struggling
dependency can receive."

---

## 4.3 CONCEPT: Bulkhead — Isolating Failure Blast Radius

### TELUGU EXPLANATION

**Bulkhead** (ship యొక్క watertight compartments నుండి పేరు వచ్చింది
— ఒక compartment నీటితో నిండినా, మిగతావి safe గా ఉంటాయి) — ఒక
service కి **బహుళ downstream dependencies** ఉంటే, **ఒక్కో దానికి
వేరే, పరిమిత resource pool** (threads, connections) కేటాయించడం —
ఒక dependency slow అయినా, అది **మిగతా dependencies కి కేటాయించిన
resources** ని వినియోగించదు:

```java
@Bulkhead(name = "recommendationService", type = Bulkhead.Type.THREADPOOL)
List<Product> getRecommendations(String userId) {
    return recommendationClient.getRecommendations(userId);
}
```

**Book 1 Chapter 10 తో direct సారూప్యత:** ఇది ఖచ్చితంగా, ఒక
`ThreadPoolExecutor` ని service కి కేటాయించడం లాంటిదే — `recommendationService`
కి కేటాయించిన thread pool నిండిపోయినా (recommendationService slow
అయితే), **`paymentService`/`inventoryService` కి కేటాయించిన వేరే
thread pools ప్రభావితం కావు** — Chapter 3 లో మనం చూసిన "ఒక్క slow
dependency, మొత్తం service ని block చేయడం" సమస్యకి ఇదే ఖచ్చితమైన
పరిష్కారం.

**రెండు రకాలు:** **Thread Pool Bulkhead** (ప్రతి dependency కి వేరే
thread pool — strongest isolation, కానీ ఎక్కువ resource overhead) vs
**Semaphore Bulkhead** (ఒకే thread pool, కానీ ఒక్కో dependency కి
concurrent calls సంఖ్య పరిమితం చేయడం — lighter-weight).

### ENGLISH INTERVIEW ANSWER

"Bulkhead isolation, named after a ship's watertight compartments,
allocates separate, bounded resource pools per downstream dependency,
so one slow dependency can't consume the resources needed to call
others. This is directly parallel to Book 1 Chapter 10's dedicated
thread pool per workload type — a thread-pool bulkhead gives each
dependency its own thread pool entirely, the strongest isolation at the
cost of more total threads/memory; a semaphore bulkhead shares one pool
but caps how many concurrent calls any single dependency can occupy, a
lighter-weight alternative. This is exactly the fix for Chapter 3's
scenario where a slow recommendation service could exhaust threads that
the payment or inventory calls also needed — with bulkhead isolation in
place, `recommendationService`'s pool can fill up completely while
payment and inventory calls proceed entirely unaffected."

---

## 4.4 CONCEPT: Composing All Four Patterns Together

### TELUGU EXPLANATION

Production లో, ఒక్క downstream call కి **అన్ని patterns కలిపి** apply
చేయడం సాధారణం — Resilience4j **recommended composition order** ఇది:

```java
@Bulkhead(name = "paymentService")          // 1. మొదట resource isolation
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallback") // 2. తర్వాత fail-fast if broken
@Retry(name = "paymentService")              // 3. తర్వాత, circuit closed అయితేనే, retry
@TimeLimiter(name = "paymentService")        // 4. ప్రతి attempt కి timeout (Chapter 3)
PaymentResponse charge(ChargeRequest request) { ... }
```

**ప్రతి pattern వేరే సమస్యని పరిష్కరిస్తుంది:**
- **Bulkhead:** "ఈ dependency, నా మిగతా resources ని తినేయకుండా చూడు."
- **Circuit Breaker:** "ఇది ఇప్పటికే broken అయితే, అస్సలు try చేయకు."
- **Retry:** "ఇది transient failure అయితే, జాగ్రత్తగా మళ్ళీ try చేయి."
- **Timeout:** "ఏ ఒక్క attempt కూడా అనంతంగా wait చేయకూడదు."

### ENGLISH INTERVIEW ANSWER

"In production, these four patterns compose together, each solving a
distinct problem: bulkhead ensures this dependency can't starve resources
needed elsewhere, circuit breaker ensures we stop trying entirely once a
dependency is known to be broken, retry handles transient failures
carefully within whatever the circuit breaker allows, and timeout bounds
every individual attempt so nothing waits forever. Getting the
composition order right matters — bulkhead and circuit breaker generally
wrap outside retry, so a broken circuit stops retries from happening at
all, rather than retry attempts happening first and only then being
counted (potentially incorrectly) toward the circuit's failure
threshold. I think of this stack as answering, in order: can I even
attempt this (bulkhead capacity available)? Should I even attempt this
(circuit closed)? If it fails, is it worth trying again (retry)? And no
matter what, how long am I willing to wait for any single attempt (timeout)?"

---

## 4.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| A downstream call fails | "Let's add retry" | Asks: is it idempotent? What's the failure type? Could retries create a traffic storm? |
| Repeated downstream failures | Keeps retrying indefinitely | Trips a circuit breaker, fails fast, gives the dependency room to recover |
| One slow dependency among several | Doesn't isolate it, risking shared thread pool exhaustion | Uses bulkhead isolation so it can't affect calls to other dependencies |
| Combining retry + circuit breaker | Doesn't consider ordering | Structures circuit breaker to wrap retry, capping retry-driven load during an outage |

---

## 4.6 COMMON MISTAKES

1. Adding retry without considering idempotency, failure type, or
   traffic-storm risk.
2. Not using a circuit breaker at all, letting a service hammer an
   already-failing dependency indefinitely.
3. Getting retry/circuit-breaker composition order wrong, letting retries
   inflate the circuit's failure count or bypass the circuit's protection.
4. Not isolating dependencies with bulkheads, allowing one slow
   dependency to exhaust resources needed for unrelated calls.
5. Not configuring a meaningful fallback for a circuit breaker, letting
   the "protection" just turn one error into a different, equally
   unhelpful error for the end user.

---

## 4.7 INTERVIEW QUESTION BANK — CHAPTER 4

**Basic:** 1. What are the three circuit breaker states? 2. What is a
"retry storm"?

**Intermediate:** 3. Why does retry/circuit-breaker composition order
matter? 4. What's the difference between thread-pool and semaphore bulkheads?

**Senior:** 5. Design the complete resilience configuration (all four
patterns) for a payment service call, justifying each setting. 6. Why is
a circuit breaker fallback method as important as the circuit breaker itself?

**Architect:** 7. You're setting resilience standards across 50
microservices with varying criticality. What would you standardize
(default timeout ranges, retry budgets, circuit breaker thresholds) versus
leave per-service?

**Scenario:** 8. During an incident, a downstream service degrades
(50% of calls slow, not failing outright), and the overall system
experiences a much larger outage than the 50% degradation would suggest.
Diagnose using this chapter's material.

**Trick:** 9. "A circuit breaker eliminates the need for timeouts, since
it stops calling failing services anyway." True or false?

<details><summary>Key answers</summary>

- Q5: Bulkhead: dedicated thread pool sized to expected payment call
  volume, isolated from other dependencies. Circuit breaker: failure
  rate threshold (e.g., 50% over a sliding window of 20 calls), wait
  duration in open state (e.g., 30 seconds) before half-open testing,
  with a fallback that queues the payment for async retry rather than
  failing the whole order. Retry: only for genuinely transient failure
  types (timeout, 503), never for a definitive decline, with exponential
  backoff+jitter, max 2-3 attempts, wrapped inside the circuit breaker so
  an open circuit stops retries too. Timeout: connect/read timeouts set
  based on the payment gateway's documented expected latency plus a
  reasonable margin, not a generic default.
- Q6: Without a meaningful fallback, tripping the circuit just converts
  "the call failed with a downstream error" into "the call failed with a
  `CallNotPermittedException`" — still a failure from the caller's
  perspective, with no actual improvement in user experience; the real
  value of a circuit breaker is enabling a *graceful degraded response*
  (a cached value, a queued-for-later status, a sensible default) instead
  of blindly propagating failure, which requires a thoughtfully designed
  fallback, not just tripping.
- Q7: Standardize: minimum required timeout configuration (no service
  ships without explicit timeouts), a retry budget ceiling
  (organization-wide policy, e.g., no more than 10-20% of traffic as
  retries), and mandatory circuit breaker + fallback for any
  cross-service call. Leave per-service: the specific timeout durations,
  failure-rate thresholds, and wait durations, since these depend on each
  service's actual latency profile and criticality — a payment service
  and an analytics-logging service warrant different specific numbers
  even under the same overall policy framework.
- Q8: This is the retry-storm/cascading-failure amplification pattern —
  with 50% of calls slow (not failing), many callers likely retried
  (each slow call looking like a candidate for retry), multiplying
  effective load on the already-degraded service well beyond its actual
  50% capacity reduction, likely also exhausting caller thread pools
  waiting on slow responses (if timeouts were too generous) — the
  combination of retry amplification and thread pool exhaustion without
  bulkhead isolation turned a 50% degradation into a much larger outage.
- Q9: False — even with a circuit breaker, calls made while the circuit
  is closed (normal operation, before enough failures accumulate to
  trip it) still need a timeout, since a single slow-but-not-yet-failing
  call can still block a thread for an unbounded time without one;
  timeout and circuit breaker solve different, complementary problems —
  timeout bounds a single call's duration, circuit breaker stops future
  calls once a pattern of failure is detected.

</details>

---

## 4.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does the circuit breaker's Half-Open state exist instead of just going straight back to Closed after the wait duration?
- **Coding Check:** Implement a Feign-client-wrapped call with `@Bulkhead`, `@CircuitBreaker`, `@Retry`, and `@TimeLimiter` composed correctly, plus a meaningful fallback method.
- **Explanation Check:** Explain in English, using a concrete number (e.g., "3 retries on a 50%-failure-rate circuit breaker with a 10-call window"), how retry can inflate a circuit breaker's failure count if ordered incorrectly.
- **Real-World Check:** Your team's service calls five downstream dependencies from one shared thread pool, and an incident with one dependency caused total service unavailability. Redesign using this chapter's material.
- **Senior Check:** When would you deliberately choose NOT to add a circuit breaker to a particular downstream call?
- **Master Check:** Design the complete resilience strategy for an order-checkout flow calling Inventory (critical, must succeed), Recommendations (non-critical, nice-to-have), and Analytics logging (fire-and-forget) — specify different resilience configurations appropriate to each dependency's actual criticality.

<details><summary>Answers</summary>

- Real-World Check: Apply bulkhead isolation (thread-pool or semaphore)
  per dependency instead of one shared pool — the one failing
  dependency's calls exhausting a dedicated, bounded pool would no longer
  affect the threads available for the other four dependencies, directly
  fixing the "one dependency took down everything" failure mode.
- Senior Check: For a genuinely low-stakes, rarely-called, or
  non-critical dependency where the operational overhead of configuring
  and monitoring a circuit breaker isn't justified by the actual risk —
  e.g., an occasional, non-blocking analytics call where a simple
  timeout and fire-and-forget error handling is sufficient without the
  added complexity of full circuit breaker state management.
- Master Check: Inventory (critical): full stack — bulkhead, circuit
  breaker with a conservative fallback (e.g., "assume unavailable, don't
  oversell" rather than optimistically proceeding), retry for transient
  failures only, strict timeout — since correctness here directly
  affects whether the order can be trusted. Recommendations
  (non-critical): circuit breaker with fallback returning an empty/
  default recommendation list (never blocks checkout), short timeout,
  no retry (not worth the latency cost for a nice-to-have). Analytics
  logging (fire-and-forget): short timeout, no retry, no circuit breaker
  even necessary — failures here should be silently swallowed (perhaps
  logged locally) since they must never affect the checkout flow's success at all.

</details>

---

## 4.9 CHEAT SHEET

| Pattern | Solves | Rule |
|---|---|---|
| Circuit Breaker | Stop hammering an already-broken dependency | Closed → Open (fail fast) → Half-Open (test) → Closed/Open |
| Retry | Recover from transient failures | Idempotent operations only, backoff+jitter, bounded attempts, budget-capped |
| Bulkhead | Isolate one dependency's failure from affecting others | Dedicated thread pool or semaphore per dependency |
| Timeout | Bound how long any single call can block | Explicit, per-dependency, never rely on defaults |
| Composition order | Bulkhead → Circuit Breaker → Retry → Timeout (per attempt) | An open circuit should stop retries too |
| Fallback | Makes circuit breaker actually useful | Graceful degradation, not just a different error |

---

*(Continues to Chapter 5 — Distributed Transactions & the Saga Pattern.)*
