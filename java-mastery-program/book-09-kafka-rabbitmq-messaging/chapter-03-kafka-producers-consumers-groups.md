# CHAPTER 3 — KAFKA PRODUCERS & CONSUMERS, CONSUMER GROUPS

---

## 3.1 CONCEPT: The Producer's Job — Serialization, Partitioning, Batching

### TELUGU EXPLANATION

**Producer, ఒక message పంపేటప్పుడు, మూడు నిర్ణయాలు తీసుకుంటుంది:**

1. **Serialization:** Java object ని bytes గా మార్చడం (ఉదా:
   `JsonSerializer`, `Avro` — Avro, schema evolution కి బాగా
   suit అవుతుంది, పెద్ద systems లో schema registry తో కలిపి వాడతారు).
2. **Partitioning:** Chapter 2 లో చూసినట్టు, key hash ఆధారంగా ఏ
   partition కి పంపాలో నిర్ణయించడం.
3. **Batching:** Producer, individual messages ని వెంటనే పంపకుండా,
   ఒక చిన్న time window (`linger.ms`) లో **batch** గా collect చేసి,
   ఒకేసారి పంపుతుంది — ఇది throughput పెంచుతుంది (fewer network
   round-trips), కానీ latency కొద్దిగా పెరుగుతుంది. ఇది Book 6
   Chapter 7 లో చూసిన JDBC batch inserts కి direct సారూప్యత —
   **batching అనేది ఎప్పుడూ throughput vs latency trade-off**.

**Spring Kafka లో `KafkaTemplate`:** ఈ మూడు concerns ని abstract
చేస్తుంది — developer, `kafkaTemplate.send(topic, key, value)`
call చేస్తే సరిపోతుంది, configuration ద్వారా serializer/batching
control చేయవచ్చు.

### ENGLISH INTERVIEW ANSWER

"A producer makes three decisions for every message: how to serialize
the Java object into bytes — JSON is simple, Avro with a schema
registry handles schema evolution better in larger systems — how to
partition it based on the key's hash, and how to batch it with other
pending messages before actually sending over the network, controlled
by `linger.ms`. Batching is the same throughput-versus-latency
trade-off we saw with JDBC batch inserts in Book 6: waiting briefly to
collect a batch means fewer network round-trips and higher overall
throughput, at the cost of a small added latency per individual
message. In Spring, `KafkaTemplate` abstracts all three concerns behind
a simple `send(topic, key, value)` call, with the actual serializer and
batching behavior controlled through configuration."

---

## 3.2 CONCEPT: Consumer Groups — How Kafka Achieves Both Broadcast and Parallelism

### TELUGU EXPLANATION

**ఇది Kafka architecture యొక్క అత్యంత elegant idea:** ఒకే topic కి,
**consumer group** అనే concept ద్వారా, **రెండు** వేర్వేరు use
cases ని ఒకేసారి support చేయవచ్చు:

- **వేర్వేరు consumer groups, ఒకే topic చదివితే:** ప్రతి group,
  **తన own, స్వతంత్రమైన copy** ఆ messages అన్నిటినీ చూస్తుంది —
  ఇది **pub/sub (broadcast)** behavior (ఉదా: "billing" group, మరియు
  "analytics" group, రెండూ ఒకే `order-events` topic ని, స్వతంత్రంగా
  చదవగలవు).
- **ఒకే consumer group లో, అనేక consumer instances ఉంటే:** Kafka,
  topic యొక్క partitions ని ఆ instances మధ్య **distribute**
  చేస్తుంది — ప్రతి partition, group లో **ఒక్క instance కి మాత్రమే**
  assign అవుతుంది (ఇది Chapter 2.8 Senior Check లో చూసిన
  "3 partitions, 5 instances" scenario కి కారణం) — ఇది **queue-like,
  load-balanced** behavior.

**Practical rule:** Partition count, ఒక consumer group లో **maximum
useful parallel consumers** ని నిర్ణయిస్తుంది — 3 partitions ఉంటే,
4వ consumer instance ఎప్పుడూ idle గా ఉంటుంది.

### ENGLISH INTERVIEW ANSWER

"Consumer groups are how Kafka elegantly supports both broadcast and
load-balanced consumption from the same topic. If two different
consumer groups both subscribe to the same topic, each group gets its
own complete, independent view of all the messages — that's pub/sub
broadcast behavior, useful when a billing service and an analytics
service both need every OrderPlaced event independently. Within a
single consumer group, if there are multiple consumer instances, Kafka
distributes the topic's partitions across them so each partition is
owned by exactly one instance in that group at a time — that's
load-balanced, queue-like behavior, since each message is only
processed once by the group as a whole. This is also why partition
count caps the useful parallelism within one group: with 3 partitions,
a 4th consumer instance in the same group simply sits idle, since there's
no partition left to assign it."

