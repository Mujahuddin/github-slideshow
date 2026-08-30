# CHAPTER 7 — REDIS CACHING PATTERNS & PITFALLS

---

## 7.1 CONCEPT: Cache-Aside — The Default Pattern, and What It Actually Requires

### TELUGU EXPLANATION

**Cache-Aside (Lazy Loading) — అత్యంత సాధారణ pattern:**

1. Application, cache లో data ఉందా అని చెక్ చేస్తుంది (`GET`).
2. **Cache hit:** Data ని నేరుగా return చేస్తుంది.
3. **Cache miss:** Database నుండి data ని fetch చేసి, cache లో
   store చేసి (`SET`, TTL తో), తర్వాత return చేస్తుంది.

**ఇది simple గా కనిపిస్తుంది, కానీ correct గా implement చేయడానికి,
జాగ్రత్తగా ఆలోచించాల్సిన decisions ఉన్నాయి:**
- **TTL ఎంత పెట్టాలి:** చాలా తక్కువ TTL → cache benefit తక్కువ
  (frequent misses); చాలా ఎక్కువ TTL → stale data ఎక్కువసేపు
  ఉండిపోతుంది.
- **Cache invalidation, write జరిగినప్పుడు:** Database update
  అయినప్పుడు, ఆ కేష్ entry ని **invalidate** (delete) చేయాలి, లేకపోతే
  **stale data** ఇంకా serve అవుతూనే ఉంటుంది (TTL expire అయ్యేదాకా) —
  ఇది Book 7 Chapter 6 లో చూసిన **second-level cache multi-instance
  staleness problem** కి **direct సారూప్యం**, ఇక్కడ application-level
  cache కోసం.

### ENGLISH INTERVIEW ANSWER

"Cache-aside, also called lazy loading, is the default caching pattern:
check the cache first, return on a hit, and on a miss, fetch from the
database, populate the cache with a TTL, then return. It sounds simple,
but getting it right requires real decisions. TTL length is a genuine
trade-off — too short and you get frequent misses, undermining the
caching benefit; too long and stale data persists longer than it
should. And critically, when the underlying data is updated, that
cache entry needs to be explicitly invalidated — otherwise stale data
keeps being served until the TTL naturally expires. This is directly
the same staleness problem we saw with JPA's second-level cache in Book
7 Chapter 6, just at the application-cache layer instead of the
ORM layer — multiple instances or code paths that don't consistently
invalidate on write will serve inconsistent, stale data to different
requests."

---

## 7.2 CONCEPT: Write-Through and Write-Behind — Alternatives with Different Trade-Offs

### TELUGU EXPLANATION

**Write-Through:** Application, data ని **database మరియు cache
రెండింటికీ, ఒకేసారి (synchronous గా)** రాస్తుంది — cache, ఎప్పుడూ
database తో **in sync** గా ఉంటుంది (cache-aside యొక్క invalidation
సమస్య ఇక్కడ ఉండదు), కానీ ప్రతి write, **రెండు operations** చేయాలి
కాబట్టి, write latency కొద్దిగా పెరుగుతుంది.

**Write-Behind (Write-Back):** Application, data ని **cache కి
మాత్రమే వెంటనే** రాసి, database write ని **asynchronous గా,
తర్వాత** చేస్తుంది — fastest writes, కానీ cache crash అయితే
(database write జరగకముందే), **data loss** risk (Chapter 1 యొక్క
at-most-once delivery కి సారూప్యమైన risk).

**Senior-level decision framework:** Read-heavy, write-infrequent
workloads కి **cache-aside** సరిపోతుంది (most common default).
Consistency ముఖ్యమైన, write-heavy workloads కి **write-through**
సరిపోతుంది. Write latency అత్యంత critical, కొంత data loss risk
acceptable అయిన workloads కి మాత్రమే **write-behind** పరిగణించాలి.

### ENGLISH INTERVIEW ANSWER

"Write-through writes to both the database and cache synchronously on
every write, keeping the cache always in sync and sidestepping
cache-aside's invalidation problem entirely, at the cost of slightly
higher write latency since every write does two operations. Write-behind
writes only to the cache immediately, deferring the database write to
happen asynchronously — this gives the fastest possible write latency,
but introduces a real data loss risk if the cache fails before that
deferred database write completes, similar in spirit to the
at-most-once delivery risk from Book 9 Chapter 1. My decision framework:
cache-aside is the right default for read-heavy, infrequently-written
workloads, which covers most caching use cases. Write-through fits
write-heavy workloads where consistency between cache and database
matters more than the small added write latency. Write-behind should
only be considered when write latency is genuinely the critical
constraint and the business can tolerate some risk of losing very
recent writes on a cache failure — a much narrower set of legitimate use cases."

---

## 7.3 CONCEPT: Cache Stampede and Cache Penetration — The Two Failure Modes

