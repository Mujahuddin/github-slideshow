# CHAPTER 2 — KAFKA ARCHITECTURE: TOPICS, PARTITIONS, OFFSETS

---

## 2.1 CONCEPT: Kafka as an Append-Only Log, Not a Traditional Queue

### TELUGU EXPLANATION

**అత్యంత ముఖ్యమైన mental model shift:** చాలామంది Kafka ని "queue"
లా ఊహించుకుంటారు (message consume అయ్యాక, delete అవుతుంది అని) —
ఇది **తప్పు**. Kafka, ఒక **append-only log** — messages, ఒక file
కి కొత్త lines add అయినట్టు, **ఎప్పటికీ order లో** add అవుతూ ఉంటాయి,
మరియు configured **retention period** (ఉదా: 7 రోజులు) వరకు,
**consume అయ్యినా కూడా, delete అవ్వవు**.

**దీని practical implications:**
- **అనేక consumers, ఒకే message ని, స్వతంత్రంగా, వేర్వేరు
  సమయాల్లో** చదవగలరు — ఒక consumer చదివేసినంత మాత్రాన, ఆ message
  మరో consumer కి "అదృశ్యం" కాదు (traditional queue లో లాగా).
- **Replay సాధ్యం:** ఒక bug fix తర్వాత, గత 3 రోజుల messages ని
  **మళ్ళీ** process చేయాలంటే, offset ని వెనక్కి reset చేస్తే సరిపోతుంది
  (Chapter 3) — data ఇంకా log లో ఉంది కాబట్టి.
- Kafka, **broker-side** గా "ఈ consumer చదివిందా లేదా" అని
  track చేయదు — ఆ బాధ్యత **consumer దే** (Chapter 3, offset commits).

### ENGLISH INTERVIEW ANSWER

"The single most important mental model correction for Kafka is that
it's an append-only log, not a traditional queue — messages aren't
deleted once consumed. They're retained for a configured retention
period regardless of whether any consumer has read them. This has real
consequences: multiple independent consumers can read the same message
at their own pace without one consumer's read affecting another's view,
and messages can be replayed — if a consumer had a bug for the last 3
days, fixing the bug and resetting its offset backward lets it
reprocess that entire window, since the data is still physically
present in the log. It also means Kafka itself doesn't track
per-consumer 'read' state the way a traditional queue tracks message
deletion — tracking position is the consumer's responsibility, which we
cover in Chapter 3's offset commits."

---

## 2.2 CONCEPT: Topics and Partitions — The Unit of Parallelism and Ordering

### TELUGU EXPLANATION

**Topic:** ఒక logical category (ఉదా: "order-events") — ఇది ఒకటి లేదా
అంతకంటే ఎక్కువ **partitions** గా విభజించబడి ఉంటుంది.

**Partition — Kafka architecture యొక్క "atom":**
- ప్రతి partition, **తనలో తానే** ఒక ordered, append-only log.
- Kafka, **partition లోపల మాత్రమే** ordering ని guarantee
  చేస్తుంది — **topic మొత్తం మీద కాదు్.** అంటే, partition 0 లో
  messages A, B, C ఈ order లోనే వస్తాయి, కానీ partition 0 లోని
  message, partition 1 లోని message తో ఎలాంటి ordering relationship
  ఉండదు.
- **ఎందుకు partitions అవసరం:** Parallelism కోసం — ఒక topic కి 6
  partitions ఉంటే, ఒకేసారి **6 consumers** (ఒకే consumer group లో,
  Chapter 3) parallel గా చదవగలరు, throughput linear గా పెరుగుతుంది.

**Message key, partition ని ఎలా నిర్ణయిస్తుంది:** Producer, ఒక
message పంపేటప్పుడు ఒక **key** ఇస్తే (ఉదా: `orderId`), Kafka ఆ key
యొక్క hash ఆధారంగా **ఏ partition కి పంపాలో** నిర్ణయిస్తుంది —
**అదే key ఉన్న అన్ని messages, ఎప్పుడూ అదే partition కి** వెళ్తాయి.
ఇది **ordering guarantee** ఇవ్వడానికి కీలకం: ఉదాహరణకి, ఒకే
`orderId` కి సంబంధించిన అన్ని events (created, paid, shipped),
ఒకే partition లో, correct order లో ఉంటాయి.

**Key లేకపోతే:** Round-robin (లేదా sticky partitioning, newer Kafka
versions లో) గా distribute అవుతుంది — ఏ ordering guarantee ఉండదు.

### ENGLISH INTERVIEW ANSWER

"A topic is a logical category split into one or more partitions, and a
partition is Kafka's real unit of ordering and parallelism. Kafka only
guarantees ordering within a single partition — never across the whole
topic — so messages in partition 0 arrive in the order they were
written, but there's no ordering relationship between partition 0 and
partition 1 at all. Partitions exist for parallelism: a topic with 6
partitions can be read by up to 6 consumers in the same consumer group
simultaneously, scaling throughput close to linearly. The producer's
message key determines which partition a message lands in — Kafka
hashes the key to pick a partition, and critically, the same key always
maps to the same partition. This is how you get ordering guarantees
where they matter: keying every event by `orderId` ensures all events
for a given order — created, paid, shipped — land in the same
partition and are therefore strictly ordered relative to each other,
even though the topic as a whole has no global order. Without a key,
messages are spread round-robin with no ordering guarantee at all."