---

## 3.3 CONCEPT: Rebalancing — What Happens When Group Membership Changes

### TELUGU EXPLANATION

**Rebalancing:** ఒక consumer group లో, instances add అయినా (scale
up), remove అయినా (crash, scale down, deploy), Kafka **partitions ని
మళ్ళీ redistribute** చేస్తుంది — ఇది **rebalance**.

**Rebalancing యొక్క hidden cost:** Rebalance జరుగుతున్నప్పుడు,
**ఆ group లోని అన్ని consumers, temporarily processing ఆపేస్తాయి**
(stop-the-world, Book 1 Chapter 11 లో చూసిన GC pause కి conceptual
సారూప్యత) — partitions తిరిగి assign అయ్యేదాకా. Frequent rebalancing
(ఉదా: consumers, slow processing వలన "session timeout" దాటిపోయి,
group నుండి బయటకి పంపబడటం, తర్వాత మళ్ళీ join అవ్వడం) **throughput
ని తీవ్రంగా దెబ్బతీస్తుంది** — దీన్ని **"rebalancing storm"** అంటారు.

**Common causes, senior-level troubleshooting:**
- Consumer processing time, `max.poll.interval.ms` కంటే ఎక్కువ
  అవ్వడం (Kafka, ఆ consumer "dead" అని అనుకుని, group నుండి
  తొలగిస్తుంది).
- Frequent deploys/restarts (ప్రతి deploy, ఒక rebalance ట్రిగ్గర్
  చేస్తుంది).
- **Fix:** `max.poll.records` తగ్గించడం (ఒక్కసారి తక్కువ messages
  fetch చేయడం, batch processing time తగ్గించడానికి), heavy processing
  ని async గా offload చేయడం, `max.poll.interval.ms` ని realistic గా
  పెంచడం.

### ENGLISH INTERVIEW ANSWER

"Rebalancing is what happens when consumer group membership changes —
an instance joins, crashes, or is removed — and Kafka redistributes
partitions among the remaining instances. The hidden cost is that
during a rebalance, every consumer in that group temporarily stops
processing until partitions are reassigned, conceptually similar to a
stop-the-world GC pause from Book 1. Frequent, unnecessary rebalancing —
a 'rebalancing storm' — seriously hurts throughput, and the most common
cause I've seen in practice is a consumer's processing time per poll
exceeding `max.poll.interval.ms`, which makes Kafka assume that consumer
is dead and evict it from the group, triggering a rebalance; when it
comes back, it rejoins, triggering another one. The fix is usually to
reduce `max.poll.records` so each poll fetches a smaller, faster-to-process
batch, offload genuinely heavy processing to happen asynchronously
outside the poll loop, and set `max.poll.interval.ms` to a realistic
value for the actual processing time required."

---

## 3.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Multiple services need the same events | Has each service poll the topic with the same consumer group ID by mistake | Uses separate consumer group IDs per service for independent broadcast consumption |
| Scaling consumers | Adds instances without checking partition count | Ensures partition count actually supports the desired parallelism before scaling |
| Diagnosing slow throughput | Doesn't suspect rebalancing | Checks for frequent rebalances caused by slow processing exceeding poll interval |
| Long-running message processing | Does heavy work directly in the poll loop | Offloads heavy work asynchronously or tunes poll settings to avoid false "dead consumer" evictions |

---

## 3.5 COMMON MISTAKES

1. Accidentally using the same consumer group ID across unrelated
   services, causing each to only see a subset of messages instead of
   the full broadcast they expected.
2. Scaling consumer instances beyond the topic's partition count,
   leaving instances permanently idle.
3. Doing slow, blocking work directly in the poll loop, triggering
   `max.poll.interval.ms` timeouts and rebalancing storms.
4. Not monitoring consumer lag (how far behind the latest offset a
   consumer group is), missing a consumer that's silently falling behind.
5. Ignoring rebalancing entirely when diagnosing throughput problems.

---

## 3.6 INTERVIEW QUESTION BANK — CHAPTER 3

**Basic:** 1. What is a consumer group? 2. What determines whether two
consumers see the same messages or split them between themselves?

**Intermediate:** 3. What is a rebalance, and why does it pause
processing? 4. How does batching in the producer affect throughput and
latency?

**Senior:** 5. A consumer group's throughput drops sharply and
intermittently. You suspect rebalancing. How do you confirm this, and
what would you change?

**Architect:** 6. Design the consumer group topology for a system where
Billing, Analytics, and Fraud Detection all need every `OrderPlaced`
event, and each needs to scale its own processing independently.

**Scenario:** 7. A team scales their consumer group from 3 to 10
instances to "handle more load," but throughput doesn't improve. The
topic has 4 partitions. Diagnose.

**Trick:** 8. "Adding more consumer instances to a group always
increases throughput." True or false?

