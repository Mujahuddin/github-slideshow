# 📘 BOOK 07 — JAVA 8+ MODERN JAVA MASTERY
## Lambdas, Streams, Functional Programming to Java 21 LTS (Telugu + English)

**Series:** Java Interview + Development Mastery Series
**Book Number:** 07 of 24
**Java Versions Covered:** **Java 8** (deep focus — lambdas, streams, Optional, default/static interface methods, Date/Time API, CompletableFuture basics), **Java 11 LTS** (`var`, new String/Collection methods, HTTP Client), **Java 21 LTS** (records, sealed classes, pattern matching, switch expressions, text blocks, virtual threads)
**Prerequisites:** Book 01–02 (classes, interfaces, polymorphism), Book 05 (Collections — Streams operate over them), Book 06 (Generics — functional interfaces are generic)
**Next Book:** `08_Java_Multithreading_Concurrency_Part_1.md`

---

## 📖 How to Use This Book / ఈ పుస్తకాన్ని ఎలా ఉపయోగించాలి

**Telugu:** ఇది Java history లో అత్యంత ముఖ్యమైన మార్పు — Java 8. ఇక్కడ నుండి Java **imperative** (ఏమి చేయాలో step-by-step చెప్పడం) programming నుండి **functional/declarative** (ఏమి కావాలో చెప్పడం, ఎలా అనేది library కి వదిలేయడం) style వైపు మారింది. ఈ పుస్తకం Java 8 ని అత్యంత లోతుగా, తర్వాత Java 11, Java 21 LTS features ని కవర్ చేస్తుంది.

**English:** Java 8 is the single most important shift in Java's history — from purely **imperative** programming (spelling out every step) toward a **functional/declarative** style (saying what you want, letting the library handle how). This book covers Java 8 in maximum depth, then Java 11 and Java 21 LTS features, clearly labeled by version throughout.

---

## 🎯 Learning Objectives

1. Understand why Java 8 fundamentally changed how Java code is written.
2. Master lambda expressions and the functional interfaces behind them.
3. Master the full Stream API — intermediate and terminal operations, collectors.
4. Master `Optional` as a null-handling design pattern.
5. Use the modern `java.time` Date/Time API correctly.
6. Understand `CompletableFuture` basics for asynchronous composition.
7. Know Java 11's practical everyday additions.
8. Master Java 21 LTS: records, sealed classes, pattern matching, switch expressions, text blocks, and virtual threads.
9. Know when NOT to use streams, and avoid common functional-style mistakes.

---

## 📑 Table of Contents

| Ch. | Title |
|---|---|
| 1 | Why Java 8? The Functional Programming Shift |
| 2 | Lambda Expressions Deep Dive |
| 3 | Functional Interfaces: Predicate, Consumer, Supplier, Function |
| 4 | Method References |
| 5 | Default & Static Interface Methods |
| 6 | Stream API — Fundamentals & Intermediate Operations |
| 7 | Stream Terminal Operations & Collectors |
| 8 | Parallel Streams |
| 9 | Optional — Deep Dive |
| 10 | Java 8 Date/Time API |
| 11 | CompletableFuture Basics |
| 12 | Java 11 LTS — Practical Everyday Features |
| 13 | Java 21 LTS — Records & Sealed Classes |
| 14 | Java 21 LTS — Pattern Matching, Switch Expressions, Text Blocks |
| 15 | Java 21 LTS — Virtual Threads (Preview) |
| 16 | Streams in Production — Pitfalls + Mini Project |
| — | Final Revision Notes, Cheat Sheet, Interview Bank, Revision Plans, Mastery Checklist |

---

# CHAPTER 1 — Why Java 8? The Functional Programming Shift

### Telugu Explanation
Java 8 కి ముందు, ఒక collection ని process చేయాలంటే, **ఎలా** iterate చేయాలో (for loop, index management) మీరే explicit గా రాయాల్సి వచ్చేది — దీన్ని **imperative style** అంటారు. Java 8 **functional/declarative style** తీసుకొచ్చింది — మీరు **ఏమి** కావాలో చెప్తే (filter this, transform that), internal iteration/optimization library handle చేస్తుంది.

### Professional English Explanation
Before Java 8, processing a collection meant explicitly writing **how** to iterate — index management, loop control (**imperative style**). Java 8 introduced a **functional/declarative style**: you describe **what** you want (filter this, transform that, collect results), and the library handles the how, internal iteration, and — critically — makes parallelization (Ch.8) possible without rewriting your logic.

### Java Code — Imperative vs Functional Side by Side

```java
import java.util.*;
import java.util.stream.*;

public class WhyJava8Demo {
    public static void main(String[] args) {
        List<String> names = List.of("Ravi", "Anitha", "Bob", "Kiran", "Alice");

        // IMPERATIVE (pre-Java 8 style): explicit iteration, explicit accumulation
        List<String> longNamesImperative = new ArrayList<>();
        for (String name : names) {
            if (name.length() > 4) {
                longNamesImperative.add(name.toUpperCase());
            }
        }
        Collections.sort(longNamesImperative);
        System.out.println("Imperative result: " + longNamesImperative);

        // FUNCTIONAL/DECLARATIVE (Java 8+): describe WHAT, not HOW
        List<String> longNamesFunctional = names.stream()
                .filter(name -> name.length() > 4)
                .map(String::toUpperCase)
                .sorted()
                .collect(Collectors.toList());
        System.out.println("Functional result: " + longNamesFunctional);
    }
}
```

### Output
```
Imperative result: [ANITHA, KIRAN]
Functional result: [ANITHA, KIRAN]
```

### Internal Working
- The functional pipeline (`filter` → `map` → `sorted` → `collect`) is built from **functional interfaces** (Ch.3) — `filter` takes a `Predicate<T>`, `map` takes a `Function<T,R>` — each a single-abstract-method interface that a lambda expression (Ch.2) can implement inline.
- This declarative structure is exactly what makes **parallel streams** (Ch.8) possible with a one-word change (`.parallelStream()`) — the imperative for-loop version has no equivalent one-line parallelization path, since the iteration logic and business logic are tangled together.
- Java 8 didn't add a new paradigm to the JVM itself — it's all still compiled to regular bytecode calling regular methods; lambdas are syntactic sugar (Ch.2) backed by `invokedynamic`, not a fundamentally new execution model.

### Real-World Example
Telugu: Real backend code లో, DB నుండి వచ్చిన records ని filter చేసి, transform చేసి, group చేసి report generate చేయడం — ఇలాంటి pipelines Java 8 style లో చాలా readable గా, less error-prone గా ఉంటాయి, పాత nested for-loop code కంటే.
English: Filtering, transforming, grouping, and reporting over records fetched from a database is exactly the kind of pipeline that reads dramatically more clearly in Java 8's declarative style than in the equivalent nested nested for-loops — a genuine, measurable maintainability win in real backend code.

### Interview Answer
"Java 8 shifted Java from a purely imperative style — explicitly writing iteration logic — toward a functional/declarative style, where you describe the transformation pipeline (filter, map, reduce) and the library handles iteration internally. This is enabled by lambda expressions and functional interfaces, and it's what makes parallel streams possible without rewriting business logic."

### Cross Questions
- Q: Did Java 8 change how the JVM executes code at a fundamental level? → A: No — lambdas compile to regular bytecode using `invokedynamic`; it's a language-level and API-level shift (new syntax, new `java.util.function`/`java.util.stream` APIs), not a new JVM execution model.
- Q: Is functional-style code always faster than imperative code? → A: Not necessarily — for simple, small collections, a plain for-loop can be faster due to lower overhead; functional style's real wins are readability, composability, and easy parallelization (Ch.8, Ch.16 covers when NOT to use streams).
- Q: What two new packages define most of Java 8's functional programming support? → A: `java.util.function` (functional interfaces, Ch.3) and `java.util.stream` (Streams, Ch.6–8).

### Coding Exercise
**L1:** Rewrite a simple for-loop (filter + transform + collect) in both imperative and functional style, side by side.
**L2:** Time both versions (via `System.nanoTime()`) on a small (100 elements) and large (1,000,000 elements) list, and compare.
**L3:** Identify 3 places in a piece of your own past code where a functional pipeline would have been clearer than the imperative version you wrote.
**L4 (Interview):** Explain, precisely, what "declarative" means in the context of Java 8 streams.
**L5 (Senior):** Explain why the functional style specifically enables easy parallelization, referencing the coupling between iteration logic and business logic in the imperative version.
**L6 (Mastery):** Explain, from memory, that Java 8 lambdas are syntactic sugar over `invokedynamic`, not a new JVM paradigm.

---

# CHAPTER 2 — Lambda Expressions, Deep Dive

### Telugu Explanation
Lambda expression అనేది ఒక **anonymous function** — name లేని, ఒక functional interface (Ch.3) ని implement చేసే inline code block. Syntax: `(parameters) -> expression` లేదా `(parameters) -> { statements; }`. Book 01, Ch.14 లో మనం చూసిన anonymous classes కి ఇది చాలా concise replacement.

### Professional English Explanation
A lambda expression is an anonymous function — an inline implementation of a functional interface's single abstract method, with no name of its own. Syntax: `(parameters) -> expression` or `(parameters) -> { statements; }`. It's a dramatically more concise replacement for the anonymous-class pattern shown in Book 01, Ch.14.

### Java Code

```java
import java.util.function.*;

public class LambdaDeepDiveDemo {
    interface Greeting { String greet(String name); }             // custom functional interface

    public static void main(String[] args) {
        // Anonymous class (pre-Java 8, Book 01 Ch.14) vs Lambda (Java 8+)
        Greeting anonGreeting = new Greeting() {
            @Override public String greet(String name) { return "Hello, " + name; }
        };
        Greeting lambdaGreeting = name -> "Hello, " + name;          // equivalent, far more concise

        System.out.println(anonGreeting.greet("Ravi"));
        System.out.println(lambdaGreeting.greet("Ravi"));

        // Various lambda syntax forms
        Runnable noArgs = () -> System.out.println("No-arg lambda running");
        Function<Integer, Integer> square = x -> x * x;                 // single param, no parens needed (optional)
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;    // multiple params need parens
        Function<Integer, Integer> multiStatement = x -> {                // block body needs explicit return
            int result = x * 2;
            return result + 1;
        };

        noArgs.run();
        System.out.println("Square: " + square.apply(5));
        System.out.println("Add: " + add.apply(3, 4));
        System.out.println("Multi-statement: " + multiStatement.apply(5));

        // Effectively final variable capture
        int base = 100;
        Function<Integer, Integer> addBase = x -> x + base;               // 'base' captured - must be effectively final
        System.out.println("With captured variable: " + addBase.apply(5));
        // base = 200;                                                       // would make 'base' non-effectively-final -> compile error above
    }
}
```

### Output
```
Hello, Ravi
Hello, Ravi
No-arg lambda running
Square: 25
Add: 7
Multi-statement: 11
With captured variable: 105
```

### Internal Working
- A lambda's type is always inferred from **context** — the target functional interface — not from the lambda syntax itself; the same `x -> x * x` could be a `Function<Integer,Integer>`, a `UnaryOperator<Integer>`, or any other single-abstract-method interface with a compatible signature, depending on where it's assigned/passed.
- **Variable capture**: a lambda can reference local variables from its enclosing scope, but only if they are **effectively final** (never reassigned after initialization) — exactly the same rule as local/anonymous classes (Book 01, Ch.14), because the lambda may execute later or on a different thread, after the original stack frame (Book 03, Ch.4) is gone; the compiler captures the variable's **value**, not a live reference to the variable itself.
- Unlike anonymous classes, a lambda does **not** create a new scope for `this` — `this` inside a lambda refers to the enclosing class instance, not the lambda itself (a genuine, useful difference from anonymous classes, where `this` refers to the anonymous class instance).

### Real-World Example
Telugu: Event listeners, `Comparator` implementations, `Runnable` tasks — Book 01, Ch.14 లో anonymous classes గా రాసినవి, ఇప్పుడు లోతైన boilerplate లేకుండా ఒక్క line లో రాయచ్చు.
English: Event listeners, `Comparator` implementations, and `Runnable` tasks — all written as verbose anonymous classes in Book 01, Ch.14 — collapse to a single line with lambdas, which is exactly why lambdas so quickly became the default idiom for single-method interface implementations across the entire JDK ecosystem after Java 8.

### Interview Answer
"A lambda expression is an anonymous, inline implementation of a functional interface's single abstract method — far more concise than the equivalent anonymous class. Its type is inferred from the target functional interface at the point of use, and it can capture local variables from its enclosing scope only if they're effectively final, for the same reason anonymous/local classes require it: the lambda might execute after the original stack frame is gone."

### Cross Questions
- Q: What does "effectively final" mean? → A: A local variable that is never reassigned after its initial assignment, even though it isn't explicitly declared `final` — the compiler infers this itself.
- Q: Does a lambda create its own `this`? → A: No — unlike an anonymous class, `this` inside a lambda refers to the enclosing instance, not the lambda itself; this is a genuine behavioral difference worth remembering.
- Q: Can a lambda expression exist without a target functional interface? → A: No — a lambda's type is always determined by context (the variable type, method parameter type, or cast it's assigned/passed to); it cannot stand alone with an inferred "lambda type."

### Tricky Questions
- Q: If a lambda captures a local variable and that variable's *field* (not the reference itself) is mutated later, is that allowed? → A: Yes — "effectively final" only restricts reassigning the *reference/primitive variable itself*; if the captured variable is a mutable object reference, mutating that object's internal state after capture is fully legal (the reference itself was never reassigned).
- Q: Can a lambda throw a checked exception? → A: Only if the functional interface's abstract method signature declares that checked exception via `throws` — most standard `java.util.function` interfaces (Ch.3) don't, which is a well-known practical friction point when using checked-exception-throwing code inside standard lambdas.

### Coding Exercise
**L1:** Convert 3 anonymous class examples from Book 01, Ch.14 into equivalent lambda expressions.
**L2:** Write a lambda capturing a local variable, then demonstrate the "effectively final" compile error by attempting to reassign it.
**L3:** Demonstrate that `this` inside a lambda refers to the enclosing instance, contrasted with an anonymous class where it doesn't.
**L4 (Interview):** Explain why lambdas require captured local variables to be effectively final.
**L5 (Senior):** Explain the practical friction of using checked-exception-throwing code inside a standard `java.util.function` lambda, and describe 2 ways to work around it.
**L6 (Mastery):** Explain, from memory, how a lambda's type is inferred purely from context, with no standalone "lambda type" of its own.

---

# CHAPTER 3 — Functional Interfaces: Predicate, Consumer, Supplier, Function

### Telugu Explanation
Functional interface అంటే **exactly ఒక్క abstract method** ఉన్న interface (`default`/`static` methods ఎన్ని ఉన్నా పర్వాలేదు) — దీన్నే **SAM (Single Abstract Method)** interface అంటారు, `@FunctionalInterface` annotation తో optionally mark చేయచ్చు. `java.util.function` package నాలుగు ప్రధాన functional interfaces ఇస్తుంది: `Predicate<T>` (boolean test), `Consumer<T>` (input తీసుకుని, ఏమీ return చేయదు), `Supplier<T>` (input తీసుకోదు, value return చేస్తుంది), `Function<T,R>` (input తీసుకుని, output return చేస్తుంది).

### Professional English Explanation
A functional interface has **exactly one abstract method** (regardless of how many `default`/`static` methods it has) — called a **SAM (Single Abstract Method)** interface, optionally marked with `@FunctionalInterface` for compiler-enforced documentation. `java.util.function` provides four core interfaces: `Predicate<T>` (tests a condition, returns `boolean`), `Consumer<T>` (accepts input, returns nothing), `Supplier<T>` (accepts nothing, returns a value), and `Function<T,R>` (accepts input, returns a transformed output).

### Java Code

