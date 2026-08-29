# CHAPTER 3 — BEAN SCOPES & LIFECYCLE

---

## 3.1 CONCEPT: Bean Scopes — How Many Instances Exist

### TELUGU EXPలanaTION

**Scope** అంటే, Spring container ఒక bean యొక్క **ఎన్ని instances**
create చేస్తుంది, మరియు వాటి **lifetime** ఏమిటి అని నిర్ణయించడం.

| Scope | అర్థం | ఎప్పుడు వాడాలి |
|---|---|---|
| **singleton** (default) | container కి **ఒక్క instance మాత్రమే**, అందరూ దాన్నే share చేస్తారు | Stateless services (చాలా వరకు `@Service`/`@Repository` beans) |
| **prototype** | `getBean()`/injection జరిగిన ప్రతిసారి **కొత్త instance** | Mutable, non-thread-safe, per-use state అవసరమైనప్పుడు |
| **request** (web-aware) | ప్రతి HTTP request కి ఒక instance | Request-scoped data (ఉదా: request context, correlation ID holder) |
| **session** (web-aware) | ప్రతి HTTP session కి ఒక instance | User-session-specific state |

**అత్యంత ముఖ్యమైన senior-level insight — Singleton + Mutable State =
Thread-Safety Bug:** Default scope **singleton** — అంటే, ఒకే instance ని
**అన్ని threads (అన్ని concurrent HTTP requests) share చేస్తాయి**. ఒక
singleton bean లో **mutable instance field** ఉంటే, ఇది Book 1 Chapter 9
లో మనం నేర్చుకున్న **race condition** కి direct దారి తీస్తుంది:

```java
// ❌ DANGEROUS — singleton bean తో mutable instance state
@Service
class OrderCounter {
    private int count = 0; // ❌ shared mutable state, అన్ని requests ఇదే field ని share చేస్తాయి!

    void increment() { count++; } // race condition — Book 1 Chapter 9 సమస్యే
}
```

**Fix:** Singleton beans ని **stateless** గా design చేయండి (local
variables వాడండి, instance fields కాదు), లేదా state నిజంగా అవసరమైతే,
**thread-safe** structures (`AtomicInteger`, `ConcurrentHashMap`) వాడండి
— ఇది Book 1 Chapter 10 direct application.

### ENGLISH INTERVIEW ANSWER

"Scope determines instance count and lifetime — singleton, one instance
shared by the whole container, is the default and correct choice for the
vast majority of Spring beans, which should be stateless. The critical
thing to internalize is that 'singleton' doesn't mean 'thread-safe' — it
means 'shared by every concurrent request/thread,' so any mutable instance
state on a singleton bean is exactly the shared-mutable-state race
condition scenario from concurrency fundamentals. I design singleton
services to be stateless, keeping any per-operation state as local
variables rather than fields; if a bean genuinely needs mutable state
scoped to a single use, prototype scope is the tool, and if it needs
thread-safe shared state, I reach for atomics or concurrent collections
rather than assuming the container somehow makes access safe — it doesn't."

**Interviewer follow-up:** "How do you inject a prototype-scoped bean into
a singleton-scoped bean correctly?" — This is a well-known gotcha: naive
constructor injection captures the prototype bean **once**, at the
singleton's creation time, permanently — you get what's effectively a
singleton behavior anyway. The fix is `ObjectProvider<T>` (or a
`@Lookup` method), which fetches a **fresh** prototype instance on each call.

---

## 3.2 CONCEPT: The Bean Lifecycle — Where Your Code Can Hook In

### TELUGU EXPLANATION

ఒక bean యొక్క life ఇలా ఉంటుంది:

```
1. Instantiation (constructor call)
        ↓
2. Dependency Injection (fields/setters populate అవుతాయి)
        ↓
3. Aware interfaces (BeanNameAware, ApplicationContextAware, మొదలైనవి — అరుదుగా అవసరం)
        ↓
4. BeanPostProcessor.postProcessBeforeInitialization()  ← @Autowired, @Value ఇక్కడే resolve అవుతాయి internally
        ↓
5. @PostConstruct method (లేదా InitializingBean.afterPropertiesSet())
        ↓
6. BeanPostProcessor.postProcessAfterInitialization()  ← AOP proxies ఇక్కడే create అవుతాయి! (Chapter 4)
        ↓
7. Bean READY — application వాడుకుంటుంది
        ↓
   (Container shutdown అయినప్పుడు...)
        ↓
8. @PreDestroy method (లేదా DisposableBean.destroy())
```

