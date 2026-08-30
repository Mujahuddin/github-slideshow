# CHAPTER 6 — REDIS DATA STRUCTURES & USE CASES

---

## 6.1 CONCEPT: Redis Is Not "Just a Cache" — It's a Data Structure Server

### TELUGU EXPLANATION

**ఇది ఈ chapter యొక్క మొదటి mindset correction:** చాలామంది Redis
ని "ఒక simple key-value cache" గా మాత్రమే ఆలోచిస్తారు — ఇది Redis
యొక్క నిజమైన శక్తి ని underestimate చేస్తుంది. Redis, **rich data
structures** ని native గా support చేస్తుంది (Strings మాత్రమే కాదు) —
ప్రతి structure, ఒక **నిర్దిష్ట use case** కోసం optimize చేయబడింది,
ఆ structure ని సరైన సందర్భంలో వాడితేనే, Redis యొక్క నిజమైన value
వస్తుంది.

**ఐదు ముఖ్యమైన data structures, ఐదు వేర్వేరు use cases:**

1. **String:** సరళమైన `key → value` (cache entries, counters
   `INCR`/`DECR`).
2. **Hash:** ఒక key లోపల, multiple field-value pairs (ఒక object ని
   represent చేయడానికి, ఉదా: `user:123` hash లో `name`, `email`
   fields) — full object ని serialize/deserialize చేయకుండా, individual
   fields ని update చేయగలగడం advantage.
3. **List:** Ordered sequence (ఉదా: ఒక simple job queue,
   `LPUSH`/`RPOP` తో).