```java
import java.util.function.*;
import java.util.*;

public class FunctionalInterfacesDemo {
    public static void main(String[] args) {
        Predicate<Integer> isEven = n -> n % 2 == 0;
        System.out.println("isEven.test(4): " + isEven.test(4));
        System.out.println("isEven.negate().test(4): " + isEven.negate().test(4));           // built-in default method
        Predicate<Integer> isPositive = n -> n > 0;
        System.out.println("isEven.and(isPositive).test(-4): " + isEven.and(isPositive).test(-4));

        Consumer<String> printer = s -> System.out.println("Consumed: " + s);
        Consumer<String> logger = s -> System.out.println("Logged: " + s);
        printer.andThen(logger).accept("event happened");                                     // chained consumers

        Supplier<Double> randomSupplier = Math::random;                                        // no input, produces a value
        System.out.println("Supplied random (0-1): " + (randomSupplier.get() >= 0));

        Function<String, Integer> length = String::length;
        Function<Integer, Integer> doubleIt = n -> n * 2;
        System.out.println("length.andThen(doubleIt).apply('hello'): " + length.andThen(doubleIt).apply("hello"));
        System.out.println("length.compose(doubleIt is invalid here - types must chain): ");

        // Specialized variants (avoid autoboxing overhead, Book 01 Ch.12)
        IntPredicate isEvenPrimitive = n -> n % 2 == 0;               // no boxing to Integer needed
        BiFunction<Integer, Integer, Integer> multiply = (a, b) -> a * b;
        UnaryOperator<Integer> increment = n -> n + 1;                  // Function<T,T> specialization
        BinaryOperator<Integer> max = Integer::max;                     // BiFunction<T,T,T> specialization

        System.out.println("BiFunction multiply(3,4): " + multiply.apply(3, 4));
        System.out.println("UnaryOperator increment(5): " + increment.apply(5));
        System.out.println("BinaryOperator max(3,4): " + max.apply(3, 4));
    }
}
```

### Output
```
isEven.test(4): true
isEven.negate().test(4): false
isEven.and(isPositive).test(-4): false
Consumed: event happened
Logged: event happened
Supplied random (0-1): true
length.andThen(doubleIt).apply('hello'): 10
length.compose(doubleIt is invalid here - types must chain): 
BiFunction multiply(3,4): 12
UnaryOperator increment(5): 6
BinaryOperator max(3,4): 6
```

### Core Functional Interfaces Table

| Interface | Abstract Method | Purpose | Common Use |
|---|---|---|---|
| `Predicate<T>` | `boolean test(T t)` | Test a condition | `stream.filter(predicate)` |
| `Consumer<T>` | `void accept(T t)` | Consume input, no return | `stream.forEach(consumer)` |
| `Supplier<T>` | `T get()` | Produce a value, no input | Lazy value creation, `Optional.orElseGet()` |
| `Function<T,R>` | `R apply(T t)` | Transform input to output | `stream.map(function)` |
| `UnaryOperator<T>` | `T apply(T t)` | `Function<T,T>` specialization | `list.replaceAll(operator)` |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | `BiFunction<T,T,T>` specialization | `stream.reduce(operator)` |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two-input transformation | `map.merge(key, value, biFunction)` |

### Internal Working
- Every one of these interfaces provides useful **default methods** (Ch.5) for composition — `Predicate.and()`/`.or()`/`.negate()`, `Function.andThen()`/`.compose()`, `Consumer.andThen()` — all implemented in terms of the single abstract method, demonstrating exactly why default methods (introduced in the same Java 8 release) were necessary to enrich these interfaces without breaking every existing implementer.
- Primitive-specialized variants (`IntPredicate`, `IntFunction<R>`, `ToIntFunction<T>`, `IntUnaryOperator`, etc.) exist specifically to **avoid autoboxing overhead** (Book 01, Ch.12) in performance-sensitive numeric pipelines — a real, deliberate design detail, not just API bloat.
- `@FunctionalInterface` is not required for a lambda-compatible interface to work — it's purely a compiler-enforced **documentation** annotation that causes a compile error if the interface accidentally gains a second abstract method later.

### Real-World Example
Telugu: Service layer validation logic లో `Predicate<Order>` chains (`isValidAmount.and(hasValidCustomer).and(isInStock)`) వాడి readable, composable validation rules build చేస్తారు — production code లో ఇది extremely common pattern.
English: Composable validation chains (`isValidAmount.and(hasValidCustomer).and(isInStock)`) built from `Predicate<T>` are an extremely common real production pattern for expressing business rules readably — a direct, practical payoff of these interfaces' default composition methods.

### Interview Answer
"A functional interface has exactly one abstract method, letting a lambda implement it inline. `java.util.function` provides `Predicate<T>` (test), `Consumer<T>` (accept, no return), `Supplier<T>` (produce, no input), and `Function<T,R>` (transform), plus specializations like `UnaryOperator`, `BinaryOperator`, `BiFunction`, and primitive variants (`IntPredicate`, etc.) to avoid autoboxing overhead."

### Cross Questions
- Q: Can a functional interface have multiple default methods? → A: Yes — only the *abstract* method count is restricted to exactly one; `default` and `static` methods (Ch.5) don't count toward that limit.
- Q: What's the difference between `Function.andThen()` and `Function.compose()`? → A: `f.andThen(g)` applies `f` first, then `g` to its result; `f.compose(g)` applies `g` first, then `f` to its result — the composition order is reversed between the two.
- Q: Why do primitive-specialized functional interfaces (`IntPredicate`, `ToIntFunction`) exist? → A: To avoid the overhead of autoboxing primitives into wrapper objects (Book 01, Ch.12) on every call in performance-sensitive numeric processing pipelines.

### Tricky Questions
- Q: Is `@FunctionalInterface` required for a lambda to be usable with that interface? → A: No — any interface with exactly one abstract method works with lambdas regardless of the annotation; the annotation only adds compile-time protection against accidentally breaking that property later.
- Q: Can `Runnable` be used as a lambda target? → A: Yes — `Runnable`'s `run()` method (no args, no return) qualifies it as a functional interface, even though it predates Java 8 by many years; Java 8 retroactively made many pre-existing single-method interfaces lambda-compatible.

### Coding Exercise
**L1:** Write a `Predicate<String>` chain combining 3 conditions with `.and()`/`.or()`/`.negate()`.
**L2:** Write a `Function<T,R>` pipeline using `.andThen()` to chain 3 transformations.
**L3:** Rewrite a numeric-heavy loop using `IntPredicate`/`IntFunction` instead of their boxed equivalents, and explain the performance rationale.
**L4 (Interview):** Fill in the full core functional interfaces table from memory (interface, method, purpose).
**L5 (Senior):** Design a composable validation framework for an `Order` domain using chained `Predicate<Order>`s with meaningful named predicates.
**L6 (Mastery):** Explain, from memory, the difference between `andThen()` and `compose()` on `Function`, with a worked example showing different results for the same two functions.

---

# CHAPTER 4 — Method References

### Telugu Explanation
Method reference అనేది ఒక **already-existing method** ని, ఒక new syntax (`::`) వాడి, functional interface ki నేరుగా point చేయడం — lambda `x -> someMethod(x)` రాయడం కంటే compact గా. నాలుగు రకాలు: **static method** (`ClassName::staticMethod`), **instance method of a particular object** (`object::instanceMethod`), **instance method of an arbitrary object of a particular type** (`ClassName::instanceMethod`), **constructor reference** (`ClassName::new`).

### Professional English Explanation
A method reference points directly at an already-existing method using `::` syntax, as a more compact alternative to a lambda that merely calls that method. Four forms: **static method** (`ClassName::staticMethod`), **instance method of a particular object** (`object::instanceMethod`), **instance method of an arbitrary object of a particular type** (`ClassName::instanceMethod`), and **constructor reference** (`ClassName::new`).

### Java Code

```java
import java.util.*;
import java.util.function.*;

public class MethodReferencesDemo {
    static boolean isLongWord(String s) { return s.length() > 5; }    // for static method reference

    static class Greeter {
        String prefix;
        Greeter(String prefix) { this.prefix = prefix; }               // for constructor reference
        String greet(String name) { return prefix + ", " + name; }      // for "particular object" reference
    }

    public static void main(String[] args) {
        // 1. Static method reference
        Predicate<String> longWord = MethodReferencesDemo::isLongWord;
        System.out.println("Static ref: " + longWord.test("elephant"));

        // 2. Instance method reference of a PARTICULAR object
        Greeter greeter = new Greeter("Hello");
        Function<String, String> greetFn = greeter::greet;
        System.out.println("Particular-object ref: " + greetFn.apply("Ravi"));

        // 3. Instance method reference of an ARBITRARY object of a particular type
        Function<String, Integer> lengthFn = String::length;             // equivalent to: s -> s.length()
        System.out.println("Arbitrary-object ref: " + lengthFn.apply("hello"));

        BiFunction<String, String, Boolean> equalsIgnoreCase = String::equalsIgnoreCase;   // s1.equalsIgnoreCase(s2)
        System.out.println("Arbitrary-object BiFunction ref: " + equalsIgnoreCase.apply("Hi", "hi"));

        // 4. Constructor reference
        Function<String, Greeter> greeterFactory = Greeter::new;
        System.out.println("Constructor ref: " + greeterFactory.apply("Namaste").greet("Anitha"));

        List<String> names = new ArrayList<>(List.of("banana", "Apple", "cherry"));
        names.sort(String::compareToIgnoreCase);                          // method reference as Comparator
        System.out.println("Sorted with method ref comparator: " + names);
    }
}
```

### Output
```
Static ref: true
Particular-object ref: Hello, Ravi
Arbitrary-object ref: 5
Arbitrary-object BiFunction ref: true
Constructor ref: Namaste, Anitha
Sorted with method ref comparator: [Apple, banana, cherry]
```

### Internal Working
- The compiler determines which of the four forms applies based on the **target functional interface's signature** matched against the referenced method's actual signature — e.g., `String::length` matches `Function<String,Integer>` because `length()` takes no explicit args (the "arbitrary object" itself becomes the implicit first parameter) and returns an `int` (auto-boxed to `Integer`).
- Method references are **not** a separate runtime mechanism from lambdas — the compiler translates both to the same `invokedynamic`-based mechanism; a method reference is purely a more concise **syntax** for a lambda that does nothing but delegate to an existing method.
- The "arbitrary object of a particular type" form is the one that most often confuses learners: `String::compareToIgnoreCase` used as a `Comparator<String>` means "call `.compareToIgnoreCase()` **on the first argument, passing the second as its parameter**" — the instance becomes an implicit parameter, effectively turning a 1-arg instance method into a 2-arg functional interface.

### Real-World Example
Telugu: `list.forEach(System.out::println)`, `stream.map(String::toUpperCase)`, `stream.collect(Collectors.toList())` లోపల constructor references — ఇవన్నీ production Java 8+ code లో అత్యంత frequently కనిపించే idioms.
English: `list.forEach(System.out::println)`, `stream.map(String::toUpperCase)`, and constructor references inside collectors are among the single most common idioms you'll see in any modern Java 8+ codebase — recognizing all four forms fluently is a genuine daily-productivity skill, not just an interview topic.

### Interview Answer
"A method reference (`::`) is a more concise alternative to a lambda that only delegates to an existing method — static methods, instance methods of a specific object, instance methods of an arbitrary object of a given type, and constructors. The compiler translates all four forms to the same underlying mechanism as lambdas; it's purely syntactic convenience."

### Cross Questions
- Q: How does the compiler know which of the 4 method reference forms applies? → A: By matching the target functional interface's parameter/return signature against candidate methods of that name — the "arbitrary object" form is distinguished from "particular object" by whether the class name or an instance reference precedes `::`.
- Q: Can a method reference point to an overloaded method? → A: Yes — the compiler resolves which overload based on the target functional interface's exact signature, exactly like normal overload resolution (Book 02, Ch.7).
- Q: Is `String::length` equivalent to `s -> s.length()`? → A: Yes, functionally identical — it's the "arbitrary object of a particular type" form, compiling to the same behavior.

### Tricky Questions
- Q: Can `String::new` be used as both a `Supplier<String>` and a `Function<char[], String>`? → A: Yes — a constructor reference is polymorphic over the target functional interface, matching whichever constructor overload fits the required signature (no-arg constructor for `Supplier<String>`, the `char[]` constructor for `Function<char[],String>`).
- Q: Does `object::instanceMethod` capture `object` eagerly (at the time the reference is created) or lazily (at call time)? → A: Eagerly — the specific object reference is captured immediately when the method reference expression is evaluated, similar to how a lambda captures effectively-final local variables (Ch.2).

### Coding Exercise
**L1:** Convert 4 lambda expressions (one matching each method reference form) into their method reference equivalents.
**L2:** Use `String::compareToIgnoreCase` and `String::compareTo` as `Comparator<String>`s and compare sorting results on mixed-case input.
**L3:** Use a constructor reference (`ClassName::new`) inside a `Stream.map()` to build objects from a `List<String>` of names.
**L4 (Interview):** Explain the difference between the "particular object" and "arbitrary object of a particular type" method reference forms with an example each.
**L5 (Senior):** Review a codebase full of lambdas that only delegate to an existing method (`x -> x.toUpperCase()`) and refactor them to method references, explaining the readability benefit.
**L6 (Mastery):** Explain, from memory, all 4 method reference forms and correctly classify 4 unlabeled examples.

---

# CHAPTER 5 — Default & Static Interface Methods

### Telugu Explanation
Java 8 కి ముందు, interface methods అన్నీ abstract గా ఉండాల్సి వచ్చేది — ఒక existing interface కి కొత్త method add చేస్తే, దాన్ని implement చేసే **అన్ని** classes break అయిపోయేవి. **Default methods** (`default` keyword తో, body తో) ఈ సమస్యని పరిష్కరించాయి — కొత్త method add చేసినా, existing implementers break అవ్వరు, fallback implementation వాడతారు. **Static methods** interface meీద directly utility methods గా వాడతారు.

### Professional English Explanation
Before Java 8, all interface methods had to be abstract — adding a new method to an existing interface would break **every** implementing class. **Default methods** (marked `default`, with a body) solved this: adding one to an existing interface doesn't break existing implementers, who simply inherit the fallback implementation unless they choose to override it. **Static interface methods** provide utility methods directly on the interface itself, without needing a separate utility class.

### Java Code

```java
public class DefaultStaticMethodsDemo {

    interface Vehicle {
        void drive();                                        // abstract - must be implemented

        default void honk() {                                   // default - optional to override
            System.out.println("Beep beep! (default horn sound)");
        }

        static Vehicle createDefault() {                         // static - utility/factory method
            return () -> System.out.println("Generic vehicle driving...");    // lambda implementing the SAM
        }
    }

    static class Car implements Vehicle {
        @Override public void drive() { System.out.println("Car driving on the road"); }
        // honk() not overridden - uses the interface's default implementation
    }

    static class SportsCar implements Vehicle {
        @Override public void drive() { System.out.println("SportsCar speeding down the highway"); }
        @Override public void honk() { System.out.println("VROOOM HORN! (custom horn)"); }    // overrides default
    }

    // Diamond problem scenario: two interfaces with the SAME default method
    interface A { default String identify() { return "A"; } }
    interface B { default String identify() { return "B"; } }
    static class C implements A, B {
        @Override public String identify() { return A.super.identify() + B.super.identify(); }  // MUST override, can call both explicitly
    }

    public static void main(String[] args) {
        Vehicle car = new Car();
        car.drive();
        car.honk();                                                // uses default

        Vehicle sportsCar = new SportsCar();
        sportsCar.drive();
        sportsCar.honk();                                           // uses overridden version

        Vehicle.createDefault().drive();                              // static factory method call

        System.out.println("Diamond resolved: " + new C().identify());
    }
}
```

### Output
```
Car driving on the road
Beep beep! (default horn sound)
SportsCar speeding down the highway
VROOOM HORN! (custom horn)
Generic vehicle driving...
Diamond resolved: AB
```