**`@PostConstruct` ఎప్పుడు వాడాలి:** Dependencies **ఇంజెక్ట్ అయిన
తర్వాత మాత్రమే** run అవ్వాల్సిన initialization logic కి — ఉదా: ఒక
connection pool ని dependencies (config values) ఆధారంగా warm-up చేయడం.
**Constructor లో ఇది ఎందుకు చేయకూడదు:** constructor call అయ్యే సమయానికి,
setter-injected/field-injected dependencies ఇంకా populate అయ్యి
ఉండకపోవచ్చు (constructor injection అయితే ఇది సమస్య కాదు, ఎందుకంటే
dependencies అప్పటికే constructor parameters గా present ఉంటాయి — ఇది
Chapter 1 లో constructor injection favor చేయడానికి **మరో కారణం**).

**`BeanPostProcessor` ఎందుకు ముఖ్యం (senior/architect-level):** ఇది
Spring యొక్క అనేక "magic"-లా కనిపించే features వెనుక ఉన్న **actual
mechanism** — `@Autowired` resolution, `@Value` injection, మరియు
**AOP proxies create చేయడం** (Chapter 4) — అన్నీ `BeanPostProcessor`
implementations ద్వారానే జరుగుతాయి. ఇది framework internals ఎలా పని
చేస్తాయో అర్థం చేసుకోవడానికి కీలకమైన concept.

### ENGLISH INTERVIEW ANSWER

"The lifecycle matters because it tells you exactly where your
initialization logic is safe to run. `@PostConstruct` runs after all
dependency injection is complete, which is why it's the right place for
logic that needs injected dependencies already populated — doing the same
work in the constructor is only safe if you're using constructor
injection, since with field/setter injection the dependencies aren't set
yet when the constructor body runs. The `BeanPostProcessor` hooks are
where Spring's own 'magic' actually lives — `@Autowired`/`@Value`
resolution and AOP proxy creation are all implemented as
`BeanPostProcessor`s under the hood, which is worth knowing not because
you'll write one often, but because it demystifies what's actually
happening when Spring 'processes annotations' — it's not compiler magic,
it's ordinary post-processing hooks the container calls for you."

---

## 3.3 CONCEPT: Circular Dependencies — Diagnosis and Fix

### TELUGU EXPLANATION

**Circular dependency:** Bean A కి Bean B కావాలి, Bean B కి Bean A
కావాలి. Constructor injection తో ఇది **startup వద్దే** fail అవుతుంది
(`BeanCurrentlyInCreationException`) — ఎందుకంటే, A ని create చేయాలంటే
B పూర్తిగా ready గా ఉండాలి, B ని create చేయాలంటే A పూర్తిగా ready గా
ఉండాలి — **deadlock లాంటి పరిస్థితి**, resolve చేయలేనిది.

```java
@Service
class ServiceA {
    ServiceA(ServiceB b) { } // B కావాలి
}

@Service
class ServiceB {
    ServiceB(ServiceA a) { } // A కావాలి — CIRCULAR!
}
```

**ఎందుకు ఇది ఎప్పుడూ ఒక design problem (workaround కాదు, root cause fix
కావాలి):** ఇది సాధారణంగా **SRP violation** ని సూచిస్తుంది (Book 1
Chapter 2) — రెండు classes ఒకదానికొకటి tightly ఆధారపడి ఉన్నాయంటే, అవి
నిజానికి ఒకే responsibility ని పంచుకుంటున్నాయని అర్థం, లేదా వాటి మధ్య
shared logic ని **మూడో class** గా extract చేయాలని సూచన.

**"Fixes" (వీటిని జాగ్రత్తగా వాడాలి, root cause పరిష్కారం కాదు):**
1. **Extract a third bean** `SharedLogic` గా, రెండూ దీని మీద depend
   అవ్వేలా చేయండి — **ఇదే నిజమైన fix**.
