# CHAPTER 7 — ACTUATOR, LOGGING, PRODUCTION CONFIGURATION

---

## 7.1 CONCEPT: Spring Boot Actuator — Production Observability, Built In

### TELUGU EXPLANATION

**Actuator** Spring Boot యొక్క production-readiness features ఇచ్చే
module — HTTP endpoints ద్వారా, running application గురించి సమాచారం
బయటపెడుతుంది:

| Endpoint | ఇచ్చే సమాచారం |
|---|---|
| `/actuator/health` | Application "UP" గా ఉందా (DB connection, disk space వంటివి కూడా చెక్ చేస్తుంది) |
| `/actuator/metrics` | JVM metrics (heap usage, GC — Book 1 Chapter 1/11 direct connection), HTTP request counts, custom metrics |
| `/actuator/info` | Build version, git commit వంటి static metadata |
| `/actuator/env` | Active configuration properties (⚠️ **sensitive** — section 7.2 చూడండి) |
| `/actuator/loggers` | Runtime లో log levels **మార్చడానికి** — restart అవసరం లేకుండా! |

**`/actuator/health` ఎందుకు ముఖ్యం (Kubernetes connection):**
Kubernetes **liveness/readiness probes** (Book 12 లో వివరంగా) ఖచ్చితంగా
ఈ endpoint నే hit చేస్తాయి, pod healthy గా ఉందో లేదో నిర్ణయించడానికి.
Custom `HealthIndicator` రాయవచ్చు, మీ specific dependencies (ఉదా: ఒక
downstream service reachable గా ఉందా) చెక్ చేయడానికి:

```java
@Component
class DownstreamServiceHealthIndicator implements HealthIndicator {
    private final RestClient restClient;

    @Override
    public Health health() {
        try {
            restClient.get().uri("/ping").retrieve().toBodilessEntity();
            return Health.up().build();
        } catch (Exception e) {
            return Health.down(e).withDetail("service", "payment-gateway").build();
        }
    }
}
```

### ENGLISH INTERVIEW ANSWER

"Actuator gives me production observability out of the box —
`/health` for liveness/readiness (which is exactly what Kubernetes probes
hit to decide whether to route traffic to or restart a pod), `/metrics`
for JVM and application-level metrics, and `/loggers` for runtime log
level changes without a restart, which is genuinely useful during a live
incident when you need more verbose logging temporarily. I write custom
`HealthIndicator`s when the default checks (disk space, DB connectivity)
don't cover what actually matters for this service — like whether a
critical downstream dependency is reachable, since a service that's
technically 'up' but can't reach a required dependency shouldn't be
reported healthy to the orchestrator."

---

## 7.2 CONCEPT: Securing Actuator — A Real, Common Misconfiguration

### TELUGU EXPLANATION

**ఇది production లో genuinely జరిగే security incident:** Default గా,
అనేక Actuator endpoints (`/env`, `/heapdump`, `/threaddump`) **sensitive
సమాచారం** బయటపెడతాయి — environment variables (secrets సహా!), heap
contents. **Actuator endpoints ని public గా exposed చేయడం, ఒక
నిజమైన, బాగా-known vulnerability class** (CVE databases లో దీనికి
సంబంధించిన entries ఉన్నాయి — Book 15 Security లో వివరంగా).

**Production checklist:**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics # అవసరమైనవి మాత్రమే — "*" (అన్నీ) ఎప్పుడూ వద్దు
  endpoint:
    health:
      show-details: when-authorized # unauthenticated users కి full details చూపించకండి