### Internal Working
- Default methods were added **specifically to evolve the Collections Framework (Book 05) without breaking every existing implementation** — e.g., `List.sort()`, `Map.getOrDefault()`, `Map.forEach()`, `Iterable.forEach()` are all Java 8 default methods added to interfaces that existed since Java 1.2/1.5, and every pre-existing `List`/`Map` implementation automatically gained these methods for free.
- The **diamond problem** for default methods (two interfaces providing conflicting default implementations of the same signature) is resolved by the compiler **forcing** the implementing class to explicitly override that method — Java refuses to silently pick one, unlike some other languages' multiple-inheritance resolution rules. Inside that override, `InterfaceName.super.methodName()` lets you explicitly invoke a specific parent interface's default implementation.
- Resolution rule when there's no ambiguity: a class's own implementation (or a superclass's concrete method) always wins over any interface default; among interfaces, a more specific (sub-)interface's default wins over a more general one.

### Real-World Example
Telugu: `Comparator.reversed()`, `Comparator.thenComparing()` (Book 05, Ch.10) — ఇవన్నీ Java 8 default methods, `Comparator` interface కి add అయ్యి, existing `Comparator` implementations break కాకుండా. `Iterable.forEach()` కూడా ఇదే కారణం తో add అయ్యింది.
English: `Comparator.reversed()`/`.thenComparing()` (Book 05, Ch.10) and `Iterable.forEach()` are real, concrete examples of Java 8 default methods added to decades-old interfaces without breaking a single existing implementation anywhere in the Java ecosystem — the entire practical justification for the feature, seen in the exact APIs you already used in Book 05.

### Interview Answer
"Default methods let interfaces provide a method body, so new methods can be added to existing interfaces without breaking implementers — this was essential for evolving the Collections Framework in Java 8 (e.g., `List.sort()`, `Map.forEach()`). Static interface methods provide utility/factory methods directly on the interface. When two interfaces provide conflicting default methods, the implementing class must explicitly override and resolve the conflict, optionally calling a specific parent via `InterfaceName.super.method()`."

### Cross Questions
- Q: Why were default methods introduced specifically in Java 8, alongside lambdas and streams? → A: Because the Stream API (Ch.6) needed to add new methods (like `stream()`) to the existing `Collection` interface without breaking every pre-existing implementation across the Java ecosystem — default methods were the enabling mechanism for the entire Java 8 functional API rollout.
- Q: What happens if a class implements two interfaces with the SAME default method signature and doesn't override it? → A: Compile error — Java refuses to guess which default to use; the implementing class must explicitly resolve the conflict.
- Q: Can a default method be `private`? → A: Not a `default` method itself, but Java 9 added **private interface methods** (both static and instance) specifically to let default/static methods share common helper logic without exposing it publicly.

### Tricky Questions
- Q: If a class extends another class with a concrete method AND implements an interface with a default method of the same signature, which wins? → A: The class's (or superclass's) concrete method always wins — "class wins over interface" is an unconditional rule, avoiding any ambiguity with the class hierarchy.
- Q: Can a static interface method be inherited/overridden by implementing classes? → A: No — static interface methods belong to the interface itself, are not inherited by implementing classes, and must be called via the interface name (`Vehicle.createDefault()`), analogous to how static methods aren't truly overridden in classes (Book 01, Ch.9).

### Coding Exercise
**L1:** Add a default method to a custom interface and demonstrate both an implementer using it as-is and one overriding it.
**L2:** Add a static factory method to a custom interface, mirroring `Vehicle.createDefault()`.
**L3:** Reproduce the diamond-problem scenario with your own two interfaces, resolving it using `InterfaceName.super.method()`.
**L4 (Interview):** Explain why default methods were essential specifically for the Stream API's introduction.
**L5 (Senior):** Review an interface used across a large codebase and design a new capability as a default method (with a sensible fallback) instead of an abstract method, explaining the backward-compatibility reasoning.
**L6 (Mastery):** Explain, from memory, the full resolution rule set: class-wins-over-interface, and the forced-override rule for conflicting interface defaults.

---

# CHAPTER 6 — Stream API: Fundamentals & Intermediate Operations

### Telugu Explanation
Stream అనేది ఒక **data source నుండి elements sequence** ప్రాసెస్ చేయడానికి ఒక abstraction — collection కాదు, data ని **store చేయదు**, ఒక్కసారి మాత్రమే traverse చేయచ్చు (**lazy, one-time-use**). Stream operations రెండు రకాలు: **Intermediate** (`filter`, `map`, `sorted`, `distinct`, `limit`, `flatMap` — మరో stream return చేస్తాయి, **lazy** గా execute అవుతాయి) మరియు **Terminal** (Ch.7 లో — actual result produce చేస్తాయి, pipeline ని trigger చేస్తాయి).

### Professional English Explanation
A `Stream` is an abstraction for processing a sequence of elements from a data source — it is **not** a collection, stores no data itself, and can be traversed exactly once (**lazy, single-use**). Stream operations split into **Intermediate** (`filter`, `map`, `sorted`, `distinct`, `limit`, `skip`, `flatMap` — return another `Stream`, execute **lazily**) and **Terminal** operations (Ch.7 — produce an actual result and trigger the entire pipeline to run).

### Java Code

```java
import java.util.*;
import java.util.stream.*;

public class StreamIntermediateDemo {
    record Employee(String name, String department, double salary) {}    // Java 16+ record, previewed here (Ch.13)

    public static void main(String[] args) {
        List<Employee> employees = List.of(
                new Employee("Ravi", "Engineering", 70000),
                new Employee("Anitha", "Engineering", 85000),
                new Employee("Kiran", "Sales", 60000),
                new Employee("Bob", "Sales", 60000),
                new Employee("Alice", "HR", 55000)
        );

        List<String> engineeringNamesSorted = employees.stream()
                .filter(e -> e.department().equals("Engineering"))    // intermediate: keep matching elements
                .map(Employee::name)                                     // intermediate: transform each element
                .sorted()                                                  // intermediate: sort
                .collect(Collectors.toList());                              // terminal (Ch.7)
        System.out.println("Engineering names, sorted: " + engineeringNamesSorted);

        List<Double> distinctSalaries = employees.stream()
                .map(Employee::salary)
                .distinct()                                                 // intermediate: remove duplicates
                .sorted(Comparator.reverseOrder())
                .limit(2)                                                     // intermediate: keep only first 2
                .collect(Collectors.toList());
        System.out.println("Top 2 distinct salaries: " + distinctSalaries);

        // flatMap: flattening nested structures
        List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3, 4), List.of(5));
        List<Integer> flat = nested.stream()
                .flatMap(List::stream)                                        // one-to-many: List<Integer> -> Stream<Integer>
                .collect(Collectors.toList());
        System.out.println("Flattened: " + flat);

        // Laziness demonstration
        System.out.println("Building pipeline (nothing runs yet)...");
        Stream<String> lazyPipeline = employees.stream()
                .peek(e -> System.out.println("peek saw: " + e.name()))       // intermediate, for debugging
                .filter(e -> e.salary() > 65000)
                .map(Employee::name);
        System.out.println("Pipeline built, but NOTHING has executed yet.");
        List<String> result = lazyPipeline.collect(Collectors.toList());        // NOW it all executes
        System.out.println("Result: " + result);
    }
}
```

### Output
```
Engineering names, sorted: [Anitha, Ravi]
Top 2 distinct salaries: [85000.0, 70000.0]
Flattened: [1, 2, 3, 4, 5]
Building pipeline (nothing runs yet)...
Pipeline built, but NOTHING has executed yet.
peek saw: Ravi
peek saw: Anitha
peek saw: Kiran
peek saw: Bob
peek saw: Alice
Result: [Ravi, Anitha]
```

### Internal Working
- **Laziness** is the single most important Stream mental model: intermediate operations don't execute anything by themselves — they just build up a description of the pipeline. Only when a **terminal** operation is invoked does the entire pipeline actually run, processing each element through every intermediate step **one at a time** (not stage-by-stage across the whole collection) — this is why the `peek` output interleaves rather than running `filter` across everything first.
- A stream can only be consumed **once** — calling a terminal operation twice on the same stream throws `IllegalStateException("stream has already been operated upon or closed")`; you must build a fresh stream (`.stream()` again) for each pipeline.
- `flatMap` is for the "one-to-many" case (each input element maps to zero or more output elements, flattened into a single stream), while `map` is strictly "one-to-one" — confusing these two is a very common mistake worth internalizing early.

### Real-World Example
Telugu: Nested JSON structures (ఉదా. `List<Order>`, ప్రతి order కి `List<OrderItem>`) ని process చేసేటప్పుడు, అన్ని order items ని ఒకే flat stream గా చూడాలంటే `flatMap` వాడతారు — ఇది e-commerce/reporting code లో అత్యంత common pattern.
English: Processing nested structures (a `List<Order>`, each with its own `List<OrderItem>`) and needing a single flat stream of all items across all orders is the classic real-world `flatMap` use case — extremely common in e-commerce and reporting code.

### Interview Answer
"A Stream processes a sequence of elements from a source without storing data itself, and can only be traversed once. Intermediate operations (`filter`, `map`, `sorted`, `flatMap`, `distinct`, `limit`) are lazy and return another Stream, building up a pipeline description; nothing actually executes until a terminal operation (Ch.7) is invoked, at which point each element flows through the entire pipeline one at a time."

### Cross Questions
- Q: Why is Stream laziness important? → A: It allows short-circuiting (e.g., `findFirst()` can stop after one match without processing the rest) and lets the JVM potentially fuse/optimize the whole pipeline, rather than materializing an intermediate collection at every stage.
- Q: What happens if you call a terminal operation twice on the same stream? → A: `IllegalStateException` — a stream is single-use; you must create a new stream from the source for each pipeline execution.
- Q: What's the difference between `map` and `flatMap`? → A: `map` transforms each element one-to-one; `flatMap` transforms each element into a stream of zero-or-more elements and flattens all of them into one combined stream.

### Tricky Questions
- Q: Does an intermediate operation like `filter()` actually run when you call it, or only later? → A: Only later — calling `.filter(...)` just builds up the pipeline description; the actual predicate testing happens element-by-element only when a terminal operation drives the whole pipeline.
- Q: If you build a stream pipeline but never call a terminal operation, does anything execute? → A: No — absolutely nothing runs; the pipeline is inert until a terminal operation (Ch.7) triggers it, which is why forgetting the terminal operation (a common beginner mistake) silently does nothing.

### Coding Exercise
**L1:** Build a stream pipeline filtering, mapping, and sorting a list of custom objects.
**L2:** Use `flatMap` to flatten a `List<List<String>>` into a single sorted, distinct `List<String>`.
**L3:** Add `.peek()` calls at 2 points in a pipeline to observe exactly when and in what order elements are processed.
**L4 (Interview):** Explain Stream laziness and why nothing executes until a terminal operation is called.
**L5 (Senior):** Refactor a nested-loop flattening algorithm (list of lists processed with nested for-loops) into a clean `flatMap`-based pipeline.
**L6 (Mastery):** Explain, from memory, why streams are single-use and what exception occurs if you violate that.

---

# CHAPTER 7 — Stream Terminal Operations & Collectors

### Telugu Explanation
Terminal operations Stream pipeline ని **actually execute చేసి**, final result produce చేస్తాయి: `collect()`, `forEach()`, `reduce()`, `count()`, `anyMatch()`/`allMatch()`/`noneMatch()`, `findFirst()`/`findAny()`. `Collectors` utility class `collect()` తో కలిపి వాడే powerful, composable collection strategies ఇస్తుంది: `toList()`, `toMap()`, `joining()`, `groupingBy()`, `partitioningBy()`.

### Professional English Explanation
Terminal operations actually execute the stream pipeline and produce a final result: `collect()`, `forEach()`, `reduce()`, `count()`, `anyMatch()`/`allMatch()`/`noneMatch()`, `findFirst()`/`findAny()`. The `Collectors` utility class provides powerful, composable strategies to pair with `collect()`: `toList()`, `toMap()`, `joining()`, `groupingBy()`, and `partitioningBy()`.

### Java Code

```java
import java.util.*;
import java.util.stream.*;

public class StreamTerminalCollectorsDemo {
    record Employee(String name, String department, double salary) {}

    public static void main(String[] args) {
        List<Employee> employees = List.of(
                new Employee("Ravi", "Engineering", 70000),
                new Employee("Anitha", "Engineering", 85000),
                new Employee("Kiran", "Sales", 60000),
                new Employee("Bob", "Sales", 60000),
                new Employee("Alice", "HR", 55000)
        );

        // reduce: combine all elements into one result
        double totalSalary = employees.stream()
                .mapToDouble(Employee::salary)
                .reduce(0.0, Double::sum);                                     // identity + accumulator
        System.out.println("Total salary (reduce): " + totalSalary);

        // Matching operations (short-circuiting)
        System.out.println("Any Sales employee? " + employees.stream().anyMatch(e -> e.department().equals("Sales")));
        System.out.println("All earn > 50000? " + employees.stream().allMatch(e -> e.salary() > 50000));
        System.out.println("None earn > 100000? " + employees.stream().noneMatch(e -> e.salary() > 100000));

        // Collectors.joining
        String allNames = employees.stream().map(Employee::name).collect(Collectors.joining(", ", "[", "]"));
        System.out.println("Joined names: " + allNames);

        // Collectors.groupingBy - group by department
        Map<String, List<Employee>> byDept = employees.stream()
                .collect(Collectors.groupingBy(Employee::department));
        System.out.println("Grouped by department: " + byDept.keySet());

        // groupingBy with downstream collector - count per department
        Map<String, Long> countByDept = employees.stream()
                .collect(Collectors.groupingBy(Employee::department, Collectors.counting()));
        System.out.println("Count per department: " + countByDept);

        // groupingBy with downstream - average salary per department
        Map<String, Double> avgSalaryByDept = employees.stream()
                .collect(Collectors.groupingBy(Employee::department, Collectors.averagingDouble(Employee::salary)));
        System.out.println("Avg salary per department: " + avgSalaryByDept);

        // partitioningBy - splits into exactly TWO groups: true/false
        Map<Boolean, List<Employee>> partitionedBySalary = employees.stream()
                .collect(Collectors.partitioningBy(e -> e.salary() > 65000));
        System.out.println("High earners: " + partitionedBySalary.get(true).size()
                + ", Others: " + partitionedBySalary.get(false).size());

        // toMap - build a Map directly
        Map<String, Double> nameToSalary = employees.stream()
                .collect(Collectors.toMap(Employee::name, Employee::salary));
        System.out.println("Name to salary map size: " + nameToSalary.size());
    }
}
```

### Output
```
Total salary (reduce): 330000.0
Any Sales employee? true
All earn > 50000? true
None earn > 100000? true
Joined names: [Ravi, Anitha, Kiran, Bob, Alice]
Grouped by department: [Engineering, Sales, HR]
Count per department: {Engineering=2, HR=1, Sales=2}
Avg salary per department: {Engineering=77500.0, HR=55000.0, Sales=60000.0}
High earners: 2, Others: 3
Name to salary map size: 5
```

### Internal Working
- `reduce(identity, accumulator)` folds the stream into a single value by repeatedly applying the accumulator function — `identity` is both the starting value AND the correct result for an empty stream, which is why it must be a true "no-op" value for the operation (0 for sum, 1 for product, empty string for concatenation).
- `anyMatch`/`allMatch`/`noneMatch`/`findFirst`/`findAny` are **short-circuiting** terminal operations — they can stop processing as soon as the answer is determined (e.g., `anyMatch` stops at the first match), which is a genuine performance benefit over always processing the entire stream, especially combined with laziness (Ch.6).
- `groupingBy(classifier, downstream)` is a **two-level** collector: the classifier decides the group (map key), and the downstream collector decides how each group's elements are combined (`counting()`, `averagingDouble()`, `toList()` is the default if omitted, `mapping()`, `summingDouble()`, etc.) — this composability is what makes `Collectors` so powerful for real reporting logic.
- `toMap()` throws `IllegalStateException` at runtime if the key function produces **duplicate keys** — a well-known gotcha requiring a merge function (`Collectors.toMap(keyFn, valueFn, (v1, v2) -> v1)`) when duplicates are possible.

