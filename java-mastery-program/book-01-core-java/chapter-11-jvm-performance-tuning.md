# CHAPTER 11 — JVM PERFORMANCE AND TUNING

> This chapter ties together Chapters 1, 9, and 10 into the practical
> skill of tuning a real running JVM — the capstone of Book 1's internals track.

---

## 11.1 CONCEPT: The Tuning Mindset — Measure, Don't Guess

### TELUGU EXPLANATION

JVM tuning లో అతిపెద్ద mistake: **measure చేయకుండా flags మార్చడం**.
"Heap size పెంచేద్దాం," "GC algorithm మార్చేద్దాం" అని ఊహాజనితంగా flags
touch చేయడం — ఇది Junior thinking. Senior engineer ఎప్పుడూ ఇలా అడుగుతారు:

1. **దేని కోసం optimize చేస్తున్నాం?** — Throughput (units/sec)? Latency
   (p99 response time)? Startup time (container cold-start)? Memory
   footprint (cost, density per node)? — ఇవి తరచుగా **ఒకదానికొకటి trade-off**
   (ఉదా: throughput-optimized GC latency కోల్పోవచ్చు).
2. **Baseline ఏంటి?** — మార్పు చేసేముందు, current metrics (GC pause times,
   throughput, memory usage) record చేయాలి.
3. **మార్పు తర్వాత measurable improvement ఉందా?** — A/B comparison,
   controlled load test.

### ENGLISH INTERVIEW ANSWER

"My rule for JVM tuning is: never change a flag without first knowing what
metric I'm optimizing for and having a baseline measurement. Throughput,
latency, memory footprint, and startup time are all real, and they often
trade off against each other — a GC tuned for maximum throughput can
introduce longer occasional pauses, hurting p99 latency. So before touching
`-Xmx` or switching collectors, I establish what SLA actually matters for
this specific service, capture current GC logs and metrics as a baseline,
make one change at a time, and validate under realistic load — not
theoretical reasoning alone."

---

## 11.2 CONCEPT: Key JVM Flags and What They Actually Control

### TELUGU EXPLANATION

| Flag | Controls | Senior note |
|---|---|---|
| `-Xms` / `-Xmx` | Initial / max heap size | Setting `-Xms == -Xmx` avoids heap resize pauses at runtime (common production practice) |
| `-Xss` | Per-thread stack size | Lower it to fit more threads in constrained memory; raise it if legitimate deep recursion needs headroom |
| `-XX:MaxMetaspaceSize` | Metaspace cap | Set explicitly in containers to convert a classloader leak into a catchable OOM instead of unbounded native memory growth |
| `-XX:+UseG1GC` / `-XX:+UseZGC` | GC algorithm | Default is G1 since Java 9; choose ZGC for strict low-latency SLAs on large heaps |
| `-XX:MaxGCPauseMillis` | G1's target pause time (a goal, not a hard guarantee) | Lower target trades off some throughput |
| `-Xlog:gc*:file=gc.log` | GC logging | Always enable in production — near-zero overhead, invaluable for incident review |
| `-XX:+HeapDumpOnOutOfMemoryError` | Auto heap dump on OOM | Should be **default-on** in every production service — you cannot root-cause a leak after the fact without this |
| `-XX:NativeMemoryTracking=summary` | Off-heap memory visibility | Essential when investigating "JVM uses more memory than -Xmx suggests" in containers |

### ENGLISH INTERVIEW ANSWER

"A handful of flags matter far more than the rest in practice. I always set
`-Xms` equal to `-Xmx` in production — letting the heap resize at runtime
causes avoidable pauses and makes capacity planning fuzzier. I always enable
`-XX:+HeapDumpOnOutOfMemoryError`, because without a heap dump captured at
the moment of failure, root-causing a memory leak after the fact is close
to guesswork. I enable GC logging (`-Xlog:gc*`) by default too — it's
near-zero overhead and is the first thing I want available when a latency
incident happens, rather than something I have to remember to turn on after
the fact. Beyond that, I explicitly cap Metaspace in containerized
deployments, since an unbounded native-memory growth there won't show up in
heap metrics at all and can silently push a container over its memory limit."