```

అదనంగా, Actuator endpoints ని **Spring Security తో protect చేయాలి**
(Chapter 6) — `/actuator/**` ని admin-only గా, లేదా internal
network/service mesh కి మాత్రమే accessible గా configure చేయాలి, public
internet కి కాదు.

### ENGLISH INTERVIEW ANSWER

"Actuator endpoints are a genuinely real production security
misconfiguration risk — `/env` can leak environment variables including
secrets, `/heapdump` can expose in-memory sensitive data. My default
production posture is to expose only the specific endpoints actually
needed — `health`, `info`, `metrics` — never a blanket wildcard, and to
put Actuator behind Spring Security so only authenticated, appropriately-
privileged callers (often just internal monitoring infrastructure, not
public internet at all) can reach it. This is exactly the kind of default-
insecure-configuration issue that shows up in real vulnerability
scanners and security audits, which is why I treat 'did we lock down
Actuator' as a standard item on any production readiness checklist."

---

## 7.3 CONCEPT: Structured Logging and Correlation IDs

### TELUGU EXPLANATION

**Junior logging:** `System.out.println("Processing order " + orderId);`
— **ఎప్పుడూ production code లో వాడకూడదు** — no log levels, no
structure, `System.out` synchronous గా, buffering లేకుండా, performance
impact చూపిస్తుంది.

**Senior logging:** SLF4J (facade) + Logback (implementation, Spring
Boot default) వాడి, **structured, leveled** logging:

```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

void placeOrder(Order order) {
    log.info("Placing order for customerId={}", order.getCustomerId()); // parameterized — string concatenation కాదు
    try {
        // ...
    } catch (PaymentException e) {
        log.error("Payment failed for orderId={}", order.getId(), e); // exception ని కూడా pass చేయండి — full stack trace log అవుతుంది
    }
}
```

**ఎందుకు parameterized logging (`{}`) వాడాలి, string concatenation
కాదు:** `log.info("Order " + orderId)` — string concatenation **ఎప్పుడూ**
జరుగుతుంది, log level DEBUG/TRACE disabled అయినా కూడా (Book 1 Chapter
3 string concatenation performance సూత్రం ఇక్కడ). `log.info("Order
{}", orderId)` — log level enable అయితేనే string నిజంగా build అవుతుంది,
లేకపోతే ఆ cost avoid అవుతుంది.

**Correlation IDs (Distributed Tracing యొక్క పునాది, Book 8 లో వివరంగా):**
ఒక request, microservices అనేకం గుండా వెళ్తే, ప్రతి service తన own
logs రాస్తుంది — వీటిని **ఒకే request కి సంబంధించినవి** అని కలపడం ఎలా?
**Correlation ID** (ఒక unique ID, request మొదట్లో generate అయ్యి,
ప్రతి downstream call కి header గా propagate అవుతుంది) దీన్ని
పరిష్కరిస్తుంది. Java లో ఇది **MDC (Mapped Diagnostic Context)** ద్వారా
implement అవుతుంది — ఇది thread-local storage (Book 1 Chapter 9/10
`ThreadLocal` concept తో సంబంధం), ప్రతి log statement కి automatic గా
correlation ID attach చేస్తుంది, మీరు ప్రతిసారి manual గా పంపాల్సిన
అవసరం లేకుండా.

```java
// ఒక filter/interceptor request మొదట్లో ఇలా చేస్తుంది
MDC.put("correlationId", UUID.randomUUID().toString());
try {
    filterChain.doFilter(request, response); // ఈ request processing అంతటా, ప్రతి log line కి ఇది attach అవుతుంది
} finally {
    MDC.clear(); // ముఖ్యం! — Book 1 Chapter 10 "ThreadLocal ని clear చేయండి" సూత్రం (pooled threads లో leak అవ్వకుండా)
}
```

### ENGLISH INTERVIEW ANSWER

"I never use `System.out.println` in production code — SLF4J with
Logback gives me proper log levels, structured output, and asynchronous
appenders that don't block the request thread. I always use parameterized
logging — `log.info(\"Order {}\", orderId)` — instead of string
concatenation, because concatenation happens unconditionally regardless
of whether that log level is even enabled, while parameterized logging
only builds the string if the level is active, directly connecting to
Book 1's string-concatenation-cost material. For correlation IDs, I use
MDC to attach a request-scoped ID to every log line automatically without
threading it through every method signature manually — and critically, I
always clear the MDC in a `finally` block, because in a thread-pooled
environment (Book 1 Chapter 10), a `ThreadLocal`-backed value like MDC
that isn't cleared can leak into the next, unrelated request handled by
the same pooled thread — a real, subtle bug where request A's correlation
ID shows up in request B's logs."

---

## 7.4 CONCEPT: Production Configuration Checklist

### TELUGU EXPLANATION

Book 1 Chapter 11 (JVM tuning) మరియు ఈ book యొక్క అన్ని chapters ని
ఒకచోట కలిపి, ఒక **production readiness checklist** (ఇది Book 16
Production Debugging కి కూడా పునాది):

- ✅ **Graceful shutdown:** `server.shutdown=graceful` — in-flight
  requests complete అయ్యేవరకు wait చేసి, తర్వాతే shutdown అవ్వడం
  (Kubernetes rolling deployments కి ముఖ్యం, Book 12).
- ✅ **Connection pool sizing:** DB connection pool (HikariCP, Boot
  default) సరిగ్గా size చేయడం — Book 1 Chapter 10 thread pool sizing
  సూత్రాలే వర్తిస్తాయి.
- ✅ **Actuator secured, minimal exposure** (section 7.2).
- ✅ **GC logging + heap dump on OOM enabled** (Book 1 Chapter 11).
- ✅ **Structured logging with correlation IDs** (section 7.3).
- ✅ **Externalized, validated configuration** (Chapter 2).
- ✅ **Centralized exception handling, sanitized error responses**
  (Chapter 4).

### ENGLISH INTERVIEW ANSWER

"I think of production readiness as a checklist that pulls together
almost every chapter of this book: externalized and validated
configuration, centralized and sanitized exception handling, secured and
minimally-exposed Actuator endpoints, structured logging with correlation
IDs that are properly cleared per-request, and the JVM-level items from
Book 1 — GC logging, heap-dump-on-OOM, sensible connection pool sizing.
None of these individually are complicated, but missing even one of them
is a common, recurring root cause I've seen in real production incidents
— which is exactly why I treat this as a concrete checklist to verify
before any service goes live, not just general awareness."

---

## 7.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Adding logging | `System.out.println` or string-concatenated `log.info` | Parameterized SLF4J logging, correct log level |
| Actuator endpoints | Exposes everything (`include: "*"`) for convenience | Exposes only what's needed, secures the rest |
| Debugging a multi-service request | Greps logs per-service, manually correlates by timestamp | Uses a correlation ID present in every service's logs |
| Kubernetes pod restarts during deploy | Doesn't configure graceful shutdown, requests get dropped | Configures `server.shutdown=graceful` |

---

## 7.6 COMMON MISTAKES

1. Exposing all Actuator endpoints (`include: "*"`) in production without
   securing them.
2. Using string concatenation in log statements instead of parameterized logging.
3. Forgetting to clear MDC in a `finally` block, leaking correlation IDs
   across requests on pooled threads.
4. Logging at the wrong level — DEBUG-worthy details logged at INFO,
   flooding production logs, or genuinely important events logged at
   DEBUG and never seen.
5. Not configuring graceful shutdown, causing dropped requests during
   deployments/scaling events.

---

## 7.7 INTERVIEW QUESTION BANK — CHAPTER 7

**Basic:** 1. What does `/actuator/health` provide, and why does
Kubernetes care about it? 2. Why is parameterized logging preferred over
string concatenation?

**Intermediate:** 3. Why must Actuator endpoints be secured in
production? 4. What is MDC, and why must it be cleared after each request?

**Senior:** 5. Design a custom `HealthIndicator` for a service with two
critical dependencies (a database and a downstream payment API) — should
either dependency being down mark the whole service unhealthy? Why or why not?
6. Explain how correlation IDs connect Book 1's `ThreadLocal` material to
a real distributed-tracing use case.

**Architect:** 7. You're standardizing observability across 50
microservices. What logging format, correlation ID propagation strategy,
and Actuator exposure policy would you mandate organization-wide, and how
would you enforce it (a shared starter library, perhaps)?

**Scenario:** 8. A security scan flags that `/actuator/env` is publicly
accessible on a production service and returns database credentials in
plain text. What's the immediate fix, and what's the broader process fix
to prevent recurrence across other services?

**Trick:** 9. "As long as your logs contain the information you need,
the log level you use them at doesn't matter." True or false?

<details><summary>Key answers</summary>

- Q5: Depends on the dependency's role — if the database is required for
  virtually every request, its failure should mark the service `DOWN`
  (readiness probe should fail, stopping traffic routing to this
  instance). If the payment API is only needed for a subset of operations
  and the service can still serve other requests meaningfully without it,
  marking the whole service `DOWN` for a single non-critical dependency
  failure would cause unnecessary, overly broad service disruption —
  a more nuanced approach might report it as a degraded but still "up"
  status, or use a separate, non-blocking health group.
- Q6: MDC is implemented using `ThreadLocal` storage under the hood — the
  same mechanism from Book 1 Chapter 9/10, with the same "must be cleared"
  discipline in pooled-thread environments. The distributed tracing
  connection: within one service, MDC carries the correlation ID
  per-thread through all log statements for a request; across services,
  that same correlation ID is propagated via an HTTP header, so external
  tools (Book 8's distributed tracing systems) can stitch together the
  full request path across every service it touched using that shared ID.
- Q7: A shared internal starter library providing pre-configured
  structured JSON logging (for log aggregation tooling), a standard
  correlation-ID-propagating filter/interceptor applied consistently, and
  a locked-down default Actuator security configuration that individual
  teams would have to deliberately override (rather than deliberately
  opt into) — making the secure, standardized behavior the path of least
  resistance for every team, rather than relying on every team
  remembering to configure it correctly independently.
- Q8: Immediate fix: restrict `/actuator/env` (and audit all other
  Actuator endpoints) behind authentication immediately, and rotate the
  exposed credentials since they must now be considered compromised.
  Broader process fix: add an automated security scan/policy check in the
  CI/CD pipeline (Book 14) that fails a build/deployment if Actuator
  endpoints are exposed without proper security configuration, so this
  class of misconfiguration is caught before reaching production across
  all services, not just this one.
- Q9: False — log level determines what's actually visible/searchable
  under normal production log verbosity settings, what triggers
  alerting (if alerts are tied to ERROR-level logs, for instance), and
  the volume/cost of log storage; content alone doesn't matter if it's
  logged at a level nobody is actually watching or that floods the system
  with noise, burying genuinely important signals.

</details>

---

## 7.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does parameterized logging avoid unnecessary string-building cost when a log level is disabled?
- **Coding Check:** Implement a servlet filter that generates a correlation ID, puts it in MDC, and clears it in a `finally` block — verify (in reasoning or by running it) that it doesn't leak across requests on a pooled thread.
- **Explanation Check:** Explain in English why exposing `/actuator/env` publicly is a genuine security vulnerability, not just a minor information leak.
- **Real-World Check:** Your team's Kubernetes pods are dropping in-flight requests during every deployment rollout. Diagnose and fix using this chapter's production checklist.
- **Senior Check:** When would you choose to log something at WARN instead of ERROR, given that both might represent "something unexpected happened"?
- **Master Check:** Design the complete observability strategy for a new microservice: what gets logged at each level (ERROR/WARN/INFO/DEBUG), what custom health indicators are needed, what metrics are tracked, and how correlation IDs flow from an incoming request through to any outgoing downstream calls.

<details><summary>Answers</summary>

- Real-World Check: Missing `server.shutdown=graceful` configuration —
  the fix ensures the application stops accepting new requests but lets
  in-flight ones complete before the process actually terminates, aligned
  with Kubernetes's pod termination grace period, so a rolling deployment
  doesn't abruptly cut off requests mid-processing.
- Senior Check: WARN for things that are unexpected but self-recovering
  or non-critical (a retry succeeded on the second attempt, a fallback
  path was used successfully) — signal worth noticing but not paging
  someone at 3am for; ERROR for things that represent an actual failure
  requiring attention or indicating a real problem (a request failed
  entirely, a critical dependency is unreachable with no successful fallback).
- Master Check: ERROR for unhandled/unexpected failures and critical
  dependency outages; WARN for degraded-but-recovered situations (retry
  succeeded, fallback used); INFO for key business events (order placed,
  payment processed) at a volume sustainable for production; DEBUG for
  detailed diagnostic information enabled only when actively
  investigating an issue (via `/actuator/loggers` at runtime, section
  7.1); custom health indicators for each critical downstream dependency,
  distinguishing "critical, marks service DOWN" from "degraded, service
  stays UP" per Q5's reasoning; metrics tracking request latency/count per
  endpoint and any critical business counters; correlation ID generated
  at the edge (first filter), stored in MDC for all local logging, and
  propagated as an HTTP header on every outgoing call to downstream
  services, continuing the trace across service boundaries.

</details>

---

## 7.9 CHEAT SHEET

| Concept | Rule |
|---|---|
| `/actuator/health` | Kubernetes liveness/readiness probes hit this directly |
| Actuator exposure | Only `health`, `info`, `metrics` (or as needed) — never `"*"` unsecured |
| Logging | SLF4J + parameterized `{}` placeholders — never string concatenation or `System.out` |
| MDC / correlation ID | `ThreadLocal`-backed — always clear in `finally` on pooled threads |
| Graceful shutdown | `server.shutdown=graceful` — essential for zero-dropped-requests deploys |
| Production checklist | Config validated + exceptions sanitized + Actuator secured + logging structured + JVM flags set (Book 1 Ch. 11) |

---

*(Continues to Chapter 8 — Testing Spring Boot Applications.)*