---

## 2.3 CONCEPT: Offsets — A Consumer's Position in the Log

### TELUGU EXPLANATION

**Offset:** ఒక partition లో, ప్రతి message కి ఒక **sequential,
increasing integer ID** ఉంటుంది (0, 1, 2, 3...) — ఇది ఆ message,
ఆ partition లో ఎక్కడ ఉందో సూచిస్తుంది.

**Consumer, తన "position" ని ఎలా track చేస్తుంది:** ప్రతి consumer
(లేదా consumer group, Chapter 3), ప్రతి partition కోసం, తను
**ఎక్కడిదాకా process చేసిందో** (ఏ offset వరకు) ఒక ప్రత్యేక internal
Kafka topic (`__consumer_offsets`) లో **commit** చేస్తుంది. Consumer
restart అయితే, ఆ committed offset నుండి **మళ్ళీ మొదలుపెడుతుంది**.

**ఇది ఎందుకు ముఖ్యం, Chapter 1 delivery semantics కి ఎలా
కలుస్తుంది:** Message process చేసి, **offset commit చేసేముందు**
consumer crash అయితే — restart అయ్యాక, అదే offset నుండి మళ్ళీ మొదలై,
**అదే message ని మళ్ళీ** process చేస్తుంది (at-least-once,
duplicate). Offset commit చేసి, **process చేసేముందు** crash అయితే
— ఆ message **కోల్పోతుంది** (at-most-once, message skip). సరైన
order: **process ముందు, commit తర్వాత** — ఇది at-least-once
ఇస్తుంది, duplicate risk తో, idempotency అవసరం (Chapter 1).

### ENGLISH INTERVIEW ANSWER

"Every message within a partition has a sequential offset — a simple
increasing integer marking its position. A consumer's progress is
tracked by committing offsets: after processing messages up to a
point, it commits that offset (to Kafka's internal
`__consumer_offsets` topic), so on restart it resumes from there rather
than from the beginning. This directly connects to Chapter 1's delivery
semantics: if a consumer commits the offset before actually finishing
processing, and then crashes, that message is effectively lost — it
will never be reprocessed, which is at-most-once behavior. If it
processes the message fully and commits the offset only afterward, a
crash between those two steps causes the same message to be reprocessed
on restart, which is at-least-once behavior with a duplicate. The
correct default is process-then-commit, accepting the duplicate-delivery
possibility and handling it with an idempotent consumer, rather than
commit-then-process, which risks silent message loss."

---

## 2.4 CONCEPT: Replication — Surviving Broker Failure

### TELUGU EXPLANATION

**Replication factor:** ప్రతి partition, ఒకటి కంటే ఎక్కువ brokers
మీద **copy** గా store అవుతుంది (ఉదా: replication factor 3 అంటే, 3
brokers మీద ఒకే partition data ఉంటుంది). ఒక్క broker, **leader**
గా పనిచేస్తుంది (అన్ని reads/writes ఆ broker ద్వారానే), మిగతావి
**followers** గా (leader data ని replicate చేసుకుంటూ ఉంటాయి).

**Leader fail అయితే:** ఒక follower, కొత్త leader గా **automatically
elect** అవుతుంది (ZooKeeper/KRaft consensus ద్వారా) — producer/consumer
లకు ఇది transparent గా జరుగుతుంది.

**`acks` config తో సంబంధం (Chapter 4 లో లోతుగా):** Producer, leader
నుండి write acknowledgment కోసం ఎంతమంది replicas confirm చేయాలో
నిర్ణయించగలదు — ఇది durability vs latency trade-off.

### ENGLISH INTERVIEW ANSWER

"Each partition is replicated across multiple brokers for fault
tolerance — one broker acts as the leader handling all reads and
writes for that partition, while the others are followers replicating
its data. If the leader broker fails, one of the in-sync followers is
automatically elected as the new leader, and this failover is
transparent to producers and consumers — they just start talking to the
new leader. This replication mechanism is also what the producer's
`acks` setting, which we cover in depth in Chapter 4, actually controls:
how many replicas must confirm a write before the producer considers it
successful, trading off durability against latency."

---

## 2.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Mental model of Kafka | Thinks of it as a queue where messages disappear after being read | Understands it as a retained, append-only log enabling replay and independent consumers |
| Needing ordering | Assumes the whole topic is ordered | Keys messages so related events land in the same partition, ordered only there |
| Choosing partition count | Picks an arbitrary number | Sizes partitions for expected parallel consumer count and future scaling headroom |
| Consumer crash handling | Doesn't think about commit timing | Deliberately commits after processing, accepting and handling at-least-once duplicates |

---

## 2.6 COMMON MISTAKES

1. Assuming Kafka guarantees global ordering across an entire topic
   instead of only within a partition.
2. Not keying messages that need relative ordering (e.g., all events for
   one order), causing them to scatter across partitions unordered.
