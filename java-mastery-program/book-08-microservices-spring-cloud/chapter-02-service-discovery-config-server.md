# CHAPTER 2 — SERVICE DISCOVERY & CONFIG SERVER

---

## 2.1 CONCEPT: The Problem Service Discovery Solves

### TELUGU EXPLANATION

Monolith లో, ఒక module మరో module ని **direct method call** తో access
చేస్తుంది — location గురించి ఆలోచించాల్సిన అవసరం లేదు. Microservices
లో, `OrderService`, `PaymentService` ని call చేయాలంటే, దాని **network
address** (IP + port) తెలియాలి — కానీ ఈ address **స్థిరంగా ఉండదు**:

- **Cloud/container environments** లో (Book 12 Kubernetes), services
  **dynamically scale అవుతాయి** — కొత్త instances రావొచ్చు, పాతవి
  పోవొచ్చు, ప్రతిదానికి **కొత్త IP**.
- **Hardcoding IPs** ఒక్క configuration file లో — ప్రతి scale event
  కి config మార్చాల్సి వస్తుంది, ఇది **manual, error-prone, ఏ scale
  కి సరిపోదు**.

**Service Discovery** ఈ సమస్యని పరిష్కరిస్తుంది: ప్రతి service instance,
startup అయినప్పుడు తనని తాను ఒక **registry** లో **register** చేసుకుంటుంది
(name + current address తో). ఇతర services, ఆ registry ని **query**
చేసి, "OrderService యొక్క ప్రస్తుత instances ఏమిటి?" అని కనుక్కుంటాయి
— hardcoded IPs బదులు, **logical service name** వాడతాయి.

### ENGLISH INTERVIEW ANSWER

"In a monolith, one module calling another is just a method call — no
location awareness needed. In microservices, calling another service
means knowing its network address, but that address isn't stable —
container orchestration environments constantly scale instances up and
down, each getting a new IP. Hardcoding addresses in configuration would
require a manual update on every scaling event, which doesn't work at
any real scale. Service discovery solves this: each instance registers
itself with a registry on startup, announcing its name and current
address, and other services query the registry by logical service name
rather than a hardcoded address — the registry always reflects the
current, actual set of healthy instances."

---

## 2.2 CONCEPT: Client-Side vs Server-Side Discovery, and Eureka Mechanics

### TELUGU EXPLANATION

- **Client-Side Discovery** (Netflix Eureka + Spring Cloud LoadBalancer
  యొక్క పద్ధతి): Calling service (client) **స్వయంగా** registry ని
  query చేసి, **తనే** load balancing decision తీసుకుంటుంది (ఏ instance
  కి call చేయాలో).
- **Server-Side Discovery** (Kubernetes native, Book 12): ఒక **infrastructure-level
  component** (Kubernetes Service, DNS-based) discovery + load balancing
  రెండింటినీ handle చేస్తుంది — calling service కి ఇది **పూర్తిగా
  invisible** — అది ఒక్క stable DNS name కి call చేస్తుంది.

**Eureka ఎలా పని చేస్తుంది (client-side):**
1. **Registration:** Service startup అయినప్పుడు, Eureka Server కి తన
   metadata (name, IP, port) పంపుతుంది.
2. **Heartbeat:** ప్రతి 30 seconds కి (default), "నేను ఇంకా alive గా
   ఉన్నాను" అని Eureka కి signal పంపుతుంది.
3. **Eviction:** ఒక instance, కొన్ని heartbeats miss అయితే (default
   90 seconds), Eureka దాన్ని registry నుండి **తీసేస్తుంది**.
4. **Self-Preservation Mode:** ఒక subtle, senior-level topic — network
   partition వల్ల, చాలా instances **ఏకకాలంలో** heartbeat miss అయితే,
   Eureka అనుమానిస్తుంది "ఇది network సమస్యే, అన్ని instances నిజంగా
   చనిపోలేదు" అని, మరియు **eviction ని temporarily ఆపేస్తుంది** —
   ఇది false positives (healthy instances ని పొరపాటున తీసేయడం) ని
   నివారిస్తుంది, కానీ **genuinely dead instances కూడా registry లో
   ఉండిపోయే risk** తీసుకుంటుంది — ఇది **availability vs accuracy**
   మధ్య ఒక deliberate trade-off.

**Senior గమనిక — పరిశ్రమ dynamics:** Kubernetes-native environments లో,
Kubernetes యొక్క own DNS-based service discovery (Book 12) **తరచుగా
Eureka అవసరాన్ని తీసేస్తుంది** — ఇది infrastructure layer లోనే
built-in గా ఉంటుంది. Eureka ఇప్పటికీ non-Kubernetes deployments కి,
లేదా Kubernetes కి ముందు design చేయబడిన systems కి relevant.

### ENGLISH INTERVIEW ANSWER