2. **`@Lazy`** ఒక injection point మీద — Spring ఒక proxy ఇస్తుంది,
   actual bean ని మొదటిసారి వాడేటప్పుడు మాత్రమే resolve చేస్తుంది. ఇది
   **compile-time/startup-time circular dependency ని runtime వరకు
   వాయిదా వేస్తుంది మాత్రమే** — root cause అలాగే ఉంటుంది.
3. **Setter injection కి మారడం** — Spring దీన్ని silently permit
   చేస్తుంది (Chapter 1 లో ప్రస్తావించినట్టు) — **ఇది anti-pattern**,
   ఎందుకంటే ఇది design సమస్యని దాచేస్తుంది, పరిష్కరించదు.

### ENGLISH INTERVIEW ANSWER

"A circular dependency between constructor-injected beans fails fast at
startup, which I consider the correct, helpful behavior — it's surfacing a
real design problem immediately rather than letting it hide. When I hit
one, my first response isn't to reach for `@Lazy` or switch to setter
injection — both just defer or hide the underlying issue. The real fix is
almost always to recognize that two classes depending on each other
directly usually means they're sharing a responsibility that should be
extracted into a third class that both can depend on unidirectionally. I'd
only use `@Lazy` as a genuinely last-resort, well-understood workaround —
for example, wrapping a legitimately rare, framework-level circular
reference — never as a first response to a design smell."

---

## 3.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Singleton bean needs to track something per-request | Adds a mutable instance field | Uses a local variable, or a request-scoped bean, or thread-safe structures |
| Circular dependency error at startup | Adds `@Lazy` to make it go away | Refactors to extract the shared responsibility into a third bean |
| Needing initialization logic after DI | Puts it directly in the constructor with field injection | Uses `@PostConstruct`, or constructor injection where the logic can safely run in the constructor itself |
| Understanding "how does `@Autowired` work" | Treats it as unexplainable framework magic | Explains it via `BeanPostProcessor` |

---

## 3.5 COMMON MISTAKES

1. Mutable instance state on a singleton bean, causing race conditions
   under concurrent requests.
2. Injecting a prototype bean into a singleton via plain constructor
   injection, unintentionally "freezing" it into singleton-like behavior.
3. Treating `@Lazy` or setter injection as a legitimate fix for circular
   dependencies instead of a symptom-hiding workaround.
4. Doing dependency-requiring initialization work in the constructor while
   using field/setter injection, where dependencies aren't populated yet.
5. Forgetting `@PreDestroy`/`DisposableBean` for resources that need
   explicit cleanup (closing connections, stopping background threads) on
   container shutdown.

---

## 3.6 INTERVIEW QUESTION BANK — CHAPTER 3

**Basic:** 1. What is the default bean scope, and why? 2. What does
`@PostConstruct` guarantee about dependency injection timing?

**Intermediate:** 3. Why is mutable state on a singleton bean dangerous?
4. Explain the correct way to inject a prototype bean into a singleton bean.

**Senior:** 5. Walk through exactly why constructor-injected circular
dependencies fail at startup while field-injected ones don't. 6. What
role does `BeanPostProcessor` play in the bean lifecycle, and name two
Spring features implemented using it.

**Architect:** 7. You're designing a caching layer where cache entries
need per-request isolation in one code path but shared singleton behavior
in another. How would you architect the bean scoping to support both
without introducing subtle bugs?

**Scenario:** 8. A production incident shows intermittent, hard-to-
reproduce incorrect values from a service — the root cause turns out to be
a singleton `@Service` with a mutable `Map` field used as an ad-hoc cache
without synchronization. Explain the failure mode and the fix.

**Trick:** 9. "Since Spring manages the bean, all bean fields are
automatically thread-safe." True or false?

<details><summary>Key answers</summary>

- Q5: Constructor injection requires the complete dependency graph to be
  resolved *before* any object in the cycle can be fully constructed —
  Spring detects it can't finish constructing either bean and fails
  immediately with a clear error. Field/setter injection constructs the
  (still-incomplete) objects first via their no-arg-friendly paths, then
  injects fields afterward — allowing Spring to construct both
  half-finished objects and wire them into each other after the fact,
  silently "working around" what is still a real design smell.
- Q6: `BeanPostProcessor` hooks run before/after a bean's initialization
  callbacks; Spring uses it internally to implement `@Autowired`/`@Value`
  field resolution and, critically, AOP proxy creation — a bean requiring
  a proxy (e.g., for `@Transactional`) actually gets swapped for a proxy
  object in the `postProcessAfterInitialization` step.