### Real-World Example
Telugu: Sales report generate చేయడానికి — department wise employee count, average salary, total revenue — ఇవన్నీ `groupingBy` + downstream collectors తో ఒక్క, readable pipeline లో express చేయచ్చు, పాత nested-map-building code కంటే చాలా క్లుప్తంగా.
English: Generating a sales report — headcount per department, average salary per department, total revenue per category — is exactly the `groupingBy` + downstream-collector use case, expressible in one short, readable pipeline instead of the old manual nested-map-building loop code.

### Interview Answer
"Terminal operations trigger the pipeline and produce a result: `collect()`, `reduce()`, `count()`, the short-circuiting `anyMatch`/`allMatch`/`noneMatch`/`findFirst`/`findAny`. `Collectors.groupingBy()` groups elements by a classifier function, optionally with a downstream collector (`counting()`, `averagingDouble()`, `toList()`) to summarize each group; `partitioningBy()` is a specialized binary grouping into `true`/`false` buckets. `toMap()` requires care with duplicate keys, which throw at runtime unless a merge function is supplied."

### Cross Questions
- Q: Why must `reduce()`'s identity value be a genuine "no-op" for the operation? → A: Because it's also returned as-is for an empty stream — using the wrong identity (e.g., `1` for a sum) would silently produce an incorrect result for empty input.
- Q: What's the difference between `groupingBy` and `partitioningBy`? → A: `groupingBy` can produce any number of groups based on a classifier function's distinct results; `partitioningBy` always produces exactly two groups (`true`/`false`) based on a `Predicate`.
- Q: What happens if `Collectors.toMap()` encounters duplicate keys without a merge function? → A: It throws `IllegalStateException` at runtime — you must supply a third argument (a `BinaryOperator<V>` merge function) to resolve duplicates.

### Tricky Questions
- Q: Is `findFirst()` guaranteed to always return the same element as `findAny()` on the same stream? → A: For a sequential stream, they're typically equivalent; for a **parallel** stream (Ch.8), `findAny()` may return any matching element (whichever thread finds one first) for better performance, while `findFirst()` always respects encounter order at extra cost.
- Q: Can `groupingBy()`'s classifier function return `null`? → A: No — attempting to group by a key function that returns `null` throws `NullPointerException`, since the resulting `HashMap` (Book 05, Ch.6) implementation used internally handles `null` differently across JDK versions and `groupingBy` explicitly disallows it for reliability.

### Coding Exercise
**L1:** Use `reduce()` to compute the product of a list of integers, being careful to choose the correct identity value.
**L2:** Build a report using `groupingBy` + `counting()` and `groupingBy` + `averagingDouble()` on a custom domain (e.g., orders by status).
**L3:** Reproduce the `toMap()` duplicate-key `IllegalStateException` and fix it with a merge function.
**L4 (Interview):** Explain the difference between `groupingBy` and `partitioningBy` with an example each.
**L5 (Senior):** Design a full sales report (department headcount, average salary, top earner per department) using chained/nested `Collectors`.
**L6 (Mastery):** Explain, from memory, why `anyMatch`/`allMatch`/`noneMatch`/`findFirst` are short-circuiting and how that connects to Ch.6's laziness.

---

# CHAPTER 8 — Parallel Streams

### Telugu Explanation
`.parallelStream()` (లేదా `.stream().parallel()`) collection ని **multiple threads** meీద, internally **ForkJoinPool** వాడి, process చేయడానికి వీలు కల్పిస్తుంది — పెద్ద data sets meీద throughput పెంచడానికి. కానీ ఇది ఎప్పుడూ better కాదు — overhead, thread-safety, ordering సమస్యలు ఉండవచ్చు.

### Professional English Explanation
`.parallelStream()` (or `.stream().parallel()`) processes a collection across **multiple threads**, internally using the common **ForkJoinPool**, to improve throughput on large datasets. It is not automatically faster — overhead from splitting/merging work, thread-safety requirements on shared state, and ordering considerations mean parallel streams need deliberate, informed use, not blind application.

### Java Code

```java
import java.util.*;
import java.util.stream.*;

public class ParallelStreamsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = IntStream.rangeClosed(1, 10_000_000).boxed().collect(Collectors.toList());

        long startSeq = System.nanoTime();
        long sumSequential = numbers.stream().mapToLong(Integer::longValue).sum();
        long timeSeq = System.nanoTime() - startSeq;

        long startPar = System.nanoTime();
        long sumParallel = numbers.parallelStream().mapToLong(Integer::longValue).sum();
        long timePar = System.nanoTime() - startPar;

        System.out.println("Sequential sum: " + sumSequential + " in " + timeSeq / 1_000_000 + " ms");
        System.out.println("Parallel sum:   " + sumParallel + " in " + timePar / 1_000_000 + " ms");

        // DANGER: non-thread-safe shared mutable state in a parallel stream
        List<Integer> unsafeResults = Collections.synchronizedList(new ArrayList<>());   // must be synchronized!
        numbers.stream().limit(1000).parallel().forEach(unsafeResults::add);               // safe ONLY because of synchronizedList
        System.out.println("Unsafe-if-not-synchronized collection size: " + unsafeResults.size());

        // Ordering: forEach does NOT guarantee order in parallel; forEachOrdered does (at a performance cost)
        System.out.print("parallel forEach (order not guaranteed): ");
        IntStream.rangeClosed(1, 5).parallel().forEach(i -> System.out.print(i + " "));
        System.out.println();
        System.out.print("parallel forEachOrdered (order guaranteed): ");
        IntStream.rangeClosed(1, 5).parallel().forEachOrdered(i -> System.out.print(i + " "));
        System.out.println();
    }
}
```

### Output (illustrative — timings and unordered output vary by machine/run)
```
Sequential sum: 50000005000000 in 45 ms
Parallel sum:   50000005000000 in 18 ms
Unsafe-if-not-synchronized collection size: 1000
parallel forEach (order not guaranteed): 3 1 4 2 5 
parallel forEachOrdered (order guaranteed): 1 2 3 4 5 
```

### Internal Working
- Parallel streams split the source into chunks (using the `Spliterator` abstraction) and process them across the **common `ForkJoinPool`** (shared JVM-wide, sized to the number of available CPU cores by default) — this is the **same pool** used by `CompletableFuture`'s default async methods (Ch.11), which means CPU-bound parallel streams and `CompletableFuture` tasks can contend with each other if not managed carefully.
- Using a regular (non-thread-safe) collection as a shared accumulation target inside a parallel stream's `forEach` is a genuine, common concurrency bug — `ArrayList.add()` is not thread-safe (Book 05, Ch.2), and concurrent parallel writes can corrupt it silently; `collect()` with a proper `Collector` (which internally handles combining partial results safely) is almost always the correct approach instead of manual `forEach` + shared mutable list.
- Parallel streams have real overhead (splitting, thread coordination, merging results) that **only pays off for large datasets with computationally expensive per-element work** — for small collections or cheap operations, parallel streams can be **slower** than sequential due to this overhead, a genuine and common misconception to correct.

### Real-World Example
Telugu: Millions of records meీద CPU-intensive computation (image processing, complex calculations) చేయాలంటే parallel streams throughput పెంచుతాయి; కానీ చిన్న lists meీద, లేదా I/O-bound operations (DB calls, HTTP calls) కి parallel streams సరైన tool కాదు — ఆ scenarios కి `CompletableFuture` (Ch.11) లేదా proper thread pools better.
English: CPU-intensive computation (image processing, heavy calculations) over millions of records is where parallel streams genuinely help; for small lists, or I/O-bound work (database calls, HTTP calls), parallel streams are the wrong tool entirely — `CompletableFuture` (Ch.11) or a dedicated, properly-sized thread pool (Book 08) is the correct approach for I/O-bound concurrency.

### Interview Answer
"Parallel streams process a collection across multiple threads using the common ForkJoinPool, potentially improving throughput for large, CPU-intensive workloads. They are not automatically faster — splitting/merging overhead can make them slower for small collections or cheap operations, shared mutable state used inside them must be properly thread-safe, and `forEach` doesn't guarantee element order (`forEachOrdered` does, at a cost)."

### Deep Interview Answer
"A critical, often-missed detail: parallel streams share the JVM's single common `ForkJoinPool` by default — the same pool `CompletableFuture.supplyAsync()` (Ch.11) uses without an explicit executor. Running blocking I/O operations inside a parallel stream can starve that shared pool, degrading unrelated parallel work elsewhere in the application (including any concurrent `CompletableFuture` chains) — this is exactly why parallel streams are recommended only for CPU-bound work, and I/O-bound concurrency should use a separate, explicitly-managed thread pool (Book 08)."

### Cross Questions
- Q: What pool do parallel streams use by default? → A: The JVM's shared common `ForkJoinPool`, sized to the number of available processors by default.
- Q: Why is using a plain `ArrayList` inside a parallel `forEach` dangerous? → A: `ArrayList` isn't thread-safe (Book 05, Ch.2); concurrent adds from multiple threads can corrupt its internal state or lose elements — use `collect()` with a proper collector, or a genuinely thread-safe/concurrent collection instead.
- Q: When should you avoid parallel streams? → A: For small collections, cheap per-element operations, I/O-bound work, or when you're already inside another parallel/concurrent context that could contend for the same shared pool.

### Tricky Questions
- Q: Does `.parallel()` guarantee actual multi-threaded execution for every stream? → A: Not strictly — for very small streams, the JVM/ForkJoinPool implementation may still execute everything on the calling thread, since the overhead of splitting wouldn't be worth it; "parallel" is a hint enabling parallelism, not an absolute guarantee of multi-threaded execution for trivial cases.
- Q: If you run a blocking I/O call inside a `.parallelStream().forEach()`, what's the risk beyond just "it's the wrong tool"? → A: It can starve the shared common `ForkJoinPool`, since a limited number of pool threads get blocked waiting on I/O instead of doing CPU work — this can degrade the performance of *other, unrelated* parallel streams or `CompletableFuture` chains running elsewhere in the same JVM at the same time.

### Coding Exercise
**L1:** Benchmark sequential vs parallel stream sum on a small (1,000) and large (10,000,000) list, and compare the results.
**L2:** Reproduce the unsafe shared-`ArrayList` bug in a parallel `forEach`, then fix it using `Collectors.toList()` via `collect()` instead.
**L3:** Compare `forEach` vs `forEachOrdered` output ordering on a parallel `IntStream`.
**L4 (Interview):** Explain why parallel streams are not automatically faster, and name at least 2 scenarios where they should be avoided.
**L5 (Senior):** Explain the shared common-`ForkJoinPool` risk between parallel streams and `CompletableFuture`, and how you'd mitigate it in a production service doing both.
**L6 (Mastery):** Explain, from memory, why I/O-bound work should never run inside a parallel stream's default configuration.

---

# CHAPTER 9 — Optional: Deep Dive

### Telugu Explanation
`Optional<T>` అనేది "value ఉండొచ్చు, ఉండకపోవచ్చు" అని **explicitly, type-level లో** express చేయడానికి Java 8 తీసుకొచ్చిన container class — `null` return చేయడం బదులు, method signature లోనే "ఈ value missing అవ్వొచ్చు" అని caller కి తెలియజేస్తుంది, NullPointerException risk ని design-level లో తగ్గిస్తుంది.

### Professional English Explanation
`Optional<T>` is a container type introduced in Java 8 to **explicitly, at the type level**, signal that a value might be absent — instead of returning `null` (which callers can easily forget to check), a method's return type of `Optional<T>` documents in the signature itself that the value may be missing, reducing `NullPointerException` risk by design rather than by convention.

### Java Code

```java
import java.util.*;

public class OptionalDeepDiveDemo {
    record User(String id, String email) {}

    static Optional<User> findUserById(String id, List<User> users) {
        return users.stream().filter(u -> u.id().equals(id)).findFirst();     // findFirst() already returns Optional
    }

    public static void main(String[] args) {
        List<User> users = List.of(new User("U1", "ravi@example.com"), new User("U2", "anitha@example.com"));

        Optional<User> found = findUserById("U1", users);
        Optional<User> notFound = findUserById("U999", users);

        // Idiomatic consumption patterns
        System.out.println("isPresent: " + found.isPresent());
        System.out.println("isEmpty: " + notFound.isEmpty());                  // Java 11+

        found.ifPresent(u -> System.out.println("Found email: " + u.email()));
        notFound.ifPresentOrElse(
                u -> System.out.println("Found: " + u.email()),
                () -> System.out.println("No user found - ifPresentOrElse fallback")   // Java 9+
        );

        String email = notFound.map(User::email).orElse("no-reply@default.com");        // safe default
        System.out.println("Email with default: " + email);

        String emailLazy = notFound.map(User::email).orElseGet(() -> computeExpensiveDefault());  // lazy default
        System.out.println("Email with lazy default: " + emailLazy);

        try {
            notFound.orElseThrow(() -> new NoSuchElementException("User not found!"));    // custom exception
        } catch (NoSuchElementException e) {
            System.out.println("Caught: " + e.getMessage());
        }

        // Chaining with map/filter, avoiding nested null checks entirely
        String domain = found
                .map(User::email)
                .filter(e -> e.contains("@"))
                .map(e -> e.substring(e.indexOf('@') + 1))
                .orElse("unknown");
        System.out.println("Domain: " + domain);
    }

    static String computeExpensiveDefault() {
        System.out.println("(computing expensive default - only happens when actually needed)");
        return "computed@default.com";
    }
}
```

### Output
```
isPresent: true
isEmpty: true
Found email: ravi@example.com
No user found - ifPresentOrElse fallback
Email with default: no-reply@default.com
(computing expensive default - only happens when actually needed)
Email with lazy default: computed@default.com
Domain: example.com
```

### Internal Working & Best Practices
- **`orElse()` vs `orElseGet()`**: `orElse(value)` **always evaluates its argument eagerly**, even when the `Optional` is present and the default won't be used — `orElseGet(supplier)` only invokes the `Supplier` (Ch.3) **lazily**, when actually needed. This matters when the default is expensive to compute (a DB call, a heavy calculation) — using `orElse()` there wastes work on every present case, a genuine, common performance mistake.
- `Optional` chaining (`map().filter().map()`) lets you express a chain of "if present, then..." logic without any nested null checks — internally, each `map`/`filter` call simply short-circuits to an empty `Optional` the moment the value is absent or fails the filter, propagating "absence" cleanly through the whole chain.
- **Never use `Optional` as a field type, a method parameter type, or inside a collection** (`List<Optional<T>>`) — this is explicitly discouraged by the JDK's own design intent and documented by its author; `Optional`'s intended, idiomatic use is exclusively as a **method return type**, to communicate "this specific call might not have a result" to the caller.

### Real-World Example
Telugu: Repository methods (Book 05 Ch.6, Book 13) `Optional<User> findById(String id)` return చేయడం — caller code `null` check మర్చిపోయే ప్రమాదం లేకుండా, compiler/API design level లోనే "ఇది missing అవ్వొచ్చు" అని force చేస్తుంది.
English: Repository methods (Book 05, Book 13) returning `Optional<User> findById(String id)` is the exact idiomatic pattern this chapter builds toward — the API design itself, not just documentation or convention, forces callers to consciously handle the "might be missing" case, directly reducing a whole category of real NullPointerException bugs.

### Interview Answer
"`Optional<T>` explicitly signals at the type level that a value might be absent, replacing ad-hoc `null` returns that callers can forget to check. Idiomatic use is as a method return type only — never as a field, parameter, or inside a collection. `orElse()` eagerly evaluates its default even when unused; `orElseGet()` lazily computes it only when needed, which matters for expensive defaults."

