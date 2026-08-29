# CHAPTER 2 — OOP, SOLID, AND COMPOSITION VS INHERITANCE

> **Depth level:** ZERO → SENIOR/ARCHITECT. Goal: not just "define the 4 pillars"
> but decide, in real code, when a principle should bend.

---

## 2.1 WHY THIS CHAPTER EXISTS

OOP గురించి చాలామంది "4 pillars" నేర్చుకుంటారు కానీ ఇంటర్వ్యూలో "మీరు ఇక్కడ
inheritance వాడారు, ఎందుకు composition వాడలేదు?" అని అడిగితే జవాబు చెప్పలేరు.
ఈ chapter goal: pillars ని memorize చేయడం కాదు, **ఎప్పుడు ఏది వాడాలో reasoning**
నేర్పించడం — ఇది Senior/Architect interview లో నిజంగా test చేసేది.

---

## 2.2 CONCEPT: The Four Pillars

### TELUGU EXPLANATION

- **Encapsulation:** object యొక్క internal state (fields) ను hide చేసి, controlled
  access (getters/setters, validated methods) ద్వారానే మార్చనివ్వడం. Goal:
  invariants (object ఎప్పుడూ valid state లో ఉండాలి అనే rule) ని guarantee చేయడం.
- **Abstraction:** "ఏం చేస్తుంది" చూపించి "ఎలా చేస్తుంది" అనేది hide చేయడం.
  Interfaces/abstract classes ఈ purpose కోసమే.
- **Inheritance:** ఒక class మరొక class నుండి fields/behavior ని reuse చేయడం
  ("is-a" relationship). **జాగ్రత్త:** ఇది reuse కోసం మాత్రమే కాదు — ఇది ఒక
  **substitutability contract** (Liskov Substitution Principle చూడండి).
- **Polymorphism:** ఒకే method call, runtime లో actual object type బట్టి,
  వేర్వేరు behavior చూపించడం (**dynamic dispatch**). Compile-time polymorphism
  (method overloading) వేరు, runtime polymorphism (method overriding) వేరు.

### INDUSTRY TERMINOLOGY

`Encapsulation`, `Abstraction`, `Inheritance`, `Polymorphism`, `is-a`, `has-a`,
`dynamic dispatch`, `method overriding`, `method overloading`, `invariant`.

### ENGLISH INTERVIEW ANSWER

"The four pillars aren't independent tricks — they compose. Encapsulation
protects an object's invariants by controlling how state changes. Abstraction
lets callers depend on *what* a type does via an interface, not *how* it does
it, which is what enables polymorphism: at runtime, the JVM performs dynamic
dispatch based on the object's actual class, not its declared reference type.
Inheritance is the mechanism most commonly used to enable that polymorphism,
but I treat inheritance as a substitutability contract, not just a code-reuse
shortcut — if a subclass can't honestly satisfy every behavioral expectation
of its parent, it shouldn't inherit from it, per the Liskov Substitution
Principle."

### SIMPLE EXPLANATION

Encapsulation = hide state safely. Abstraction = hide "how", expose "what".
Inheritance = is-a reuse + substitutability. Polymorphism = same call,
different behavior at runtime.

### REAL-TIME EXAMPLE

ఒక payment processing system లో, `PaymentGateway` అనే interface ఉండి,
`StripeGateway`, `RazorpayGateway` అనే implementations ఉంటాయి. `OrderService`
ఏ concrete gateway వాడుతున్నామో తెలియకుండానే `PaymentGateway.charge(...)`
call చేస్తుంది — ఇది abstraction + polymorphism వల్ల సాధ్యం, కొత్త gateway
add చేయాలంటే `OrderService` code మార్చాల్సిన అవసరం లేదు.

---

## 2.3 CONCEPT: SOLID Principles

### TELUGU EXPLANATION

- **S — Single Responsibility Principle (SRP):** ఒక class కి ఒకే ఒక్క "reason
  to change" ఉండాలి. ఉదా: `OrderService` order logic చూసుకోవాలి, PDF invoice
  generate చేయడం వేరే class చూసుకోవాలి — రెండూ కలిపేస్తే, invoice format
  మారితే order logic కూడా touch అవుతుంది (unrelated risk).
