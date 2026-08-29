# CHAPTER 1 — IOC CONTAINER & DEPENDENCY INJECTION

---

## 1.1 CONCEPT: The Problem Before Spring — Tight Coupling

### TELUGU EXPLANATION

Spring అర్థం చేసుకోవడానికి, ముందు Spring **లేకుండా** code ఎలా ఉంటుందో
చూద్దాం:

```java
class OrderService {
    private final EmailNotifier notifier = new EmailNotifier(); // ❌ tight coupling

    void placeOrder(Order order) {
        // ... order logic ...
        notifier.send(order.getCustomerEmail(), "Order placed!");
    }
}
```

ఇక్కడ సమస్య ఏమిటంటే, `OrderService` **స్వయంగా** `EmailNotifier` ని
create చేసుకుంటుంది (`new EmailNotifier()`). ఇది Book 1 Chapter 2 లో
మనం నేర్చుకున్న **Dependency Inversion Principle (DIP)** ని ఉల్లంఘిస్తుంది
— `OrderService` (high-level module) `EmailNotifier` (low-level, concrete
class) మీద **direct గా depend** అవుతుంది.

**దీనివల్ల వచ్చే నిజమైన సమస్యలు:**
1. **Testability:** `OrderService` ని unit test చేయాలంటే, real
   `EmailNotifier` (బహుశా నిజంగా email పంపేది!) ఉపయోగించాల్సి వస్తుంది —
   దాన్ని mock చేయలేరు, ఎందుకంటే అది hard-coded గా create అయ్యింది.
2. **Flexibility:** రేపు `SmsNotifier` కి మారాలంటే, `OrderService` code
   నే మార్చాలి — closed for modification (OCP violation, Book 1 Chapter 2).
3. **Configuration:** ఒకే `OrderService` ని వేర్వేరు environments లో
   (dev: fake notifier, prod: real notifier) వేర్వేరుగా configure
   చేయాలంటే, code changes అవసరం.

### ENGLISH INTERVIEW ANSWER

"Before understanding Spring, it's worth seeing the exact problem it
solves: when a class directly instantiates its own dependencies with
`new`, it becomes tightly coupled to that concrete implementation, which
directly violates the Dependency Inversion Principle from SOLID. This
manifests as three concrete pains: you can't substitute a test double
without changing the class's code, you can't swap implementations without
modifying and recompiling the dependent class, and you can't configure
different behavior per environment declaratively. Spring's IoC container
exists specifically to solve this — it takes over the responsibility of
constructing objects and wiring their dependencies together, so classes
only need to declare *what* they depend on (usually via an interface),
never *how* to construct it."

---

## 1.2 CONCEPT: Inversion of Control (IoC) — The "Inversion"

### TELUGU EXPLANATION

**Inversion of Control** అంటే ఏమిటి, ఖచ్చితంగా "ఏది invert అవుతుంది"?

**సాధారణ (non-IoC) code లో:** మీ class **control** తీసుకుంటుంది —
ఎప్పుడు, ఎలా dependencies create చేయాలో అది స్వయంగా నిర్ణయిస్తుంది
(`new EmailNotifier()`).

**IoC తో:** ఈ control **invert** (తిరగబడుతుంది) — ఇప్పుడు ఒక **external
container** (Spring IoC Container) objects create చేసి, వాటి
dependencies ని wire చేసి, మీ class కి **అందిస్తుంది**. మీ class ఇక
"నేను నా dependency ని ఎలా పొందుతాను" అని ఆలోచించదు — అది కేవలం "నాకు
ఇది కావాలి" అని declare చేస్తుంది (సాధారణంగా constructor parameter
ద్వారా), container దాన్ని provide చేస్తుంది.

ఇదే **"Hollywood Principle"** అని కూడా అంటారు: **"Don't call us, we'll
call you."** — మీ code control తీసుకోదు, framework మీ code ని అవసరమైనప్పుడు
call చేస్తుంది/దానికి అవసరమైనవి అందిస్తుంది.

**Dependency Injection (DI)** అనేది IoC ని achieve చేయడానికి **ఒక
specific technique** (మరొకటి: Service Locator pattern, ఇది తక్కువ
popular, ఎందుకంటే ఇది dependency ని hide చేస్తుంది, explicit గా
express చేయదు).