### Deep Interview Answer
"`Optional` doesn't eliminate `null` from Java — it's a disciplined design pattern, not a language-enforced guarantee (you can still assign `null` to an `Optional<T>` variable itself, which defeats its purpose entirely and should never be done). Its real value is making the 'might be absent' case a first-class, impossible-to-ignore part of a method's contract, and enabling clean functional-style chaining (`map`/`filter`/`orElseGet`) that replaces nested null-check pyramids — a genuine readability and safety improvement when used according to its intended pattern, but actively harmful when misused as a general-purpose null-replacement (e.g., as a field type, which just adds an unnecessary wrapper-object allocation for no real benefit)."

### Cross Questions
- Q: What's the difference between `orElse()` and `orElseGet()`? → A: `orElse(value)` always evaluates `value` eagerly, even if unused; `orElseGet(supplier)` only calls the supplier lazily, when the Optional is actually empty — critical for expensive default computations.
- Q: Should `Optional` be used as a method parameter type? → A: No — this is explicitly discouraged; use overloading or `null` (with proper documentation) for parameters, reserving `Optional` for return types where the "might be absent" contract genuinely helps the caller.
- Q: Can an `Optional<T>` variable itself be `null`? → A: Yes, unfortunately — nothing prevents `Optional<String> opt = null;`, which is why it should never happen in well-written code, since it completely defeats `Optional`'s purpose (you'd need a null check on the Optional itself).

### Tricky Questions
- Q: Does `Optional.of(null)` work? → A: No — it throws `NullPointerException` immediately; use `Optional.ofNullable(value)` if the value might genuinely be `null` at that point, which correctly produces an empty `Optional` instead.
- Q: Is it good practice to call `.get()` directly on an `Optional` without checking `.isPresent()` first? → A: No — `.get()` throws `NoSuchElementException` if empty, and calling it unconditionally reintroduces exactly the same "forgot to check" risk `Optional` was designed to prevent; prefer `orElse`/`orElseGet`/`ifPresent`/`map` idioms instead.

### Coding Exercise
**L1:** Write a repository-style method returning `Optional<T>` and consume it with `ifPresent`, `orElse`, and `orElseThrow`.
**L2:** Demonstrate the `orElse()` vs `orElseGet()` eager-vs-lazy distinction with a "default" method that prints when it's called.
**L3:** Rewrite a nested null-check pyramid (3+ levels) using `Optional`'s `map`/`filter` chaining.
**L4 (Interview):** Explain exactly why `Optional` should never be used as a field type or method parameter.
**L5 (Senior):** Review a codebase using `Optional.get()` everywhere without checks — propose a remediation plan using the idiomatic consumption patterns from this chapter.
**L6 (Mastery):** Explain, from memory, why `Optional` is a design pattern rather than a language-level null-safety guarantee, and what its one intended use case is.

---

# CHAPTER 10 — Java 8 Date/Time API

### Telugu Explanation
Java 8 కి ముందు `java.util.Date`/`Calendar` classes **mutable, thread-unsafe, confusing** గా ఉండేవి (ఉదా. months 0-indexed). Java 8 తీసుకొచ్చిన `java.time` package (`LocalDate`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`, `Duration`, `Period`) **immutable, thread-safe, intuitive** APIs ఇస్తుంది.

### Professional English Explanation
Before Java 8, `java.util.Date`/`Calendar` were mutable, not thread-safe, and famously confusing (e.g., 0-indexed months). Java 8's `java.time` package (`LocalDate`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`, `Duration`, `Period`) provides immutable, thread-safe, intuitive replacements — modeled closely on the well-regarded third-party Joda-Time library.

### Java Code

```java
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.time.temporal.ChronoUnit;

public class DateTimeApiDemo {
    public static void main(String[] args) {
        LocalDate today = LocalDate.now();
        LocalDate birthday = LocalDate.of(1995, Month.AUGUST, 15);          // months are named/1-indexed, not 0-indexed!
        System.out.println("Today: " + today);
        System.out.println("Birthday: " + birthday);

        Period age = Period.between(birthday, today);
        System.out.println("Age: " + age.getYears() + " years, " + age.getMonths() + " months");

        LocalDateTime meeting = LocalDateTime.of(2026, 9, 1, 14, 30);
        System.out.println("Meeting: " + meeting);

        ZonedDateTime zonedMeeting = meeting.atZone(ZoneId.of("Asia/Kolkata"));
        ZonedDateTime nyTime = zonedMeeting.withZoneSameInstant(ZoneId.of("America/New_York"));
        System.out.println("Meeting in IST: " + zonedMeeting);
        System.out.println("Same instant in New York: " + nyTime);

        Duration processingTime = Duration.ofMinutes(90);
        System.out.println("Processing takes: " + processingTime.toHours() + "h " + (processingTime.toMinutes() % 60) + "m");

        // Immutability in action - every "modification" returns a NEW instance
        LocalDate nextWeek = today.plusWeeks(1);                             // does NOT modify 'today'
        System.out.println("Today unchanged: " + today + ", next week: " + nextWeek);

        long daysBetween = ChronoUnit.DAYS.between(birthday, today);
        System.out.println("Days since birthday: " + daysBetween);

        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd-MMM-yyyy");
        System.out.println("Formatted: " + today.format(formatter));
    }
}
```

### Output (illustrative — depends on actual current date)
```
Today: 2026-08-28
Birthday: 1995-08-15
Age: 31 years, 0 months
Meeting: 2026-09-01T14:30
Meeting in IST: 2026-09-01T14:30+05:30[Asia/Kolkata]
Same instant in New York: 2026-09-01T05:00-04:00[America/New_York]
Processing takes: 1h 30m
Today unchanged: 2026-08-28, next week: 2026-09-04
Days since birthday: 11336
Days since birthday: 11336
Formatted: 28-Aug-2026
```

### Internal Working
- Every `java.time` class is **immutable** — `plusWeeks()`, `plusDays()`, `withZoneSameInstant()`, etc., all return a **new instance**, never modifying the original — the same immutability discipline as `String` (Book 01, Ch.11), chosen specifically to make these classes safely shareable across threads without synchronization (a direct fix for `Date`/`Calendar`'s notorious thread-safety problems).
- `LocalDate`/`LocalTime`/`LocalDateTime` have **no time zone concept at all** — they represent a "local," zone-agnostic date/time (like a wall clock reading); `ZonedDateTime` is specifically for when time zone matters (scheduling meetings across regions, timestamps in logs).
- `Period` measures date-based amounts (years, months, days — calendar-aware, accounting for varying month lengths); `Duration` measures time-based amounts (hours, minutes, seconds — a fixed length of time) — using the wrong one for a given use case (e.g., `Duration` for "1 month," which has no fixed length) is a common, meaningful mistake.

### Real-World Example
Telugu: Multi-region applications లో meeting times, log timestamps schedule చేయాలంటే `ZonedDateTime`/`Instant` వాడాలి, `LocalDateTime` వాడితే time zone ambiguity వల్ల bugs వస్తాయి — ఇది global backend systems లో genuinely common real bug source.
English: Scheduling meetings or timestamping logs across multiple regions requires `ZonedDateTime`/`Instant`, not `LocalDateTime` — using `LocalDateTime` where zone matters is a genuinely common real bug source in global backend systems (e.g., a scheduled job running at the wrong actual moment when servers/users span time zones).

### Interview Answer
"Java 8's `java.time` package replaced the mutable, thread-unsafe, confusing `Date`/`Calendar` classes with immutable, thread-safe alternatives — `LocalDate`/`LocalTime`/`LocalDateTime` for zone-agnostic values, `ZonedDateTime` when time zone matters, `Period` for calendar-based amounts (years/months/days), and `Duration` for fixed time-based amounts (hours/minutes/seconds). Every 'modification' method returns a new instance rather than mutating in place."

### Cross Questions
- Q: Why is `java.time`'s immutability significant beyond just being a design preference? → A: It directly fixes `Date`/`Calendar`'s documented thread-safety problems — immutable objects can be freely shared across threads (Book 08) without synchronization, unlike the old mutable date classes.
- Q: What's the difference between `Period` and `Duration`? → A: `Period` represents calendar-based amounts (years, months, days) that vary in actual length (a month isn't a fixed number of seconds); `Duration` represents a fixed, exact time-based amount (hours, minutes, seconds, nanoseconds).
- Q: When should you use `Instant` instead of `LocalDateTime`? → A: `Instant` represents a single, unambiguous point on the timeline (typically UTC-based, like a timestamp), ideal for logging/storage; `LocalDateTime` has no timezone and is ambiguous across regions — use `Instant`/`ZonedDateTime` whenever the actual moment in time (not just a "wall clock" reading) matters.

### Tricky Questions
- Q: Does calling `today.plusDays(1)` change the `today` variable's value? → A: No — like all `java.time` operations, it returns a new `LocalDate` instance; you must capture the return value (`today = today.plusDays(1);`) if you intend to "update" it, exactly like `String` concatenation (Book 01, Ch.11).
- Q: Is `LocalDate.of(2024, 2, 30)` valid? → A: No — it throws `DateTimeException` at runtime, since `java.time` validates calendar correctness (February never has 30 days) unlike the old `Calendar` class, which would have silently rolled over to an unexpected date.

### Coding Exercise
**L1:** Compute your own age in years/months/days using `Period.between()`.
**L2:** Convert a `ZonedDateTime` in `Asia/Kolkata` to 3 other time zones and print each.
**L3:** Demonstrate immutability by calling 3 chained `plusX()` methods without reassigning, then print the original unchanged value.
**L4 (Interview):** Explain the difference between `Period` and `Duration`, with an example of when using the wrong one would produce an incorrect result.
**L5 (Senior):** Review a global scheduling system using `LocalDateTime` for meeting times across regions — identify the bug risk and redesign using `ZonedDateTime`/`Instant`.
**L6 (Mastery):** Explain, from memory, why `java.time` classes are immutable and how that directly fixes a specific `Date`/`Calendar` problem.

---

# CHAPTER 11 — CompletableFuture Basics

### Telugu Explanation
`CompletableFuture<T>` అనేది Java 8 తీసుకొచ్చిన **asynchronous programming** tool — ఒక task ని background లో (సాధారణంగా common `ForkJoinPool` meీద) run చేసి, దాని result meీద **non-blocking** గా further operations chain చేయడానికి వీలు కల్పిస్తుంది. దీని పూర్తి concurrency mechanics **Book 08** లో వివరంగా — ఇక్కడ basics మాత్రమే.

### Professional English Explanation
`CompletableFuture<T>` is Java 8's asynchronous programming tool — it runs a task in the background (by default on the common `ForkJoinPool`, Ch.8) and lets you chain further operations onto its eventual result **non-blockingly**. Full concurrency mechanics (thread pools, executors, blocking vs non-blocking trade-offs) are covered in **Book 08**; this chapter introduces the basic composition API.

### Java Code

```java
import java.util.concurrent.*;

public class CompletableFutureBasicsDemo {
    static String fetchUserName(String userId) {
        System.out.println("Fetching user on thread: " + Thread.currentThread().getName());
        try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return "User-" + userId;
    }

    static int fetchUserScore(String userName) {
        System.out.println("Fetching score on thread: " + Thread.currentThread().getName());
        return userName.length() * 10;
    }

    public static void main(String[] args) throws Exception {
        // Basic async execution
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> fetchUserName("U1"));
        System.out.println("Main thread continues immediately, not blocked: " + Thread.currentThread().getName());
        System.out.println("Result (blocking here only to print): " + future.get());              // .get() DOES block

        // Chaining with thenApply (transform result, non-blocking chain)
        CompletableFuture<Integer> chained = CompletableFuture.supplyAsync(() -> fetchUserName("U2"))
                .thenApply(CompletableFutureBasicsDemo::fetchUserScore);
        System.out.println("Chained score: " + chained.get());

        // Combining two independent futures
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");
        CompletableFuture<String> combined = f1.thenCombine(f2, (s1, s2) -> s1 + " " + s2);
        System.out.println("Combined: " + combined.get());

        // Exception handling
        CompletableFuture<String> risky = CompletableFuture.supplyAsync(() -> {
            if (true) throw new RuntimeException("Simulated failure");
            return "never reached";
        }).exceptionally(ex -> "Fallback value due to: " + ex.getCause().getMessage());
        System.out.println("Risky result: " + risky.get());

        // thenAccept (consume result, no return) and thenRun (ignore result entirely)
        CompletableFuture.supplyAsync(() -> "final data")
                .thenAccept(data -> System.out.println("Consumed: " + data))
                .thenRun(() -> System.out.println("Pipeline complete"))
                .get();
    }
}
```

### Output (illustrative — thread names vary)
```
Main thread continues immediately, not blocked: main
Fetching user on thread: ForkJoinPool.commonPool-worker-1
Result (blocking here only to print): User-U1
Fetching user on thread: ForkJoinPool.commonPool-worker-1
Fetching score on thread: main
Chained score: 70
Combined: Hello World
Risky result: Fallback value due to: Simulated failure
Consumed: final data
Pipeline complete
```

### Internal Working
- `supplyAsync()` submits the task to the common `ForkJoinPool` (Ch.8) by default, immediately returning a `CompletableFuture` handle **without blocking the calling thread** — the actual work happens on a separate pool thread; only calling `.get()` (or `.join()`) blocks the *calling* thread until the result is ready.
- `thenApply`/`thenAccept`/`thenRun`/`thenCombine` are all **non-blocking chaining** methods — they register a continuation to run once the prior stage completes, without the calling thread waiting; whether the continuation itself runs on the same worker thread or a new one depends on timing (if the prior stage already completed, it may run immediately on the calling thread; otherwise, the pool schedules it).
- `exceptionally()` provides a **fallback** value if any prior stage in the chain throws — analogous to a `catch` block (Book 04) for asynchronous pipelines, letting you recover gracefully instead of the exception propagating out of `.get()` as an `ExecutionException`.

### Real-World Example
Telugu: Multiple independent backend calls (user info, recommendations, notifications) ఏకకాలంలో fetch చేసి, అన్నీ complete అయ్యాక ఒక్కటే response build చేయాలంటే `CompletableFuture.allOf()`/`thenCombine()` వాడతారు — ఇది microservices (Book 16) లో అత్యంత common pattern.
English: Fetching multiple independent backend resources (user profile, recommendations, notifications) concurrently, then assembling one combined response once all complete, is exactly the `CompletableFuture.allOf()`/`thenCombine()` use case — an extremely common pattern in microservices architectures (Book 16) needing to minimize total request latency.

### Interview Answer
"`CompletableFuture<T>` runs a task asynchronously (by default on the common ForkJoinPool) and lets you chain transformations (`thenApply`), consumers (`thenAccept`), combinations of independent futures (`thenCombine`), and exception recovery (`exceptionally`) without blocking the calling thread — only `.get()`/`.join()` actually blocks, and only the thread that calls it. Full concurrency mechanics — executors, thread pools, blocking vs non-blocking design — are covered in Book 08."

### Cross Questions
- Q: Does calling `supplyAsync()` block the calling thread? → A: No — it returns immediately with a `CompletableFuture` handle; only `.get()`/`.join()` blocks, and only when actually called.
- Q: What pool does `CompletableFuture` use by default, and why does that matter? → A: The common `ForkJoinPool` (Ch.8) — the same one parallel streams use by default, meaning CPU-bound work in one can starve the other if not managed with explicit, separate executors for I/O-bound work.
- Q: What's the difference between `thenApply` and `thenAccept`? → A: `thenApply` transforms the result into a new value (returns a `CompletableFuture<R>`); `thenAccept` consumes the result without producing a new value (returns `CompletableFuture<Void>`).

### Tricky Questions
- Q: If a `CompletableFuture` chain has `exceptionally()` handling a failure, does execution continue to any `thenApply` stages that follow it? → A: Yes — once `exceptionally()` supplies a fallback value, the chain is considered "recovered," and subsequent stages proceed normally using that fallback value, exactly like code after a `catch` block continuing normally (Book 04).
- Q: Is `.get()` or `.join()` preferred, and why? → A: `.join()` is often preferred in stream/lambda contexts because it throws an unchecked `CompletionException` instead of `.get()`'s checked `ExecutionException`/`InterruptedException`, avoiding forced try-catch or `throws` declarations (Book 04) in places where that's inconvenient.

