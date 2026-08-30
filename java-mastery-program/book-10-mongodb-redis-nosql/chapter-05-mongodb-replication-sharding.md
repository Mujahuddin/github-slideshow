# CHAPTER 5 — MONGODB REPLICATION & SHARDING

---

## 5.1 CONCEPT: Replica Sets — Availability Through Redundant Copies

### TELUGU EXPLANATION

**Replica Set:** ఒకే data యొక్క **అనేక copies**, వేర్వేరు nodes
మీద maintain చేయడం — Chapter 2 (Kafka) లో చూసిన **replication factor**
concept కి, database level లో direct సారూప్యం. ఒక replica set లో:

- **Primary node:** అన్ని **writes**, ఇక్కడికే వెళ్తాయి.
- **Secondary nodes:** Primary నుండి data ని **replicate** చేసుకుంటూ
  ఉంటాయి (oplog ద్వారా — Book 6 Chapter 5 లో చూసిన **write-ahead log**
  concept కి సారూప్యం).

**Automatic Failover:** Primary fail అయితే, remaining secondaries,
ఒక **election** జరిపి, ఒకదాన్ని కొత్త primary గా promote చేస్తాయి —
ఇది Book 8 Chapter 8 (High Availability) లో చూసిన **"no single point
of failure"** principle యొక్క database-level అమలు.

**Read Preference:** Application, reads ని primary నుండి మాత్రమే
కాకుండా, **secondaries నుండి కూడా** చదవగలదు (`readPreference: secondaryPreferred`)
— ఇది read scalability పెంచుతుంది, కానీ **replication lag** వలన,
secondary నుండి చదివిన data, **కొద్దిగా stale** గా ఉండొచ్చు (Chapter
1 యొక్క CAP theorem, consistency vs availability trade-off ఇక్కడ
concrete గా కనిపిస్తుంది).

### ENGLISH INTERVIEW ANSWER

"A replica set maintains multiple copies of the same data across
different nodes — the database-level equivalent of the replication
factor concept from Kafka in Book 9. One node is the primary, handling
all writes; secondaries continuously replicate from it via the oplog,
conceptually similar to the write-ahead log from Book 6 Chapter 5. If
the primary fails, the remaining secondaries hold an election and
promote one of themselves to primary automatically — this is the
database-level implementation of the 'no single point of failure'
principle from Book 8 Chapter 8's high availability chapter. Reads can
be directed to secondaries instead of just the primary, which improves
read scalability, but introduces the concrete, practical version of
Chapter 1's CAP theorem trade-off: a secondary can lag slightly behind
the primary, so a read from a secondary can return slightly stale data
in exchange for that scalability and reduced load on the primary."

---

## 5.2 CONCEPT: Write and Read Concern — Tuning the Consistency/Availability Dial

### TELUGU EXPLANATION

**Write Concern:** ఒక write, "successful" గా considered అవ్వడానికి,
ఎంతమంది replica set members acknowledge చేయాలో నిర్ణయించడం —

- **`w: 1`:** Primary మాత్రమే acknowledge చేస్తే సరిపోతుంది (fast,
  కానీ primary fail అయితే, replicate అవ్వకముందే, ఆ write పోవచ్చు).
- **`w: majority`:** Replica set లో **majority** members acknowledge
  చేయాలి — durability పెరుగుతుంది, latency కొద్దిగా పెరుగుతుంది
  (Chapter 4 యొక్క Kafka `acks=all` కి direct సారూప్యం!).

**Read Concern:** ఒక read, ఎంత "confirmed" గా ఉన్న data ని చూడాలో
నిర్ణయించడం — `"local"` (fastest, unconfirmed data కూడా చూడొచ్చు),
`"majority"` (majority-acknowledged data మాత్రమే, roll-back
కాకుండా ఉండేలా guarantee).

**Senior-level connection:** ఇది సరిగ్గా Book 9 Chapter 4 యొక్క
Kafka `acks`/`min.insync.replicas` discussion కి **అదే mental
model, వేరే product** — ప్రతి distributed, replicated system,
ఇదే fundamental durability-vs-latency dial ని ఏదో ఒక రూపంలో
బహిర్గతం చేస్తుంది.

### ENGLISH INTERVIEW ANSWER

"Write concern controls how many replica set members must acknowledge a
write before it's considered successful — `w: 1` only requires the
primary, which is fast but risks losing an unreplicated write if the
primary fails immediately after; `w: majority` requires acknowledgment
from a majority of members, trading some latency for real durability.
This is conceptually identical to Kafka's `acks=all` and
`min.insync.replicas` from Book 9 Chapter 4 — the same
durability-versus-latency dial, just named differently in a different
product. Read concern is the read-side equivalent, controlling how
confirmed the data you're reading needs to be — `local` is fastest but
can include data that could theoretically be rolled back in an edge
case, while `majority` guarantees you're only reading data that's been
acknowledged by a majority and therefore won't be rolled back. Once you
recognize this pattern once, in any single distributed system, it
transfers directly to reasoning about every other replicated system you
encounter."

