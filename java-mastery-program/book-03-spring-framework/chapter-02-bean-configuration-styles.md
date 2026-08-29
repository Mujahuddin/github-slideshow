# CHAPTER 2 — BEAN CONFIGURATION STYLES & THE CONTAINER

---

## 2.1 CONCEPT: Three Ways to Tell Spring About a Bean

### TELUGU EXPLANATION

Spring కి "ఇది ఒక bean, దీన్ని manage చేయి" అని చెప్పడానికి **మూడు
historical styles** ఉన్నాయి:

**1. XML Configuration (legacy, పాత enterprise codebases లో కనిపిస్తుంది):**
```xml
<bean id="orderService" class="com.example.OrderService">
    <constructor-arg ref="emailNotifier"/>
</bean>
```
ఇది verbose, refactoring-unsafe (class name string గా ఉంటుంది, IDE
rename చేస్తే XML break అవుతుంది) — **కొత్త projects లో వాడకూడదు**,
కానీ enterprise interviews లో legacy codebase చదవాల్సి రావొచ్చు
కాబట్టి గుర్తించగలగాలి.

**2. Java Configuration (`@Configuration` + `@Bean`) — explicit, type-safe:**
```java
@Configuration
class AppConfig {
    @Bean
    Notifier emailNotifier() {
        return new EmailNotifier();
    }

    @Bean
    OrderService orderService(Notifier notifier) { // parameter గా ఇచ్చిన bean automatic గా inject అవుతుంది
        return new OrderService(notifier);
    }
}
```
**ఎప్పుడు వాడాలి:** మీరు **మీ own code కాని** classes ని (third-party
libraries, మీరు `@Component` add చేయలేని classes) bean గా register
చేయాల్సి వచ్చినప్పుడు — ఇదే దీని **ప్రధాన, ఇప్పటికీ అవసరమైన use case**.

**3. Component Scanning (`@Component` కుటుంబం + `@ComponentScan`) — ఇప్పటి డిఫాల్ట్:**
```java
@Service // @Component యొక్క ఒక specialization — "ఇది a service-layer bean" అని సూచిస్తుంది
class OrderService {
    private final Notifier notifier;
    OrderService(Notifier notifier) { this.notifier = notifier; } // Spring ఇక్కడ constructor injection చేస్తుంది
}

@Component
class EmailNotifier implements Notifier { /* ... */ }
```
Spring **classpath scan** చేసి, `@Component` (లేదా దాని
specializations) ఉన్న అన్ని classes ని కనుక్కుని, automatic గా beans
గా register చేస్తుంది.

**Stereotype annotations — ఎందుకు `@Service`/`@Repository`/`@Controller`
వేర్వేరు పేర్లతో ఉన్నాయి, అన్నీ `@Component` యొక్క specializations అయినా:**
- **`@Component`:** generic, layer-agnostic.
- **`@Service`:** business/service layer — **semantic clarity** కోసం.
- **`@Repository`:** data access layer — semantic clarity + **ఒక
  ప్రత్యేక technical benefit**: Spring ఈ bean మీద automatic గా
  **exception translation** apply చేస్తుంది (database-specific exceptions
  ని Spring యొక్క unified `DataAccessException` hierarchy గా మారుస్తుంది
  — Book 6/7 లో వివరంగా చూద్దాం).
- **`@Controller`/`@RestController`:** web layer — Spring MVC request
  mapping కోసం ప్రత్యేకంగా గుర్తించబడుతుంది.

### ENGLISH INTERVIEW ANSWER

"Modern Spring code uses component scanning with stereotype annotations as
the default — `@Service`, `@Repository`, `@Controller`, all specializations
of `@Component`. I still reach for explicit `@Configuration`/`@Bean`
methods specifically when I need to register a bean for a class I don't
own — a third-party library class I can't annotate directly — or when
constructing the bean genuinely requires custom logic beyond what
component scanning's implicit constructor-call can express. XML
configuration is something I need to be able to *read*, since a lot of
enterprise Java codebases still have it from before Java Config existed,
but I wouldn't introduce it in new code — it's verbose and not
refactoring-safe, since class references are strings the compiler can't
check. The stereotype annotations aren't purely cosmetic either —
`@Repository` specifically enables Spring's automatic exception
translation, converting a JDBC/JPA-specific exception into Spring's
unified `DataAccessException` hierarchy, which is a real technical
behavior difference, not just a naming convention."