### TELUGU EXPLANATION

**Cache Stampede (Thundering Herd):** ఒక **popular** cache entry
expire అయిన క్షణంలో, **అనేక concurrent requests** ఒకేసారి cache
miss అయి, **అన్నీ ఒకేసారి database కి** hit చేస్తాయి — database
మీద sudden, massive load spike (Book 8 Chapter 4 యొక్క **retry
storm** కి సారూప్యమైన dynamics, వేరే root cause తో).

**పరిష్కారం:**
- **Lock-based:** మొదటి request మాత్రమే database ని query చేసి,
  cache ని repopulate చేస్తుంది; మిగతా requests, ఆ మొదటి request
  పూర్తయ్యేదాకా **wait** చేస్తాయి (లేదా stale data ని momentarily
  serve చేస్తాయి).
- **Probabilistic early expiration:** TTL expire అవ్వడానికి కొంచెం
  ముందే, **కొన్ని requests మాత్రమే** (randomly) refresh ట్రిగ్గర్
  చేస్తాయి, మిగతావి ఇంకా cached value వాడతాయి — spike ని smooth
  చేస్తుంది.

**Cache Penetration:** Requests, **cache లో ఎప్పుడూ ఉండని** keys
కోసం వస్తే (ఉదా: invalid/non-existent IDs, లేదా ఒక malicious actor
ఉద్దేశపూర్వకంగా random IDs తో attack చేస్తే) — ప్రతి request,
cache miss అయ్యి, **నేరుగా database కి** వెళ్తుంది, cache యొక్క
protective benefit **పూర్తిగా bypass** అవుతుంది.

**పరిష్కారం:** "Null values" ని కూడా (చిన్న TTL తో) cache చేయడం —
"ఈ ID లేదు" అనే fact నే cache చేసి, repeated invalid requests ని
database కి చేరకుండా ఆపడం. Bloom filters (advanced), అసలు
database కి వెళ్ళే ముందే, "ఈ key ఖచ్చితంగా లేదు" అని fast గా
తెలుసుకోవడానికి కూడా వాడతారు.

### ENGLISH INTERVIEW ANSWER

"Cache stampede, or thundering herd, happens when a popular cache
entry expires and many concurrent requests all miss at the same
instant, all hitting the database simultaneously — a sudden load spike
with dynamics similar to Book 8 Chapter 4's retry storm, just from a
different root cause. The fix is either a lock-based approach, where
only the first request actually queries the database and repopulates
the cache while others wait or briefly get stale data, or probabilistic
early expiration, where only a small random subset of requests near the
TTL's expiration trigger a refresh, smoothing out what would otherwise
be a spike. Cache penetration is a different failure mode: requests for
keys that will never exist in the cache — invalid or non-existent IDs,
or a deliberate attack using random IDs — always miss and go straight
to the database, completely bypassing the cache's protective benefit.
The standard fix is caching the 'this doesn't exist' result itself,
with a short TTL, so repeated requests for the same invalid ID get
served from cache instead of hitting the database every time; a Bloom
filter is a more advanced technique for quickly ruling out
definitely-non-existent keys before even touching the database."

---

## 7.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Implementing caching | Adds cache-aside without an invalidation strategy | Explicitly designs cache invalidation alongside every write path |
| Choosing a caching pattern | Always uses cache-aside by default | Weighs cache-aside, write-through, and write-behind against consistency/latency needs |
| A popular cache entry expiring | Doesn't anticipate a load spike | Designs for cache stampede with locking or probabilistic early expiration |
| Requests for non-existent data | Lets every miss hit the database | Caches negative results to prevent cache penetration |

---

## 7.5 COMMON MISTAKES

1. Implementing cache-aside without a corresponding cache invalidation
   step on writes, serving stale data indefinitely until TTL expiry.
2. Choosing write-behind for data where losing recent writes on a
   crash would be unacceptable.
3. Not anticipating cache stampede for high-traffic cache entries with
   a shared expiration time.
4. Letting invalid/non-existent-key requests repeatedly hit the
   database instead of caching negative results.
5. Setting a uniform TTL for all cache entries regardless of how
   frequently the underlying data actually changes.

---

## 7.6 INTERVIEW QUESTION BANK — CHAPTER 7

**Basic:** 1. What is cache-aside, and what are its three steps?
2. What is cache stampede?

**Intermediate:** 3. Why does write-through avoid the invalidation
problem cache-aside has? 4. What is cache penetration, and how is it
different from cache stampede?

**Senior:** 5. Design a caching strategy for a product-detail page that
gets 50,000 requests/minute for the most popular products, with prices
that change a few times a day.

**Architect:** 6. Your team's cache invalidation logic has become
scattered across a dozen different write paths, and stale-data bugs
keep appearing. Propose an architectural fix.