### Coding Exercise
**L1:** Chain `supplyAsync()` → `thenApply()` → `thenAccept()` for a simple string-processing pipeline.
**L2:** Combine two independent `CompletableFuture`s with `thenCombine()`.
**L3:** Reproduce an exception inside a `CompletableFuture` chain and recover using `exceptionally()`.
**L4 (Interview):** Explain why `supplyAsync()` doesn't block the calling thread but `.get()` does.
**L5 (Senior):** Design a service method fetching 3 independent pieces of data concurrently and combining them into one response, using `CompletableFuture.allOf()` conceptually.
**L6 (Mastery):** Explain, from memory, the shared-common-pool risk between `CompletableFuture` and parallel streams (Ch.8), and how you'd mitigate it in production (previewing Book 08).

---

# CHAPTER 12 — Java 11 LTS: Practical Everyday Features

### Telugu Explanation
Java 11 (LTS) Java 8 తర్వాత రెండవ major LTS release — చాలా revolutionary features లేకపోయినా, daily-use quality-of-life improvements చాలా ఇచ్చింది: `var` (local variable type inference, actually Java 10 లో వచ్చింది కానీ Java 11 తో పాటు widely adopted), కొత్త `String` methods (`isBlank()`, `strip()`, `repeat()`, `lines()`), కొత్త `Collection.toArray()` overload, standard `HttpClient` (built-in, `java.net.http`).

### Professional English Explanation
Java 11 (LTS) is the second major LTS release after Java 8 — not revolutionary, but full of practical, daily-use improvements: `var` (local variable type inference, technically Java 10 but widely adopted alongside 11), new `String` methods (`isBlank()`, `strip()`, `repeat()`, `lines()`), a new `Collection.toArray(IntFunction)` overload, and a standard, built-in `HttpClient` (`java.net.http`) replacing the old, clunky `HttpURLConnection`.

### Java Code

```java
import java.util.*;
import java.util.stream.*;

public class Java11FeaturesDemo {
    public static void main(String[] args) {
        // var: local variable type inference (Java 10, standard practice by Java 11)
        var list = new ArrayList<String>();              // inferred as ArrayList<String>
        var message = "Hello Java 11";                     // inferred as String
        list.add(message);
        System.out.println(list);

        // New String methods
        String blank = "   ";
        System.out.println("isBlank(): " + blank.isBlank());                     // true - whitespace-only check
        System.out.println("' hi '.strip(): '" + " hi ".strip() + "'");           // unicode-aware trim
        System.out.println("'ab'.repeat(3): " + "ab".repeat(3));

        String multiLine = "line1\nline2\nline3";
        multiLine.lines().forEach(line -> System.out.println("Line: " + line));    // Stream<String> of lines

        // New Collection.toArray(IntFunction) overload - cleaner than the old array-typed overload
        String[] array = list.toArray(String[]::new);
        System.out.println("Array: " + Arrays.toString(array));

        // var in enhanced for-loops and try-with-resources (also valid, less commonly needed)
        for (var item : list) System.out.println("Item via var: " + item);

        System.out.println("Java 11 also added a built-in HttpClient (java.net.http) -"
                + " see Book 09/16 for real HTTP client usage examples.");
    }
}
```

### Output
```
[Hello Java 11]
isBlank(): true
' hi '.strip(): 'hi'
'ab'.repeat(3): ababab
Line: line1
Line: line2
Line: line3
Array: [Hello Java 11]
Item via var: Hello Java 11
Java 11 also added a built-in HttpClient (java.net.http) - see Book 09/16 for real HTTP client usage examples.
```