- **O — Open/Closed Principle (OCP):** class extension కోసం open గా, modification
  కోసం closed గా ఉండాలి. కొత్త payment type add చేయాలంటే, existing code
  మార్చకుండా, కొత్త class add చేయగలగాలి (Strategy pattern ద్వారా).
  Trade-off గుర్తుంచుకోండి: ప్రతి small change కి OCP వాడితే over-engineering.
- **L — Liskov Substitution Principle (LSP):** Subtype ని parent type బదులు
  వాడితే program correctness మారకూడదు. Classic violation ఉదాహరణ: `Square
  extends Rectangle` — `setWidth`/`setHeight` behavior break అవుతుంది
  (Square లో రెండూ ఒకేసారి మారాలి), కాబట్టి client code surprise అవుతుంది.
- **I — Interface Segregation Principle (ISP):** Client లు తమకి అవసరం లేని
  methods మీద depend అవ్వకూడదు. పెద్ద "fat" interface కంటే, చిన్న చిన్న
  focused interfaces మంచివి.
- **D — Dependency Inversion Principle (DIP):** High-level modules low-level
  modules మీద directly depend కాకూడదు — రెండూ abstractions మీద depend
  అవ్వాలి. ఇదే Spring యొక్క **Dependency Injection** కి theoretical పునాది
  (Book 3 లో వివరంగా చూద్దాం).

### INDUSTRY TERMINOLOGY

`SRP`, `OCP`, `LSP`, `ISP`, `DIP`, `Strategy Pattern`, `fat interface`,
`high-level module`, `low-level module`.

### ENGLISH INTERVIEW ANSWER

"SOLID gives me a vocabulary for *why* a design is fragile, not just that it
is. If I find myself touching the same class for unrelated reasons, that's an
SRP violation. If adding a new variant requires editing an existing
`switch`/`if-else` chain instead of adding a new class, that's an OCP
violation — usually fixed with a Strategy or polymorphism-based design. LSP
tells me a class hierarchy is wrong when a subclass has to throw
`UnsupportedOperationException` or silently change behavior to fit the parent
contract. ISP tells me an interface is doing too much when implementers are
forced to implement methods they don't need. And DIP is the principle behind
constructor injection in Spring — services depend on interfaces, and the
concrete implementation is wired in externally, which is what makes the code
testable via mocking and swappable in production."

### SIMPLE EXPLANATION

SRP: one reason to change. OCP: extend without modifying. LSP: subtype must
behave like its parent,不 surprise. ISP: small focused interfaces. DIP:
depend on abstractions, not concretions.

### REAL-TIME EXAMPLE

ఒక enterprise reporting service లో, కొత్త export format (Excel) add చేయాల్సి
వచ్చినప్పుడు, existing `if (format.equals("PDF")) {...} else if
(format.equals("CSV")) {...}` block లో ఇంకో `else if` add చేయడం OCP violation.
బదులుగా, `ReportExporter` interface + `PdfExporter`, `CsvExporter`,
`ExcelExporter` classes + a registry/map — కొత్త format add చేయాలంటే కొత్త
class రాయడమే సరిపోతుంది, existing code touch అవదు.

---

## 2.4 COMPOSITION VS INHERITANCE — THE SENIOR-LEVEL DECISION

### TELUGU EXPLANATION

Junior engineer తరచుగా code reuse కోసం inheritance వాడేస్తారు — "ఇది కూడా
ఇలాంటిదే, extend చేసేద్దాం" అని. Senior engineer ముందు అడిగే ప్రశ్న:
**"ఇది నిజంగా 'is-a' relationship నా, లేక కేవలం 'behavior share చేయాలి'
అన్న అవసరమా?"** రెండోది అయితే, **composition** (ఒక class లోపల మరో class
యొక్క object ని field గా hold చేయడం, దాని methods ని delegate చేయడం)
వాడాలి.

**Favor composition over inheritance** అనే principle కారణాలు:
1. Inheritance **tight coupling** create చేస్తుంది — parent class internal
   మార్పు, subclasses ని break చేయవచ్చు (**fragile base class problem**).
2. Java single inheritance మాత్రమే support చేస్తుంది — ఒకసారి ఒక class ని
   extend చేస్తే, వేరే class ని extend చేయలేరు. Composition తో, multiple
   behaviors ని multiple objects ద్వారా combine చేయవచ్చు.