"Client-side discovery, Eureka's model, has the calling service query the
registry directly and make its own load-balancing decision among the
returned instances. Server-side discovery, Kubernetes' native model,
handles both discovery and load balancing at the infrastructure level —
the calling service just hits a stable DNS name and never sees the
underlying instance addresses at all. Eureka's mechanics: instances
register on startup, send heartbeats roughly every 30 seconds, and get
evicted from the registry after missing heartbeats for about 90 seconds.
The one genuinely subtle piece worth knowing is self-preservation mode —
if Eureka sees an unusually large fraction of instances missing
heartbeats simultaneously, it suspects a network partition rather than
mass instance death, and temporarily stops evicting anyone, deliberately
trading some registry accuracy (potentially keeping genuinely-dead
instances listed a bit longer) for avoiding a much worse outcome: mass
incorrect eviction during a transient network blip. I'd also note that in
Kubernetes-native environments, Kubernetes' own DNS-based service
discovery often makes Eureka unnecessary — Eureka remains most relevant
for non-Kubernetes deployments or systems architected before Kubernetes
became the default."

---

## 2.3 CONCEPT: Spring Cloud Config Server — Centralized, Externalized Configuration at Scale

### TELUGU EXPLANATION

Book 4 Chapter 2 లో మనం `application.yml` + environment variables
చూశాము — ఇది **ఒక్క service** కి బాగుంటుంది. **50 microservices**
ఉంటే, ప్రతిదానికి విడిగా config manage చేయడం (ముఖ్యంగా shared
settings — DB connection patterns, common feature flags) **repetitive,
inconsistent** అవుతుంది.

**Spring Cloud Config Server:** ఒక **centralized config repository**
(సాధారణంగా Git-backed) — అన్ని services, startup అయినప్పుడు, config
server నుండి తమ config ని fetch చేస్తాయి:

```yaml
# Config Server లో: application.yml (అన్ని services కి common)
# order-service.yml (Order Service కి మాత్రమే specific)
```

```properties
# ప్రతి client service లో (bootstrap.yml / application.yml)
spring.config.import=configserver:http://config-server:8888
spring.application.name=order-service # ఇది ఏ config file వాడాలో నిర్ణయిస్తుంది
```

**Dynamic Refresh — production లో genuinely useful feature:**
`@RefreshScope` annotation తో mark చేసిన beans, config మారినప్పుడు,
**service ని restart చేయకుండానే** కొత్త values తీసుకుంటాయి —
`/actuator/refresh` endpoint call చేయడం ద్వారా (లేదా, బహుళ services
కి ఏకకాలంలో, **Spring Cloud Bus** వాడి, ఒక message broker ద్వారా
broadcast చేయడం ద్వారా).

### ENGLISH INTERVIEW ANSWER

"With dozens of microservices, managing configuration per-service
independently becomes repetitive and inconsistent, especially for
settings shared across many services. Spring Cloud Config Server
centralizes configuration in one place — typically a Git repository —
and every service fetches its configuration from it at startup, using
its `spring.application.name` to determine which config file applies to
it, layered with shared common configuration. The genuinely useful
production feature here is dynamic refresh: beans annotated
`@RefreshScope` can pick up configuration changes without a restart,
triggered via the `/actuator/refresh` endpoint, or broadcast to many
service instances simultaneously via Spring Cloud Bus over a message
broker — letting an operational team change a feature flag or a
threshold across a whole fleet of instances without a redeploy, which is
a meaningfully different operational capability than Book 4's
static-at-startup configuration model."

---

## 2.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Calling another service | Hardcodes its IP/hostname in config | Uses service discovery (or Kubernetes DNS) with a logical service name |
| Managing config for 30 services | Duplicates common settings across each service's own config | Centralizes shared config via a Config Server |
| Needing a config change live | Redeploys every affected service | Uses `@RefreshScope` + `/actuator/refresh` (or Spring Cloud Bus) |
| Choosing a discovery mechanism | Defaults to Eureka regardless of environment | Recognizes Kubernetes-native DNS discovery may already solve this |

---

## 2.5 COMMON MISTAKES

1. Hardcoding service addresses instead of using discovery, breaking
   under any scaling event.
2. Introducing Eureka in a Kubernetes environment without recognizing
   Kubernetes' own discovery might already suffice, adding unnecessary complexity.
3. Duplicating shared configuration across many services instead of
   centralizing it in a Config Server.
4. Not understanding Eureka's self-preservation mode, and being confused
   when "dead" instances remain in the registry during a network issue.
5. Forgetting `@RefreshScope` on a bean that needs to pick up dynamic
   config changes, then being confused why a config update via
   `/actuator/refresh` has no effect.

---

## 2.6 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. What problem does service discovery solve? 2. What is
Spring Cloud Config Server used for?

**Intermediate:** 3. Explain the difference between client-side and
server-side discovery. 4. What does `@RefreshScope` do?