---

## 2.2 CONCEPT: Resolving Ambiguity — `@Qualifier` and `@Primary`

### TELUGU EXPLANATION

రెండు (లేదా ఎక్కువ) beans ఒకే type (interface) implement చేస్తే, Spring
"ఏది inject చేయాలో" **నిర్ణయించలేక** `NoUniqueBeanDefinitionException`
throw చేస్తుంది. దీన్ని పరిష్కరించడానికి రెండు మార్గాలు:

**1. `@Primary`:** ఒక bean ని "default choice" గా mark చేయడం:
```java
@Component
@Primary // ambiguity ఉన్నప్పుడు, దీన్నే default గా ఎంచుకోండి
class EmailNotifier implements Notifier { }

@Component
class SmsNotifier implements Notifier { }
```

**2. `@Qualifier`:** injection point వద్ద, **ఖచ్చితంగా ఏ bean కావాలో
పేరుపెట్టి చెప్పడం** — ఇది `@Primary` కంటే **ఎక్కువ specific**, మరియు
ఒకటి కంటే ఎక్కువ non-default beans వాడాల్సి వస్తే అవసరం:
```java
class OrderService {
    OrderService(@Qualifier("smsNotifier") Notifier notifier) { // పేరుతో ఖచ్చితంగా ఎంచుకోవడం
        this.notifier = notifier;
    }
}
```

**Senior rule:** `@Primary` ఒక్క, **స్పష్టమైన default** ఉన్నప్పుడు
బాగుంటుంది. **బహుళ injection points వేర్వేరు specific
implementations** కావాలంటే, `@Qualifier` వాడటం స్పష్టత ఇస్తుంది.

### ENGLISH INTERVIEW ANSWER

"When multiple beans satisfy the same type, Spring can't guess which one
you want and fails fast with `NoUniqueBeanDefinitionException` rather than
picking arbitrarily — which I consider a feature, not a limitation; silent
arbitrary selection would be a much worse failure mode. `@Primary`
designates one implementation as the sensible default when ambiguity
arises, which works well when there genuinely is one obvious default.
`@Qualifier` is more explicit and precise — I use it at the specific
injection point to name exactly which bean I want, which is necessary the
moment more than one non-default implementation needs to be wired in
different places."

---

## 2.3 CONCEPT: `ApplicationContext` vs `BeanFactory`

### TELUGU EXPలanaTION

`BeanFactory` అనేది Spring IoC container యొక్క **most basic** interface
— **lazy initialization** (bean మొదటిసారి `getBean()` call అయినప్పుడు
మాత్రమే create అవుతుంది). `ApplicationContext` అనేది `BeanFactory` ని
**extend** చేస్తుంది, మరియు enterprise applications కి అవసరమైన extra
features add చేస్తుంది:

- **Eager singleton initialization** (default గా — startup వద్దే అన్ని
  singleton beans create అవుతాయి, "fail fast" behavior కోసం — ఒక bean
  configuration తప్పుగా ఉంటే, startup వద్దే తెలిసిపోతుంది, మొదటి
  request వచ్చినప్పుడు కాదు).
- **Event publication** (`ApplicationEventPublisher`, Chapter 5 చూడండి).
- **Internationalization (i18n) support**.
- **AOP integration** ఇంకా సులభంగా.

**Practical గమనిక:** దాదాపు ప్రతి real Spring application
`ApplicationContext` వాడుతుంది (Spring Boot దీన్నే internally వాడుతుంది)
— `BeanFactory` ని నేరుగా వాడటం చాలా అరుదు, కానీ ఈ తేడా ఇంటర్వ్యూలో
అడగడం classic.