3. Composition runtime లో behavior మార్చగలదు (constructor/setter ద్వారా వేరే
   implementation inject చేయవచ్చు) — inheritance compile-time fixed.

### INDUSTRY TERMINOLOGY

`is-a` vs `has-a`, `fragile base class problem`, `tight coupling`,
`favor composition over inheritance`, `delegation`.

### ENGLISH INTERVIEW ANSWER

"I default to composition and reach for inheritance only when there's a
genuine, stable 'is-a' relationship that I expect to hold for the life of the
system — for example, `Circle is-a Shape`. The moment I'm inheriting purely
to reuse a few methods, without a true substitutability relationship, I
switch to composition: hold the behavior as a field, delegate to it. This
avoids the fragile base class problem, where a change deep in a base class
ripples unpredictably into subclasses several levels down, and it keeps
options open — for example, Java's single inheritance means committing to
one base class forecloses other reuse, while composing multiple small
collaborators doesn't."

### SIMPLE EXPLANATION

is-a + stable + substitutable → inheritance is fine. Otherwise → composition
(hold a reference, delegate).

### REAL-TIME EXAMPLE

Notification system: `EmailNotifier`, `SmsNotifier`, `PushNotifier` — ఇవి
"is-a Notifier" కాదు నిజంగా behavior మాత్రమే వేరు. కాబట్టి `NotificationService`
ఒక `List<NotificationChannel>` ని compose చేసి, ప్రతి channel కి delegate
చేస్తుంది — ఇది **Strategy pattern**, composition ఆధారంగా.

---

## 2.5 CODE: DEMONSTRATING LSP VIOLATION AND THE COMPOSITION FIX

```java
// ❌ Classic LSP violation
class Rectangle {
    protected int width, height;
    void setWidth(int w) { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}

class Square extends Rectangle {
    @Override void setWidth(int w) { width = w; height = w; }  // surprise!
    @Override void setHeight(int h) { width = h; height = h; } // surprise!
}

// A client written against Rectangle's contract breaks silently for Square:
void resizeAndCheck(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.area() == 50; // FAILS for Square — area() is 100, not 50
}
```

**Explanation:** `Square` technically compiles as a `Rectangle`, but it
violates the *behavioral* contract clients rely on (setting width shouldn't
affect height). This is exactly what LSP warns against — inheritance based
on a superficial "a square is a kind of rectangle" geometric truth, without
checking behavioral substitutability.

```java
// ✅ Fix: no inheritance relationship needed; both implement a Shape abstraction
interface Shape {
    int area();
}

final class Rectangle implements Shape {
    private final int width, height;
    Rectangle(int width, int height) { this.width = width; this.height = height; }
    public int area() { return width * height; }
}

final class Square implements Shape {
    private final int side;
    Square(int side) { this.side = side; }
    public int area() { return side * side; }
}
```

**Design notes:** Making both classes `final` and immutable (fields set only
in the constructor) sidesteps the mutable-setter problem entirely — this is
also a preview of Chapter 3's immutability theme. `Shape` is the abstraction
(DIP) that client code depends on; `resizeAndCheck`-style mutation logic
simply shouldn't exist against these types once they're immutable.

**Interviewer follow-ups:**
- "Could you fix the original hierarchy without removing inheritance?" (You
  could make `Rectangle` immutable and give `Square` its own constructor
  instead of inheriting mutable setters — but the cleaner senior answer is
  usually to question whether the inheritance relationship should exist at
  all.)
- "What's the cost of making these `final`?" (You lose the ability to
  subclass for extension, which is usually desirable for simple value-like
  types — this is itself an OCP/LSP-aware design choice.)

---

## 2.6 SENIOR VS JUNIOR THINKING TABLE

| Situation | Junior | Senior/Architect |
|---|---|---|
| Need shared behavior across two unrelated classes | "Make one extend the other" | "Extract an interface/behavior class, use composition" |
| Adding a new variant to a `switch` on type | "Add another case" | "Is this OCP violation recurring? Consider polymorphism/Strategy" |
| Interface has 10 methods, most implementers only use 3 | "Implement all 10, throw `UnsupportedOperationException` for the rest" | "Split into smaller interfaces (ISP)" |
| A subclass overrides a method to do nothing / throw | "That's fine, it's still technically 'a' subtype" | "This is an LSP smell — re-examine the hierarchy" |

---

## 2.7 COMMON MISTAKES