- Q7: Use two different beans/scopes for the two needs rather than
  forcing one bean to serve both — a request-scoped bean (or a
  singleton service accepting request-scoped data as method parameters,
  which is often simpler and avoids scope-proxy complexity) for the
  per-request isolation need, and a genuinely stateless or properly
  synchronized singleton for the shared case; conflating the two in one
  bean is exactly how subtle scope-related bugs get introduced.
- Q8: This is precisely the singleton-plus-mutable-state race condition —
  concurrent requests reading/writing the unsynchronized `Map` corrupt its
  internal state or produce stale/incorrect reads under concurrent
  modification (Book 1 Chapter 4's HashMap-under-concurrency material);
  fix by replacing the plain `Map` with a `ConcurrentHashMap` at minimum,
  or better, using a proper caching library (Caffeine) with defined
  eviction semantics instead of an ad-hoc field.
- Q9: False — Spring manages the bean's *lifecycle and wiring*, not
  thread-safety of whatever state you put inside it; a singleton bean's
  mutable fields are exactly as unsafe under concurrent access as any
  other shared mutable object, unless you explicitly make them so (atomics,
  concurrent collections, synchronization).

</details>

---

## 3.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does the default singleton scope make sense for the majority of Spring service beans, and what property must those beans have to make it safe?
- **Coding Check:** Write a singleton `@Service` that safely tracks an in-memory request counter using `AtomicLong`, and explain why the naive `int` version would be a bug in production.
- **Explanation Check:** Explain in English why "Spring will fail fast at startup" for a constructor-injected circular dependency is a feature, not an inconvenience to work around.
- **Real-World Check:** Your team wants to add a per-user "recently viewed items" cache to a singleton `@Service`. Design the correct approach (scope choice and/or data structure) that avoids both cross-user data leakage and thread-safety bugs.
- **Senior Check:** When, if ever, would `@Lazy` on a circular dependency be a legitimate, defensible choice rather than a workaround to avoid?
- **Master Check:** Design a `@PostConstruct`-based warm-up routine for a singleton bean that pre-loads a cache from a database at startup — explain why this logic belongs in `@PostConstruct` rather than the constructor, tying the answer explicitly to the injection style used.

<details><summary>Answers</summary>

- Real-World Check: Do NOT store per-user state as a singleton's instance
  field (this leaks across users/requests, a real security-adjacent bug,
  not just a correctness one) — instead, key the cache by user ID in a
  properly bounded, thread-safe structure (`ConcurrentHashMap` with an
  eviction policy, or better, an external cache like Redis for true
  per-user isolation at scale) so it functions correctly as shared
  infrastructure rather than accidentally-shared per-user state.
- Senior Check: Rarely, and only when the cycle is a genuinely
  unavoidable framework-level interaction (e.g., certain AOP
  self-referencing proxy scenarios) that has been consciously analyzed and
  accepted, documented with a comment explaining why, rather than reached
  for as a quick fix for an ordinary application-level design smell.
- Master Check: With constructor injection, all dependencies are already
  populated by the time the constructor runs, so warm-up logic requiring
  those dependencies is arguably safe directly in the constructor; but
  `@PostConstruct` is still preferred by convention because it clearly
  separates "object construction" from "initialization behavior," keeps
  constructors simple/side-effect-free (matching general OOP best practice
  independent of Spring), and remains correct even if the class were later
  changed to use field/setter injection for some fields.

</details>

---

## 3.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Default scope | Singleton — one shared instance, must be stateless or thread-safe |
| Prototype scope | New instance per request/injection — for mutable, per-use state |
| Injecting prototype into singleton | Use `ObjectProvider<T>` or `@Lookup`, not plain constructor injection |
| `@PostConstruct` | Runs after DI completes — safe place for dependency-requiring init logic |
| `@PreDestroy` | Runs on container shutdown — cleanup resources here |
| `BeanPostProcessor` | The real mechanism behind `@Autowired`, `@Value`, and AOP proxies |
| Circular dependency | A design smell — extract shared logic to a third bean; `@Lazy`/setter injection are workarounds, not fixes |

---

*(Continues to Chapter 4 — Aspect-Oriented Programming (AOP).)*