**Scenario:** 7. A cached "top news article" entry expires at midnight,
and every server experiences a CPU/database spike at exactly 12:00 AM
daily. Diagnose and fix.

**Trick:** 8. "As long as I set a TTL on every cache entry, I never need
explicit cache invalidation on write." True or false?

<details><summary>Key answers</summary>

- Q5: Cache-aside with a TTL matched to how often prices actually
  change (e.g., a few hours, refreshed proactively when a price update
  occurs) combined with explicit cache invalidation in the price-update
  code path (so a price change is reflected immediately, not just
  eventually via TTL expiry). Given 50,000 requests/minute concentrated
  on popular products, add probabilistic early expiration or a
  lock-based refresh to prevent a stampede when a popular product's
  cache entry does expire.
- Q6: Centralize cache invalidation behind a single, well-tested
  component or event-driven mechanism — e.g., every write path publishes
  a "data changed" event (or calls one shared invalidation service/method)
  rather than each write path independently remembering to invalidate
  the cache; this converts a scattered, easy-to-forget responsibility
  into one enforced code path, reducing the chance any future write
  path is added without invalidation.
- Q7: Classic cache stampede — many concurrent requests around midnight
  all experience the same cache miss simultaneously and all hit the
  database at once. Fix: apply probabilistic early expiration (refresh
  the cache entry slightly before the exact expiration time, staggered
  randomly) or a lock-based refresh (only one request actually queries
  the database on miss, others wait briefly or get the stale value)
  instead of a hard, simultaneous expiration for all clients.
- Q8: False — a TTL only bounds *how long* staleness can persist in the
  worst case, but it does nothing to make a cache reflect a write
  *immediately*. Without explicit invalidation on write, users can see
  outdated data for the entire TTL window after every update, which is
  often unacceptable; TTL and explicit invalidation solve different
  problems and are typically used together, not as substitutes for each other.

</details>

---

## 7.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain why cache-aside without invalidation reproduces Book 7 Chapter 6's second-level cache staleness problem.
- **Coding Check:** N/A for this conceptual chapter — instead, sketch the pseudocode for a cache-aside `getProduct(id)` function, including where invalidation would need to hook in on an `updateProduct` call.
- **Explanation Check:** Explain to a teammate the difference between cache stampede and cache penetration, and why each needs a different fix.
- **Real-World Check:** Your team's API is receiving repeated requests for a `userId` that was deleted last week, and each request still hits the database. Diagnose and propose a fix.
- **Senior Check:** Why is probabilistic early expiration often preferred over a strict locking approach for preventing cache stampede at very high request volumes?
- **Master Check:** Design the full caching architecture for a news website's homepage, which shows a personalized "top 10 for you" list (unique per user) alongside a global "trending now" list (shared across all users), considering invalidation, stampede, and penetration for each.

<details><summary>Answers</summary>

- Real-World Check: This is cache penetration — the deleted user's ID
  will never be found in the cache, so every request bypasses it and
  hits the database directly. Fix: cache a "not found" marker for that
  ID with a short TTL when a lookup returns nothing, so subsequent
  requests for the same deleted ID are served (as a fast "not found"
  response) from the cache instead of repeatedly querying the database.
- Senior Check: A locking approach means every concurrent request
  except the first has to wait (or poll) for the lock to release, which
  under very high request volume can itself create a bottleneck or
  latency spike as a large number of requests queue up behind the single
  refreshing request. Probabilistic early expiration avoids this
  entirely by spreading refreshes out randomly before the hard
  expiration, so no single moment ever sees a synchronized mass of
  requests missing at once, and no request has to explicitly wait on another.
- Master Check: "Trending now" (shared, moderate update frequency,
  extremely high read volume) → cache-aside with probabilistic early
  expiration to prevent stampede, since it's a single shared hot key hit
  by effectively all traffic; explicit invalidation when the trending
  computation updates. "Top 10 for you" (per-user, unique cache key per
  user) → cache-aside with a moderate TTL; stampede risk is far lower
  per-key since traffic for any single user's specific key is much
  lower than the shared trending key, but cache penetration is still a
  risk for invalid/deleted user IDs, warranting negative-result caching
  there too.

</details>

---

## 7.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Cache-aside | Read from cache, populate on miss — must pair with explicit invalidation on write |
| Write-through | Synchronous dual write — always consistent, slightly higher write latency |
| Write-behind | Async database write — fastest, but real data-loss risk on cache failure |
| Cache stampede | Many concurrent misses on one expired popular key — fix with locking or probabilistic early expiration |
| Cache penetration | Repeated misses for non-existent keys — fix by caching negative results |
| TTL alone | Bounds worst-case staleness; does NOT replace explicit invalidation on write |
| Default choice | Cache-aside for most read-heavy workloads; write-through when consistency matters more |

---

*(Continues to Chapter 8 — Choosing the Right Data Store: Polyglot Persistence.)*