### ENGLISH INTERVIEW ANSWER

"`BeanFactory` is the root container interface, lazy by default — beans
are created only when first requested. `ApplicationContext` extends it
with enterprise features: event publishing, internationalization, easier
AOP integration, and critically, eager initialization of singleton beans
at startup by default. That eager initialization is actually a deliberate
'fail fast' design choice — if a bean's configuration is broken, I want
the application to fail at startup, loudly, not on the first user request
that happens to touch that bean. In practice, essentially all real
applications — including everything Spring Boot auto-configures — use
`ApplicationContext`; `BeanFactory` is mostly discussed for interview
completeness and historical/architectural understanding rather than direct
day-to-day use."

---

## 2.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Registering a third-party class as a bean | Doesn't know how, since `@Component` can't be added to it | Writes an explicit `@Bean` method in a `@Configuration` class |
| Multiple implementations of an interface | Confused by `NoUniqueBeanDefinitionException`, adds `@Primary` to all of them (doesn't fix it) | Uses `@Primary` for the sensible default, `@Qualifier` for explicit named injection elsewhere |
| Bean startup failure timing | Assumes any misconfiguration will show up "eventually" | Understands `ApplicationContext`'s eager singleton init means broken config fails loudly at startup |
| Reading legacy XML config | Can't parse it, blocked | Recognizes it as an older config style, translates it mentally to the Java Config equivalent |

---

## 2.5 COMMON MISTAKES

1. Adding `@Primary` to every implementation to "fix" ambiguity errors —
   this defeats the purpose; only one implementation should be primary.
2. Forgetting that `@Repository`'s exception translation is a real
   technical feature, not just a naming convention — using `@Component`
   instead loses this behavior silently.
3. Writing a `@Bean` method for a class you actually own and could
   annotate directly with `@Component` — unnecessary indirection when
   component scanning would suffice.
4. Assuming `ApplicationContext` beans are lazy by default like
   `BeanFactory` — singleton beans are eager by default in
   `ApplicationContext`, a source of real startup-time surprises if not understood.

---

## 2.6 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. What's the difference between `@Component`, `@Service`, and
`@Repository`? 2. When would you use `@Bean` instead of `@Component`?

**Intermediate:** 3. How does `@Qualifier` differ from `@Primary`, and
when would you need both in the same application? 4. What technical
(not just semantic) benefit does `@Repository` provide?

**Senior:** 5. Why does `ApplicationContext` eagerly initialize singleton
beans by default, and why is this considered a feature rather than a
performance cost? 6. You need to conditionally register a bean only when
a certain property is set. How would you achieve this (hint: think ahead
to Spring Boot's `@ConditionalOnProperty`, but describe the underlying
Java Config mechanism first)?

**Architect:** 7. You're migrating a large legacy application from XML
configuration to Java Config/component scanning incrementally, without a
big-bang rewrite. What migration strategy would you use, and how do XML
and annotation-based configuration coexist during the transition?

**Scenario:** 8. A new bean throws `NoUniqueBeanDefinitionException` at
startup after a teammate added a second implementation of an existing
interface. What are the two valid fixes, and how do you decide which one
to apply?

**Trick:** 9. "Component scanning is strictly better than Java
Configuration and should always be preferred." True or false?

<details><summary>Key answers</summary>

- Q5: Because startup-time failure is far cheaper and easier to diagnose
  than a failure surfacing on a random production request hours or days
  after deployment — eager initialization trades a slightly slower startup
  for the guarantee that a broken configuration is caught immediately, in
  a controlled context (deployment/startup), not in front of a real user.
- Q6: At the plain Java Config level, this can be done with an `@Bean`
  method guarded by a conditional check (e.g., reading an environment
  property directly and returning `null`/throwing, or more idiomatically,
  Spring's `@Conditional` annotation with a custom `Condition`
  implementation) — Spring Boot's `@ConditionalOnProperty` (Book 4) is
  built on exactly this underlying `@Conditional` mechanism.
- Q7: XML and annotation-based configuration can coexist — `@ImportResource`
  lets a Java `@Configuration` class import an existing XML file, allowing
  incremental migration module-by-module rather than all at once; convert
  and remove XML files as their owning modules are touched for other
  reasons, verifying behavior with tests at each step.
- Q8: Fix 1: add `@Primary` to whichever implementation should be the
  default when no other qualifier is specified. Fix 2: add `@Qualifier`
  at every injection point that needs a specific one. Decision: if there's
  a clear "usual" choice and only occasional exceptions, use `@Primary` +
  `@Qualifier` only at the exception points; if there's no natural default
  and every injection point genuinely needs to be explicit, use
  `@Qualifier` everywhere and skip `@Primary` entirely.
- Q9: False — component scanning can't register classes you don't own
  (no source to annotate) and can't express custom construction logic;
  `@Bean` methods remain necessary and are not inferior, just suited to a
  different, still-common situation.

</details>

---

## 2.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why can't you use `@Component` to register a bean for a class from an external JAR you don't control the source of?
- **Coding Check:** Write a `@Configuration` class registering two `DataSource`-like beans (mocked/simplified) with `@Primary` on one and `@Qualifier`-based injection for the other into a consuming class.
- **Explanation Check:** Explain in English why `@Repository`'s exception translation matters for a service layer that needs to react differently to "duplicate key" vs "connection failure" errors.
- **Real-World Check:** Your team's startup time has grown as the bean count increased, and someone proposes making beans lazy by default (`@Lazy` globally) to speed up startup. What trade-off are they introducing, tying back to section 2.3?
- **Senior Check:** When would you deliberately reach for XML configuration in a brand-new project in 2024+ (if ever)?
- **Master Check:** Design a small plugin-style system using `@Qualifier`-annotated beans and a `Map<String, Notifier>` injection (Spring supports collecting all beans of a type into a Map keyed by bean name) to dynamically dispatch to a notifier by a runtime string key (e.g., from a database column) — explain why this is more flexible than a hardcoded `@Qualifier` at a single injection point.

<details><summary>Answers</summary>

- Real-World Check: Making beans lazy by default trades startup speed for
  losing the "fail fast" guarantee from section 2.3 — a broken bean's
  misconfiguration would now only surface when something first actually
  requests it, potentially in production under real user load, rather than
  at a controlled startup moment; the trade-off should be made
  deliberately and narrowly (lazy-loading specific expensive, rarely-used
  beans) rather than applied globally.
- Senior Check: Essentially never for genuinely new code — the one
  legitimate case is integrating with an old library or tool that only
  ships XML-based Spring configuration samples/support, where importing
  that XML via `@ImportResource` is more practical than hand-translating
  it.
- Master Check: `@Autowired Map<String, Notifier> notifiersByBeanName`
  lets Spring inject every `Notifier` bean, keyed by bean name,
  automatically — the dispatch logic (`notifiersByBeanName.get(channelKey)`)
  can then select at runtime based on external data (e.g., a user's
  preferred channel stored in a database), rather than being fixed at
  compile time by a single `@Qualifier` annotation, which is essential when
  the specific implementation to use isn't known until runtime.

</details>

---

## 2.8 CHEAT SHEET

| Need | Mechanism |
|---|---|
| Register your own class as a bean | `@Component` (or a stereotype: `@Service`/`@Repository`/`@Controller`) |
| Register a third-party/externally-owned class | `@Bean` method in a `@Configuration` class |
| Data-access layer bean, want exception translation | `@Repository` specifically |
| Pick a sensible default among multiple implementations | `@Primary` |
| Pick a specific implementation at an injection point | `@Qualifier("beanName")` |
| Inject all beans of a type, keyed by name | `Map<String, T>` injection |
| Legacy config style, read-only in modern work | XML `<bean>` definitions |

---

*(Continues to Chapter 3 — Bean Scopes & Lifecycle.)*