### Internal Working
- `var` is **compile-time type inference only** — it is NOT dynamic typing; the compiler determines the concrete type at the declaration site and that type is fixed forever after (identical in spirit to Book 01, Ch.2's discussion) — `var` cannot be used for fields, method parameters, or return types, only local variables (and lambda parameters, since Java 11 specifically).
- `strip()` differs from the older `trim()` by being **Unicode-aware** — `trim()` only recognizes characters ≤ ` ` as whitespace, while `strip()` uses `Character.isWhitespace()`, correctly handling a wider range of Unicode whitespace characters.
- The built-in `HttpClient` (`java.net.http`) was added specifically because the old `HttpURLConnection` API was widely regarded as clunky and outdated, and most real projects had been relying on third-party libraries (Apache HttpClient, OkHttp) just for basic HTTP calls — Java 11 finally gave the JDK a modern, fluent, and asynchronous-capable (`CompletableFuture`-returning, Ch.11) HTTP client out of the box.

### Real-World Example
Telugu: `var` వాడితే code verbosity తగ్గుతుంది (`var list = new ArrayList<Employee>();` vs `ArrayList<Employee> list = new ArrayList<Employee>();`), కానీ readability కోసం, type obvious గా లేని చోట్ల `var` avoid చేయాలి అనేది common team style guideline.
English: `var` reduces verbosity (`var list = new ArrayList<Employee>();` instead of repeating the type twice) but common team style guidelines recommend avoiding it where the inferred type isn't immediately obvious from the right-hand side, to preserve readability — a genuine, practical style debate you'll encounter on real teams.

### Interview Answer
"Java 11 (LTS) brought `var` for local variable type inference (compile-time only, not dynamic typing), useful new `String` methods (`isBlank()`, `strip()`, `repeat()`, `lines()`), and a modern, built-in `HttpClient` replacing the old `HttpURLConnection` — all practical, incremental improvements rather than a paradigm shift like Java 8."

### Cross Questions
- Q: Is `var` the same as dynamic typing (like in Python/JavaScript)? → A: No — the type is fully determined and fixed at compile time based on the initializer expression; `var` is purely a syntactic convenience, not a change to Java's static type system.
- Q: Where can `var` NOT be used? → A: Fields, method parameters (except lambda parameters since Java 11), method return types, and any local variable without an initializer to infer the type from (e.g., `var x;` is illegal).
- Q: How does `strip()` differ from the older `trim()`? → A: `strip()` is Unicode-aware (uses `Character.isWhitespace()`); `trim()` only strips characters with code points ≤ ` `, missing some legitimate Unicode whitespace characters.

### Tricky Questions
- Q: Can you write `var x = null;`? → A: No — the compiler cannot infer a meaningful type from `null` alone; this is a compile error.
- Q: Since Java 11, can `var` be used for lambda parameters, and why would you want that? → A: Yes (`(var x, var y) -> x + y`) — primarily so you can attach annotations to lambda parameters (which requires an explicit type or `var`, but not full type inference without either), a relatively niche but real use case.

### Coding Exercise
**L1:** Rewrite 5 verbose variable declarations using `var` where the type remains clear from context.
**L2:** Use `isBlank()`, `strip()`, `repeat()`, and `lines()` on various string inputs and compare with pre-Java-11 equivalents.
**L3:** Use the `Collection.toArray(String[]::new)` overload and compare it to the older `toArray(new String[0])` idiom.
**L4 (Interview):** Explain why `var` is not dynamic typing, using the compile-time inference argument.
**L5 (Senior):** Establish a team style guideline for when to use (and avoid) `var`, with 3 concrete examples of each.
**L6 (Mastery):** Explain, from memory, exactly where `var` can and cannot be used.

---

# CHAPTER 13 — Java 21 LTS: Records & Sealed Classes

### Telugu Explanation
**Records** (Java 16 standard, previewed earlier) data-carrier classes రాయడానికి boilerplate (constructor, getters, `equals()`, `hashCode()`, `toString()`) పూర్తిగా eliminate చేస్తాయి — ఒక్క line declaration తో అన్నీ auto-generate అవుతాయి, **immutable by design**. **Sealed classes** (Java 17 standard) ఒక class/interface ని ఎవరు extend/implement చేయగలరో **explicitly restrict** చేయడానికి వీలు కల్పిస్తాయి — exhaustive pattern matching (Ch.14) కి foundation.

### Professional English Explanation
**Records** (standardized in Java 16) eliminate the boilerplate of data-carrier classes — constructor, accessors, `equals()`, `hashCode()`, `toString()` — all auto-generated from a one-line declaration, and are **immutable by design**. **Sealed classes/interfaces** (standardized in Java 17) let you explicitly restrict which classes may extend/implement a type, forming the foundation for exhaustive pattern matching (Ch.14).

### Java Code — Records

```java
public class RecordsDemo {
    record Point(int x, int y) {                        // auto: constructor, getters (x()/y()), equals, hashCode, toString

        Point {                                             // compact canonical constructor - validation without repeating fields
            if (x < 0 || y < 0) throw new IllegalArgumentException("Coordinates must be non-negative");
        }

        double distanceFromOrigin() {                        // records CAN have additional methods
            return Math.sqrt(x * x + y * y);
        }

        static Point origin() { return new Point(0, 0); }     // and static methods too
    }

    public static void main(String[] args) {
        Point p1 = new Point(3, 4);
        Point p2 = new Point(3, 4);

        System.out.println("p1: " + p1);                      // auto-generated toString(): Point[x=3, y=4]
        System.out.println("p1.x(): " + p1.x() + ", p1.y(): " + p1.y());   // auto-generated accessors
        System.out.println("p1.equals(p2): " + p1.equals(p2));               // auto-generated, value-based equals
        System.out.println("p1.hashCode() == p2.hashCode(): " + (p1.hashCode() == p2.hashCode()));
        System.out.println("Distance: " + p1.distanceFromOrigin());
        System.out.println("Origin: " + Point.origin());

        try {
            new Point(-1, 5);
        } catch (IllegalArgumentException e) {
            System.out.println("Validation works: " + e.getMessage());
        }
    }
}
```

### Java Code — Sealed Classes

```java
public class SealedClassesDemo {

    sealed interface Shape permits Circle, Square, Triangle {}     // explicitly closed set of implementers

    record Circle(double radius) implements Shape {}
    record Square(double side) implements Shape {}
    record Triangle(double base, double height) implements Shape {}

    // NOTE: any OTHER class attempting "implements Shape" would be a COMPILE ERROR - not in the permits list

    static double area(Shape shape) {
        return switch (shape) {                                       // exhaustive pattern matching (Ch.14 detail)
            case Circle c -> Math.PI * c.radius() * c.radius();
            case Square s -> s.side() * s.side();
            case Triangle t -> 0.5 * t.base() * t.height();
            // NO default needed - compiler KNOWS these are the only 3 possible implementers
        };
    }

    public static void main(String[] args) {
        System.out.println("Circle area: " + area(new Circle(5)));
        System.out.println("Square area: " + area(new Square(4)));
        System.out.println("Triangle area: " + area(new Triangle(6, 8)));
    }
}
```

### Output
```
p1: Point[x=3, y=4]
p1.x(): 3, p1.y(): 4
p1.equals(p2): true
p1.hashCode() == p2.hashCode(): true
Distance: 5.0
Origin: Point[x=0, y=0]
Validation works: Coordinates must be non-negative
Circle area: 78.53981633974483
Square area: 16.0
Triangle area: 48.72
```

### Internal Working
- A `record` is compiler sugar that generates: a canonical constructor, `private final` fields for every component, public accessor methods (named exactly like the component, e.g., `x()`, not `getX()`), and `equals()`/`hashCode()`/`toString()` implemented by value (comparing/hashing/printing every component) — exactly the boilerplate Book 01, Ch.15 and Book 02 taught you to write by hand, now automatic, and directly connecting to that chapter's immutability discussion (records are inherently immutable — no setters are generated, ever).
- The **compact canonical constructor** (`Point { ... }`, no parameter list repeated) lets you add validation/normalization logic without restating every field assignment — the implicit field assignments still happen automatically after the compact constructor's body runs.
- `sealed` + `permits` is what makes a `switch` over the sealed type **exhaustive** — the compiler knows the complete, closed set of possible subtypes and can verify every case is handled at compile time, eliminating the need for a `default` branch (and the corresponding risk of silently missing a case if a new subtype is added later without updating every switch).

### Real-World Example
Telugu: DTOs (Data Transfer Objects), API request/response bodies, value objects (Money, Address) — ఇవన్నీ records తో minimal boilerplate తో, guaranteed immutability తో రాయచ్చు. Sealed interfaces `PaymentMethod` (CreditCard, UPI, NetBanking మాత్రమే permitted) వంటి fixed-domain modeling కి ideal — Book 02 Ch.6's enum-based state modeling కి ఒక powerful, richer alternative.
English: DTOs, API request/response bodies, and value objects (Money, Address) are now written as records with minimal boilerplate and guaranteed immutability. Sealed interfaces are ideal for modeling a fixed, closed domain (`PaymentMethod` permitting only `CreditCard`, `UPI`, `NetBanking`) — a richer, more powerful alternative to Book 02, Ch.6's enum-based state modeling when each variant needs different associated data, not just a name.

### Interview Answer
"Records (Java 16) eliminate data-carrier boilerplate — one-line declarations auto-generate an immutable class with a canonical constructor, accessors, and value-based `equals()`/`hashCode()`/`toString()`. Sealed classes/interfaces (Java 17) explicitly restrict which types may extend/implement them via `permits`, which is what allows `switch` pattern matching (Ch.14) over them to be exhaustive and compiler-verified, with no `default` branch needed."

### Cross Questions
- Q: Are records mutable? → A: No — every component becomes a `private final` field with only a read accessor generated; there are no setters, by design.
- Q: Can a record implement an interface? → A: Yes — as shown, records can implement interfaces (including sealed ones) and add their own additional methods beyond the auto-generated ones.
- Q: What happens if you try to create a class implementing a sealed interface that isn't in its `permits` list? → A: Compile error — sealed types enforce a closed, explicitly-declared set of implementers at compile time.

### Tricky Questions
- Q: Can a record have additional instance fields beyond its declared components? → A: No — a record's state is *exactly* its declared components; you cannot add extra instance fields (though you can add static fields and instance/static methods).
- Q: Is a sealed interface's `permits` clause always required to be written explicitly? → A: Not if all permitted implementations are nested within the same file/class as the sealed type itself — in that case, the compiler can infer the `permits` list automatically; it's only mandatory to write explicitly when permitted subtypes live in separate files.

### Coding Exercise
**L1:** Convert a hand-written value class (with constructor, getters, `equals`/`hashCode`/`toString`) into an equivalent record.
**L2:** Add a compact canonical constructor with validation to a record, and a static factory method.
**L3:** Build a sealed interface with 3 record implementers and an exhaustive `switch` computing something per type (like the area example).
**L4 (Interview):** Explain exactly what boilerplate a record generates, and what it deliberately does NOT allow (setters, extra fields).
**L5 (Senior):** Redesign a `PaymentMethod` domain (currently an enum with an awkward "extra data" field abused for different payment types) using a sealed interface with per-type records instead.
**L6 (Mastery):** Explain, from memory, why sealed types make exhaustive switch pattern matching possible, and what compile-time guarantee that gives you when a new variant is added later.

---

# CHAPTER 14 — Java 21 LTS: Pattern Matching, Switch Expressions, Text Blocks

### Telugu Explanation
**Pattern matching for `instanceof`** (Java 16) `instanceof` check తో పాటు cast ని ఒకే line లో combine చేస్తుంది. **Switch expressions** (Java 14) `switch` ని value-returning expression గా, fall-through లేకుండా వాడనిస్తాయి. **Pattern matching for `switch`** (Java 21) `switch` ని types meీద, even record components meీద (record patterns/destructuring) branch చేయనిస్తుంది. **Text blocks** (Java 15) multi-line strings (JSON, SQL, HTML) ని readable గా రాయనిస్తాయి.

### Professional English Explanation
**Pattern matching for `instanceof`** (Java 16) combines a type check and cast into one expression. **Switch expressions** (Java 14) let `switch` be used as a value-producing expression with no fall-through. **Pattern matching for `switch`** (Java 21) extends `switch` to branch on type, including **record patterns** (destructuring a record's components directly in the case label). **Text blocks** (Java 15) allow readable multi-line string literals (JSON, SQL, HTML) without escape-character clutter.

### Java Code

```java
public class PatternMatchingDemo {

    sealed interface Shape permits Circle, Rectangle {}
    record Circle(double radius) implements Shape {}
    record Rectangle(double width, double height) implements Shape {}

    // Pattern matching for instanceof (Java 16): check + cast in one step
    static void describeOld(Object obj) {
        if (obj instanceof String s && s.length() > 5) {         // 's' is usable immediately, no separate cast
            System.out.println("Long string: " + s.toUpperCase());
        }
    }

    // Switch expression (Java 14): value-producing, no fall-through
    static String dayType(int day) {
        return switch (day) {
            case 1, 2, 3, 4, 5 -> "Weekday";
            case 6, 7 -> "Weekend";
            default -> throw new IllegalArgumentException("Invalid day: " + day);
        };
    }

    // Pattern matching for switch + record patterns (Java 21): type AND destructuring in one case label
    static double area(Shape shape) {
        return switch (shape) {
            case Circle(double r) -> Math.PI * r * r;                       // destructures Circle directly
            case Rectangle(double w, double h) when w == h -> w * w;          // guarded pattern: special case for squares
            case Rectangle(double w, double h) -> w * h;
        };
    }

    public static void main(String[] args) {
        describeOld("Hello World");
        System.out.println("Day 3: " + dayType(3));
        System.out.println("Day 6: " + dayType(6));

        System.out.println("Circle area: " + area(new Circle(3)));
        System.out.println("Square-as-rectangle area: " + area(new Rectangle(4, 4)));
        System.out.println("Rectangle area: " + area(new Rectangle(4, 6)));

        // Text block (Java 15): readable multi-line strings
        String json = """
                {
                  "name": "Ravi",
                  "role": "Engineer",
                  "active": true
                }""";
        System.out.println(json);
    }
}
```

### Output
```
Long string: HELLO WORLD
Day 3: Weekday
Day 6: Weekend
Circle area: 28.274333882308138
Square-as-rectangle area: 16.0
Rectangle area: 24.0
{
  "name": "Ravi",
  "role": "Engineer",
  "active": true
}
```

### Internal Working
- Pattern matching for `instanceof` (`obj instanceof String s`) introduces `s` as a new, **effectively final, scoped variable** — usable directly in the same expression (via `&&`) or in the following block — eliminating the old two-step "check, then manually cast" pattern that was both verbose and a source of copy-paste cast errors.
- Switch **expressions** (`->` arrow form) differ fundamentally from switch **statements** (`:` colon form, Book 01, Ch.4): they don't fall through, each arm produces a value directly (or, for multi-statement arms, uses `yield`), and — critically, combined with sealed types (Ch.13) — the compiler can verify **exhaustiveness**, refusing to compile if a `sealed` type's `switch` doesn't cover every permitted subtype (unless a `default` is present).
- **Record patterns** (Java 21) destructure a record's components directly in the `case` label (`case Circle(double r) ->`) — this is genuinely new capability beyond simple type-pattern matching, letting you access a matched record's fields without any additional accessor calls, and can be **nested** for complex, deeply-structured data.
- **Text blocks** (`"""..."""`) automatically manage indentation (stripping "incidental" leading whitespace common to all lines) and require no `\n`/`\"` escaping for embedded newlines/quotes — purely a readability improvement with no new runtime behavior; a text block still compiles down to a regular `String`.

### Real-World Example
Telugu: Domain events processing (Book 16/17) లో sealed interface + record pattern switch వాడితే, ప్రతి event type కి exhaustive, type-safe handling code రాయచ్చు — కొత్త event type add అయితే, compiler ప్రతి existing switch ని check చేసి, మీరు handle చేయకపోతే compile error ఇస్తుంది. Text blocks SQL queries (Book 09), JSON payloads (Book 10) readable గా embed చేయడానికి production code లో విస్తృతంగా వాడతారు.
English: Processing domain events (Book 16/17) with a sealed interface + record-pattern switch gives exhaustive, type-safe handling — adding a new event type later forces the compiler to flag every existing switch that doesn't yet handle it, a powerful correctness guarantee as a system evolves. Text blocks are now widely used in production for embedding readable SQL queries (Book 09) and JSON payloads (Book 10) directly in Java source.

### Interview Answer
"Java 21 combines several incremental language features into a genuinely powerful whole: pattern matching for `instanceof` eliminates the separate check-then-cast step; switch expressions (Java 14) make `switch` a value-producing, non-fall-through construct; pattern matching for switch plus record patterns (Java 21) let you branch on type and destructure record components in one case label, with the compiler enforcing exhaustiveness when combined with sealed types (Ch.13); and text blocks (Java 15) make multi-line string literals readable without escape-character clutter."

### Cross Questions
- Q: What variable scoping rule applies to `instanceof` pattern matching? → A: The introduced variable (`s` in `obj instanceof String s`) is effectively final and scoped to wherever the compiler can prove the pattern matched — the same expression via `&&`, or the following block for a plain `if`.
- Q: How does the compiler verify a switch expression is exhaustive over a sealed type? → A: It checks the switch's case labels against the sealed type's `permits` list (Ch.13); if every permitted subtype has a matching case (or there's a `default`), it compiles — otherwise, it's a compile error.
- Q: Do text blocks support string interpolation like some other languages? → A: No — Java text blocks are purely a multi-line string literal syntax improvement; string formatting still requires `String.format()`/`formatted()` or concatenation, not inline interpolation.

### Tricky Questions
- Q: In `case Rectangle(double w, double h) when w == h -> w * w;`, what happens if a `Rectangle` doesn't satisfy the `when` guard? → A: The switch falls through to try the *next* matching case label in order (here, the plain `Rectangle(double w, double h) ->` case) — guarded patterns are evaluated top-to-bottom like any other switch case ordering.
- Q: Can pattern matching for `switch` be used on a non-sealed type? → A: Yes — you can pattern-match on any type hierarchy, but exhaustiveness checking (no `default` required) is only possible for `sealed` types (or a small set of other provably-complete cases, like matching all subtypes of an enum); a non-sealed type's switch still requires a `default` branch to be exhaustive as far as the compiler can verify.

### Coding Exercise
**L1:** Rewrite an old-style `instanceof` + explicit cast block using pattern matching for `instanceof`.
**L2:** Convert a traditional fall-through `switch` statement into a modern arrow-form switch expression.
**L3:** Build a sealed interface with 3 record implementers and use record patterns (with at least one `when` guard) in an exhaustive switch.
**L4 (Interview):** Explain the difference between switch statements and switch expressions, including exhaustiveness.
**L5 (Senior):** Redesign an event-processing system's type-checking logic (currently a chain of `instanceof` checks) using a sealed interface + record-pattern switch, and explain the compile-time safety benefit when a new event type is added.
**L6 (Mastery):** Explain, from memory, how sealed types + pattern-matching switch together provide a compile-time-verified exhaustiveness guarantee that plain `instanceof` chains never could.

---

# CHAPTER 15 — Java 21 LTS: Virtual Threads (Preview)

### Telugu Explanation
Virtual Threads (Java 21, **Project Loom**) అనేవి **JVM-managed, lightweight threads** — OS thread కి 1:1 map అవ్వకుండా, వేలాది/లక్షలాది virtual threads ఒక చిన్న number of OS ("carrier") threads meీద run అవ్వగలవు. ఇవి ముఖ్యంగా **I/O-bound** workloads కోసం design చేయబడ్డాయి — traditional platform threads కంటే వేలరెట్లు ఎక్కువ concurrency సాధించగలవు, thread-per-request style code ని almost unchanged గా ఉంచుతూనే. పూర్తి concurrency mechanics **Book 08** లో.

### Professional English Explanation
Virtual Threads (Java 21, **Project Loom**) are JVM-managed, lightweight threads that don't map 1:1 to OS threads — potentially hundreds of thousands of virtual threads can run atop a small pool of OS ("carrier") threads. They're designed specifically for **I/O-bound** workloads, enabling orders-of-magnitude more concurrency than traditional platform threads while keeping simple, familiar thread-per-request-style code largely unchanged. Full concurrency mechanics are covered in **Book 08** — this is a structural preview from the "what's new in modern Java" perspective.

### Java Code

```java
import java.util.concurrent.*;
import java.util.*;

public class VirtualThreadsPreviewDemo {
    public static void main(String[] args) throws InterruptedException {
        // Traditional platform thread (heavyweight, 1:1 with OS thread)
        Thread platformThread = new Thread(() -> System.out.println("Platform thread: " + Thread.currentThread()));
        platformThread.start();
        platformThread.join();

        // Virtual thread (Java 21) - lightweight, JVM-managed
        Thread virtualThread = Thread.ofVirtual().start(() ->
                System.out.println("Virtual thread: " + Thread.currentThread()));
        virtualThread.join();

        // Creating MANY virtual threads cheaply - impractical with platform threads at this scale
        int taskCount = 10_000;
        var latch = new CountDownLatch(taskCount);
        long start = System.currentTimeMillis();

        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < taskCount; i++) {
                executor.submit(() -> {
                    try { Thread.sleep(50); } catch (InterruptedException ignored) {}   // simulated I/O wait
                    latch.countDown();
                });
            }
        }
        latch.await();
        System.out.println(taskCount + " virtual thread tasks (each sleeping 50ms) completed in "
                + (System.currentTimeMillis() - start) + " ms - impossible to do this cheaply with 10,000 platform threads");
    }
}
```

### Output (illustrative — timing/thread details vary by JVM/machine)
```
Platform thread: Thread[#21,Thread-0,5,main]
Virtual thread: VirtualThread[#23]/runnable@ForkJoinPool-1-worker-1
10000 virtual thread tasks (each sleeping 50ms) completed in 187 ms - impossible to do this cheaply with 10,000 platform threads
```

### Internal Working (Structural Preview — Full Detail in Book 08)
- A **platform thread** is a thin wrapper directly around an OS thread — expensive to create (typically ~1MB stack reserved per thread) and limited in number (thousands, not millions, before exhausting OS/JVM resources).
- A **virtual thread** is scheduled by the **JVM itself**, not the OS — when a virtual thread performs a blocking operation (like I/O or `Thread.sleep()`), the JVM **unmounts** it from its carrier (platform) thread, freeing that carrier to run other virtual threads, and **remounts** the virtual thread onto a (possibly different) carrier once the blocking operation completes — this is precisely why 10,000 virtual threads sleeping concurrently finish in roughly the sleep duration, not 10,000× that duration serialized through a small platform-thread pool.
- Virtual threads are specifically NOT intended for **CPU-bound** work — since carrier threads are still limited to roughly the number of CPU cores, virtual threads provide no benefit (and some overhead) for compute-heavy tasks; they shine specifically for I/O-bound, blocking-style code (network calls, database queries, file I/O) at massive concurrency scale, letting developers keep simple, sequential-looking blocking code instead of needing complex reactive/async rewrites for scalability.

### Real-World Example
Telugu: Web server ప్రతి incoming request కి ఒక thread కేటాయించే traditional "thread-per-request" model, high-concurrency (10,000+ simultaneous requests) వద్ద platform threads తో scale అవ్వదు — Virtual Threads దీన్ని almost unchanged code తో solve చేస్తాయి, thread-per-request simplicity ని కొనసాగిస్తూనే.
English: The traditional "thread-per-request" web server model doesn't scale to tens of thousands of concurrent requests with platform threads (resource exhaustion); virtual threads solve exactly this, letting simple, blocking-style thread-per-request code scale to massive concurrency without needing a rewrite to reactive/async style — a major reason Spring Boot (Book 12) added first-class virtual thread support starting with Spring Boot 3.2.

### Interview Answer
"Virtual threads (Java 21, Project Loom) are lightweight, JVM-managed threads that don't map 1:1 to OS threads — the JVM unmounts a virtual thread from its carrier platform thread whenever it blocks (I/O, sleep), freeing the carrier for other virtual threads, and remounts it once unblocked. This enables massive I/O-bound concurrency (hundreds of thousands of virtual threads) using simple, familiar blocking-style code, without the complexity of reactive/async rewrites — but they provide no benefit for CPU-bound work, which is still limited by actual core count."

### Cross Questions
- Q: Are virtual threads faster than platform threads for CPU-bound computation? → A: No — CPU-bound work is still ultimately limited by the number of carrier (platform) threads, roughly the CPU core count; virtual threads' advantage is specifically for I/O-bound, blocking workloads at high concurrency.
- Q: Do you need to rewrite blocking code into reactive/async style to benefit from virtual threads? → A: No — that's precisely their point; ordinary blocking-style code (a plain `Thread.sleep()` or blocking I/O call) automatically benefits, since the JVM handles the unmount/remount transparently.
- Q: How many virtual threads can reasonably run concurrently, roughly? → A: Potentially hundreds of thousands to millions, limited mainly by available heap memory (each virtual thread's stack is much smaller and more dynamically-sized than a platform thread's), unlike platform threads which are typically limited to low thousands.

### Tricky Questions
- Q: If a virtual thread performs CPU-bound work without ever blocking, does it ever get unmounted from its carrier? → A: No — it stays pinned to its carrier thread for the entire duration of that CPU-bound work, exactly like a platform thread would, since there's no blocking operation to trigger the unmount; this is why virtual threads don't help CPU-bound workloads.
- Q: Can a virtual thread get "pinned" to its carrier even during a blocking operation, defeating the whole benefit? → A: Yes — this can happen in specific scenarios (e.g., blocking inside a `synchronized` block in some JDK versions, or certain native method calls) where the JVM cannot safely unmount the virtual thread; this is a known, documented caveat that production teams need to be aware of when adopting virtual threads (fully explored in Book 08).

### Coding Exercise
**L1:** Create and run both a platform thread and a virtual thread, printing `Thread.currentThread()` for each.
**L2:** Launch 10,000 virtual threads each sleeping 100ms using `Executors.newVirtualThreadPerTaskExecutor()`, and measure total completion time.
**L3:** Attempt (conceptually, in comments if you can't easily reproduce it) the same 10,000-task experiment with a fixed platform-thread pool of size 100, and reason about why it takes much longer.
**L4 (Interview):** Explain the unmount/remount mechanism that makes virtual threads efficient for I/O-bound work.
**L5 (Senior):** Evaluate whether a given production service (described as CPU-bound image processing vs I/O-bound API gateway) would benefit from migrating to virtual threads, and justify your answer.
**L6 (Mastery):** Explain, from memory, why virtual threads help I/O-bound but not CPU-bound workloads, and name at least one "pinning" caveat that can defeat the benefit.

---

# CHAPTER 16 — Streams in Production: Pitfalls + Mini Project

### Telugu Explanation
Streams powerful గా ఉన్నా, ప్రతిచోటా వాడాలని కాదు. ఈ chapter లో streams ఎప్పుడు **వాడకూడదో**, common mistakes ఏమిటో, మరియు ఈ పుస్తకం మొత్తం ఒక్క mini project లో ఎలా కలపాలో చూద్దాం.

### Professional English Explanation
Streams are powerful, but not universally appropriate. This chapter covers when NOT to use streams, common functional-style mistakes, and combines the entire book into one mini project.

### When NOT to Use Streams

```java
import java.util.*;
import java.util.stream.*;

public class StreamPitfallsDemo {
    public static void main(String[] args) {
        // PITFALL 1: Streams for simple, single-pass operations - a for-loop is often clearer AND faster
        List<Integer> nums = List.of(1, 2, 3, 4, 5);
        int sumStream = nums.stream().mapToInt(Integer::intValue).sum();     // fine, but...
        int sumLoop = 0;
        for (int n : nums) sumLoop += n;                                       // ...equally clear, no stream overhead
        System.out.println("Both sums equal: " + (sumStream == sumLoop));

        // PITFALL 2: Using streams with SIDE EFFECTS in map() - anti-pattern, breaks functional purity
        List<String> results = new ArrayList<>();
        nums.stream().map(n -> {
            results.add("processed-" + n);            // BAD: side effect inside map() - fragile, hard to reason about
            return n * 2;
        }).forEach(n -> {});                              // collect() should be used instead of side-effecting map()
        System.out.println("Side-effect anti-pattern result (works but fragile): " + results);

        // PITFALL 3: Overly complex, unreadable stream chains ("stream golf")
        // BAD (contrived example of an overly clever one-liner):
        // String result = nums.stream().filter(n->n%2==0).map(n->n*n).reduce(0,Integer::sum)+"" ;
        // GOOD: break into named, readable steps
        List<Integer> evens = nums.stream().filter(n -> n % 2 == 0).collect(Collectors.toList());
        List<Integer> squared = evens.stream().map(n -> n * n).collect(Collectors.toList());
        int total = squared.stream().reduce(0, Integer::sum);
        System.out.println("Readable multi-step result: " + total);

        // PITFALL 4: Checked exceptions inside lambdas (a real, common friction point)
        List<String> fileNames = List.of("a.txt", "b.txt");
        // fileNames.stream().map(name -> Files.readString(Path.of(name)))   // won't compile - checked IOException
        //         .forEach(System.out::println);
        // Fix: wrap in a helper method that converts checked to unchecked (Book 04, Ch.3)
        System.out.println("Checked exceptions inside lambdas need wrapping - see Book 04, Ch.3.");
    }
}
```

### Common Pitfalls Summary Table

| Pitfall | Why It's a Problem | Fix |
|---|---|---|
| Streams for trivial single-pass loops | Adds overhead/indirection with no readability gain | Plain for-loop is fine |
| Side effects inside `map()`/`filter()` | Breaks functional purity, fragile, hard to reason about, unsafe in parallel | Use `collect()`/`forEach()` for side effects, not `map()` |
| Overly clever one-liner chains ("stream golf") | Hard to read/debug/maintain | Break into named intermediate variables |
| Checked exceptions inside standard lambdas | Doesn't compile (functional interfaces don't declare `throws`) | Wrap in a helper that converts to unchecked (Book 04, Ch.3) |
| Reusing a consumed stream | `IllegalStateException` (Ch.6) | Build a fresh stream per pipeline execution |
| Parallel streams for small/I/O-bound work | Overhead outweighs benefit, can starve shared pool (Ch.8) | Use sequential streams, or dedicated executors (Book 08) |

### Real-World Example
Telugu: Code review లో "stream golf" (overly compressed, unreadable one-liner chains) ఒక common feedback point — readability maintainability కంటే ఎక్కువ ముఖ్యం production teams కి, clever code కంటే.
English: "Stream golf" — cramming an entire complex transformation into one unreadable one-liner — is a genuinely common real code-review pushback; production teams consistently value readable, debuggable, step-by-step pipelines over maximally compressed clever ones.

### Interview Answer
"Streams shouldn't be used everywhere — trivial single-pass loops are often clearer as plain for-loops, side effects inside `map()`/`filter()` break functional purity and are unsafe in parallel, overly compressed one-liner chains hurt readability, checked exceptions don't compile inside standard lambdas without a wrapper, and parallel streams should be reserved for genuinely CPU-bound, large-scale work. Recognizing when NOT to use a stream is as important a skill as using one well."

### 🏗️ Mini Project: Order Analytics Pipeline

**Goal:** Combine every concept in this book into one realistic reporting pipeline.

**Requirements:**
1. Model the domain with **records** (Ch.13): `record Order(String id, String customerId, OrderStatus status, LocalDate orderDate, List<OrderItem> items)`, `record OrderItem(String productName, int quantity, double unitPrice)`.
2. `OrderStatus` as a **sealed interface or enum** (Ch.13) — `PENDING`, `SHIPPED`, `DELIVERED`, `CANCELLED`.
3. Build a `List<Order>` of sample data (10+ orders across several customers/statuses/dates).
4. Using Streams (Ch.6–8): compute (a) total revenue per customer (`groupingBy` + `summingDouble`, Ch.7), (b) order count per status (`groupingBy` + `counting()`), (c) the top 3 highest-value orders (`sorted` + `limit`), (d) all distinct product names ordered across all orders (`flatMap`, Ch.6).
5. Use `Optional<Order>` (Ch.9) for a `findMostRecentOrder(customerId)` method.
6. Use the `java.time` API (Ch.10) to filter orders within the last 30 days.
7. Use pattern matching + switch (Ch.14) to compute a status-specific message per order.
8. Wrap one "slow" computation (e.g., a simulated report-generation step) in a `CompletableFuture` (Ch.11), demonstrating non-blocking composition.
9. Deliberately avoid all 6 pitfalls from this chapter's table.

**Concepts Reinforced:** Every chapter in this book, applied together in one cohesive, realistic reporting pipeline.

**Stretch Goal:** Add a second pipeline variant using `.parallelStream()` (Ch.8) for the revenue computation over a large generated dataset (100,000+ orders), and benchmark it against the sequential version.

---

# 📌 FINAL REVISION NOTES

- **Java 8's shift**: imperative → functional/declarative, enabled by lambdas + functional interfaces + `invokedynamic` (no new JVM paradigm).
- **Lambdas**: anonymous SAM implementations; capture effectively-final locals by value; `this` refers to the enclosing instance.
- **Core functional interfaces**: `Predicate` (test), `Consumer` (accept), `Supplier` (get), `Function` (apply) — plus primitive/Unary/Binary specializations to avoid autoboxing.
- **Method references**: 4 forms, all compiled the same way as lambdas — pure syntax sugar.
- **Default/static interface methods**: enabled evolving `Collection`/`Comparator` without breaking implementers; diamond conflicts force explicit override.
- **Streams**: lazy, single-use; intermediate ops build a pipeline description, terminal ops execute it; `flatMap` for one-to-many, `map` for one-to-one.
- **Collectors**: `groupingBy` (N groups) vs `partitioningBy` (exactly 2); `toMap` needs a merge function for duplicate keys.
- **Parallel streams**: use the shared common `ForkJoinPool`; only worth it for large, CPU-bound work; never for I/O-bound work (pool starvation risk).
- **Optional**: return-type-only idiom, never a field/parameter; `orElseGet` is lazy, `orElse` is eager.
- **java.time**: immutable, thread-safe; `Period` for calendar amounts, `Duration` for fixed time amounts; use `ZonedDateTime`/`Instant` when zone matters.
- **CompletableFuture**: async, non-blocking chaining; only `.get()`/`.join()` block; shares the common pool with parallel streams.
- **Java 11**: `var` (compile-time inference only), new String methods, built-in `HttpClient`.
- **Java 21**: records (immutable data carriers), sealed types (closed hierarchies enabling exhaustive switch), pattern matching + record patterns, text blocks, virtual threads (I/O-bound concurrency via JVM-managed unmount/remount, not CPU-bound speedup).
- **Stream pitfalls**: avoid trivial-loop streams, side effects in `map()`, "stream golf," unwrapped checked exceptions, stream reuse, and misapplied parallelism.

---

# 🗒️ CHEAT SHEET

```
Java 8 shift: imperative -> declarative, via lambdas + java.util.function + java.util.stream
Lambda: (args) -> expr/block; captures effectively-final locals BY VALUE; 'this' = enclosing instance
Predicate<T>.test | Consumer<T>.accept | Supplier<T>.get | Function<T,R>.apply | UnaryOp/BinaryOp/BiFunction
Method refs: Class::staticMethod | obj::instanceMethod | Class::instanceMethod | Class::new (4 forms, = lambdas)
default method: body in interface, doesn't break implementers | static method: interface utility, not inherited
Stream: lazy, single-use | intermediate (filter/map/sorted/flatMap) lazy | terminal (collect/reduce/forEach) executes
groupingBy: N groups by classifier(+downstream) | partitioningBy: exactly true/false | toMap: needs merge fn for dup keys
Parallel stream: shared common ForkJoinPool, CPU-bound large data only, NEVER I/O-bound (pool starvation)
Optional: RETURN TYPE ONLY, never field/param | orElse=eager | orElseGet=lazy | never Optional.of(null)
java.time: immutable, thread-safe | Period=calendar amount | Duration=fixed time | ZonedDateTime/Instant when zone matters
CompletableFuture: supplyAsync (non-blocking) | .get()/.join() blocks | thenApply/thenAccept/thenCombine/exceptionally
Java 11: var (compile-time infer only, not dynamic typing) | isBlank/strip/repeat/lines | java.net.http.HttpClient
Java 21: record (immutable data class, auto equals/hashCode/toString) | sealed+permits (closed hierarchy)
         pattern matching switch + record patterns (destructuring, exhaustive with sealed)  | text blocks """...."""
         Virtual Threads: JVM-managed, unmount on block, MASSIVE I/O concurrency, NO CPU-bound benefit
Stream pitfalls: trivial loops | side effects in map() | "stream golf" | checked exceptions in lambdas | reuse | bad parallelism
```

---

# 🎤 INTERVIEW QUESTION BANK — Java 8+ Modern Java

**Beginner**
1. What is a lambda expression, and what problem does it solve?
2. What are the four core functional interfaces in `java.util.function`?
3. What is the difference between intermediate and terminal stream operations?
4. What is `Optional`, and where should it be used?
5. What is a record, and what boilerplate does it eliminate?

**Intermediate**
6. Explain the difference between `map()` and `flatMap()`.
7. Explain `groupingBy()` vs `partitioningBy()` with an example each.
8. Why were default methods introduced in Java 8, and what problem did they solve for the Collections Framework?
9. What is the difference between `orElse()` and `orElseGet()`?
10. Explain why streams are lazy and single-use.

**Advanced**
11. Explain why parallel streams are not automatically faster, and describe the shared-ForkJoinPool risk with CompletableFuture.
12. Explain how sealed classes enable exhaustive pattern matching in switch expressions.
13. Why do virtual threads help I/O-bound workloads but not CPU-bound ones?
14. Explain the practical friction of using checked exceptions inside standard Java 8 lambdas, and how to work around it.
15. Explain the difference between switch statements and switch expressions, including exhaustiveness checking.

**Senior/Architect**
16. Design a full reporting pipeline using Streams/Collectors for a domain of your choice, explaining every collector choice.
17. Evaluate whether a given production service should adopt virtual threads, parallel streams, both, or neither — justify with the CPU-bound vs I/O-bound distinction.
18. Review a codebase full of "stream golf" one-liners and side-effecting `map()` calls — propose a remediation plan.
19. Design a domain model using sealed interfaces + records instead of an enum with an awkward "extra data" field, explaining the migration and its benefits.
20. Explain, end-to-end, how you would decide between `CompletableFuture`, virtual threads, and traditional thread pools for a new I/O-heavy microservice feature.

*(Full short/professional/deep-senior answer scaffolding for every question across the series lives in Book 23 — Java Interview Master Book.)*

---

# 🔁 CROSS-QUESTION ENGINE — Sample Chains

**Q: What is a Stream?**
→ Q: Is it a data structure? → Q: What's the difference between intermediate and terminal operations? → Q: Why is it lazy? → Q: Can you reuse a stream after a terminal operation? → Q: When would parallel streams NOT help?

**Q: What is Optional?**
→ Q: Where should it be used, and where should it NOT be used? → Q: Difference between orElse and orElseGet? → Q: What happens with Optional.of(null)? → Q: Is Optional a substitute for proper null-checking discipline everywhere?

**Q: What are Virtual Threads?**
→ Q: How do they differ from platform threads? → Q: Why do they help I/O-bound but not CPU-bound work? → Q: What is "pinning," and when does it happen? → Q: Would you use them for a CPU-bound image-processing service?

---

# 🏋️ CONSOLIDATED EXERCISES (All Levels)

**L1 — Beginner:** Redo each chapter's L1 exercise unaided.
**L2 — Intermediate:** Redo each chapter's L2/L3 exercises, explaining every choice out loud in Telugu.
**L3 — Advanced:** Build a full Stream + Collectors reporting pipeline over a custom dataset using at least 5 different collectors.
**L4 — Interview:** Answer all 20 Interview Bank questions from memory, under 90 seconds each.
**L5 — Senior:** Complete the Chapter 16 mini project fully, including the parallel-stream stretch goal.
**L6 — Mastery:** Teach Chapters 6–7 (Streams/Collectors), 9 (Optional), and 15 (Virtual Threads) out loud, from memory, using your own fresh examples.

---

# 🗓️ ONE-DAY REVISION PLAN (≈7 hours)

| Time | Focus |
|---|---|
| 0:00–0:30 | Ch.1–2: Why Java 8, lambdas — reproduce imperative vs functional side by side |
| 0:30–1:00 | Ch.3–4: Functional interfaces, method references |
| 1:00–1:20 | Ch.5: Default/static interface methods |
| 1:20–1:30 | Break |
| 1:30–2:30 | Ch.6–7: Streams fundamentals + Collectors — highest-density block, re-read twice |
| 2:30–3:00 | Ch.8: Parallel streams — memorize the shared-pool risk |
| 3:00–3:40 | Ch.9: Optional deep dive |
| 3:40–4:10 | Ch.10: Date/Time API |
| 4:10–4:40 | Ch.11: CompletableFuture basics |
| 4:40–5:00 | Ch.12: Java 11 features |
| 5:00–5:00 | Break |
| 5:00–6:00 | Ch.13–14: Records, sealed classes, pattern matching, switch, text blocks |
| 6:00–6:30 | Ch.15: Virtual threads |
| 6:30–7:00 | Ch.16 + Interview Bank — pitfalls table, answer questions from memory |

---

# 🗓️ ONE-WEEK MASTER REVISION PLAN

| Day | Focus |
|---|---|
| 1 | Ch.1–2 (why Java 8, lambdas) — convert 5 anonymous classes to lambdas |
| 2 | Ch.3–5 (functional interfaces, method refs, default/static methods) — build composable predicate chains |
| 3 | Ch.6–7 (Streams fundamentals, Collectors) — build a full groupingBy/partitioningBy report |
| 4 | Ch.8–9 (parallel streams, Optional) — benchmark parallel vs sequential; refactor null-check pyramids |
| 5 | Ch.10–11 (Date/Time, CompletableFuture) — build a multi-timezone scheduler; chain async calls |
| 6 | Ch.12–15 (Java 11, records/sealed, pattern matching, virtual threads) — migrate an enum-based model to sealed+records |
| 7 | Ch.16 + Mini Project + full mock interview: all 20 bank questions + cross-question chains, Telugu and English, unaided |

---

# ✅ FINAL MASTERY CHECKLIST

- [ ] I can explain why Java 8 represents a paradigm shift and what didn't change at the JVM level.
- [ ] I can write lambdas correctly, including variable capture rules.
- [ ] I can use all 4 core functional interfaces and their primitive/Unary/Binary specializations.
- [ ] I can use all 4 method reference forms fluently.
- [ ] I can explain why default/static interface methods were introduced.
- [ ] I can build multi-stage Stream pipelines and explain laziness and single-use semantics.
- [ ] I can use `groupingBy`, `partitioningBy`, and `toMap` correctly, including edge cases.
- [ ] I can explain when parallel streams help and when they hurt.
- [ ] I can use `Optional` idiomatically and explain its one intended use case.
- [ ] I can use `java.time` correctly, including timezone-aware scheduling.
- [ ] I can chain `CompletableFuture` operations and explain blocking vs non-blocking calls.
- [ ] I can use Java 21 records, sealed types, and pattern-matching switch together for exhaustive, type-safe domain modeling.
- [ ] I can explain why virtual threads help I/O-bound but not CPU-bound workloads, including the pinning caveat.
- [ ] I can identify and fix all 6 stream anti-patterns from Chapter 16.
- [ ] I built the Order Analytics Pipeline mini project, including the parallel-stream stretch goal.
- [ ] I answered the full Interview Question Bank confidently in both Telugu and English.

**Once every box is checked, you are ready for `08_Java_Multithreading_Concurrency_Part_1.md`.**