---

## 5.3 CONCEPT: Sharding — Horizontal Scaling Through Data Partitioning

### TELUGU EXPLANATION

**Sharding:** Data ని, **అనేక shards (వేర్వేరు machines/replica
sets)** మధ్య horizontally distribute చేయడం — ఒక్క machine కి
అమిత మైన data/traffic పెరిగినప్పుడు, vertical scaling (bigger
machine) కి బదులుగా, horizontal గా scale చేయడానికి. ఇది Book 9
Chapter 2 లో చూసిన **Kafka partitions** concept కి **నేరుగా
సారూప్యం** — ఒకే logical collection, physical గా అనేక shards
గా విభజించబడి ఉంటుంది.

**Shard Key — అత్యంత critical నిర్ణయం:** ఒక field (లేదా fields
కలయిక) ఎంచుకోవడం, ప్రతి document ని ఏ shard కి route చేయాలో
నిర్ణయించడానికి — Kafka యొక్క **message key** (partition routing
కోసం) కి direct సారూప్యం. తప్పు shard key ఎంచుకుంటే:

- **Hot shard problem:** ఒక్క shard కి ఎక్కువ traffic వెళ్తే
  (ఉదా: shard key గా `createdAt` వాడి, అన్ని ఇటీవలి writes ఒకే
  shard కి వెళ్తే) — మిగతా shards idle గా ఉంటాయి, ఒక్క shard
  overload అవుతుంది (Book 9 Chapter 2 యొక్క "unkeyed messages" hot
  partition సమస్యకి సారూప్యం).
- **సరైన shard key:** High cardinality (అనేక distinct values),
  query patterns లో frequently వాడేది (ఉదా: `customerId` ఒక
  multi-tenant system లో), write load ని evenly distribute చేసేది.

**Senior-level caution:** Shard key, **ఒకసారి ఎంచుకున్నాక,
మార్చడం చాలా కష్టం** (major operational effort అవసరం) — ఇది Book 9
Chapter 2 యొక్క "partition count మార్చడం కష్టం" caution కి
సారూప్యం, **ఇంకా ఎక్కువ consequential** గా.

### ENGLISH INTERVIEW ANSWER

"Sharding horizontally distributes data across multiple shards — each
potentially its own replica set — when a single machine's capacity for
data or traffic is exceeded, choosing horizontal scale over vertical.
This is directly analogous to Kafka partitions from Book 9 Chapter 2: a
single logical collection is physically split across multiple shards.
The shard key is the critical decision — the field or fields determining
which shard a document routes to, playing the same role as a Kafka
message key. A poorly chosen shard key creates a hot shard problem: if,
say, `createdAt` is used as the shard key, all recent writes concentrate
onto whichever shard currently owns the newest time range, leaving other
shards idle while that one is overloaded — the same imbalance problem as
unkeyed Kafka messages clustering unevenly. A good shard key has high
cardinality, is frequently used in query patterns — like `customerId` in
a multi-tenant system — and distributes write load evenly. I'd stress in
an interview that changing a shard key after the fact is a genuinely
major operational undertaking, an even more consequential version of
Book 9's caution about changing Kafka partition counts — so getting the
shard key right up front deserves serious analysis before committing to it."

---

## 5.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Understanding replica sets | Thinks of them only as a backup mechanism | Understands them as the mechanism for both durability and automatic failover |
| Reading from secondaries | Doesn't consider replication lag | Weighs read scalability against acceptable staleness explicitly |
| Choosing write concern | Uses the default without considering the trade-off | Chooses `w: majority` for critical writes, matching the durability need |
| Choosing a shard key | Picks a convenient or recently-used field | Analyzes cardinality and query patterns to avoid a hot shard |

---

## 5.5 COMMON MISTAKES

1. Treating a replica set purely as backup, without understanding its
   role in automatic failover and read scaling.
2. Reading from secondaries for data that genuinely needs strong
   consistency, without accounting for replication lag.
3. Using `w: 1` for writes where durability genuinely matters.
4. Choosing a low-cardinality or monotonically-increasing shard key
   (like a timestamp), creating a hot shard.
5. Treating the shard key decision as easily reversible later.

---

## 5.6 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. What is a replica set, and what are primary/secondary
roles? 2. What is sharding, and why would you use it?

**Intermediate:** 3. What is replication lag, and what practical
problem can it cause? 4. Why is a monotonically increasing field
(like a timestamp) usually a bad shard key?

**Senior:** 5. Design the write/read concern configuration for a
banking application's account balance updates versus its "recent
activity feed" reads.

**Architect:** 6. A team's single-shard MongoDB deployment is
approaching capacity limits. Walk through your process for evaluating
and choosing a shard key before migrating to a sharded cluster.

**Scenario:** 7. After sharding by `createdAt`, the team notices one
shard consistently has 10x the CPU load of the others. Diagnose and propose a fix.

**Trick:** 8. "Adding more secondaries to a replica set always
improves write throughput." True or false?