1. Using inheritance purely for code reuse without a real "is-a" relationship.
2. Deep inheritance hierarchies (4+ levels) — fragile, hard to reason about.
3. Overusing interfaces/abstraction for code that will never have more than
   one implementation ("speculative generality" — a real anti-pattern, not
   a virtue).
4. Violating encapsulation by exposing mutable internal collections directly
   (`return this.items;` instead of an unmodifiable view or a copy).
5. Treating SOLID as absolute law rather than a set of trade-offs — applying
   DIP/OCP everywhere adds indirection that isn't always worth its cost in a
   small, stable codebase.

---

## 2.8 INTERVIEW QUESTION BANK — CHAPTER 2

### BASIC
1. Explain the four pillars of OOP with one real example each.
2. What is the difference between method overloading and overriding?

### INTERMEDIATE
3. Explain each SOLID principle with a code smell that indicates its violation.
4. Why is "favor composition over inheritance" good general advice? When would you break it?

### SENIOR
5. Show a real hierarchy from a project you've worked on (or would design)
   that violates LSP, and how you'd redesign it.
6. When does applying DIP add unnecessary complexity, and how do you decide?

### ARCHITECT
7. You're designing a plugin system where third parties will implement your
   interfaces. How does ISP guide the interface design, and what happens if
   you get it wrong after the plugin ecosystem already exists?

### SCENARIO
8. A junior engineer submits a PR where `PromoDiscount extends Discount` and
   overrides `apply()` to sometimes return a negative discount (a surcharge).
   What do you flag in review?

### TRICK
9. "More interfaces always means better design." True or false — defend your answer.

---

## 2.9 CHAPTER MASTERY CHECKPOINTS

- **Knowledge Check:** Name the SOLID principle violated when a `ReportGenerator` class also handles emailing the report and writing audit logs.
- **Coding Check:** Refactor a `Bird` class hierarchy where `Penguin extends Bird` and `fly()` throws an exception, into a design that doesn't violate LSP.
- **Explanation Check:** In English, explain to a junior why `favor composition over inheritance` is the default, not an absolute rule.
- **Real-World Check:** Your `PaymentProcessor` interface has grown to 15 methods over 2 years as new payment types were added. New implementers only need 3–4 of them. Diagnose and propose a fix.
- **Senior Check:** When would you deliberately choose inheritance over composition even knowing the trade-offs?
- **Master Check:** Design (on paper) a notification system supporting Email/SMS/Push/Slack that can add a new channel without modifying existing code, using SOLID principles explicitly. State which principle each design decision serves.

<details>
<summary>Answers</summary>

- Knowledge Check: SRP — three unrelated reasons to change (report format, email delivery, audit logging).
- Coding Check: Split into `Bird` (common traits) and a `Flyable` interface implemented only by flying birds; `Penguin` implements `Bird` but not `Flyable`.
- Real-World Check: ISP violation — split `PaymentProcessor` into smaller interfaces like `Chargeable`, `Refundable`, `Recurring`, so implementers only take on what they need.
- Senior Check: When the relationship is genuinely stable and behavioral substitutability is guaranteed for the foreseeable life of the system — e.g., a small, closed set of `Shape` subtypes in a geometry library you fully control, where the "is-a" contract is mathematically enforced, not just superficially true.
- Master Check: `NotificationChannel` interface (DIP: `NotificationService` depends on the abstraction) with `EmailChannel`/`SmsChannel`/etc. implementations (OCP: new channel = new class, no edits to `NotificationService`); each channel class has a single responsibility (SRP); if some channels need delivery-receipt tracking and others don't, split into `NotificationChannel` and `TrackableNotificationChannel` (ISP) rather than forcing every channel to implement tracking.

</details>

---

## 2.10 CHEAT SHEET

| Principle | One-line test |
|---|---|
| SRP | "Can I describe this class's job in one sentence without 'and'?" |
| OCP | "Can I add a new case without editing existing code?" |
| LSP | "Can every client use the subtype without knowing/caring it's a subtype?" |
| ISP | "Does every implementer actually need every method?" |
| DIP | "Does my high-level class import a concrete class, or an interface?" |
| Composition vs Inheritance | "Is it truly is-a and stable, or just 'shares some code'?" |

---

*(Continues to Chapter 3 — String, Immutability, and the String Pool.)*