---

## 11.3 CONCEPT: Reading GC Logs

### TELUGU EXPLANATION

ఒక typical G1 GC log line (simplified):

```
[12.345s][info][gc] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
    28M->6M(64M) 8.219ms
```

దీన్ని ఇలా చదవాలి: **GC #42**, ఇది ఒక **Young generation pause**
(Minor GC), ఇది heap usage ని **28MB నుండి 6MB కి** తగ్గించింది
(current total heap 64MB), ఇది **8.219 milliseconds** పట్టింది.

**Senior గా ఏం చూడాలి:**
- **Pause frequency పెరుగుతోందా?** — allocation rate పెరుగుతోంది అని సూచన,
  లేదా heap చాలా చిన్నది కావొచ్చు.
- **Pause duration పెరుగుతోందా?** — Old Gen pressure పెరుగుతోంది అని
  సూచన, leak అవ్వొచ్చు లేదా genuine undersizing.
- **"Full GC" entries ఉన్నాయా?** — ఇవి చాలా ఖరీదైనవి (stop-the-world,
  మొత్తం heap); తరచుగా కనిపిస్తే, ఇది urgent investigation కావాలి.
- **"before-GC" heap usage, GC తర్వాత కూడా baseline కి తిరిగి రావట్లేదా?**
  — ప్రతి GC cycle తర్వాత minimum usage పెరుగుతూనే ఉంటే, ఇది **leak
  signature** — genuine "steady state" workload GC తర్వాత స్థిరమైన baseline
  కి తిరిగి రావాలి.

---

## 11.4 CONCEPT: Profiling — Finding the Actual Bottleneck

### TELUGU EXPLANATION

"Performance slow" అనే ఫిర్యాదు వచ్చినప్పుడు, **guess చేయకుండా profile
చేయాలి**:

- **CPU profiling** (`async-profiler`, JFR — Java Flight Recorder): CPU
  time ఎక్కడ ఖర్చు అవుతోందో (which methods, what % of samples) చూపిస్తుంది
  — **flame graphs** ద్వారా visualize చేస్తారు.
- **JFR (Java Flight Recorder):** production-safe (low overhead), JDK
  built-in profiling — CPU, allocation, lock contention, GC అన్నీ ఒకేసారి
  capture చేస్తుంది, `jcmd <pid> JFR.start` తో production లో కూడా safely
  run చేయవచ్చు.
- **Heap profiling:** `jmap`/heap dump + Eclipse MAT — memory retention
  కోసం (Chapter 1 లో చూసినట్టు).
- **Thread profiling:** `jstack`/JFR thread dumps — lock contention, thread
  starvation కోసం (Chapter 9).

**Senior rule:** "Slow" అనే word తనంతట తానుగా ఏమీ చెప్పదు — CPU-bound
(profiler CPU hotspot చూపిస్తుంది), I/O-bound (threads mostly `WAITING`/
`BLOCKED` on network/DB calls in thread dumps), లేదా GC-bound (GC logs
pause correlation చూపిస్తాయి) అని **differentiate చేయడమే మొదటి పని** —
తర్వాతే fix దిశ నిర్ణయించాలి.

### ENGLISH INTERVIEW ANSWER

"When someone reports 'the service is slow,' my first move is to classify
*what kind* of slow, because the fix is completely different depending on
the answer. I take a thread dump — if most threads are `BLOCKED` or
`WAITING` on I/O, it's likely a downstream dependency or lock contention
issue, not a JVM problem at all. I check GC logs for pause correlation with
the reported slowness — if GC pauses line up with the latency spikes, it's
memory pressure. If neither points anywhere, I reach for CPU profiling —
Java Flight Recorder is my default since it's built into the JDK and safe to
run in production with low overhead — to get an actual flame graph of where
CPU time is spent, rather than guessing based on which code 'looks'
expensive."

---

## 11.5 CODE: A REPRODUCIBLE TUNING EXERCISE

**Requirement:** Demonstrate the measurable effect of heap sizing on GC
behavior — turning theory into an experiment you can actually run.