4. **Set:** Unique, unordered elements (ఉదా: "ఈ user ఏ tags follow
   చేస్తున్నాడు" — duplicate-free, fast membership check `SISMEMBER`).
5. **Sorted Set (ZSet):** Unique elements, ఒక **score** తో ordered
   (ఉదా: leaderboards — `ZADD`, `ZRANGE`, `ZRANK` ద్వారా, O(log N)
   లో ranking operations).

### ENGLISH INTERVIEW ANSWER

"A common underestimation is treating Redis as just a simple key-value
cache — that misses most of its actual value as a data structure
server. Redis natively supports several rich structures, each optimized
for a specific access pattern. Strings are the simplest key-to-value
mapping, useful for cache entries and atomic counters via `INCR`/`DECR`.
Hashes store multiple field-value pairs under one key, letting you
represent an object and update individual fields without
serializing/deserializing the whole thing. Lists are ordered sequences,
useful for simple job queues via `LPUSH`/`RPOP`. Sets store unique,
unordered elements with fast membership checks — good for something
like 'which tags does this user follow.' And Sorted Sets combine
uniqueness with a score-based ordering, which is exactly what powers
leaderboards — `ZADD` to insert with a score, `ZRANGE` and `ZRANK` to
query rank and range in logarithmic time. Choosing the right structure
for the actual access pattern, rather than defaulting to a plain string
holding a serialized blob, is where Redis's real performance and
expressiveness comes from."

---

## 6.2 CONCEPT: Real Use Cases Mapped to Structures

### TELUGU EXPLANATION

**Rate Limiting (Book 5 Chapter 6 గుర్తుచేసుకోండి):** Redis
String తో `INCR` + `EXPIRE` కలిపి — ప్రతి request కి, ఒక key
(ఉదా: `rate_limit:user123:minute`) increment చేసి, ఒక TTL పెట్టి,
threshold దాటితే reject చేయడం. **Atomic operation** అవ్వడం ముఖ్యం
— Book 1 Chapter 9 (Concurrency) లో చూసిన race condition సమస్యలు,
ఇక్కడ Redis యొక్క single-threaded, atomic command execution వలన
avoid అవుతాయి.

**Session Store:** Hash లేదా String (JSON serialized), TTL తో —
Book 4/5 లో చూసిన stateless service design తో కలిపి, session data
ని central గా, fast గా store చేయడానికి.

**Leaderboard:** Sorted Set — `ZADD leaderboard 1500 "player1"`,
`ZREVRANGE leaderboard 0 9 WITHSCORES` (top 10), `ZRANK leaderboard
"player1"` (ఒక specific player rank) — ఇవన్నీ O(log N), massive
scale కి కూడా fast గా ఉంటాయి.

**Distributed Lock (Book 1 Chapter 9/10 concurrency concepts, distributed
systems కి extend చేయడం):** `SET key value NX EX 30` (key లేకపోతేనే
set చేయడం, atomic గా, TTL తో) — ఇది ఒక simple distributed lock
implement చేయడానికి పునాది, అనేక service instances లో, ఒక్కటే
instance ఒక task run చేయాలంటే.

**Idempotency Key Storage (Book 5 Chapter 7, Book 9 Chapter 1
గుర్తుచేసుకోండి):** ఒక processed request/message ID ని, ఒక TTL తో
Redis Set/String లో store చేయడం — "ఇది ఇంతకుముందు process
అయ్యిందా" అని fast గా check చేయడానికి.

### ENGLISH INTERVIEW ANSWER

"A few real use cases map directly onto specific structures. Rate
limiting, which we saw in Book 5 Chapter 6, uses a String with `INCR`
plus `EXPIRE` — incrementing a per-user, per-time-window key and
rejecting once it exceeds a threshold; this works correctly under
concurrent requests specifically because Redis executes commands
atomically on a single thread, avoiding the race conditions from Book 1
Chapter 9's concurrency chapter that a naive read-then-write
implementation would hit. A session store typically uses a Hash or a
JSON-serialized String with a TTL, giving stateless services (Book 4/5)
a fast, centralized place to look up session data. A leaderboard is the
canonical Sorted Set use case — `ZADD` to record a score, `ZREVRANGE`
for a top-N list, `ZRANK` for one player's specific rank, all in
logarithmic time even at large scale. A simple distributed lock can be
built from `SET key value NX EX 30` — set the key only if it doesn't
already exist, atomically, with a TTL as a safety net — ensuring only
one service instance runs a given task at a time. And idempotency key
storage, connecting back to Book 5 Chapter 7 and Book 9 Chapter 1,
stores a processed request or message ID with a TTL, giving a fast way
to check 'have I already handled this' before doing potentially
duplicate work."

---

## 6.3 CONCEPT: Persistence and Durability — Redis Is Not Purely Volatile

### TELUGU EXPLANATION

**Common misconception:** "Redis, memory లో మాత్రమే ఉంటుంది,
restart అయితే data పోతుంది" — ఇది **పాక్షికంగా మాత్రమే నిజం**,
configuration మీద ఆధారపడి:

- **RDB (snapshotting):** Periodic గా, మొత్తం dataset ని disk కి
  snapshot గా save చేయడం — fast restart, కానీ చివరి snapshot తర్వాత
  జరిగిన writes కోల్పోవచ్చు.
- **AOF (Append-Only File):** ప్రతి write operation ని ఒక log
  file కి append చేయడం (Book 6 Chapter 5 యొక్క write-ahead log కి
  సారూప్యం) — durability ఎక్కువ, కానీ file size, restart time ఎక్కువ.

**Senior-level framing:** Redis ని **purely ephemeral cache** గా
వాడాలా, **కొంత durability అవసరమైన primary store** గా వాడాలా అనేది,
ఈ persistence options ద్వారా tune చేయగల నిర్ణయం — కానీ, session
data/rate limit counters లాంటి **naturally-expiring, re-derivable**
data కి, persistence అంతగా ముఖ్యం కాదు; **గా bank balance లాంటి
critical, non-derivable data ని, Redis ని దానికి single source of
truth గా వాడటం, generally సరైనది కాదు** (Chapter 8 లో ఈ decision
framework లోతుగా చూస్తాం).

### ENGLISH INTERVIEW ANSWER

"A common misconception is that Redis is purely in-memory and loses
everything on restart — that's only partially true and depends on
configuration. RDB snapshotting periodically saves the whole dataset to
disk, giving fast restarts but potentially losing writes since the last
snapshot. AOF logs every write operation to an append-only file,
conceptually similar to the write-ahead log from Book 6 Chapter 5,
giving much stronger durability at the cost of larger file size and
slower restart replay. The senior-level framing I'd offer: whether Redis
needs strong persistence depends entirely on what it's storing. For
naturally-expiring, easily re-derivable data — rate limit counters,
cache entries, session data — losing it on a rare restart is a minor,
recoverable inconvenience, so persistence tuning matters less. For
something like a critical, non-derivable balance, Redis generally
shouldn't be the single source of truth at all, regardless of
persistence configuration — that's a decision we'll dig into more
fully in Chapter 8's data-store decision framework."

---

## 6.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Modeling data in Redis | Serializes everything into a JSON string | Chooses the structure (Hash, Set, Sorted Set) matching the actual access pattern |
| Implementing rate limiting | Reads a counter, checks it, then increments (non-atomic) | Uses `INCR` + `EXPIRE` for atomic, race-condition-free counting |
| Assuming Redis durability | Assumes all data is always safely persisted | Understands RDB vs AOF trade-offs and matches persistence to data criticality |
| Storing critical financial data | Considers using Redis as the source of truth for convenience | Uses Redis only for cache/derived data, keeping the durable source of truth elsewhere |

---

## 6.5 COMMON MISTAKES

1. Storing everything as a serialized JSON string instead of using the
   structure that actually fits the access pattern.
2. Implementing rate limiting or counters with a non-atomic
   read-check-increment sequence, introducing race conditions.
3. Assuming Redis data is always durable, or always volatile, without
   checking actual persistence configuration.
4. Using Redis as the single source of truth for critical,
   non-derivable data.
5. Ignoring TTLs entirely, letting temporary data (sessions, rate-limit
   counters) accumulate indefinitely.

---

## 6.6 INTERVIEW QUESTION BANK — CHAPTER 6

**Basic:** 1. Name three Redis data structures and one use case for
each. 2. What does `INCR` do, and why is it atomic?

**Intermediate:** 3. Why is `SET key value NX EX 30` useful for
implementing a distributed lock? 4. What's the difference between RDB
and AOF persistence?

**Senior:** 5. Design a rate limiter using Redis that allows 100
requests per user per minute, avoiding race conditions under concurrent requests.

**Architect:** 6. A team wants to use Redis as the primary datastore for
a shopping cart feature, with no other backing database. What would you
weigh before approving this?

**Scenario:** 7. A leaderboard implemented with a Redis List (not a
Sorted Set) is slow to compute "top 10 players" as the list grows, and
players' ranks are hard to query directly. Diagnose and fix.

**Trick:** 8. "Since Redis is single-threaded, race conditions are
impossible in any Redis-based application logic." True or false?

<details><summary>Key answers</summary>

- Q5: `INCR rate_limit:{userId}:{currentMinute}`, and if the returned
  value is 1 (the key was just created), also call `EXPIRE` on it with
  a 60-second TTL; check the returned increment value against the
  threshold of 100, rejecting the request if exceeded. Using `INCR`
  alone (rather than `GET` then `SET`) keeps the increment atomic, so
  concurrent requests from the same user can't race past each other and
  both see a stale pre-increment value.
- Q6: Weigh: is cart data acceptable to lose on a rare Redis
  restart/failure (with no other backing store, yes it would be lost);
  does the business tolerate a customer's cart disappearing
  occasionally; is there a plan for Redis high availability (replication,
  persistence tuning) if this becomes the source of truth. Generally
  I'd recommend Redis as a fast cache/session layer for the cart with a
  backing database as the actual source of truth (write-through or
  periodic sync), rather than Redis alone being the only copy of
  potentially revenue-affecting cart data.
- Q7: A Redis List has no native ranking or sorting operation — finding
  "top 10" requires reading and sorting the entire list in application
  code, which gets slower as it grows, and there's no efficient way to
  query one player's rank without scanning. Fix: switch to a Sorted Set
  keyed by player with score as the ranking metric — `ZREVRANGE` gives
  top 10 in O(log N + 10), and `ZRANK` gives any player's rank directly
  in O(log N), regardless of total leaderboard size.
- Q8: False — single-threaded *command execution* means any single
  Redis command is atomic, but a sequence of multiple separate commands
  (e.g., `GET` then `SET` in application code, with logic in between) is
  not atomic as a whole — another client's command can execute between
  them. Race conditions are still very possible at the multi-command,
  application-logic level; avoiding them requires using Redis's atomic
  single-command operations (`INCR`, `SET ... NX`) or transactions/Lua
  scripts for multi-step atomicity.

</details>

---

## 6.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain why choosing the right Redis data structure matters, rather than storing everything as a serialized string.
- **Coding Check:** N/A for this conceptual chapter — instead, sketch (in pseudocode) the Redis commands for a "top 5 most-viewed products this hour" feature.
- **Explanation Check:** Explain to a teammate why `INCR` is safe under concurrency but a `GET`-then-`SET` counter implementation isn't.
- **Real-World Check:** Your team needs a simple job queue for background image processing, expected to process a few hundred jobs per minute. Would Redis Lists be a reasonable choice, or would you recommend Kafka/RabbitMQ from Book 9? Justify.
- **Senior Check:** Why is a Redis-based distributed lock using `SET NX EX` still not perfectly safe in every edge case, and what would you tell a team relying on it for a critical operation?
- **Master Check:** Design the Redis data structure strategy for a real-time multiplayer game needing: a per-match leaderboard, a "currently online players" set, per-player session data, and a matchmaking queue.

<details><summary>Answers</summary>

- Real-World Check: For a few hundred jobs per minute with simple
  processing needs, Redis Lists are a reasonable, lightweight choice —
  `LPUSH`/`BRPOP` gives a simple, fast queue without the operational
  overhead of running Kafka or RabbitMQ. I'd recommend Book 9's brokers
  instead once the requirements grow to need delivery guarantees beyond
  at-most-once, dead-letter handling, complex routing, or much higher
  throughput — Redis Lists lack the reliability tooling (Chapter 6 of
  Book 9) that a production-critical queue usually needs.
- Senior Check: The basic `SET NX EX` lock has known edge cases — for
  example, if the process holding the lock stalls (e.g., a long GC
  pause) past the TTL, the lock can expire and be acquired by another
  process while the first is still running, leading to two processes
  believing they hold the lock simultaneously. For a truly critical
  operation, I'd tell the team to either use a more robust distributed
  locking algorithm (like Redlock, with its own accepted trade-offs) or
  design the operation itself to be safely idempotent/reentrant so a
  rare double-execution doesn't cause real harm.
- Master Check: Per-match leaderboard → Sorted Set, scored by
  points/kills. Currently-online players → Set, for fast membership
  checks and count. Per-player session data → Hash, keyed by player ID,
  with a TTL matching expected session length. Matchmaking queue →
  List (simple FIFO matchmaking) or Sorted Set (if matching needs to
  consider skill rating/wait time as a score) — the choice between List
  and Sorted Set here depends specifically on whether matchmaking needs
  simple ordering or score-based prioritization.

</details>

---

## 6.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Redis's real identity | A data structure server, not just a key-value cache |
| String | Simple values, atomic counters (`INCR`/`DECR`) |
| Hash | Object-like field-value groupings under one key |
| List | Ordered sequences, simple queues |
| Set | Unique elements, fast membership checks |
| Sorted Set | Unique elements ranked by score — leaderboards, O(log N) ranking |
| Atomicity | Single commands are atomic; multi-command sequences are not without transactions/Lua |
| RDB vs AOF | Snapshot (fast restart, some loss) vs append-log (durable, slower restart) |
| Golden rule | Never make Redis the sole source of truth for critical, non-derivable data |

---

*(Continues to Chapter 7 — Redis Caching Patterns & Pitfalls.)*