```java
// ✅ Dependency Injection — control is INVERTED
class OrderService {
    private final Notifier notifier; // interface మీద depend అవుతుంది, concrete class మీద కాదు

    // Constructor injection — Spring ఈ constructor ని చూసి, ఏదైనా Notifier bean ని ఇక్కడ inject చేస్తుంది
    OrderService(Notifier notifier) {
        this.notifier = notifier;
    }

    void placeOrder(Order order) {
        notifier.send(order.getCustomerEmail(), "Order placed!");
    }
}
```

### ENGLISH INTERVIEW ANSWER

"Inversion of Control means the responsibility for creating and wiring
objects moves from the object itself to an external container — often
summarized as the Hollywood Principle, 'don't call us, we'll call you.'
Dependency Injection is the specific technique Spring uses to achieve
IoC: instead of a class constructing its own dependencies, it declares
what it needs — typically via constructor parameters typed to an
interface — and the container supplies a concrete implementation at
runtime. This is exactly Book 1's Dependency Inversion Principle put into
practice mechanically: `OrderService` now depends only on the `Notifier`
interface, and Spring decides, based on configuration, which concrete
`Notifier` bean to hand it — `EmailNotifier` in production,
a mock in a test, without any change to `OrderService` itself."

---

## 1.3 CONCEPT: Constructor vs Setter vs Field Injection

### TELUGU EXPLANATION

Spring మూడు రకాల injection support చేస్తుంది. **ఏది ఎప్పుడు వాడాలో
అర్థం చేసుకోవడం** senior-level knowledge:

**1. Constructor Injection (బలంగా recommended, default choice):**
```java
@Service
class OrderService {
    private final Notifier notifier;
    private final PaymentGateway paymentGateway;

    OrderService(Notifier notifier, PaymentGateway paymentGateway) { // Spring 4.3+ లో @Autowired కూడా అవసరం లేదు, ఒకే constructor ఉంటే
        this.notifier = notifier;
        this.paymentGateway = paymentGateway;
    }
}
```
**ఎందుకు ఇది best:**
- Fields ని **`final`** చేయవచ్చు — Book 1 Chapter 3 immutability
  సూత్రం, thread-safety + "object ఎప్పుడూ valid state లో ఉంటుంది"
  అనే guarantee.
- **Mandatory dependencies స్పష్టంగా కనిపిస్తాయి** — constructor
  signature చూస్తే, class కి ఏం కావాలో తెలిసిపోతుంది.
- **Testability** — plain Java `new OrderService(mockNotifier,
  mockGateway)` తో, Spring container అవసరం లేకుండా, unit test రాయవచ్చు.
- **Circular dependency ని compile/startup time లోనే catch చేస్తుంది**
  (section 3.5 లో వివరంగా చూద్దాం) — setter injection దీన్ని silently
  (with a workaround) permit చేస్తుంది, ఇది నిజానికి ఒక **design smell**
  ని దాచేస్తుంది.

**2. Setter Injection (optional dependencies కి మాత్రమే):**
```java
@Autowired(required = false)
void setAuditLogger(AuditLogger auditLogger) { // optional — లేకపోయినా app పని చేస్తుంది
    this.auditLogger = auditLogger;
}
```

**3. Field Injection (❌ discouraged, కానీ చాలా tutorials/legacy code లో కనిపిస్తుంది):**
```java
@Autowired
private Notifier notifier; // ❌ ఎందుకు discouraged
```
**ఎందుకు avoid చేయాలి:**
- Field `final` కాదు — mutable, reflection ద్వారా ఎప్పుడైనా మారవచ్చు.
- **Plain `new`** తో test చేయలేరు — Spring reflection వాడి field ని
  set చేస్తుంది, కాబట్టి unit test కి Spring context (లేదా Mockito's
  `@InjectMocks`, ఇది reflection మీదే ఆధారపడుతుంది) అవసరం.
- Dependencies **hidden** గా ఉంటాయి — class ని చూసి, అది ఏం మీద depend
  అవుతుందో వెంటనే తెలియదు.

### ENGLISH INTERVIEW ANSWER

