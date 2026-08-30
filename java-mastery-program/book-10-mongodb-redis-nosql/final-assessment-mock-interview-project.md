# BOOK 10 — FINAL ASSESSMENT, NOSQL MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 8 CHAPTERS)

1. A team wants to migrate their entire relational schema to MongoDB
   "because NoSQL scales better," without a specific bottleneck
   identified. How do you respond? *(Ch. 1)*
2. Design the embedding/referencing strategy for a `BlogPost` with an
   `author` and a potentially large number of `comments`. *(Ch. 2)*
3. A dashboard's aggregation pipeline uses 3 chained `$lookup` stages
   and is slow. What would you investigate, and what alternative design
   would you consider? *(Ch. 3)*
4. Why does a compound index `{status: 1, createdAt: -1}` not help a
   query filtering only on `createdAt`? *(Ch. 4)*
5. Explain the difference between MongoDB's write concern and read
   concern, and connect this to a concept from Book 9. *(Ch. 5)*
6. A shard key choice of `createdAt` causes uneven load across shards.
   Diagnose and propose a better shard key. *(Ch. 5)*
7. Design a Redis-based rate limiter that's safe under concurrent
   requests, and explain why a naive `GET`-then-`SET` implementation
   would not be. *(Ch. 6)*
8. A cache-aside implementation has a TTL but no explicit invalidation
   on write. What problem does this cause, and which earlier chapter
   from Book 7 does it mirror? *(Ch. 7)*
9. Design the persistence architecture for a food delivery app: orders,
   a restaurant menu catalog with wildly different attributes per
   cuisine type, and real-time delivery-driver location tracking. *(Ch. 8)*
10. Trace a single "place order" flow through this book's stack: an
    order write to a relational system of record, a denormalized
    MongoDB read model kept in sync, and a Redis-cached order-summary —
    naming which chapter's concept governs each part and how
    consistency is maintained across all three.

<details>
<summary>Answer Key</summary>

1. I'd push back and ask what the specific bottleneck actually is —
   without an identified pain point (schema volatility, horizontal
   write scale needs, or a genuinely document-shaped access pattern),
   a wholesale migration trades away ACID guarantees and join
   expressiveness for an unverified, vague benefit — the same
   "solve a problem you don't have yet" mistake as premature
   microservices adoption in Book 8, applied to persistence.
2. Embed the author's minimal display info if desired for read
   convenience, but reference the full `Author` document by ID since
   author data is shared across many posts and updated independently.
   For comments, apply the Subset Pattern: embed the most recent N
   comments directly in the post for fast common-case reads, with the
   full comment history in a separate, referenced `comments` collection,
   since comment count is unbounded and risks the 16MB document limit
   if fully embedded.
3. Investigate whether `$match` stages are placed early and whether the
   fields they filter on are indexed; also investigate whether this
   read pattern would be better served by a precomputed, denormalized
   read-model collection updated via change streams or a background
   job — directly the same idea as CQRS from Book 8 Chapter 8 — turning
   the live 3-way join into a single fast read.
4. Because of the leftmost-prefix rule — the compound index's B-Tree is
   effectively organized by `status` first; a query that doesn't filter
   on `status` at all can't use the index's sorted structure to
   efficiently narrow down on `createdAt` alone.
5. Write concern controls how many replica set members must acknowledge
   a write before it's considered successful (durability); read concern
   controls how confirmed the data returned by a read must be
   (consistency). This is the exact same durability/consistency dial as
   Kafka's `acks` and `min.insync.replicas` from Book 9 Chapter 4 — the
   same underlying trade-off, expressed in a different product's terminology.
6. `createdAt` is monotonically increasing, so all new writes
   concentrate onto whichever shard currently owns the newest range,
   creating a hot shard while other shards sit comparatively idle. A
   better shard key has high cardinality and distributes writes evenly
   — e.g., a hashed `_id`, or a compound key combining a business
   dimension (like `customerId`) with time, avoiding the
   monotonically-increasing-alone problem.
7. Use `INCR` on a per-user, per-time-window key, checking the returned
   value against a threshold, with `EXPIRE` set when the key is first
   created. A naive `GET`-then-`SET` approach isn't atomic — under
   concurrent requests, two requests can both `GET` the same
   pre-increment value, both decide they're under the limit, and both
   `SET` an incremented value that overwrites rather than accumulates,
   undercounting actual request volume — the same race condition
   category as Book 1 Chapter 9's concurrency hazards.