```java
public class AllocationBenchmark {
    public static void main(String[] args) throws InterruptedException {
        List<byte[]> shortLived = new ArrayList<>();
        long start = System.currentTimeMillis();
        long allocations = 0;

        while (System.currentTimeMillis() - start < 30_000) { // run for 30 seconds
            shortLived.add(new byte[1024]); // allocate 1KB
            allocations++;
            if (shortLived.size() > 1000) {
                shortLived.clear(); // let most objects become garbage quickly
            }
        }
        System.out.println("Total allocations: " + allocations);
    }
}
```

**Experiment:** Run this twice with different heap sizes and GC logging:

```
java -Xms64m -Xmx64m -Xlog:gc*:file=small-heap.log AllocationBenchmark
java -Xms512m -Xmx512m -Xlog:gc*:file=large-heap.log AllocationBenchmark
```

**Expected observation:** `small-heap.log` shows many more, more frequent
Young GC pauses (Eden fills up faster with less room), while
`large-heap.log` shows fewer, less frequent pauses — but total memory
footprint is higher. This is the throughput-vs-footprint trade-off made
concrete and measurable, not theoretical — exactly what a senior engineer
should be able to demonstrate, not just describe.

**Interviewer follow-up:** "Which configuration is 'better'?" — Neither,
unconditionally — it depends on whether this service runs on a
memory-constrained container (favoring the smaller heap despite more
frequent pauses) or a latency-sensitive service with headroom to spare
(favoring the larger heap for fewer pauses). This is the correct senior
answer: context-dependent, not a universal rule.

---

## 11.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| "Service is slow" | Increase heap size and hope | Classify first: CPU-bound, I/O-bound, or GC-bound — then act |
| GC pause complaints | Switch to the "best" GC blindly | Check current pause times/frequency against actual SLA first — may not even be the bottleneck |
| Production OOM | Just raise `-Xmx` | Heap dump analysis first — leak vs genuine undersizing are different problems with different fixes |
| Container memory limit exceeded despite `-Xmx` set below it | Confused, "JVM ignoring my flag" | Check Metaspace, thread stacks, direct buffers, JIT code cache — memory outside the heap |

---

## 11.7 COMMON MISTAKES

1. Changing multiple JVM flags at once, making it impossible to attribute
   any observed effect to a specific change.
2. Tuning based on theory/blog posts without measuring the actual workload.
3. Not enabling GC logging and heap-dump-on-OOM by default in production —
   discovering this gap only after an incident, too late to have useful data.
4. Confusing "high memory usage" with "memory leak" — a service that uses a
   large but *stable* amount of heap under steady load is not leaking; only
   *unbounded growth over time* is a leak signature.
5. Assuming a container's memory limit only needs to account for `-Xmx`,
   ignoring Metaspace, thread stacks, and native/direct memory.

---

## 11.8 INTERVIEW QUESTION BANK — CHAPTER 11

**Basic:** 1. What's the first thing you should do before changing any JVM
tuning flag? 2. What does `-XX:+HeapDumpOnOutOfMemoryError` do and why
enable it by default?

**Intermediate:** 3. How do you read a GC log line to determine if pause
times are trending upward? 4. What's the difference between throughput and
latency as tuning goals, and how might optimizing for one hurt the other?

**Senior:** 5. A service's container is OOM-killed by Kubernetes even
though `-Xmx` is comfortably below the pod's memory limit. Walk through your
investigation (tie this back to Chapter 1). 6. When would you use JFR over
`jstack`/`jmap`, and vice versa?

**Architect:** 7. You're standardizing JVM tuning defaults across 50
microservices with different workload profiles (some CPU-bound batch jobs,
some latency-sensitive APIs). What baseline defaults would you set
org-wide, and what would you leave per-service?

**Scenario:** 8. After a "successful" tuning change (switching GC
algorithms), throughput metrics improved but p99 latency got worse, and
customers are complaining. What happened, and what's your next step?

**Trick:** 9. "A larger heap always means better performance." True or false?

<details><summary>Key answers</summary>