3. Committing a consumer's offset before finishing processing, risking
   silent message loss on a crash.
4. Under-provisioning partitions, capping the maximum useful number of
   parallel consumers in a group.
5. Over-provisioning partitions far beyond any realistic consumer count,
   adding broker overhead without a throughput benefit.

---

## 2.7 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. What is the difference between a topic and a partition?
2. What is an offset?

**Intermediate:** 3. How does a message key affect which partition it's
routed to, and why does that matter for ordering? 4. What happens when a
partition's leader broker fails?

**Senior:** 5. You need all events for a given `userId` to be processed
in order, but the topic has 12 partitions. How do you achieve this?

**Architect:** 6. How would you decide the number of partitions for a
new high-throughput topic, considering both current and future scale?

**Scenario:** 7. A consumer commits its offset immediately after
receiving a batch of messages, before processing them, to "avoid
duplicate processing." What's wrong with this, and what will happen if
it crashes mid-batch?

**Trick:** 8. "Increasing a topic's partition count is always safe and
reversible." True or false?

<details><summary>Key answers</summary>

- Q5: Use `userId` as the message key — Kafka's hashing ensures every
  message with the same key lands in the same partition, so all of a
  given user's events are strictly ordered relative to each other,
  regardless of how many total partitions the topic has (the 12
  partitions still provide parallelism across *different* users).
- Q6: Consider the maximum realistic number of parallel consumers you'd
  ever want in one consumer group (partitions cap useful parallelism),
  expected message throughput and per-partition throughput limits, and
  some headroom for future growth — but not excessive headroom, since
  more partitions means more open file handles and replication overhead
  per broker, and (per Q8) partition count can't be safely decreased later.
- Q7: This is committing before processing, which favors at-most-once
  over at-least-once — if the consumer crashes partway through the
  batch, the committed offset already reflects messages that were never
  actually processed, so on restart the consumer resumes past them,
  silently losing whatever wasn't finished. The stated goal ("avoid
  duplicate processing") should instead be solved with an idempotent
  consumer, not by risking message loss.
- Q8: False, and this is a genuinely dangerous trap — increasing
  partitions is possible but changes the key-to-partition hash mapping
  for existing keys going forward, which can break ordering guarantees
  for a key whose messages now split across old and new partition
  assignments; decreasing partition count isn't supported at all
  (a topic must be deleted and recreated). Partition count should be
  planned deliberately up front, not treated as a casually adjustable knob.

</details>

---

## 2.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does Kafka only guarantee ordering within a partition, not across a whole topic?
- **Coding Check:** N/A for this conceptual chapter — instead, sketch the partition key you'd choose for an `order-events` topic and a `page-view-events` topic, and justify each choice.
- **Explanation Check:** Explain to a teammate why Kafka is better described as a "distributed commit log" than a "queue."
- **Real-World Check:** Your team wants to replay the last 2 days of events for a topic to backfill a new consumer. How is this possible with Kafka but would not be with a traditional queue?
- **Senior Check:** A topic has 3 partitions and your consumer group has 5 consumer instances. What happens to the extra 2 instances?
- **Master Check:** Design the topic and partitioning strategy for a multi-tenant SaaS platform's audit-log system, where each tenant's events must be strictly ordered, but different tenants can be processed independently and in parallel.

<details><summary>Answers</summary>

- Real-World Check: Because Kafka retains messages for a configured
  retention period regardless of whether they were already consumed,
  the last 2 days of data are still physically present in the log; a
  new consumer can simply set its starting offset back to the
  appropriate point in time and read through that history. A
  traditional queue that deletes messages on consumption would have
  nothing left to replay.
- Senior Check: Since a partition can only be actively consumed by one
  consumer instance within a given consumer group at a time, only 3 of
  the 5 instances will be assigned partitions and do any work — the
  remaining 2 sit idle as standby capacity, ready to pick up a partition
  immediately if one of the active 3 fails (covered further in
  Chapter 3's rebalancing).
- Master Check: Key every audit event by `tenantId`, ensuring all of a
  given tenant's events land in the same partition and are strictly
  ordered relative to each other. Choose a partition count well above
  the expected number of tenants needing simultaneous processing
  capacity (to allow parallelism across tenants), while accepting that
  multiple tenants will share a partition via hash collisions — which is
  fine, since ordering is only required within a tenant, not across
  tenants sharing a partition.

</details>

---

## 2.9 CHEAT SHEET

| Concept | Rule |
|---|---|
| Kafka's structure | Append-only, retained log — not a delete-on-read queue |
| Ordering guarantee | Within a partition only, never across a whole topic |
| Partition count | Caps maximum useful parallel consumers in a group |
| Message key | Same key → same partition, always — the mechanism for relative ordering |
| Offset | A consumer's tracked position within a partition |
| Commit timing | Process, then commit — accept at-least-once duplicates, handle with idempotency |
| Replication | Leader handles reads/writes; followers replicate; automatic failover on leader loss |
| Partition count changes | Increasing is possible but risks reordering; decreasing requires recreating the topic |

---

*(Continues to Chapter 3 — Kafka Producers & Consumers, Consumer Groups.)*
