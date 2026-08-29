# CHAPTER 6 — API GATEWAY PATTERNS & RATE LIMITING

---

## 6.1 CONCEPT: What an API Gateway Actually Does (and Doesn't Do)

### TELUGU EXPలanaTION

**API Gateway** అనేది client requests మరియు backend services (Book 8
Microservices) మధ్య ఉండే ఒక **single entry point**. దీని **నిజమైన
responsibilities:**

- **Routing:** `/api/orders/**` → Order Service, `/api/customers/**` →
  Customer Service — client కి internal service topology **తెలియకుండా**.
- **Authentication termination:** JWT validation ఒక్కసారి, gateway
  వద్దే చేసి, downstream services కి "ఇది authenticated request" అని
  తెలియజేయడం (ప్రతి service తనే auth logic repeat చేయాల్సిన అవసరం లేదు).
- **Rate limiting** (section 6.2) — **అన్ని services కి centralized గా**.
- **Request/response transformation** — ఉదా: legacy backend response ని
  కొత్త client-expected format కి మార్చడం.
- **Aggregation (BFF pattern):** ఒక mobile app కి, ఒకే screen కోసం
  3 వేర్వేరు services నుండి డేటా కావాలంటే, gateway (లేదా ఒక dedicated
  **Backend For Frontend**) వాటిని **ఒకే response** గా combine చేయవచ్చు
  — client multiple round trips చేయాల్సిన అవసరం లేకుండా.

**ఏం చేయకూడదు (సాధారణ architectural mistake):** **Business logic**
ని gateway లో పెట్టకూడదు — ఇది gateway ని ఒక **"god object"** గా
మార్చేస్తుంది, ప్రతి business rule మార్పుకి gateway deploy చేయాల్సి
వస్తుంది, ఇది Book 1 Chapter 2 SRP సూత్రాన్ని ఉల్లంఘిస్తుంది.

### ENGLISH INTERVIEW ANSWER

"An API Gateway is the single entry point for client traffic into a
microservices architecture, handling routing so clients don't need to
know internal service topology, authentication termination so JWT
validation happens once instead of being duplicated in every service,
centralized rate limiting, and sometimes response aggregation via a
Backend-For-Frontend pattern, combining multiple backend calls into one
client-facing response. What I actively push back on is putting business
logic in the gateway — that turns it into an unmaintainable god object
that every business rule change requires redeploying, which violates
single responsibility at an architectural level. The gateway's job is
cross-cutting infrastructure concerns, not domain logic."

---

## 6.2 CONCEPT: Rate Limiting at the Gateway — Distributed, Not Per-Instance

### TELUGU EXPLANATION

Book 2 Chapter 16 లో మనం Token Bucket rate limiter ని **ఒక్క JVM instance
లోపల** (in-memory, `synchronized`) implement చేశాం. **Gateway level
లో, ఇది సరిపోదు** — production లో **అనేక gateway instances** ఉంటాయి
(load balancer వెనుక), ప్రతి దానికి తన own in-memory counter ఉంటే,
**real rate limit ని enforce చేయలేరు** (ఉదా: "100 req/min per user"
అనుకుంటే, 5 gateway instances ఉంటే, actual limit **500 req/min**
అయిపోతుంది, ప్రతి instance తన సొంత counter track చేస్తే).

**పరిష్కారం: Distributed rate limiting (Redis-backed):**
```
INCR user:123:requests
EXPIRE user:123:requests 60  (మొదటిసారి మాత్రమే)
```
Redis **atomic** `INCR` command వాడి, **అన్ని gateway instances**
ఒకే, **shared counter** ని update చేస్తాయి — ఇది Book 1 Chapter 9-10
లో మనం నేర్చుకున్న "shared mutable state కి synchronization అవసరం"
సూత్రం, **distributed system స్థాయి** కి extend అయ్యింది (Redis
ఇక్కడ centralized "lock" లాంటి పాత్ర పోషిస్తుంది, atomic operations ద్వారా).

**Senior-level trade-off:** Redis **ఒక additional network hop**, మరియు
**potential single point of failure** (Redis down అయితే?). Production
systems సాధారణంగా **fail-open** (Redis unreachable అయితే, rate limiting
skip చేసి request ని అనుమతించడం) లేదా **fail-closed** (reject చేయడం)
policy ని **deliberately** ఎంచుకుంటాయి — business context బట్టి (a
payment API fail-closed కావొచ్చు, ఒక low-stakes read API fail-open
కావొచ్చు).

### ENGLISH INTERVIEW ANSWER