<details><summary>Key answers</summary>

- Q5: Account balance updates: `w: majority` write concern and
  `readConcern: majority` for any read that influences a financial
  decision — durability and consistency matter far more than latency
  here, and losing or misreading a balance update is unacceptable.
  Recent activity feed reads: `readPreference: secondaryPreferred` with
  a `local` read concern — slight staleness is an acceptable trade-off
  for reduced load on the primary and better read scalability, since an
  activity feed being a few hundred milliseconds behind has no real
  business consequence.
- Q6: Analyze current and projected query patterns to identify fields
  frequently used in filters (candidates for shard key inclusion);
  measure the cardinality of candidate fields to ensure even
  distribution potential; simulate or estimate write distribution across
  hypothetical shard values to catch hot-shard risks before committing;
  and weigh a compound shard key (e.g., combining a low-cardinality
  business field with a high-cardinality field) if a single field
  doesn't satisfy both query-pattern relevance and even distribution.
- Q7: This is the classic hot shard problem from a monotonically
  increasing shard key — since `createdAt` values always increase, all
  new documents route to whichever shard currently owns the newest
  range, concentrating all write load onto one shard while others sit
  comparatively idle. Fix: choose a different or compound shard key with
  better distribution properties (e.g., hashed sharding on `_id`, or a
  compound key combining a business identifier with the timestamp) —
  though changing a shard key on an already-sharded, populated
  collection is a major migration effort, underscoring why this should
  be caught before initial sharding rather than after.
- Q8: False — write throughput is fundamentally bounded by the primary,
  since all writes go through it; adding secondaries improves
  read scalability (if reads are distributed to them) and durability/
  failover resilience, but doesn't increase how many writes the primary
  itself can accept and process. Increasing write throughput requires
  sharding (distributing write load across multiple primaries), not
  adding more secondaries to one replica set.

</details>

---

## 5.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain how a replica set's automatic failover works, and connect it to Book 8's "no single point of failure" principle.
- **Coding Check:** N/A for this conceptual chapter — instead, write the write concern and read concern settings you'd choose for (a) an inventory-reservation write, (b) a "trending products" read.
- **Explanation Check:** Explain to a teammate why write concern and read concern are the same underlying consistency/durability dial as Kafka's acks and min.insync.replicas, just in a different product.
- **Real-World Check:** Your team is designing a multi-tenant SaaS platform and needs to choose a shard key for the primary `events` collection. What would you check before deciding, and what would disqualify a candidate field?
- **Senior Check:** Why does sharding solve a fundamentally different problem than adding replica set secondaries, even though both involve "adding more machines"?
- **Master Check:** Design the full replication and sharding strategy for a global e-commerce platform needing both high write throughput for order events and low-latency reads for product catalog browsing across multiple regions.

<details><summary>Answers</summary>

- Real-World Check: Check the cardinality of `tenantId` (likely high,
  good), whether queries are predominantly scoped to a single tenant
  (favoring `tenantId` inclusion in the shard key for query routing
  efficiency), and whether any single tenant's write volume could
  dominate and create a hot shard on its own (in which case a compound
  key adding another dimension, like a hashed field, might be needed). A
  field like `eventType` would likely be disqualified for low
  cardinality and uneven distribution (some event types vastly more
  common than others).
- Senior Check: Adding secondaries to a replica set adds *redundant
  copies* of the exact same data, which helps durability, failover, and
  read scaling, but every write still funnels through the single
  primary. Sharding *partitions* the data itself across multiple
  independent primaries, so write capacity actually scales with the
  number of shards — genuinely different problems (durability/read-scale
  vs. write-scale) that happen to share the surface-level description
  of "adding more machines."
- Master Check: Order events: shard by a high-cardinality,
  well-distributed key like a hashed `orderId` or a compound
  `(region, orderId)` key for write scalability, with `w: majority`
  write concern for durability given the financial nature of orders.
  Product catalog: since it's read-heavy and changes far less
  frequently, a replica set per region (or a sharded cluster with
  region-aware shard key if catalog size demands it) with
  `readPreference: nearest` or `secondaryPreferred` to serve reads from
  geographically close secondaries, accepting minor staleness for much
  lower read latency across regions.

</details>

---

## 5.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Replica set | Multiple copies for durability + automatic failover, not just backup |
| Primary/Secondary | Writes go to primary; secondaries replicate via oplog |
| Read from secondaries | Improves read scalability; introduces replication-lag staleness |
| Write concern (`w`) | Durability dial — `majority` for critical writes, same idea as Kafka's acks |
| Read concern | Consistency dial for reads — `majority` avoids reading rollback-able data |
| Sharding | Horizontal write/data scaling — analogous to Kafka partitions |
| Shard key | Critical, hard-to-change decision — needs high cardinality, even distribution |
| Hot shard | Caused by low-cardinality or monotonically increasing shard keys |

---

*(Continues to Chapter 6 — Redis Data Structures & Use Cases.)*