<details><summary>Key answers</summary>

- Q5: Check the broker/consumer logs for rebalance events and their
  frequency, and check consumer lag metrics for sawtooth patterns
  coinciding with rebalances; if confirmed, check `max.poll.interval.ms`
  against actual observed processing time per poll batch, and consider
  reducing `max.poll.records` or moving heavy work off the poll thread
  to reduce time-per-poll below that threshold.
- Q6: Give each of the three services its own consumer group ID
  (e.g., `billing-service-group`, `analytics-group`,
  `fraud-detection-group`) subscribed to the same `order-events` topic —
  each group gets a full, independent copy of every event, and each
  group's own instance count and partition assignment can scale
  independently based on that service's own processing needs, without
  affecting the others.
- Q7: The topic has only 4 partitions, so at most 4 consumer instances
  in one group can ever be actively assigned work — the other 6 of the
  10 instances are permanently idle, contributing nothing. The fix is
  increasing partition count (accepting Chapter 2's caveat about
  reordering existing keys) or accepting that 4 is the current
  parallelism ceiling for this group.
- Q8: False — throughput can only scale up to the number of partitions
  in the topic; beyond that, additional instances in the same group sit
  idle. It's also possible for excessive instances to actively hurt
  throughput indirectly by increasing the chance and cost of
  rebalancing during scaling events.

</details>

---

## 3.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain the difference between how consumer groups achieve broadcast (pub/sub) behavior versus load-balanced (queue-like) behavior from the same topic.
- **Coding Check:** N/A for this conceptual chapter — instead, sketch the consumer group IDs and instance counts you'd configure for a topic with 8 partitions serving 2 independent downstream services, one needing high parallelism and one needing very little.
- **Explanation Check:** Explain to a teammate why increasing `max.poll.records` can, counterintuitively, cause more rebalancing rather than less.
- **Real-World Check:** Your consumer occasionally calls a slow third-party API as part of processing each message, sometimes taking 40 seconds. `max.poll.interval.ms` is set to the default 5 minutes but you're seeing rebalances during traffic spikes. Diagnose and propose a fix.
- **Senior Check:** Why does Kafka pause all processing in a consumer group during a rebalance, instead of only reassigning affected partitions incrementally?
- **Master Check:** Design a strategy to safely deploy a new version of a Kafka consumer service with zero message loss and minimal rebalancing disruption, given that each deploy currently restarts all instances at once.

<details><summary>Answers</summary>

- Real-World Check: During traffic spikes, more messages queue up per
  poll batch, and if enough of them individually take close to 40
  seconds, the cumulative batch processing time can exceed 5 minutes,
  triggering the `max.poll.interval.ms` timeout and eviction. Fix:
  reduce `max.poll.records` so batches are smaller and finish well
  within the interval regardless of spikes, and/or make the third-party
  API call asynchronous/non-blocking relative to the poll loop so slow
  calls don't directly consume poll-interval budget.
- Senior Check: (Classic eager rebalancing, versus newer
  cooperative/incremental rebalancing protocols) Traditional eager
  rebalancing revokes all partition assignments across the whole group
  and reassigns from scratch, which is simpler to reason about
  correctness-wise but pauses everyone; newer cooperative rebalancing
  protocols (available in modern Kafka clients) can reassign only the
  specific partitions that actually need to move, letting unaffected
  consumers keep processing — a meaningful production tuning consideration.
- Master Check: Use a rolling deployment strategy (restart instances one
  or a few at a time, not all simultaneously) combined with a
  cooperative rebalancing assignor to minimize the blast radius of each
  individual instance's restart-triggered rebalance; ensure graceful
  shutdown handling commits the current offset before the instance
  exits, so no in-flight work is silently lost; and consider increasing
  `session.timeout.ms` slightly during deploys if instance startup time
  is non-trivial, to avoid premature eviction of a still-starting instance.

</details>

---

## 3.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Producer responsibilities | Serialize, partition (by key), and batch (`linger.ms`) messages |
| Different consumer groups, same topic | Each gets a full, independent copy — broadcast/pub-sub |
| Same consumer group, multiple instances | Partitions split across them — load-balanced, each message processed once |
| Parallelism ceiling | Bounded by partition count, regardless of instance count |
| Rebalance trigger | Group membership change — instance joins, crashes, or is evicted |
| Rebalance cost | Processing pauses group-wide (or partition-wide, with cooperative rebalancing) until reassignment completes |
| Common rebalance cause | Processing time exceeding `max.poll.interval.ms`, causing false "dead consumer" eviction |
| Fix for rebalancing storms | Reduce `max.poll.records`, offload heavy work, tune `max.poll.interval.ms` realistically |

---

*(Continues to Chapter 4 — Kafka Delivery Semantics & Reliability.)*