"A single-JVM rate limiter, like the token bucket we built in Book 2, is
correct within one instance but breaks down completely at the gateway
level, where multiple gateway instances sit behind a load balancer —
each with its own independent in-memory counter would let the actual
aggregate rate limit balloon to instance-count times the intended limit.
The fix is a shared, centralized counter, typically Redis-backed, using
atomic operations like `INCR` so all instances coordinate against one
source of truth — this is the same 'shared mutable state needs
synchronization' principle from Book 1's concurrency chapters, just
extended from threads-in-one-JVM to instances-in-a-distributed-system,
with Redis's atomicity playing the role a lock would play locally. The
real trade-off I'd flag is what happens if Redis itself is unreachable —
fail-open (allow requests through, prioritizing availability) or
fail-closed (reject requests, prioritizing the rate limit guarantee) is a
deliberate business decision, not a default to leave unexamined; a
payment API might reasonably choose fail-closed, while a low-stakes
read-only API might choose fail-open."

---

## 6.3 CONCEPT: Gateway as a Potential Single Point of Failure

### TELUGU EXPLANATION

Gateway **అన్ని traffic** కి single entry point అయినందున, ఇది ఒక
**critical, high-availability infrastructure** గా design చేయాలి —
కేవలం "ఒక extra service" కాదు. Practical considerations:

- **Multiple gateway instances**, load balanced (గేట్‌వేకే ఒక load
  balancer అవసరం).
- Gateway యొక్క own **downstream call timeouts, circuit breakers**
  (Book 8 లో వివరంగా) — ఒక slow/failing backend service, gateway ని
  మొత్తం block చేయకూడదు.
- Gateway **stateless** గా ఉండాలి (Book 4 Chapter 6/Chapter 1 సూత్రం)
  — ఏ instance ని అయినా, ఏ request నైనా handle చేయగలగాలి.

### ENGLISH INTERVIEW ANSWER

"Because the gateway sees 100% of external traffic, it has to be treated
as critical, highly-available infrastructure, not just another service —
a gateway outage takes down access to everything behind it. In practice
this means running multiple gateway instances behind their own load
balancer, keeping the gateway itself stateless so any instance can serve
any request, and implementing timeouts and circuit breakers for calls to
downstream services specifically so one slow or failing backend service
can't cascade into blocking the entire gateway's request-handling
capacity — a preview of Book 8's resilience patterns, applied at exactly
the point in the architecture where a failure has the widest possible blast radius."

---

## 6.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Where to put shared auth/rate-limiting logic | Duplicates it in every microservice | Centralizes it at the API Gateway |
| Business logic placement | Tempted to add a quick business rule at the gateway for convenience | Keeps the gateway to cross-cutting infrastructure concerns only |
| Rate limiting across multiple gateway instances | Uses per-instance in-memory counters | Uses a shared, atomic, Redis-backed counter |
| Redis unavailability during rate limiting | Doesn't consider it, service just breaks unpredictably | Deliberately chooses fail-open or fail-closed based on the specific API's risk profile |

---

## 6.5 COMMON MISTAKES

1. Putting business logic in the API Gateway.
2. Using per-instance in-memory rate limiting behind a load balancer,
   silently multiplying the effective rate limit.
3. Not planning for gateway high availability, treating it as a single,
   ordinary service.
4. Not deciding (and testing) fail-open vs fail-closed behavior when the
   rate-limiting backend (Redis) is unreachable.
5. Letting a single slow downstream service block the gateway's capacity
   to serve unrelated requests, with no timeout/circuit breaker in place.

---

## 6.6 INTERVIEW QUESTION BANK — CHAPTER 6

**Basic:** 1. Name three responsibilities of an API Gateway. 2. Why
shouldn't business logic live in the gateway?

**Intermediate:** 3. Why does per-instance in-memory rate limiting break
down at the gateway level? 4. What does Redis's `INCR` provide that a
plain read-then-write counter wouldn't?

**Senior:** 5. Design the fail-open vs fail-closed decision for three
different APIs: a public content-read API, a payment-processing API, and
an internal admin API. 6. Explain the Backend-For-Frontend (BFF) pattern
and when it's worth the added complexity of a dedicated aggregation layer.

**Architect:** 7. You're designing gateway infrastructure for a platform
with wildly different traffic patterns across services (some
latency-sensitive, some bulk/batch). How would you architect rate
limiting and routing to avoid one service's traffic pattern degrading another's?

**Scenario:** 8. After scaling the API Gateway from 2 to 10 instances to
handle load, a team notices the effective rate limit per user
increased 5x. Diagnose and fix.

**Trick:** 9. "Centralizing authentication at the API Gateway means
individual microservices no longer need any security logic at all." True or false?

<details><summary>Key answers</summary>

- Q5: Public content-read API: fail-open — availability matters more than
  strict rate enforcement for low-stakes reads. Payment-processing API:
  fail-closed — allowing unlimited, unrated requests through during a
  Redis outage risks real financial/fraud exposure, worse than temporary
  reduced availability. Internal admin API: likely fail-open, given lower
  request volume/risk and trusted internal users, though this is a
  judgment call based on the specific admin actions exposed.