"I default to constructor injection for all mandatory dependencies, and
I'd flag field injection in any code review. Constructor injection lets me
make fields `final`, which gives both thread-safety and a guarantee that
the object is never in a partially-constructed, invalid state. It also
makes dependencies visible in the class's public contract — the
constructor signature itself documents what's required — and it makes the
class trivially testable with plain `new`, no Spring container or
reflection tricks needed. Field injection hides dependencies, prevents
`final`, and couples your tests to Spring's reflection machinery. Setter
injection has a legitimate, narrow use case: genuinely optional
dependencies that the class can function without — but if I find myself
reaching for setter injection for a dependency the class actually
requires, that's usually a sign the dependency should be in the
constructor instead."

**Interviewer follow-up:** "If constructor injection is best, why does
field injection still appear so often, including in tutorials?" — It's
visually the shortest to write and looks clean in slide/demo code, but
that brevity trades away testability and immutability — exactly the kind
of "looks simpler, is actually worse" trap a senior engineer should
recognize and correct rather than copy.

---

## 1.4 CONCEPT: The IoC Container Itself — `ApplicationContext`

### TELUGU EXPLANATION

**`ApplicationContext`** అనేది Spring యొక్క IoC container యొక్క
నిజమైన implementation (interface, వేర్వేరు implementations ఉన్నాయి —
`AnnotationConfigApplicationContext`, `ClassPathXmlApplicationContext`
మొదలైనవి). ఇది చేసే పని:

1. **Bean definitions చదవడం** (annotations, Java Config, లేదా XML నుండి).
2. **Beans ని instantiate చేయడం** (constructors call చేయడం).
3. **Dependencies ని wire చేయడం** (constructor/setter/field injection).
4. **Bean lifecycle ని manage చేయడం** (Chapter 3 చూడండి).
5. Application కి **beans ని అందించడం** (`getBean()`, లేదా, సాధారణంగా,
  అవి automatic గా other beans లోకి inject అవుతాయి).

**"Bean" అనే పదం అర్థం:** ఏదైనా object ని Spring IoC container create
చేసి, manage చేస్తే, దాన్ని **"bean"** అంటారు — ఇది కేవలం ఒక
terminology, ప్రత్యేకమైన class type కాదు.

### ENGLISH INTERVIEW ANSWER

"`ApplicationContext` is the concrete IoC container — it reads bean
definitions from wherever they're declared (annotations with component
scanning, Java `@Configuration` classes, or legacy XML), instantiates
those beans, resolves and injects their dependencies, manages their full
lifecycle, and makes them available to the rest of the application. A
'bean' is simply the term for any object whose lifecycle Spring manages —
it's not a special type, just a role a plain Java object plays once the
container is responsible for creating and wiring it."

---

## 1.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Adding a new dependency to a class | Field injection, `@Autowired private X x;` | Constructor injection, `final` field |
| Class needs an optional dependency | Makes it mandatory via constructor anyway | Setter injection with `required = false`, or an `Optional<X>` constructor parameter |
| "Why does Spring exist" | "It manages beans" | Explains the concrete tight-coupling/testability/DIP problem it solves |
| Testing a service with dependencies | Spins up a full Spring context for a unit test | Constructs the class directly with mocked dependencies via constructor injection — no Spring needed |

---

## 1.6 COMMON MISTAKES

1. Using field injection by habit/tutorial-copying instead of constructor injection.
2. Making a mandatory dependency a setter-injected, nullable field —
   allowing the object to exist in a half-configured, invalid state.
3. Depending on a concrete class instead of an interface, silently
   reintroducing the tight coupling DI was meant to eliminate.
4. Believing "using `@Autowired` = using DI correctly" — DI is a design
   principle; `@Autowired` is just Spring's mechanism, and it can still be
   used to build a tightly-coupled, hard-to-test design if dependencies
   are on concrete classes or injected as mutable fields.
5. Confusing "Spring manages this object" with "this object needs to be a
   POJO with getters/setters" — modern constructor-injected, immutable
   beans are the idiomatic style, not JavaBean-style mutable objects.

---

## 1.7 INTERVIEW QUESTION BANK — CHAPTER 1

**Basic:** 1. What problem does Dependency Injection solve? 2. What is a
"bean" in Spring?

**Intermediate:** 3. Compare constructor, setter, and field injection —
when would you use each? 4. What is the "Hollywood Principle" and how does
it relate to IoC?

**Senior:** 5. Why does constructor injection make circular dependencies
easier to catch than field injection? 6. Explain, with a code example, why
field-injected classes are harder to unit test.