8. Without explicit invalidation on write, an update to the underlying
   data doesn't remove or refresh the cached entry — stale data
   continues being served to every reader until the TTL naturally
   expires, no matter how quickly the underlying data actually changed.
   This directly mirrors Book 7 Chapter 6's second-level cache
   multi-instance staleness problem, just at the application-cache
   layer instead of the ORM layer.
9. Orders → relational database, for ACID transactional guarantees
   around payment and fulfillment. Restaurant menu catalog → MongoDB,
   matching Chapter 2's variable-attribute-per-category use case
   directly (a "biryani" item and a "pizza" item have very different
   natural attributes). Real-time driver location → Redis, given the
   very high write frequency, latest-value-only nature, and need for
   extremely fast reads (potentially using geospatial commands for
   "nearest driver" queries).
10. The order write commits to the relational system of record first
    (Ch. 1, Book 6's ACID guarantees) → an Outbox-pattern event (Book 8
    Ch. 6) is published in the same local transaction → a consumer
    updates the MongoDB read model asynchronously, applying the
    embedding/referencing choices from Chapter 2 for the order-summary
    shape → the Redis cache entry for that order summary is explicitly
    invalidated (Ch. 7) rather than left to expire via TTL alone, so
    the next read repopulates it from the now-updated MongoDB read
    model rather than serving a stale cached value — three stores, one
    system of record, kept in sync via event-driven propagation rather
    than direct synchronous dual writes.

</details>

---

## PART B — MOCK INTERVIEW: NOSQL ROUND

**Interviewer:** "Your team's PostgreSQL database is under increasing
write load from a new 'user activity feed' feature that logs every
click, view, and interaction — millions of events per day, with a
schema that keeps needing new columns as new event types are added.
Walk me through how you'd approach this."

**Model answer:** "This has two separate signals worth pulling apart.
The schema churn — constantly adding columns for new event types — is
exactly the kind of variable-shape problem Chapter 1 and 2 describe as
a poor fit for a fixed relational schema; a document model would let
each event type have its own natural shape without forcing sparse,
ever-growing nullable columns onto every event. The write volume is a
separate, additional signal favoring a NoSQL approach with a better
horizontal-scaling story than we currently have. I'd propose moving
this specific feature's data to MongoDB, likely applying the Bucket
Pattern from Chapter 2 to group events into time-window documents
rather than one document per event, both for write efficiency and to
naturally cap document growth. I wouldn't propose migrating anything
else in the system — this is a polyglot persistence decision (Chapter
8) driven by this feature's specific shape and volume, not a signal to
move the whole database off PostgreSQL."

**Follow-up:** "The team is worried about needing to query 'all activity
for a user across the last 30 days' for a compliance report. Does your
design support that?"
(Yes, with an index on `(userId, windowStart)` for the bucketed
collection, per Chapter 4's ESR-rule reasoning — an equality filter on
`userId` and a range filter on the time window is exactly the query
shape a well-designed compound index supports efficiently.)

---

**Interviewer:** "You're told a production incident caused a Redis
cache-aside implementation to serve prices that were two hours stale
after a bulk price update ran. Walk me through your root-cause
process and the fix."

**Model answer:** "First, I'd confirm the TTL setting — if it's set
to something like 4 hours, that alone would partially explain up to a
4-hour staleness window, but 'exactly 2 hours after a bulk update' is a
more specific signal pointing at missing cache invalidation rather than
just a long TTL. I'd check whether the bulk price-update code path
actually calls a cache-invalidation step for each affected product, or
whether it only writes to the database and relies on TTL expiry alone —
which is precisely the cache-aside pitfall from Chapter 7. If that's
confirmed, the fix is adding explicit cache invalidation (or a
cache-refresh) to the bulk update's code path for every affected key,
so a price change is reflected immediately rather than waiting out
the TTL. I'd also check whether this bulk update is a common enough
operation to justify centralizing invalidation logic behind a single
shared mechanism, per the architectural fix from Chapter 7's mastery
checkpoint, to prevent this class of bug from being reintroduced by a
future write path that also forgets to invalidate."