- Q6: BFF creates a dedicated aggregation/adaptation layer per
  client type (mobile BFF, web BFF), each shaped around that specific
  client's needs — worth it when different clients need meaningfully
  different data shapes/aggregations from the same backend services, and
  a single generic gateway response would otherwise become a bloated,
  compromise-shaped payload serving no client well; not worth it for
  simple systems with one primary client type.
- Q7: Consider separate gateway routing/scaling tiers or dedicated rate
  limit buckets per service/traffic class, so a burst of bulk/batch
  traffic to one service doesn't consume shared gateway capacity or rate
  limit budget that latency-sensitive services need — this might mean
  physically separate gateway deployments for different traffic classes,
  not just configuration within one shared gateway.
- Q8: Classic per-instance in-memory rate limiting — going from 2 to 10
  instances, each with its own independent counter, multiplied the
  effective aggregate limit by 5x exactly as the numbers suggest. Fix:
  migrate to a shared, Redis-backed atomic counter (section 6.2) so the
  rate limit is enforced consistently regardless of instance count.
- Q9: False — gateway-level authentication (verifying identity) doesn't
  eliminate the need for authorization logic within services,
  particularly data-level/object-level authorization (Book 4 Chapter 6's
  "broken object-level authorization" material) — a service still must
  verify that the authenticated caller is allowed to access the *specific*
  resource being requested, which the gateway generally cannot know without
  business-specific context it shouldn't hold (per section 6.1's "no
  business logic in the gateway" principle).

</details>

---

## 6.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does Redis's atomic `INCR` matter for distributed rate limiting, connecting to Book 1's race-condition material?
- **Coding Check:** Implement a distributed rate limiter using Redis `INCR`/`EXPIRE` (or a Lua script for full atomicity) callable from a Spring Boot gateway filter.
- **Explanation Check:** Explain in English why "we centralized auth at the gateway" doesn't mean individual services can skip all authorization logic.
- **Real-World Check:** Your team's gateway currently does JWT validation, rate limiting, AND applies a discount-calculation business rule for a specific client. A new engineer asks if this is good architecture. Answer and justify.
- **Senior Check:** When would you choose per-instance (non-distributed) rate limiting deliberately, accepting its limitations?
- **Master Check:** Design the complete gateway architecture for a platform with a public API (needs strict per-API-key rate limiting), an internal service mesh (Book 8 preview — service-to-service calls that shouldn't go through the same external gateway), and a partner API (needs contract-tested, versioned endpoints from Chapters 2 and 5). Would these all share one gateway or need separate infrastructure?

<details><summary>Answers</summary>

- Real-World Check: The discount-calculation business rule should NOT be
  in the gateway — that's domain logic belonging in a service (likely a
  Pricing or Order service), not cross-cutting infrastructure; explain
  that keeping it in the gateway couples an unrelated infrastructure
  component to business rule changes, makes the rule harder to test in
  isolation, and violates the separation this chapter establishes between
  gateway responsibilities and business logic.
- Senior Check: For a single-instance service, or a service with sticky
  session routing where the same client always hits the same instance
  (rare, and generally discouraged for statelessness reasons) — outside
  those narrow cases, distributed rate limiting is almost always correct
  once more than one instance exists.
- Master Check: Likely separate infrastructure for each: the public API
  gateway handles external, per-API-key rate limiting and strict
  contract/version enforcement; internal service-to-service traffic
  typically bypasses the external gateway entirely, using a service mesh
  (Book 8) for internal routing/resilience instead, since external-gateway
  concerns like public rate limiting and API-key auth don't apply
  internally; the partner API might share the public gateway's
  infrastructure but with its own dedicated routing rules, versioning,
  and contract tests (Chapters 2 and 5) layered on top — the underlying
  principle being that different traffic classes with different trust
  levels and requirements generally warrant distinctly configured (if not
  fully separate) infrastructure paths.

</details>

---

## 6.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Gateway responsibilities | Routing, auth termination, rate limiting, transformation, aggregation (BFF) |
| Gateway anti-pattern | Never put business logic in the gateway |
| Rate limiting at scale | Must be distributed (Redis-backed atomic counter), never per-instance in-memory |
| Redis unavailability | Deliberately choose fail-open vs fail-closed per API's risk profile |
| Gateway availability | Must be highly available itself — multiple instances, stateless, circuit breakers on downstream calls |
| Gateway auth ≠ full authorization | Services still need data-level authorization checks (Book 4 Ch. 6) |

---

*(Continues to Chapter 7 — API Evolution: Backward Compatibility & Idempotency.)*