**Architect:** 7. You're designing a plugin architecture where third-party
modules provide implementations of your interfaces, loaded and wired at
runtime. How do IoC/DI principles apply beyond a single Spring
application context — what challenges arise (classloading, versioning)?

**Scenario:** 8. A code reviewer rejects a PR because a new `@Service`
class uses `@Autowired` on private fields directly. The author argues
"it's shorter and it works." How do you explain the actual engineering
cost, not just "it's a style rule"?

**Trick:** 9. "Using `@Autowired` anywhere in your code automatically
means you're following the Dependency Inversion Principle." True or false?

<details><summary>Key answers</summary>

- Q5: Constructor injection requires ALL dependencies to be resolved
  before the object can be constructed at all — if beans A and B depend on
  each other via constructors, Spring cannot construct either first,
  and fails fast at startup with a clear circular-dependency error. Field/
  setter injection allows Spring to construct the (empty) object first and
  inject fields afterward, which can silently "work around" a circular
  dependency by injecting a partially-initialized bean — hiding what is
  usually still a real design problem.
- Q6: A field-injected class like `class OrderService { @Autowired private
  Notifier notifier; }` has no public constructor accepting a `Notifier` —
  a plain unit test can't do `new OrderService(mockNotifier)`; it must
  either stand up a Spring test context or use reflection-based tools
  (like Mockito's `@InjectMocks`) to force a value into the private field,
  both heavier and more indirect than simply calling a constructor.
- Q9: False — `@Autowired` is a mechanism, not a guarantee of good design.
  If the field/constructor parameter is typed to a *concrete class*
  instead of an interface/abstraction, you still have a direct dependency
  on a low-level implementation detail, which is exactly what DIP argues
  against, regardless of whether Spring is "managing" the wiring.

</details>

---

## 1.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why can constructor-injected fields be `final`, but field-injected ones cannot?
- **Coding Check:** Refactor a field-injected `@Service` class (with 3 dependencies) into constructor injection, and write a plain JUnit test for it with zero Spring annotations.
- **Explanation Check:** Explain in English, to someone who has only used `new` to create objects, what "control" is being "inverted" in Inversion of Control.
- **Real-World Check:** Your team's codebase has 200 classes using field injection. Leadership asks whether migrating to constructor injection is worth the effort. Make the business case, not just the "best practice" case.
- **Senior Check:** When, if ever, is field injection an acceptable trade-off in production code?
- **Master Check:** Design a class with one mandatory dependency (constructor-injected) and one genuinely optional dependency (setter-injected), and justify in one sentence each why each dependency belongs in its category.

<details><summary>Answers</summary>

- Real-World Check: Frame it around what field injection actually costs
  today — every one of those 200 classes requires spinning up a Spring
  context (or reflection-based mocking) just to unit test, which slows the
  test suite and makes tests more fragile/coupled to Spring's wiring
  behavior; migrating incrementally (only touching files already being
  modified for other reasons, rather than a risky big-bang rewrite) captures
  the testability win without a large, disruptive refactor.
- Senior Check: Rare, but arguably acceptable in throwaway prototype code
  never intended for production, or in very specific framework-integration
  contexts where the framework itself mandates field injection (some legacy
  JPA/CDI interop scenarios) — even then, treated as a known trade-off, not a default.
- Master Check: e.g., `PaymentGateway` (constructor-injected — the service
  cannot function at all without a way to charge payments, so it's a
  correctness-critical, non-optional dependency) and `MetricsReporter`
  (setter-injected, `required=false` — the service should still function
  correctly, just without metrics, if no reporter is configured, since
  metrics are an operational nicety, not a functional requirement).

</details>

---

## 1.9 CHEAT SHEET

| Concept | One-liner |
|---|---|
| IoC | Object creation/wiring control moves to an external container |
| DI | The technique Spring uses to achieve IoC — dependencies declared, not self-constructed |
| Bean | Any object whose lifecycle the Spring container manages |
| Constructor injection | Default choice — `final` fields, visible mandatory deps, plain-Java testable |
| Setter injection | Use only for genuinely optional dependencies |
| Field injection | Avoid — hides deps, blocks `final`, harder to test |
| `ApplicationContext` | The concrete IoC container implementation |

---

*(Continues to Chapter 2 — Bean Configuration Styles & the Container.)*