**Senior:** 5. Explain Eureka's self-preservation mode and the trade-off
it represents. 6. Design the config strategy for 20 microservices with
both shared settings (logging format, common timeouts) and
service-specific settings.

**Architect:** 7. Your organization is migrating from a VM-based
deployment (using Eureka) to Kubernetes. What changes regarding service
discovery, and what would you need to verify before removing Eureka
entirely?

**Scenario:** 8. During a network partition affecting part of your
service fleet, Eureka's registry shows all instances as healthy, but
several are actually unreachable. Explain why, referencing this
chapter's material.

**Trick:** 9. "Service discovery eliminates the need for load balancing." True or false?

<details><summary>Key answers</summary>

- Q6: A base `application.yml` in the Config Server repository holds
  shared settings (logging format, common timeout defaults, shared
  feature flags); each service has its own `{service-name}.yml`
  overlaying only its service-specific settings (database connection
  details, service-specific feature flags) — mirroring exactly Book 4
  Chapter 2's profile-overlay pattern, just centralized across services
  instead of per-service profiles.
- Q7: Kubernetes' native DNS-based service discovery and its Service/
  Endpoint objects replace Eureka's registration/heartbeat/eviction
  mechanism entirely — services call each other via stable Kubernetes
  Service DNS names instead of querying a Eureka registry. Before
  removing Eureka, verify no remaining code paths directly depend on
  Eureka-specific APIs/metadata, and that load balancing (now handled by
  Kubernetes Services) behaves correctly for all existing traffic patterns.
- Q8: This is self-preservation mode in effect — Eureka detected an
  unusually high proportion of missed heartbeats (consistent with a
  network partition) and deliberately stopped evicting instances to
  avoid mass false-positive removal, at the cost of the registry
  temporarily showing genuinely-unreachable instances as still healthy —
  exactly the availability-over-accuracy trade-off from section 2.2.
- Q9: False — service discovery answers "what are the current healthy
  instances of this service," which is a prerequisite for load
  balancing, but a separate decision (round-robin, least-connections, etc.)
  still needs to be made about which of the returned instances to
  actually route a given request to; discovery and load balancing are
  complementary, not substitutes for each other.

</details>

---

## 2.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why can't microservices in a dynamically-scaling environment rely on hardcoded IP addresses?
- **Coding Check:** Configure a Spring Boot service to register with Eureka and another service to discover and call it via a logical service name (using Spring Cloud LoadBalancer).
- **Explanation Check:** Explain in English why Eureka's self-preservation mode is a deliberate design trade-off, not a bug.
- **Real-World Check:** Your organization runs 40 services with substantial configuration duplication across them, and a recent incident occurred because one service's logging config drifted from the standard. Design the fix.
- **Senior Check:** When would you still choose Eureka-style client-side discovery even in a Kubernetes environment?
- **Master Check:** Design the complete configuration and discovery architecture for a system migrating from 5 to 50 microservices over the next year — what would you put in place now to avoid configuration drift and discovery bottlenecks as the fleet grows?

<details><summary>Answers</summary>

- Real-World Check: Centralize configuration via a Spring Cloud Config
  Server (or equivalent) with shared base configuration for
  cross-cutting settings like logging format, eliminating the
  possibility of per-service drift on settings that should be uniform —
  the specific incident (logging config drift) is a direct, textbook
  case for centralized shared configuration.
- Senior Check: When you need client-side load-balancing logic more
  sophisticated or customizable than what Kubernetes Services provide
  out of the box, or when running a hybrid environment where not
  everything is Kubernetes-native yet — a genuine but increasingly rare case.
- Master Check: Establish a Config Server with clear shared-vs-specific
  configuration conventions from the start (before the fleet grows,
  since retrofitting this discipline onto 50 already-drifted services is
  much harder); adopt Kubernetes-native service discovery early if
  targeting Kubernetes, avoiding Eureka's additional operational
  component entirely; establish `@RefreshScope`/Spring Cloud Bus
  conventions for dynamic config changes so operational teams have a
  consistent mechanism across all services as the fleet grows, rather
  than each team improvising its own approach.

</details>

---

## 2.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Service discovery | Solves dynamic addressing — services register, callers query by logical name |
| Client-side discovery (Eureka) | Calling service queries registry, makes its own load-balancing decision |
| Server-side discovery (Kubernetes) | Infrastructure handles discovery + load balancing — invisible to the caller |
| Eureka self-preservation | Stops evicting during suspected network partitions — availability over accuracy |
| Config Server | Centralized, Git-backed config — shared + per-service overlay |
| `@RefreshScope` | Enables live config updates without a restart, via `/actuator/refresh` or Spring Cloud Bus |
| Kubernetes environments | Native DNS discovery often makes Eureka unnecessary |

---

*(Continues to Chapter 3 — Inter-Service Communication & Feign.)*