- Q7: Org-wide baseline: GC logging always on, heap-dump-on-OOM always on,
  `-Xms == -Xmx`, an explicit Metaspace cap, and JFR continuous recording
  enabled (very low overhead) as a safety net for post-incident analysis.
  Per-service: heap size, GC algorithm choice, and pause-time targets, since
  those genuinely depend on each service's workload profile and SLA.
- Q8: Likely switched to a throughput-optimized collector (e.g., Parallel
  GC) that reduces total GC overhead but takes longer, less frequent
  pauses — improving average throughput while worsening tail latency. Next
  step: revert or switch to a latency-focused collector (G1 with a lower
  pause-time target, or ZGC) and re-measure both throughput and p99 latency
  together, since they were traded off against each other, not both improved.
- Q9: False — a larger heap can mean fewer but longer GC pauses (especially
  for Full GCs, which scale with heap size), higher memory cost, and slower
  startup/warm-up; "better" depends entirely on the workload and the metric
  that matters for that service.

</details>

---

## 11.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Name three things you should capture as a baseline BEFORE changing any JVM flag.
- **Coding Check:** Run the allocation benchmark from 11.5 with two different heap sizes yourself, and write two sentences comparing the resulting GC logs.
- **Explanation Check:** Explain in English, to a non-technical stakeholder, why "just add more memory" isn't always the right fix for a slow service.
- **Real-World Check:** Your team's standard is to always enable `-XX:+HeapDumpOnOutOfMemoryError` in production. A teammate objects that heap dumps are huge and slow to write during an actual OOM, making the outage worse. How do you respond?
- **Senior Check:** How would you decide between tuning G1's `-XX:MaxGCPauseMillis` target lower vs. migrating to ZGC entirely?
- **Master Check:** Design a full production readiness checklist item for "JVM observability" that a new service must satisfy before launch — list the specific flags/settings and why each is there.

<details><summary>Answers</summary>

- Real-World Check: Acknowledge the trade-off is real but argue it's still
  worth it — an OOM is already a severe failure event; the heap dump write
  happens once, at the moment of crash, and provides the only real evidence
  to prevent recurrence, versus repeating the same unexplained OOM
  indefinitely without root cause. Mitigate by writing dumps to fast local
  disk and ensuring adequate disk space is provisioned.
- Senior Check: Try lowering G1's pause target first (cheap, no algorithm
  change, measure the result) — only migrate to ZGC if G1 genuinely cannot
  hit the required pause-time ceiling even after tuning, since ZGC trades
  some throughput/CPU overhead for its latency guarantees and migrating
  algorithms is a bigger, riskier change than tuning parameters.
- Master Check: `-Xms == -Xmx` (stable heap, no resize pauses),
  `-XX:+HeapDumpOnOutOfMemoryError` + a writable dump path (root-cause
  capability), `-Xlog:gc*:file=...` with log rotation (always-on GC
  visibility), an explicit `-XX:MaxMetaspaceSize` (container memory
  safety), and JFR continuous recording enabled (general-purpose
  post-incident profiling) — each tied to a specific "we couldn't diagnose
  X without this" scenario from earlier in this chapter.

</details>

---

## 11.10 CHEAT SHEET

| Goal | Flag/Practice |
|---|---|
| Avoid resize pauses | `-Xms == -Xmx` |
| Root-cause OOMs | `-XX:+HeapDumpOnOutOfMemoryError` |
| Always-available GC visibility | `-Xlog:gc*:file=gc.log` |
| Cap native-memory class metadata leaks | `-XX:MaxMetaspaceSize` |
| Low-overhead production profiling | JFR (`jcmd <pid> JFR.start`) |
| Classify "slow" before fixing | Thread dump (I/O?) → GC logs (memory?) → CPU profile (compute?) |

---

## BOOK 1 — CHAPTER 11 COMPLETE

*(Book 1's chapter content (11 chapters) is now complete. See
`final-assessment-mock-interview-project.md` in this directory for the
cumulative Final Assessment, the Core Java Mock Interview round, and the
Book 1 capstone Project Assignment.)*