**Follow-up:** "What if invalidating 10,000 product cache entries in a
bulk update itself risks a cache stampede once they all expire or get
invalidated together?"
(A reasonable follow-up concern — staggering invalidation slightly, or
relying on probabilistic early expiration/lock-based refresh from
Chapter 7's stampede mitigation for the highest-traffic subset of those
products, rather than a synchronized mass invalidation-and-refetch.)

---

**Interviewer:** "Explain to me, as if I'm new to the team, why we use
three different databases in this system instead of just picking the
one that's 'best' and using it everywhere."

**Model answer:** "I'd start by pointing out that 'best' isn't a
context-free property of a database — it depends entirely on the shape
of the data and how it's accessed. Our orders and payments need ACID
transactions and clear relational structure, so PostgreSQL is the right
fit there — MongoDB or Redis would either lose transactional guarantees
we depend on or add complexity without benefit. Our product catalog has
wildly different attributes per category, which is awkward to model
with a fixed relational schema but very natural in MongoDB's flexible
document model. And our session data, rate limiting, and caching need
extremely fast, structure-specific operations that Redis is built for,
but none of that data needs to be durable or transactional the way
orders do. Using one database everywhere would mean accepting a poor
fit somewhere — either forcing flexible catalog data into rigid
relational columns, or risking financial data's correctness in a store
that doesn't guarantee it the way we need. Three databases sounds more
complex on the surface, but each one is actually solving the specific
problem it's good at, which is simpler in practice than forcing one
tool to do everything adequately."

---

## PART C — CAPSTONE PROJECT: "POLYGLOT E-COMMERCE PLATFORM"

**Goal:** A persistence architecture demonstrating every chapter of
Book 10 working together, building directly on Book 6's relational
foundation and Book 8's Outbox pattern.

**Requirements:**

1. Design and justify the NoSQL-vs-relational decision (Ch. 1) for each
   of: orders, product catalog (variable attributes per category), and
   session data.
2. Implement the product catalog in MongoDB with a deliberate
   embedding/referencing strategy for product-to-reviews, applying the
   Subset or Outlier Pattern for a simulated "viral product" with tens
   of thousands of reviews (Ch. 2).
3. Build an aggregation pipeline computing "revenue by category, last
   30 days" with `$match` placed early and `$sort` correctly ordered
   before `$limit`; then redesign it as a precomputed read model and
   compare query latency (Ch. 3).
4. Design and justify a compound index for the product catalog's most
   common query pattern (category equality + price range + rating
   sort), applying the ESR rule; verify with `explain()` that it
   produces an `IXSCAN`, not a `COLLSCAN` (Ch. 4).
5. Configure a MongoDB replica set (or document the design if a live
   cluster isn't available) with `w: majority` for order-related writes
   and `secondaryPreferred` reads for the catalog; design a shard key
   for the catalog collection and justify it against the hot-shard risk (Ch. 5).
6. Implement Redis-based rate limiting (Ch. 6, Book 5 Ch6), a session
   store, and a Sorted-Set-based "top sellers" leaderboard.
7. Implement cache-aside for product detail pages with explicit
   invalidation on price/stock updates, and add a mitigation for cache
   stampede on the top 10 best-selling products (Ch. 7).
8. Write a one-page architecture decision record (ADR) justifying the
   full polyglot persistence architecture, naming the system of record
   and the Outbox-pattern-based sync mechanism keeping MongoDB read
   models and Redis caches consistent with it (Ch. 8).

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Technology fit | Each store is justified by actual data shape/access pattern, not familiarity |
| Schema design | Embedding/referencing choices avoid the 16MB limit and unnecessary duplication |
| Query/index correctness | `explain()` confirms `IXSCAN` usage; aggregation stage ordering is correct |
| Replication/sharding reasoning | Write/read concern and shard key choices are justified against real trade-offs |
| Caching correctness | No stale reads survive a write; stampede mitigation is demonstrated under simulated load |
| Consistency architecture | The ADR clearly identifies one system of record with an event-driven sync mechanism, not direct dual writes |

---

*(This completes BOOK 10 — MONGODB + REDIS + NOSQL. Book 11 — DOCKER —
moves from data storage technologies to containerization, packaging
these polyglot services and their dependencies for consistent
deployment across environments.)*
