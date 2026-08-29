# CHAPTER 4 — KAFKA DELIVERY SEMANTICS & RELIABILITY

---

## 4.1 CONCEPT: The `acks` Setting — Producer-Side Durability Control

### TELUGU EXPLANATION

**`acks`, Chapter 2.4 replication concept ని producer perspective
నుండి control చేసే setting:**

- **`acks=0`:** Producer, broker నుండి ఎలాంటి confirmation కోసం
  wait చేయదు — fastest, కానీ broker fail అయితే message **శాశ్వతంగా
  పోతుంది**, producer కి తెలియదు కూడా (Chapter 1's at-most-once).
- **`acks=1`:** Producer, **leader** broker write చేసిందని confirm
  అయ్యేదాకా wait చేస్తుంది — leader write చేసిన వెంటనే, replicate
  అయ్యేముందే leader fail అయితే, ఇప్పటికీ **data loss** సాధ్యం.
- **`acks=all` (లేదా `-1`):** Producer, **అన్ని in-sync replicas**
  (ISR) write చేసిందని confirm అయ్యేదాకా wait చేస్తుంది — highest
  durability, కానీ highest latency. Production systems లో, durability
  ముఖ్యమైనప్పుడు (ఉదా: payment events), **ఇదే సరైన default**.

**Senior-level nuance:** `acks=all` కూడా, `min.insync.replicas`
config తో కలిపి చూడాలి — replication factor 3, కానీ
`min.insync.replicas=1` అయితే, ఆచరణలో durability guarantee బలహీనంగా
ఉంటుంది (ఒక్క replica confirm అయితేనే సరిపోతుంది). `min.insync.replicas=2`
(replication factor 3 తో), genuine strong durability ఇస్తుంది.

### ENGLISH INTERVIEW ANSWER

"`acks` controls how much durability a producer demands before
considering a write successful, directly built on the replication model
from Chapter 2. `acks=0` doesn't wait for any confirmation at all —
fastest, but a broker failure can silently lose the message.
`acks=1` waits for just the partition leader to confirm the write, which
is still vulnerable to data loss if the leader fails before followers
replicate it. `acks=all` waits for every in-sync replica to confirm,
giving the strongest durability at the cost of higher latency — this is
the right default for anything where losing a message is unacceptable,
like payment events. The nuance senior engineers need to know is that
`acks=all` alone isn't the full picture — it has to be paired with
`min.insync.replicas`; with a replication factor of 3 but
`min.insync.replicas=1`, you can still lose durability in practice
because only one replica actually needs to confirm. Setting
`min.insync.replicas=2` with replication factor 3 is what actually
delivers strong, genuine durability."

---

## 4.2 CONCEPT: Idempotent Producer — Eliminating Duplicate Writes from Retries

### TELUGU EXPLANATION

**సమస్య:** Producer, `acks=all` తో write చేసి, broker actually
write చేసినా, **acknowledgment network లో పోతే** — producer, "fail
అయింది" అని అనుకుని **retry** చేస్తుంది — ఫలితంగా, ఆ message
**partition లో రెండుసార్లు** కనిపిస్తుంది (broker-level duplicate,
Chapter 1 యొక్క consumer-level duplicate కి అదనంగా).

**పరిష్కారం — Idempotent Producer (`enable.idempotence=true`):**
ప్రతి producer కి ఒక unique ID, ప్రతి message కి ఒక sequence number
ఇచ్చి, broker, **అదే producer నుండి, అదే sequence number ఉన్న
message ని** రెండోసారి చూస్తే, **silently discard** చేస్తుంది —
ఇది **broker-level, producer-retry-caused duplicates** ని పరిష్కరిస్తుంది
(consumer-level idempotency ని ఇది replace చేయదు — అది వేరే
సమస్య, Chapter 1).

### ENGLISH INTERVIEW ANSWER

"Even with `acks=all`, a producer can end up retrying unnecessarily —
the broker wrote the message successfully, but the acknowledgment
itself got lost over the network, so the producer assumes failure and
resends. Without protection, this creates a genuine duplicate in the
partition itself, separate from and in addition to the consumer-level
duplicate problem from Chapter 1. Enabling the idempotent producer
(`enable.idempotence=true`) solves this specific case: each producer
gets a unique ID, each message a sequence number, and the broker
recognizes and silently discards a message it's already seen from that
same producer with that same sequence number. This is a genuinely
different, broker-level fix — it doesn't replace the need for an
idempotent consumer to handle at-least-once delivery to the application
layer, it just eliminates one specific source of duplication caused by
producer-side retries."

---

## 4.3 CONCEPT: Kafka Transactions — Exactly-Once Across Read-Process-Write

### TELUGU EXPLANATION

**Use case:** ఒక Kafka Streams-style application, ఒక topic నుండి
చదివి, process చేసి, ఫలితాన్ని **మరో topic కి** రాస్తుంది
("read-process-write"). ఇక్కడ genuine exactly-once కావాలంటే: input
consume చేయడం, output produce చేయడం, **రెండూ ఒకే atomic transaction
లో** జరగాలి — ఏదో ఒకటి fail అయితే, **రెండూ rollback** అవ్వాలి.

**Kafka Transactions (`transactional.id` + `read_committed` isolation
level):** ఇది సరిగ్గా ఇదే guarantee ఇస్తుంది — producer, ఒక
transaction begin చేసి, ఒకటి లేదా అంతకంటే ఎక్కువ topics కి write
చేసి, input topic యొక్క offset commit **కూడా అదే transaction లో
భాగం** గా చేసి, transaction commit చేస్తుంది. Downstream consumers,
`isolation.level=read_committed` set చేస్తే, **committed transactions
యొక్క messages మాత్రమే** చూస్తాయి, aborted transaction యొక్క
messages వాళ్ళకి కనిపించవు.

**Critical caveat, senior-level:** ఈ guarantee, **Kafka-to-Kafka**
flow కి మాత్రమే వర్తిస్తుంది. Consumer, ఒక external database కి
write చేసినా, ఒక external API call చేసినా, ఆ side effect, Kafka
transaction boundary లో భాగం **కాదు** — అక్కడ ఇప్పటికీ, Chapter 1
యొక్క idempotent consumer pattern అవసరం.

### ENGLISH INTERVIEW ANSWER

"Kafka transactions solve exactly-once specifically for the
read-process-write pattern — an application consuming from one topic,
processing, and producing to another. The transaction ties together
producing to the output topic and committing the input topic's offset
as one atomic unit: either both happen or neither does, so a failure
partway through can't leave you having produced output without
advancing the input offset, or vice versa. Downstream consumers opt
into this guarantee with `isolation.level=read_committed`, so they only
ever see messages from committed transactions, never from aborted ones.
The critical caveat I'd make sure an interviewer knows I understand:
this exactly-once guarantee only covers Kafka-to-Kafka flows. The moment
a consumer's processing has an external side effect — writing to a
different database, calling a third-party API — that side effect is
outside the transaction boundary entirely, and you're back to needing
Chapter 1's idempotent consumer pattern for that specific effect."

---

## 4.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Choosing `acks` | Uses the client default without considering it | Chooses `acks=all` with `min.insync.replicas` tuned for genuinely important data |
| Producer retries | Doesn't consider broker-level duplicates from retries | Enables idempotent producer to eliminate retry-caused duplicates |
| Hearing "Kafka transactions" | Assumes it makes any external side effect exactly-once | Knows it only covers Kafka-to-Kafka read-process-write flows |
| Designing a consumer with external effects | Trusts the broker's guarantees alone | Still implements consumer-side idempotency for anything leaving Kafka |

---

## 4.5 COMMON MISTAKES

1. Using `acks=1` or `acks=0` for data where message loss is genuinely
   unacceptable, like financial events.
2. Setting `acks=all` without also setting `min.insync.replicas`
   appropriately, weakening the actual durability guarantee.
3. Assuming Kafka's idempotent producer eliminates the need for an
   idempotent consumer.
4. Believing Kafka transactions make an external API call or database
   write exactly-once just because the consumer reads from a
   transactional Kafka topic.
5. Not setting `isolation.level=read_committed` on a consumer reading
   from a topic produced to transactionally, silently seeing uncommitted
   (and potentially aborted) data.

---

## 4.6 INTERVIEW QUESTION BANK — CHAPTER 4

**Basic:** 1. What does `acks=all` mean? 2. What problem does the
idempotent producer solve?

**Intermediate:** 3. Why isn't `acks=all` alone sufficient for strong
durability, and what else must be configured? 4. What is
`isolation.level=read_committed` for?

**Senior:** 5. Design the producer and consumer configuration for a
payment-event pipeline where losing or duplicating an event is
unacceptable.

**Architect:** 6. Explain the actual boundary of Kafka's exactly-once
guarantee to a team that wants to rely on it for a consumer that also
writes to an external database.

**Scenario:** 7. A team enables Kafka transactions for their
read-process-write service and declares the system "fully exactly-once
end to end," including a step that calls an email-sending API. What's
wrong with this claim?

**Trick:** 8. "Enabling `enable.idempotence=true` on the producer
means my consumer no longer needs to worry about duplicate processing."
True or false?

<details><summary>Key answers</summary>

- Q5: Producer: `acks=all`, `enable.idempotence=true`, and a replication
  factor of at least 3 with `min.insync.replicas=2` on the topic.
  Consumer: process-then-commit offset ordering (Chapter 2), plus an
  idempotent processing step (e.g., a durable "already processed"
  record keyed by a payment/event ID) since at-least-once delivery to
  the consumer is still possible even with all producer-side protections
  in place.
- Q6: I'd explain that Kafka's transactional exactly-once guarantee
  covers only the read-input-offset-commit and produce-to-output-topic
  cycle within Kafka itself — the moment the consumer's processing
  writes to an external database or calls an external API, that action
  is outside the transaction boundary, and a crash after the external
  write but before the Kafka-side commit (or vice versa) can still cause
  a duplicate or inconsistent external effect. The team still needs to
  make that specific external write idempotent, independent of the
  Kafka transaction.
- Q7: The email-sending API call is an external side effect completely
  outside Kafka's transactional boundary — Kafka transactions guarantee
  nothing about it. If the consumer sends the email and then crashes
  before completing its Kafka-side commit, on restart it will reprocess
  the message and could send the email again; the "fully exactly-once
  end to end" claim is false for any step involving an external,
  non-Kafka side effect.
- Q8: False — the idempotent producer only eliminates duplicates caused
  by the *producer* retrying a write to the broker. It does nothing
  about the separate at-least-once delivery semantics between the
  broker and the *consumer*, which can still redeliver a message the
  consumer already successfully processed (e.g., if the offset commit
  is lost after processing). Consumer-side idempotency remains necessary.

</details>

---

## 4.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain the difference between `acks=1` and `acks=all`, and why `acks=all` alone still isn't sufficient without `min.insync.replicas`.
- **Coding Check:** N/A for this conceptual chapter — instead, write the producer configuration properties you'd set for a payment-events topic, with a one-line justification for each.
- **Explanation Check:** Explain to a teammate why the idempotent producer and an idempotent consumer solve two genuinely different problems, not the same one twice.
- **Real-World Check:** Your team is building an order-processing pipeline where a consumer reads OrderPlaced events and writes rows to a reporting database. Would Kafka transactions alone make this pipeline exactly-once? What else is needed?
- **Senior Check:** Why does `isolation.level=read_committed` matter specifically for a consumer reading a topic that a transactional producer writes to?
- **Master Check:** Design the full reliability configuration (producer, topic, and consumer settings) for a critical inventory-reservation pipeline, explaining what failure mode each setting specifically protects against.

<details><summary>Answers</summary>

- Real-World Check: No — Kafka transactions only cover Kafka-to-Kafka
  read-process-write flows; writing to an external reporting database is
  outside that boundary. The pipeline also needs the consumer's database
  write to be idempotent (e.g., an upsert keyed by order ID, or checking
  a processed-events table before writing) so a redelivered message
  doesn't create duplicate reporting rows.
- Senior Check: Without `read_committed`, a consumer using the default
  `read_uncommitted` isolation level would see messages from a
  transaction even before (or if) it's committed — including messages
  from a transaction that later gets aborted, which the consumer would
  have already processed as if they were valid. `read_committed`
  ensures the consumer only ever sees messages belonging to transactions
  that actually committed successfully.
- Master Check: Producer: `acks=all` (protects against accepting a write
  the cluster hasn't durably confirmed), `enable.idempotence=true`
  (protects against producer-retry-caused broker-level duplicates).
  Topic: replication factor 3 with `min.insync.replicas=2` (protects
  against losing durability when one replica is temporarily unavailable).
  Consumer: process-then-commit offsets (protects against silently
  skipping unprocessed messages), plus an idempotent reservation check
  (e.g., a unique constraint or existing-reservation lookup keyed by
  order ID) before actually reserving inventory (protects against
  double-reserving stock on a redelivered message).

</details>

---

## 4.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| `acks=0` | No confirmation; fastest, highest loss risk |
| `acks=1` | Leader-only confirmation; still loses data if leader fails before replication |
| `acks=all` | All in-sync replicas confirm; pair with `min.insync.replicas` for real durability |
| Idempotent producer | Eliminates broker-level duplicates from producer retries only |
| Kafka transactions | Exactly-once for Kafka-to-Kafka read-process-write only |
| `isolation.level=read_committed` | Consumer only sees committed transactional writes |
| External side effects | Never covered by Kafka's guarantees — always need consumer-side idempotency |
| Critical data (payments, inventory) | `acks=all` + `min.insync.replicas≥2` + idempotent producer + idempotent consumer |

---

*(Continues to Chapter 5 — RabbitMQ Architecture: Exchanges, Queues, Routing.)*
