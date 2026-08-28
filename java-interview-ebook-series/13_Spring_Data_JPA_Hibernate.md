# 📘 BOOK 13 — SPRING DATA JPA / HIBERNATE
## Object-Relational Mapping Mastery (Telugu + English)

**Series:** Java Interview + Development Mastery Series
**Book Number:** 13 of 24
**Versions Covered:** JPA 3.x (Jakarta namespace), Hibernate 6.x, Spring Data JPA 3.x
**Prerequisites:** Book 09 (JDBC — everything here sits on top of it), Book 11 (Spring Core — repositories are beans), Book 12 (Spring Boot — `@Entity`/`JpaRepository` previewed there)
**Next Book:** `14_Spring_Security.md`

---

## 📖 How to Use This Book / ఈ పుస్తకాన్ని ఎలా ఉపయోగించాలి

**Telugu:** Book 09 లో మీరు manual గా `ResultSet` ని Java objects గా map చేశారు. Book 12, Ch.7 లో `JpaRepository` ని ఒక్క preview గా చూశారు. ఈ పుస్తకం లో JPA/Hibernate **ఎలా పనిచేస్తుందో** పూర్తిగా లోతుగా నేర్చుకుంటాము — persistence context, entity lifecycle, relationships, lazy loading, N+1 problem — ఇవి senior interviews లో అత్యంత critical, అత్యంత తప్పుగా అర్థం చేసుకునే topics.

**English:** Book 09 taught manual `ResultSet`-to-object mapping. Book 12, Ch.7 gave a preview of `JpaRepository`. This book goes fully deep into how JPA/Hibernate actually works — the persistence context, entity lifecycle, relationships, lazy loading, and the N+1 problem — among the most critical, most commonly misunderstood topics in senior Java interviews.

---

## 🎯 Learning Objectives

1. Understand the JPA/Hibernate/Spring Data JPA layering.
2. Map entities correctly with `@Entity`, `@Id`, `@Column`, `@Table`.
3. Understand the persistence context, `EntityManager`, first-level cache, and dirty checking.
4. Understand entity lifecycle states and transitions.
5. Map all relationship types correctly (`@OneToOne`, `@ManyToOne`, `@OneToMany`, `@ManyToMany`).
6. Understand lazy vs eager loading and diagnose/fix the N+1 problem.
7. Understand cascading and orphan removal.
8. Master Spring Data JPA repositories: derived queries, `@Query`, JPQL.
9. Use native queries and projections when needed.
10. Understand `@Transactional` deeply, connecting to Book 09's transaction fundamentals.
11. Apply pagination, sorting, and performance optimization techniques.

---

## 📑 Table of Contents

| Ch. | Title |
|---|---|
| 1 | JPA vs Hibernate vs Spring Data JPA |
| 2 | Entity Mapping Fundamentals |
| 3 | The Persistence Context & EntityManager |
| 4 | Entity Lifecycle States |
| 5 | Relationships: OneToOne, ManyToOne, OneToMany, ManyToMany |
| 6 | Lazy vs Eager Loading & the N+1 Problem |
| 7 | Cascading & Orphan Removal |
| 8 | Spring Data JPA Repositories Deep Dive |
| 9 | Native Queries & Projections |
| 10 | @Transactional Deep Dive |
| 11 | Pagination, Sorting & Performance |
| 12 | Mini Project — E-commerce Data Model |
| — | Final Revision Notes, Cheat Sheet, Interview Bank, Revision Plans, Mastery Checklist |

---

# CHAPTER 1 — JPA vs Hibernate vs Spring Data JPA

### Telugu Explanation
**JPA (Jakarta Persistence API)** ఒక **specification** మాత్రమే (interfaces, annotations) — actual implementation కాదు. **Hibernate** JPA యొక్క అత్యంత popular **implementation** (ఇతర implementations: EclipseLink). **Spring Data JPA** Hibernate (లేదా ఏ JPA provider అయినా) meీద ఒక **additional abstraction layer** — repository interfaces నుండి boilerplate code ని పూర్తిగా తొలగిస్తుంది (Book 12, Ch.7 లో చూసినట్టు).

### Professional English Explanation
**JPA (Jakarta Persistence API)** is only a **specification** (interfaces and annotations) — not an implementation. **Hibernate** is the most popular JPA **implementation** (alternatives include EclipseLink). **Spring Data JPA** is an additional abstraction layer on top of Hibernate (or any JPA provider) — eliminating repository boilerplate entirely, as previewed in Book 12, Ch.7.

### Diagram — The Three Layers

```text
Your Code
    |
    v
Spring Data JPA  (interface Repository extends JpaRepository<T, ID> - NO implementation code written by you)
    |
    v
JPA  (Specification: @Entity, @Id, EntityManager interface, JPQL - defines the CONTRACT, not the behavior)
    |
    v
Hibernate  (Implementation: actually translates JPQL to SQL, manages the persistence context, talks to the DB)
    |
    v
JDBC  (Book 09 - Hibernate uses JDBC underneath, exactly like you used it manually)
    |
    v
Database
```

### Java Code

```java
import jakarta.persistence.*;                                        // JPA specification package
import org.springframework.data.jpa.repository.JpaRepository;         // Spring Data JPA

@Entity                                                                  // JPA annotation - defines the CONTRACT
@Table(name = "products")
class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)                // JPA annotations - Hibernate IMPLEMENTS the behavior
    Long id;
    String name;
    double price;
}

interface ProductRepository extends JpaRepository<Product, Long> {}         // Spring Data JPA - zero implementation code

public class LayeringDemo {
    public static void main(String[] args) {
        System.out.println("""
                JPA defines: @Entity, @Id, EntityManager.persist()/find()/merge(), JPQL syntax
                Hibernate implements: actually generates SQL, manages persistence context (Ch.3),
                                       dirty checking, caching, lazy-loading proxies
                Spring Data JPA adds: JpaRepository interface with auto-generated CRUD + derived
                                       queries, ZERO implementation code needed from you

                You could swap Hibernate for EclipseLink (another JPA implementation) and your
                @Entity classes / JpaRepository interfaces would still work UNCHANGED - this is
                exactly Book 02's Dependency Inversion Principle applied to persistence technology.
                """);
    }
}
```

### Internal Working
- Because JPA is a **specification**, your `@Entity` classes and Spring Data `Repository` interfaces are written against **interfaces and annotations JPA defines**, not against Hibernate-specific classes directly — this is a real, practical Dependency Inversion Principle (Book 02, Ch.13) application: in principle, switching the underlying JPA provider (Hibernate → EclipseLink) requires no changes to your entity/repository code, only a dependency and configuration swap.
- Hibernate **predates** JPA historically — it existed as its own independent ORM framework first, and later became JPA's reference/most popular implementation; this is why Hibernate still offers some Hibernate-specific extensions beyond the JPA spec (e.g., certain caching strategies, `@Filter`) that aren't portable to other JPA providers, a genuine, real trade-off between using pure-JPA-portable features versus Hibernate-specific power features.
- Spring Data JPA's `JpaRepository<T, ID>` interface, at runtime, is backed by a Spring-generated **proxy implementation** (Book 11, Ch.8's AOP/proxy mechanism) — `SimpleJpaRepository`, which internally uses the JPA `EntityManager` (Ch.3) to implement `save()`, `findById()`, etc. — this is precisely the mechanism previewed in Book 12, Ch.7, now with the full chain of responsibility visible: your interface → Spring Data's proxy → JPA's `EntityManager` contract → Hibernate's implementation → JDBC (Book 09) → the database.

### Real-World Example
Telugu: Real Spring Boot projects `spring-boot-starter-data-jpa` dependency ఒక్కటే add చేస్తారు — అది Hibernate ని (default JPA provider గా), Spring Data JPA ని, రెండింటినీ transitively pull చేస్తుంది (Book 12, Ch.2's starter mechanism). మీరు Hibernate ని నేరుగా ఎప్పుడూ import చేయరు, JPA annotations మరియు Spring Data interfaces మాత్రమే వాడతారు.
English: Real Spring Boot projects add just the `spring-boot-starter-data-jpa` dependency — it transitively pulls in Hibernate (as the default JPA provider) and Spring Data JPA together (Book 12, Ch.2's starter mechanism). You never directly import Hibernate classes yourself in typical application code — only JPA annotations and Spring Data interfaces.

### Interview Answer
"JPA is a specification (interfaces, annotations) — not an implementation. Hibernate is the most popular JPA implementation, actually generating SQL and managing the persistence context. Spring Data JPA is an additional layer on top, eliminating repository boilerplate entirely via `JpaRepository` interfaces that Spring implements automatically at runtime via a proxy backed by the JPA `EntityManager`. This layering is a direct application of Dependency Inversion — your entity/repository code depends on JPA's interfaces, not Hibernate's concrete classes, so the underlying provider is in principle swappable."

### Cross Questions
- Q: Is Hibernate required to use JPA? → A: No — JPA is a specification with multiple implementations (Hibernate, EclipseLink); Hibernate is simply the most widely used and the Spring Boot default.
- Q: What does Spring Data JPA add on top of plain JPA/Hibernate? → A: Automatic repository implementation generation — `JpaRepository` interfaces get a working CRUD implementation, plus derived query methods from method names (Book 12, Ch.7), with zero implementation code written by the developer.
- Q: What mechanism does Spring Data JPA use to provide a working implementation for an interface with no code? → A: A runtime-generated proxy (Book 11, Ch.8's proxy mechanism), specifically `SimpleJpaRepository`, which internally delegates to the JPA `EntityManager` (Ch.3).

### Tricky Questions
- Q: If you use a Hibernate-specific annotation (like `@Filter`) not part of the JPA specification, does that break the "swap providers freely" claim? → A: Yes, technically — using any Hibernate-specific extension ties your code to Hibernate specifically, reducing true portability; this is a real, deliberate trade-off teams make when a Hibernate-specific feature provides enough value to accept reduced provider-portability.
- Q: Does "Hibernate predates JPA" mean Hibernate had to change significantly to become JPA-compliant? → A: Yes — historically, Hibernate's own native API existed before JPA was standardized; modern Hibernate implements the JPA specification while still retaining some of its original native API surface for backward compatibility and Hibernate-specific power features.

### Coding Exercise
**L1:** Identify, for 5 given code snippets, whether each uses pure JPA, Hibernate-specific, or Spring Data JPA constructs.
**L2:** Research (via documentation) one Hibernate-specific feature not part of the JPA specification, and summarize it.
**L3:** Trace, in your own words, the full call path from `repository.save(entity)` down to the actual SQL executed.
**L4 (Interview):** Explain the three-layer relationship between JPA, Hibernate, and Spring Data JPA.
**L5 (Senior):** Explain the trade-off of using a Hibernate-specific feature versus staying strictly within the JPA specification.
**L6 (Mastery):** Explain, from memory, why this layering is a real-world application of Book 02's Dependency Inversion Principle.

---

# CHAPTER 2 — Entity Mapping Fundamentals

### Telugu Explanation
`@Entity` ఒక class ని database table కి map చేస్తుంది. `@Id` primary key field ని mark చేస్తుంది. `@GeneratedValue` ID generation strategy define చేస్తుంది (`IDENTITY` — DB auto-increment meీద ఆధారపడుతుంది; `SEQUENCE` — DB sequence వాడుతుంది; `AUTO` — provider decide చేస్తుంది). `@Column` field-to-column mapping customize చేస్తుంది (name, nullable, length, unique).

### Professional English Explanation
`@Entity` maps a class to a database table. `@Id` marks the primary key field. `@GeneratedValue` defines the ID generation strategy (`IDENTITY` — relies on DB auto-increment; `SEQUENCE` — uses a DB sequence; `AUTO` — provider decides). `@Column` customizes field-to-column mapping (name, nullable, length, unique).

### Java Code

```java
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "products", indexes = @Index(name = "idx_product_sku", columnList = "sku"))    // Book 09, Ch.9's indexing
class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)         // relies on DB auto-increment (e.g., MySQL AUTO_INCREMENT)
    private Long id;

    @Column(name = "product_name", nullable = false, length = 200)    // DB column name differs from field name
    private String name;

    @Column(unique = true, nullable = false)
    private String sku;

    @Column(precision = 10, scale = 2)                              // DECIMAL(10,2) - never use double for money in real systems!
    private java.math.BigDecimal price;

    @Column(name = "created_at", updatable = false)                   // set once, never updated after insert
    private LocalDateTime createdAt;

    @Enumerated(EnumType.STRING)                                        // stores the enum NAME as text, not ordinal (critical!)
    private ProductStatus status;

    @Transient                                                             // NOT persisted - computed/temporary field only
    private double discountedPriceCache;

    @Version                                                                 // optimistic locking (Book 09, Ch.6) column
    private Long version;

    protected Product() {}                                                    // JPA REQUIRES a no-arg constructor (Book 01, Ch.8)

    public Product(String name, String sku, java.math.BigDecimal price) {
        this.name = name; this.sku = sku; this.price = price;
        this.createdAt = LocalDateTime.now();
        this.status = ProductStatus.ACTIVE;
    }
    // getters omitted for brevity
}

enum ProductStatus { ACTIVE, DISCONTINUED, OUT_OF_STOCK }
```

### Internal Working
- **`protected Product() {}`** — the mandatory no-arg constructor — exists precisely because Hibernate instantiates entities via **reflection** (Book 01, Ch.8's exact "why ORMs need a no-arg constructor" discussion, now seen in its real, concrete context) before populating fields, bypassing your normal constructors entirely; `protected` (rather than `public`) is a common convention that still satisfies JPA's requirement while discouraging application code from accidentally using the "empty" constructor directly.
- **`@Enumerated(EnumType.STRING)`** vs the default `EnumType.ORDINAL` is a critical, genuinely dangerous distinction: `ORDINAL` stores the enum's **position** (0, 1, 2...) as an integer — if you ever reorder or insert a new constant in the middle of the enum declaration (Book 01, Ch.13), **every existing stored value silently means something different** after that code change, a real, serious data-corruption bug class; `STRING` stores the human-readable name, immune to this specific reordering hazard (though it does still break if you rename a constant), which is why `STRING` is the strongly recommended default in nearly all real production code.
- **`@Version`** implements **optimistic locking** (Book 09, Ch.6) — Hibernate automatically includes a `WHERE version = ?` check on every UPDATE and increments the version column on success; if another transaction modified the row (and thus its version) in between, the update affects zero rows and Hibernate throws `OptimisticLockException` — this is the JPA-level realization of Book 09, Ch.6's optimistic-locking concept, requiring zero manual `SELECT ... FOR UPDATE` SQL.
- Using `java.math.BigDecimal` (with `precision`/`scale`) instead of `double` for monetary values is a direct, practical application of Book 01, Ch.2's floating-point rounding-error warning, now seen at the persistence-mapping level where it matters most.

### Real-World Example
Telugu: Production e-commerce systems `ProductStatus` వంటి enums `EnumType.STRING` గా store చేయకపోతే, ఎప్పుడైనా ఎవరైనా ఒక కొత్త status (ఉదా. `BACKORDERED`) మధ్యలో add చేస్తే, database లో ఇప్పటికే ఉన్న products అన్నీ silently wrong status కి "మారిపోతాయి" — ఇది real, documented, painful production bug pattern.
English: In production e-commerce systems, storing enums like `ProductStatus` with the default `ORDINAL` mode means that if anyone ever inserts a new status (like `BACKORDERED`) in the middle of the enum declaration, every existing product in the database silently "changes" to the wrong status — a real, documented, and painful production bug pattern that `EnumType.STRING` entirely avoids.

### Interview Answer
"`@Entity` maps a class to a table, `@Id`/`@GeneratedValue` define the primary key and its generation strategy, `@Column` customizes column mapping. Critically, JPA requires a no-arg constructor since Hibernate instantiates entities via reflection before populating fields (Book 01, Ch.8). `@Enumerated(EnumType.STRING)` should always be preferred over the default `ORDINAL` mode, since ordinal storage silently corrupts existing data if the enum's declaration order ever changes. `@Version` implements optimistic locking automatically, and monetary fields should use `BigDecimal`, never `double`, for precision."

### Cross Questions
- Q: Why does JPA require a no-arg constructor on every entity? → A: Hibernate instantiates entities via reflection, calling the no-arg constructor first and populating fields afterward — bypassing any parameterized constructor's own logic entirely.
- Q: What's the risk of `@Enumerated(EnumType.ORDINAL)` (the default)? → A: If the enum's constant declaration order ever changes (inserting a new value in the middle, reordering), every previously-stored ordinal value silently refers to a different constant — a serious, silent data-corruption risk.
- Q: How does `@Version` implement optimistic locking without manual SQL? → A: Hibernate automatically appends a `WHERE version = ?` condition to UPDATE statements and increments the column on success; a concurrent modification causes the update to affect zero rows, and Hibernate throws `OptimisticLockException`.

### Tricky Questions
- Q: Does `@Transient` fields ever get sent to the database at all, on INSERT or UPDATE? → A: No — `@Transient` fields are completely excluded from persistence; they exist purely as in-memory, computed, or cache-style fields on the Java object.
- Q: If two entities in the same table both declare `@Column(unique = true)` on different fields with the same underlying data, does JPA prevent inserting a genuine duplicate at the application level, or only at the database level? → A: The uniqueness constraint is enforced at the **database level** (JPA translates it into a DDL constraint when Hibernate generates/validates the schema) — a genuine race condition between two concurrent inserts is still only truly prevented by the database's own unique index, not by any JPA/Hibernate application-level check.

### Coding Exercise
**L1:** Map an entity with `@Id`, `@GeneratedValue`, `@Column` customizations, and a `BigDecimal` price field.
**L2:** Add an enum field with `@Enumerated(EnumType.STRING)` and explain, in a comment, the risk if you'd used `ORDINAL` instead.
**L3:** Add `@Version` to an entity and, conceptually (or with a real test if you have a working setup), reproduce an `OptimisticLockException` from a concurrent update.
**L4 (Interview):** Explain why JPA requires a no-arg constructor, connecting it to Book 01, Ch.8.
**L5 (Senior):** Review an entity storing money as `double` and enum status as `ORDINAL` — explain both risks and provide the fix.
**L6 (Mastery):** Explain, from memory, exactly how `@Version`-based optimistic locking works at the SQL level.

---

# CHAPTER 3 — The Persistence Context & EntityManager

### Telugu Explanation
`EntityManager` JPA యొక్క core interface — entities ని persist/find/update/remove చేయడానికి. **Persistence Context** అనేది `EntityManager` నిర్వహించే ఒక **first-level cache** — ఒకే persistence context లో ఒకే entity (ఒకే ID తో) ఒక్కసారి మాత్రమే load అవుతుంది, తర్వాతి calls cached instance return చేస్తాయి. **Dirty Checking**: managed entity యొక్క field మార్చితే, `EntityManager` దాన్ని automatic గా detect చేసి, transaction commit అయినప్పుడు UPDATE SQL generate చేస్తుంది — explicit `save()` call అవసరం లేకుండా.

### Professional English Explanation
`EntityManager` is JPA's core interface for persisting/finding/updating/removing entities. The **Persistence Context** is a **first-level cache** `EntityManager` maintains — within one persistence context, the same entity (same ID) is loaded from the database only once; subsequent lookups return the cached instance. **Dirty checking**: modifying a managed entity's field is automatically detected, and an UPDATE SQL statement is generated at transaction commit — with no explicit `save()` call needed at all.

### Java Code

```java
import jakarta.persistence.*;

public class PersistenceContextDemo {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("demo-unit");
        EntityManager em = emf.createEntityManager();

        em.getTransaction().begin();

        Product product = new Product("Laptop", "SKU-001", new java.math.BigDecimal("55000.00"));
        em.persist(product);                                        // INSERT staged, not necessarily executed YET
        System.out.println("After persist(), has ID? " + (product.getId() != null));   // depends on generation strategy timing

        // FIRST-LEVEL CACHE demonstration
        Product fetched1 = em.find(Product.class, product.getId());       // might hit DB, or might be served from cache
        Product fetched2 = em.find(Product.class, product.getId());        // definitely served from the SAME persistence context cache
        System.out.println("Same instance from cache? " + (fetched1 == fetched2));    // true - IDENTITY, not just equality!

        // DIRTY CHECKING demonstration - NO explicit save()/update() call needed
        fetched1.setPrice(new java.math.BigDecimal("52000.00"));               // just mutate the managed entity's field
        // No em.persist(fetched1) or em.merge(fetched1) call here at all!

        em.getTransaction().commit();                                             // Hibernate detects the change HERE and generates UPDATE
        System.out.println("Price change was automatically persisted via dirty checking (no explicit save() call)");

        em.close();
        emf.close();
    }
}
```

### Diagram — Persistence Context as a First-Level Cache

```text
EntityManager (one per persistence context, typically one per transaction/request)
    |
    +--- Persistence Context (in-memory map: entity ID -> managed entity instance)
              |
              +-- Product#1 (id=1) <-- em.find(Product.class, 1L) returns THIS SAME OBJECT every time
              +-- Product#2 (id=2)
              +-- Order#5   (id=5)

em.find(Product.class, 1L) called AGAIN within the SAME persistence context:
    -> Hibernate checks the persistence context FIRST
    -> Found! Returns the SAME Java object reference - NO SQL query executed at all (cache hit)

At transaction commit (or em.flush()):
    -> Hibernate compares EVERY managed entity's current state against its ORIGINAL loaded snapshot
    -> Any DIFFERENCE -> UPDATE SQL generated automatically ("dirty checking")
```

### Internal Working
- The persistence context returning the **exact same object reference** (`fetched1 == fetched2`, reference equality, not just `.equals()`) for repeated `find()` calls on the same ID within one context is a genuinely important guarantee — it means you never have to worry about "which copy of this entity am I modifying" within a single transaction; every reference to that entity ID, anywhere in your call stack during that transaction, IS the same object.
- **Dirty checking** works by Hibernate maintaining an internal snapshot of each managed entity's state **at the moment it was loaded** (or persisted); at flush/commit time, it compares the entity's *current* field values against that snapshot, field by field, generating an `UPDATE` statement only for entities that actually changed, and only for the *specific columns* that changed (in some Hibernate configurations) — this is the mechanism that makes `save()` calls unnecessary for updates to already-managed (loaded within the current transaction) entities, a genuinely surprising and powerful behavior for developers coming from Book 09's explicit JDBC `UPDATE` statements.
- The persistence context is typically scoped to **one transaction** (or, in Spring's usual configuration, one `@Transactional` method call, Ch.10) — once that transaction/context ends, the cache is discarded; this is why the first-level cache, while genuinely useful, is not a general-purpose application cache (that's a different concept — a second-level cache, or an external cache like Redis, both beyond this book's scope).

### Real-World Example
Telugu: Service layer method లో `@Transactional` (Ch.10) ఉన్నప్పుడు, ఒక entity ని `find()` చేసి, దాని field మార్చి, method end అయిన తర్వాత — explicit `save()` call లేకుండానే database automatic గా update అవుతుంది. ఇది కొత్త developers కి surprising గా అనిపిస్తుంది, కానీ ఖచ్చితంగా dirty checking యొక్క expected behavior.
English: Inside a `@Transactional` (Ch.10) service method, fetching an entity via `find()`, mutating a field, and letting the method end — the database is automatically updated with zero explicit `save()` call. This surprises developers new to JPA, but it's exactly dirty checking's expected, designed behavior.

### Interview Answer
"`EntityManager` manages entities via a persistence context, which acts as a first-level cache — the same entity ID within one persistence context is loaded once and returned as the identical object reference on every subsequent lookup. Dirty checking automatically detects field changes on managed entities and generates UPDATE SQL at commit/flush time, with no explicit save call needed — Hibernate compares the entity's current state against a snapshot taken when it was loaded."

### Cross Questions
- Q: Does `em.find()` for the same ID always hit the database? → A: No — within the same persistence context, subsequent calls for an already-loaded entity ID return the cached instance without a database round-trip.
- Q: How does dirty checking know an entity changed, without an explicit save() call? → A: Hibernate keeps an internal snapshot of each managed entity's loaded state and compares it against the current state at flush/commit time, generating UPDATE SQL for any difference found.
- Q: Is the persistence context a general-purpose application-wide cache? → A: No — it's typically scoped to one transaction and discarded afterward; it's a distinct concept from a second-level cache or an external cache like Redis.

### Tricky Questions
- Q: If you call `em.find(Product.class, 1L)`, then directly modify the database row for ID 1 via a raw SQL tool in another connection, then call `em.find(Product.class, 1L)` again in the SAME persistence context, what do you get? → A: The stale, originally-cached object — the first-level cache doesn't re-query the database for an already-loaded ID within the same context, so the external change is invisible until a new persistence context/transaction begins.
- Q: Does dirty checking generate an UPDATE for an entity whose fields were changed and then changed back to their original values before commit? → A: Depends on the specific comparison — a naive snapshot comparison at commit time would find no net difference and skip the UPDATE, though this exact behavior can have nuances depending on Hibernate version/configuration; the key concept (comparison against the loaded snapshot, not a change-log) remains constant.

### Coding Exercise
**L1:** Demonstrate the first-level cache by calling `find()` twice for the same ID and comparing reference equality.
**L2:** Demonstrate dirty checking by mutating a managed entity's field and confirming the UPDATE happens without an explicit save call.
**L3:** Research (via documentation or logs) how to enable SQL logging to actually observe the generated UPDATE statement from dirty checking.
**L4 (Interview):** Explain the persistence context as a first-level cache, and what "managed" means for an entity.
**L5 (Senior):** Explain why the persistence context is scoped per-transaction, and why that's the correct scope (not application-wide).
**L6 (Mastery):** Explain, from memory, exactly how dirty checking determines which entities need an UPDATE at commit time.

---

# CHAPTER 4 — Entity Lifecycle States

### Telugu Explanation
ప్రతి entity object నాలుగు states లో ఒకదానిలో ఉంటుంది: **Transient** (`new Entity()` చేసిన వెంటనే — persistence context కి తెలియదు, DB తో సంబంధం లేదు), **Managed/Persistent** (`persist()`/`find()` తర్వాత — persistence context track చేస్తుంది, dirty checking apply అవుతుంది), **Detached** (persistence context close అయిన తర్వాత లేదా `detach()` call తర్వాత — object ఉంది కానీ ఇక track అవ్వదు), **Removed** (`remove()` call తర్వాత — commit అయినప్పుడు DELETE అవుతుంది).

### Professional English Explanation
Every entity object exists in one of four states: **Transient** (right after `new Entity()` — unknown to the persistence context, no relation to the DB), **Managed/Persistent** (after `persist()`/`find()` — tracked by the persistence context, dirty checking applies), **Detached** (after the persistence context closes, or an explicit `detach()` call — the object still exists but is no longer tracked), **Removed** (after `remove()` — will be DELETEd at commit).

### Java Code

```java
import jakarta.persistence.*;

public class EntityLifecycleDemo {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("demo-unit");
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();

        // TRANSIENT: not yet known to JPA at all
        Product product = new Product("Mouse", "SKU-002", new java.math.BigDecimal("500.00"));
        System.out.println("Transient - has ID? " + (product.getId() != null));            // false

        // MANAGED: now tracked by the persistence context
        em.persist(product);
        System.out.println("Managed - has ID? " + (product.getId() != null));                // true (depending on strategy)
        System.out.println("Is managed? " + em.contains(product));                              // true

        em.getTransaction().commit();
        em.close();                                                                                // persistence context CLOSED

        // DETACHED: em is closed, product object still exists but is no longer tracked
        product.setPrice(new java.math.BigDecimal("450.00"));                                        // legal Java, but NOT auto-persisted!
        System.out.println("Detached - price mutated but will NOT be saved automatically");

        // Re-attaching a detached entity: merge()
        EntityManager em2 = emf.createEntityManager();
        em2.getTransaction().begin();
        Product reattached = em2.merge(product);                       // merge() returns a NEW managed instance/reference!
        System.out.println("Original 'product' still managed? " + em2.contains(product));              // FALSE - a common gotcha!
        System.out.println("'reattached' is managed? " + em2.contains(reattached));                      // true
        em2.getTransaction().commit();                                                                      // NOW the price change persists
        em2.close();

        // REMOVED
        EntityManager em3 = emf.createEntityManager();
        em3.getTransaction().begin();
        Product toDelete = em3.find(Product.class, product.getId());
        em3.remove(toDelete);                                            // marked for deletion
        System.out.println("Still findable before commit (within same context)? "
                + (em3.find(Product.class, product.getId()) != null));     // false - remove() takes effect in-context immediately
        em3.getTransaction().commit();                                       // DELETE actually executed here
        em3.close();
        emf.close();
    }
}
```

### Diagram — State Transitions

```text
new Entity()
     |
     v
[TRANSIENT] --persist()--> [MANAGED] --em.close() / detach()--> [DETACHED]
                               ^  |                                   |
                               |  |remove()                           |merge()
                               |  v                                   |
                               |[REMOVED] --commit()--> (row DELETEd)  |
                               |                                       |
                               +---------------------------------------+
                                    (merge() returns a NEW managed reference,
                                     the ORIGINAL detached object stays detached!)
```

### Internal Working
- **`merge()`'s return value is critical** — it does **not** re-attach the object you passed in; it copies that object's state onto either an existing managed entity (if one with that ID is already in the persistence context) or a newly-loaded one, and returns **that** reference — this is a very common, genuine gotcha: code that calls `em.merge(detachedEntity)` and then continues using `detachedEntity` directly (instead of the returned reference) is silently working with an object that's still detached and won't have its further changes persisted.
- A **detached** entity is a completely normal Java object in every other respect — you can read its fields, pass it around, even mutate it — the *only* thing that's changed is that the persistence context no longer tracks it for dirty checking; this state commonly occurs in Spring applications when an entity is loaded inside one `@Transactional` method (Ch.10) and then returned/used outside that method's transaction boundary, after the persistence context has closed.
- `remove()` marks an entity for deletion **immediately within the current persistence context** (subsequent `find()` calls for that ID in the same context correctly return `null`/nothing), but the actual `DELETE` SQL is only sent to the database at flush/commit time — mirroring exactly Book 09, Ch.5's "changes aren't durable until commit" transaction semantics.

### Real-World Example
Telugu: Spring `@Transactional` service method లో entity fetch చేసి, method return అయ్యాక (transaction ముగిసిన తర్వాత), ఆ entity meీద `getLazyField()` access చేయడానికి ప్రయత్నిస్తే — `LazyInitializationException` వస్తుంది (Ch.6 లో వివరంగా), ఎందుకంటే entity ఇప్పుడు detached, persistence context ఇక active గా లేదు. ఇది "entity lifecycle" అర్థం చేసుకోకపోతే production లో తరచుగా కనిపించే bug.
English: Fetching an entity inside a Spring `@Transactional` service method and, after the method (and its transaction) has ended, trying to access a lazily-loaded field (Ch.6) on that now-detached entity throws `LazyInitializationException` — an extremely common real production bug that only makes sense once you understand entity lifecycle states, precisely what this chapter builds toward.

### Interview Answer
"Entities exist in 4 states: Transient (unknown to JPA), Managed/Persistent (tracked, dirty checking applies), Detached (no longer tracked, e.g., after the persistence context closes), and Removed (marked for deletion, executed at commit). A critical, common gotcha: `merge()` doesn't re-attach the object you pass in — it returns a different, newly-managed reference, and the original detached object you passed stays detached; code must use the returned reference for further changes to actually persist."

### Cross Questions
- Q: Does mutating a detached entity's field automatically get saved to the database? → A: No — detached entities aren't tracked by any persistence context, so dirty checking doesn't apply; the change stays purely in-memory until (and unless) the entity is re-attached via `merge()` and that transaction commits.
- Q: What does `merge()` actually return, and why does it matter? → A: A managed reference (either an existing managed entity with copied state, or a newly-loaded-and-updated one) — NOT the same object instance you passed in; continuing to use the original passed-in reference after `merge()` is a common bug, since it remains detached.
- Q: When does the actual DELETE SQL execute after calling `remove()`? → A: At flush/commit time, not immediately when `remove()` is called — though the entity is immediately excluded from `find()` results within that same persistence context.

### Tricky Questions
- Q: Can a transient entity ever accidentally become managed without an explicit `persist()` call? → A: Yes, in one specific case — if a transient entity is set as a field on an already-managed entity with a cascading relationship configured (`CascadeType.PERSIST`, Ch.7), it becomes managed automatically when the owning entity is persisted/flushed, without its own explicit `persist()` call.
- Q: If you call `em.detach(entity)` explicitly and then modify the entity, then call `em.merge(entity)` again in the same still-open persistence context, does it work correctly? → A: Yes — `merge()` works regardless of whether the entity was detached via context closure or explicit `detach()`; it always follows the same "find-or-load, then copy state, return the managed reference" logic.

### Coding Exercise
**L1:** Walk through all 4 lifecycle states explicitly (transient → managed → detached → re-managed via merge) with print statements confirming each transition.
**L2:** Reproduce the "using the original detached reference after merge() instead of the returned one" gotcha, and observe that its subsequent changes are lost.
**L3:** Reproduce `LazyInitializationException` (Ch.6 preview) by accessing a lazy field on a detached entity after its persistence context has closed.
**L4 (Interview):** Explain all 4 entity lifecycle states and the transitions between them.
**L5 (Senior):** Review a Spring service returning an entity from a `@Transactional` method, then accessing a lazy field on it in the controller layer — diagnose the resulting exception and propose 2 different fixes.
**L6 (Mastery):** Explain, from memory, exactly why `merge()`'s return value must be used instead of the original passed-in reference.

---

# CHAPTER 5 — Relationships: OneToOne, ManyToOne, OneToMany, ManyToMany

### Telugu Explanation
JPA relationship annotations Book 05, Ch.8's SQL joins/normalization ని Java object graph గా map చేస్తాయి. `@ManyToOne` (చాలా entities ఒక్క దానికి point చేస్తాయి — foreign key ఇక్కడే ఉంటుంది, "owning side"), `@OneToMany` (దీని opposite, usually `mappedBy` తో "inverse side"గా configure చేస్తారు), `@OneToOne`, `@ManyToMany` (ఒక junction/join table అవసరం).

### Professional English Explanation
JPA relationship annotations map Book 09, Ch.8's SQL joins/normalization onto a Java object graph. `@ManyToOne` (many entities point to one — the foreign key lives here, the "owning side"), `@OneToMany` (its opposite, usually configured as the "inverse side" via `mappedBy`), `@OneToOne`, and `@ManyToMany` (requiring a junction/join table).

### Java Code

```java
import jakarta.persistence.*;
import java.util.*;

@Entity
class Customer {
    @Id @GeneratedValue Long id;
    String name;

    @OneToOne(mappedBy = "customer", cascade = CascadeType.ALL)          // INVERSE side - no foreign key column here
    private CustomerProfile profile;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)   // Ch.7's cascading
    private List<Order> orders = new ArrayList<>();                        // INVERSE side - Order owns the FK
}

@Entity
class CustomerProfile {                                                        // OWNING side of the 1:1 - has the FK
    @Id @GeneratedValue Long id;
    String address;

    @OneToOne
    @JoinColumn(name = "customer_id")                                             // the actual foreign key column
    private Customer customer;
}

@Entity
class Order {                                                                       // OWNING side of the many-to-one
    @Id @GeneratedValue Long id;
    double total;

    @ManyToOne(fetch = FetchType.LAZY)                                                // Ch.6 - LAZY is the correct default here
    @JoinColumn(name = "customer_id")                                                   // Order table HAS a customer_id FK column
    private Customer customer;

    @ManyToMany                                                                           // needs a JUNCTION TABLE
    @JoinTable(
            name = "order_promotions",
            joinColumns = @JoinColumn(name = "order_id"),
            inverseJoinColumns = @JoinColumn(name = "promotion_id"))
    private Set<Promotion> promotions = new HashSet<>();                                    // Set, not List - no duplicates
}

@Entity
class Promotion {
    @Id @GeneratedValue Long id;
    String code;

    @ManyToMany(mappedBy = "promotions")                                                       // INVERSE side of the M:M
    private Set<Order> orders = new HashSet<>();
}
```

### Diagram — Owning Side vs Inverse Side

```text
@ManyToOne / @OneToOne with @JoinColumn: the OWNING side - has the actual FK column in ITS table
@OneToMany / @OneToOne with mappedBy:     the INVERSE side - "mirror" of the owning side, NO FK column

Order table:                          Customer table:
+----+----------+-------------+       +----+------+
| id | total    | customer_id |  <----| id | name |    customer_id in Order IS the actual foreign key
+----+----------+-------------+       +----+------+
                                        (Customer.orders is just a JAVA-level convenience view -
                                         changing THAT list alone, without setting Order.customer,
                                         does NOT update the database at all!)
```

### Internal Working
- The **owning side vs inverse side** distinction is the single most important, most-tested concept in this chapter: only the owning side's mapping (`@JoinColumn`) actually corresponds to a real foreign key column and controls what gets persisted; the inverse side (`mappedBy`) is purely a **Java-level convenience view** into the same relationship. This means a genuine, common bug: setting `customer.getOrders().add(newOrder)` **alone**, without also calling `newOrder.setCustomer(customer)`, does **nothing** to the database — because `Order.customer` (the owning side) is what actually gets written; production code must keep both sides consistent, often via a helper method on the parent entity (`addOrder(order)` that does both operations together).
- `@ManyToMany` **requires** a separate join table (`order_promotions` above) since neither `Order` nor `Promotion` can hold a single foreign key representing a many-to-many relationship — this directly mirrors Book 09, Ch.8's SQL-level junction table concept, now automated: Hibernate manages inserting/deleting rows in `order_promotions` as you modify the `Set<Promotion>` on the owning side.
- Using `Set<Promotion>` rather than `List<Promotion>` for the many-to-many collection is a deliberate, common choice — it naturally prevents duplicate associations (Book 05, Ch.4's `HashSet` uniqueness) and avoids certain known Hibernate performance issues with `List`-typed many-to-many collections under specific fetch configurations.

### Real-World Example
Telugu: E-commerce `Customer` ↔ `Order` (1:Many), `Order` ↔ `Promotion` (Many:Many), `Customer` ↔ `CustomerProfile` (1:1) — ఇది real domain modeling లో అత్యంత common relationship combination. Owning-side/inverse-side తప్పుగా అర్థం చేసుకోవడం production లో "నేను add చేశాను కానీ DB లో save అవ్వలేదు" అనే frequently-reported bug కి root cause.
English: E-commerce's `Customer` ↔ `Order` (one-to-many), `Order` ↔ `Promotion` (many-to-many), and `Customer` ↔ `CustomerProfile` (one-to-one) is an extremely common real domain-modeling combination. Misunderstanding owning-side vs inverse-side is the root cause of a frequently-reported real bug pattern: "I added it to the collection, but it never saved to the database."

### Interview Answer
"JPA relationship annotations map SQL foreign-key relationships (Book 09, Ch.8) onto a Java object graph. `@ManyToOne`/`@OneToOne` with `@JoinColumn` are the owning side, holding the actual foreign key; `@OneToMany`/`@OneToOne` with `mappedBy` are the inverse side, a Java-only convenience view. `@ManyToMany` requires a join table. The critical, commonly-tested gotcha: modifying only the inverse side's collection (adding to a parent's list) doesn't persist anything — you must also set the owning side's reference, often via a helper method keeping both sides in sync."

### Cross Questions
- Q: What determines which side of a relationship is "owning"? → A: The side that declares `@JoinColumn` (holding the actual foreign key column); the other side uses `mappedBy` to reference the owning side's field name, making it the inverse/mirror side.
- Q: Why does adding an entity only to the inverse side's collection not persist the relationship? → A: Because only the owning side's foreign key value actually gets written to the database; the inverse side's Java collection is purely a convenience view, not itself a source of truth for persistence.
- Q: Why does `@ManyToMany` require a separate join table? → A: Neither entity's table can hold a single foreign key representing a many-to-many association; a junction table with two foreign keys (one to each side) is required, exactly as in plain SQL (Book 09, Ch.8).

### Tricky Questions
- Q: If you only ever call `newOrder.setCustomer(customer)` (setting the owning side) but never add the order to `customer.getOrders()` (the inverse side's list), does the relationship still persist correctly to the database? → A: Yes — since the owning side's foreign key is what's actually written, the relationship is correctly persisted; the inverse side's in-memory collection would just be inconsistent (stale) until the `Customer` entity is freshly reloaded from the database in a new persistence context.
- Q: Is it possible for `@OneToMany` to be the owning side instead of `@ManyToOne`? → A: Yes, technically (using `@JoinColumn` directly on the `@OneToMany` side instead of `mappedBy`), but this is generally discouraged — it forces Hibernate to issue extra UPDATE statements to manage the foreign key from the "many" side's table, which is less efficient and less intuitive than the conventional `@ManyToOne`-owns-the-FK pattern shown in this chapter.

### Coding Exercise
**L1:** Model a `Customer`/`Order` one-to-many relationship, correctly identifying the owning and inverse sides.
**L2:** Reproduce the "added to inverse side only, nothing persisted" bug, then fix it with a helper method keeping both sides in sync.
**L3:** Model a `Student`/`Course` many-to-many relationship with a join table, using `Set` on both sides.
**L4 (Interview):** Explain the owning side vs inverse side distinction and its practical persistence consequence.
**L5 (Senior):** Design the full entity relationship model for a blog platform (`Author`, `Post`, `Comment`, `Tag`), specifying every relationship type and which side owns each.
**L6 (Mastery):** Explain, from memory, why `@ManyToMany` requires a join table, connecting it to Book 09, Ch.8's SQL-level normalization.

---

# CHAPTER 6 — Lazy vs Eager Loading & the N+1 Problem

### Telugu Explanation
**Lazy loading** (`FetchType.LAZY`) associated entities ని **అవసరం అయినప్పుడు మాత్రమే** load చేస్తుంది (actual field access జరిగినప్పుడు, ఒక proxy object ద్వారా). **Eager loading** (`FetchType.EAGER`) parent entity load అయినప్పుడే associated entities కూడా వెంటనే load చేస్తుంది. **N+1 Problem**: 1 query parent records fetch చేయడానికి, తర్వాత ప్రతి parent కి ఒక్కో separate query (N queries) associated data fetch చేయడానికి — total N+1 queries, catastrophic performance problem పెద్ద datasets meీద.

### Professional English Explanation
**Lazy loading** (`FetchType.LAZY`) loads associated entities **only when actually accessed** (via a proxy object, triggered on first field access). **Eager loading** (`FetchType.EAGER`) loads associated entities immediately alongside the parent. The **N+1 Problem**: 1 query fetches parent records, then a separate query executes for **each** parent to fetch its associated data (N additional queries) — a total of N+1 queries, a catastrophic performance problem at scale.

### Java Code

```java
import jakarta.persistence.*;
import java.util.List;

public class LazyEagerNPlusOneDemo {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("demo-unit");
        EntityManager em = emf.createEntityManager();

        System.out.println("--- THE N+1 PROBLEM ---");
        em.getTransaction().begin();
        List<Customer> customers = em.createQuery("SELECT c FROM Customer c", Customer.class)
                .getResultList();                                              // Query #1: fetches ALL customers

        for (Customer c : customers) {
            System.out.println(c.name + " has " + c.getOrders().size() + " orders");    // TRIGGERS a SEPARATE query PER customer!
            // If there are 100 customers, this is 1 + 100 = 101 total queries - the N+1 problem in action
        }
        em.getTransaction().commit();

        System.out.println("\n--- THE FIX: JOIN FETCH (Ch.8's JPQL) ---");
        em.getTransaction().begin();
        List<Customer> customersWithOrders = em.createQuery(
                "SELECT DISTINCT c FROM Customer c LEFT JOIN FETCH c.orders", Customer.class)
                .getResultList();                                                // ONE query, using SQL JOIN (Book 09, Ch.8)

        for (Customer c : customersWithOrders) {
            System.out.println(c.name + " has " + c.getOrders().size() + " orders");     // NO extra queries - already loaded!
        }
        em.getTransaction().commit();

        em.close();
        emf.close();
    }
}
```

### Diagram — N+1 in Action

```text
QUERY 1: SELECT * FROM customer;                          -- fetches 100 customer rows

Then, for EACH of the 100 customers, accessing c.getOrders() triggers:
QUERY 2:   SELECT * FROM orders WHERE customer_id = 1;
QUERY 3:   SELECT * FROM orders WHERE customer_id = 2;
QUERY 4:   SELECT * FROM orders WHERE customer_id = 3;
   ...
QUERY 101: SELECT * FROM orders WHERE customer_id = 100;

TOTAL: 101 queries for what should have been 1 or 2 - the N+1 problem

THE FIX (JOIN FETCH):
QUERY 1: SELECT c.*, o.* FROM customer c LEFT JOIN orders o ON o.customer_id = c.id;
TOTAL: 1 query - Hibernate populates BOTH the customers AND their orders from one result set
```

### Internal Working
- With `FetchType.LAZY` (the JPA-recommended default for `@OneToMany`/`@ManyToMany`, and the sensible choice for `@ManyToOne`/`@OneToOne` too despite their historical `EAGER` default), Hibernate returns a **proxy object** (a CGLIB-generated subclass, conceptually similar to Book 11, Ch.8's AOP proxies) in place of the real collection/entity — the proxy holds no real data initially; the **first** method call on it (like `.size()` or iterating it) triggers Hibernate to execute the actual `SELECT` query at that exact moment, transparently, behind the scenes.
- The N+1 problem occurs specifically because that lazy-loading trigger happens **inside a loop**, once per parent entity — each iteration independently triggers its own query, when a single `JOIN`-based query could have fetched everything needed in one round-trip; this is a genuinely one of the most common, most costly real Hibernate/JPA performance bugs in production, often invisible in development (small datasets, few customers) and catastrophic in production (thousands of customers = thousands of extra queries).
- `LEFT JOIN FETCH` in JPQL (Ch.8) is the standard fix — it tells Hibernate to eagerly populate the association **for this specific query**, using a genuine SQL `JOIN`, without changing the entity's default `FetchType` mapping globally (which would eagerly load that association for *every* query touching that entity, potentially over-fetching data that's not needed in other contexts) — this per-query control is exactly why `FetchType.LAZY` as the default, combined with deliberate `JOIN FETCH` where needed, is the recommended general strategy.

### Real-World Example
Telugu: Admin dashboard "అన్ని customers, వారి total order count తో" చూపించే page — N+1 problem లేకుండా design చేయకపోతే, 10,000 customers ఉంటే 10,001 queries run అవుతాయి, page load చాలా నెమ్మదిగా అవుతుంది. ఇది Hibernate వాడే teams కి అత్యంత frequently కనిపించే, అత్యంత frequently interview లో అడిగే performance bug.
English: An admin dashboard showing "all customers with their order counts" is the classic real scenario where the N+1 problem strikes — with 10,000 customers, a naive implementation runs 10,001 queries, making the page catastrophically slow. This is, in practice, one of the most commonly encountered real production bugs for teams using Hibernate, and correspondingly one of the most frequently asked interview questions.

### Interview Answer
"Lazy loading (`FetchType.LAZY`) defers loading an association until it's actually accessed, via a proxy object; eager loading (`FetchType.EAGER`) loads it immediately with the parent. The N+1 problem occurs when a loop accesses a lazy association once per parent entity, triggering N separate queries in addition to the original 1 — a common, costly real performance bug, often invisible in development with small datasets. The fix is `JOIN FETCH` in JPQL, fetching the association eagerly for that specific query via a real SQL JOIN, without changing the entity's global fetch strategy."

### Cross Questions
- Q: Why does accessing a lazy-loaded collection inside a loop cause N+1 queries specifically? → A: Each iteration's access to the lazy proxy independently triggers its own query for that one parent's data, since the proxy has no way to know it should batch across the whole loop — N parents means N separate triggered queries.
- Q: Why is `JOIN FETCH` preferred over simply changing the entity's mapping to `FetchType.EAGER`? → A: `JOIN FETCH` applies eager loading only for that specific query; changing the entity's default to `EAGER` would eagerly load that association for every query touching that entity everywhere in the application, even where the data isn't needed, causing unnecessary over-fetching elsewhere.
- Q: Why might the N+1 problem go unnoticed during development? → A: With small development datasets (a handful of records), the extra queries are cheap and fast enough not to be noticeable — the problem's severity scales directly with data volume, becoming catastrophic specifically in production with realistic data sizes.

### Tricky Questions
- Q: Does using `FetchType.LAZY` everywhere guarantee you'll never hit the N+1 problem? → A: No — `LAZY` avoids over-fetching data you don't need, but if you then access that lazy association inside a loop (as in this chapter's demo), you still trigger N+1 queries; `LAZY` alone doesn't solve N+1, it just changes *when* the problem manifests (on access, rather than always upfront) — `JOIN FETCH` (or similar batch-fetching techniques) is what actually solves it.
- Q: Can `JOIN FETCH` be used with a `WHERE` clause and pagination (Ch.11) together safely? → A: This is a well-known, genuine complication — `JOIN FETCH` combined with a to-many association and pagination can produce incorrect/duplicated result counts (since the join multiplies rows, Book 09, Ch.8's "join fan-out"), and Hibernate specifically warns about or restricts this combination; alternative techniques (a separate query for the association, or `@BatchSize`) are often needed when both pagination and fetching a to-many association are required together.

### Coding Exercise
**L1:** Reproduce the N+1 problem in a loop accessing a lazy `@OneToMany` collection, and count the queries executed (via SQL logging).
**L2:** Fix it using `LEFT JOIN FETCH` in JPQL and confirm the query count drops to 1.
**L3:** Research `@BatchSize` (a Hibernate-specific alternative mitigation) and summarize how it reduces N+1's impact without eliminating multiple queries entirely.
**L4 (Interview):** Explain the N+1 problem and its fix using JOIN FETCH.
**L5 (Senior):** Diagnose a production admin dashboard reported as "slow with more customers" — walk through how you'd identify N+1 as the root cause using SQL logging/APM tools.
**L6 (Mastery):** Explain, from memory, why JOIN FETCH is preferred over globally changing an entity's FetchType to EAGER.

---

# CHAPTER 7 — Cascading & Orphan Removal

### Telugu Explanation
**Cascading** parent entity meీద చేసే operation (persist, merge, remove) ని automatic గా associated child entities కి కూడా **propagate** చేయడం. `CascadeType.PERSIST` (parent save అయితే child కూడా save అవుతుంది), `CascadeType.REMOVE` (parent delete అయితే child కూడా delete అవుతుంది), `CascadeType.ALL` (అన్ని operations). **Orphan removal** (`orphanRemoval = true`) — parent యొక్క collection నుండి child ని తీసివేస్తే (వేరే parent కి move చేయకుండా), ఆ child automatic గా DB నుండి delete అవుతుంది.

### Professional English Explanation
**Cascading** propagates an operation (persist, merge, remove) performed on a parent entity to its associated child entities automatically. `CascadeType.PERSIST` (saving the parent also saves the child), `CascadeType.REMOVE` (deleting the parent also deletes the child), `CascadeType.ALL` (all operations). **Orphan removal** (`orphanRemoval = true`) — removing a child from a parent's collection (without reassigning it elsewhere) automatically deletes that child from the database.

### Java Code

```java
import jakarta.persistence.*;
import java.util.*;

@Entity
class Order {
    @Id @GeneratedValue Long id;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)      // OrderItems belong ENTIRELY to Order
    private List<OrderItem> items = new ArrayList<>();

    void addItem(OrderItem item) {                                                          // sync BOTH sides (Ch.5's lesson)
        items.add(item);
        item.setOrder(this);
    }

    void removeItem(OrderItem item) {
        items.remove(item);                                                                    // orphanRemoval triggers DELETE at flush
        item.setOrder(null);
    }
}

@Entity
class OrderItem {
    @Id @GeneratedValue Long id;
    String productName;
    int quantity;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
    void setOrder(Order order) { this.order = order; }
}

public class CascadingDemo {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("demo-unit");
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();

        Order order = new Order();
        OrderItem item1 = new OrderItem(); item1.productName = "Laptop"; item1.quantity = 1;
        OrderItem item2 = new OrderItem(); item2.productName = "Mouse"; item2.quantity = 2;
        order.addItem(item1);
        order.addItem(item2);

        em.persist(order);                                              // CASCADE.PERSIST: item1/item2 saved automatically!
        // We NEVER called em.persist(item1) or em.persist(item2) directly - cascading did it

        em.getTransaction().commit();
        System.out.println("Order and both items persisted via cascading, item IDs: "
                + item1.id + ", " + item2.id);

        // Orphan removal demonstration
        em.getTransaction().begin();
        Order managedOrder = em.find(Order.class, order.id);
        managedOrder.removeItem(managedOrder.items.get(0));                  // remove from the collection, no reassignment
        em.getTransaction().commit();                                          // orphanRemoval=true -> DELETE executed for item1

        System.out.println("Remaining items after orphan removal: " + managedOrder.items.size());   // 1, not 2

        em.close();
        emf.close();
    }
}
```

### Output
```
Order and both items persisted via cascading, item IDs: 1, 2
Remaining items after orphan removal: 1
```

### Internal Working
- `CascadeType.PERSIST` works by Hibernate, during the flush that processes `em.persist(order)`, walking the entity graph reachable from `order` via cascaded associations — finding `item1`/`item2` in the `items` list — and calling `persist()` on each of **them** too, automatically, before the actual INSERT statements are issued; without this cascade configuration, you'd need to explicitly call `em.persist(item1); em.persist(item2);` yourself for every child, every time.
- **`orphanRemoval = true`** is subtly, importantly different from `CascadeType.REMOVE`: `CascadeType.REMOVE` only deletes children when the **parent itself** is deleted; `orphanRemoval` additionally deletes a child the moment it's **removed from the parent's collection**, even if the parent entity itself is never deleted — this is the correct semantic for a genuine "part-of" ownership relationship (Book 02, Ch.12's **Composition**, not Aggregation) — `OrderItem`s have no independent existence or meaning outside their owning `Order`.
- This is the exact JPA-level realization of Book 02, Ch.12's composition-vs-aggregation distinction: `CascadeType.ALL` + `orphanRemoval = true` together model **composition** (the classic `Order`/`OrderItem` example used *in that very chapter*); a relationship without `orphanRemoval` (and often without full `CascadeType.ALL`) models **aggregation**, where the child genuinely can and should outlive its removal from one particular parent's collection.

### Real-World Example
Telugu: `Order`/`OrderItem` composition relationship — order delete అయితే items కూడా delete అవ్వాలి, item ని order నుండి తీసేస్తే (కొనుగోలుదారు cart నుండి item remove చేస్తే) ఆ item కూడా independently exist అవ్వకూడదు — ఇదే `CascadeType.ALL` + `orphanRemoval=true` combination యొక్క exact production use case.
English: The `Order`/`OrderItem` composition relationship — deleting an order should delete its items, and removing an item from an order (a customer removing an item from their cart) shouldn't leave that item independently existing anywhere — is exactly the real production use case for `CascadeType.ALL` + `orphanRemoval = true` together.

### Interview Answer
"Cascading propagates an operation on a parent to its associated children automatically — `CascadeType.PERSIST` saves children when the parent is saved, `CascadeType.REMOVE` deletes children when the parent is deleted. `orphanRemoval = true` goes further, deleting a child the moment it's removed from the parent's collection, even without deleting the parent itself — the correct semantic for a genuine composition relationship (Book 02, Ch.12), like `Order`/`OrderItem`, where children have no independent existence outside their parent."

### Cross Questions
- Q: What's the difference between `CascadeType.REMOVE` and `orphanRemoval = true`? → A: `CascadeType.REMOVE` deletes children only when the parent itself is deleted; `orphanRemoval` additionally deletes a child immediately when it's removed from the parent's collection, regardless of whether the parent is deleted.
- Q: Does `CascadeType.PERSIST` mean you never call `em.persist()` on the child entities directly? → A: Correct — with proper cascading configured, persisting the parent automatically persists reachable, cascaded children, with no separate `persist()` call needed for them.
- Q: How does `CascadeType.ALL` + `orphanRemoval=true` relate to Book 02's composition vs aggregation? → A: This combination is the JPA-level implementation of composition (strong ownership, child has no independent existence) — the exact concept and even the exact `Order`/`OrderItem` example from Book 02, Ch.12.

### Tricky Questions
- Q: If `OrderItem` is also referenced by a separate, independent `WishlistItem` entity (unrelated to any specific `Order`), would `orphanRemoval = true` on `Order.items` be an appropriate design? → A: No — `orphanRemoval` implies the child has no meaningful existence outside this specific parent relationship (true composition); if the same conceptual item type genuinely needs independent existence elsewhere, that's evidence the relationship is closer to aggregation, and `orphanRemoval` would incorrectly delete data that should have survived.
- Q: Does cascading apply automatically in both directions of a bidirectional relationship? → A: No — cascade configuration is set independently per relationship-side annotation; cascading `Order → OrderItem` operations doesn't imply anything cascades in the reverse direction unless separately configured (which would rarely make sense for a composition relationship anyway).

### Coding Exercise
**L1:** Build an `Order`/`OrderItem` composition relationship with `CascadeType.ALL` and verify persisting the order automatically persists its items.
**L2:** Add `orphanRemoval = true` and verify removing an item from the order's collection deletes it from the database.
**L3:** Design a relationship where `orphanRemoval` would be INAPPROPRIATE (an aggregation, not composition), and explain why in a comment.
**L4 (Interview):** Explain the difference between CascadeType.REMOVE and orphanRemoval.
**L5 (Senior):** Review a `Department`/`Employee` relationship configured with `orphanRemoval = true` — explain why this is likely a design mistake (aggregation modeled as composition) and the real-world data-loss risk.
**L6 (Mastery):** Explain, from memory, how CascadeType.ALL + orphanRemoval=true is the JPA-level realization of Book 02, Ch.12's composition concept.

---

# CHAPTER 8 — Spring Data JPA Repositories Deep Dive

### Telugu Explanation
Book 12, Ch.7 లో Spring Data JPA repositories ని preview చేశాము. ఇప్పుడు దీన్ని లోతుగా చూద్దాం: **Derived query methods** (method name నుండి query auto-generate), **`@Query`** (custom JPQL/native SQL అవసరమైనప్పుడు), **JPQL** (SQL లాంటిదే కానీ table/column లు కాకుండా entity/field పేర్లు meీద operate అవుతుంది).

### Professional English Explanation
Book 12, Ch.7 previewed Spring Data JPA repositories. Now, in depth: **Derived query methods** (query auto-generated from the method name), **`@Query`** (for custom JPQL/native SQL when needed), **JPQL** (SQL-like, but operating on entity/field names rather than tables/columns).

### Java Code

```java
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import java.util.*;

interface OrderRepository extends JpaRepository<Order, Long> {

    // DERIVED QUERIES: Spring parses the method name into a query - ZERO implementation code
    List<Order> findByCustomerId(Long customerId);
    List<Order> findByTotalGreaterThan(double amount);
    List<Order> findByCustomerIdAndStatus(Long customerId, String status);           // AND combines conditions
    List<Order> findByStatusOrderByTotalDesc(String status);                            // OrderBy + Desc in the method name itself
    boolean existsByCustomerIdAndStatus(Long customerId, String status);                   // exists-check, returns boolean
    long countByStatus(String status);                                                       // count, returns long
    Optional<Order> findFirstByCustomerIdOrderByIdDesc(Long customerId);                        // "most recent order" pattern

    // @Query with JPQL: for anything derived-query-naming can't express cleanly
    @Query("SELECT o FROM Order o WHERE o.total > :minTotal AND o.customer.name = :customerName")
    List<Order> findHighValueOrdersForCustomer(@Param("minTotal") double minTotal, @Param("customerName") String name);

    // JOIN FETCH (Ch.6's N+1 fix) expressed via @Query
    @Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);

    // Aggregate JPQL query
    @Query("SELECT o.customer.id, SUM(o.total) FROM Order o GROUP BY o.customer.id")           // Book 05, Ch.7's groupingBy, in JPQL
    List<Object[]> getTotalSpentPerCustomer();

    // Modifying query (UPDATE/DELETE) - needs @Modifying + @Transactional (Ch.10)
    @Modifying
    @Query("UPDATE Order o SET o.status = :newStatus WHERE o.id = :id")
    int updateOrderStatus(@Param("id") Long id, @Param("newStatus") String newStatus);
}
```

### Derived Query Method Naming Reference

| Method Name Pattern | Generated JPQL Equivalent |
|---|---|
| `findByX(value)` | `WHERE x = :value` |
| `findByXAndY(v1, v2)` | `WHERE x = :v1 AND y = :v2` |
| `findByXOrY(v1, v2)` | `WHERE x = :v1 OR y = :v2` |
| `findByXGreaterThan(value)` | `WHERE x > :value` |
| `findByXContaining(value)` | `WHERE x LIKE %:value%` |
| `findByXIsNull()` | `WHERE x IS NULL` |
| `findByXOrderByYDesc(value)` | `WHERE x = :value ORDER BY y DESC` |
| `countByX(value)` | `SELECT COUNT(*) WHERE x = :value` |
| `existsByX(value)` | `SELECT COUNT(*) > 0 WHERE x = :value` |
| `deleteByX(value)` | `DELETE WHERE x = :value` |

### Internal Working
- Derived query methods work via Spring parsing the method name at **application startup** (not per-call, avoiding runtime overhead) into an Abstract Syntax Tree of query conditions, then generating the corresponding JPQL — this is why a genuinely misspelled field name in a derived query method (`findByCustmerId` instead of `findByCustomerId`) fails **fast, at application startup** with a clear error, rather than silently at runtime — a real, valuable fail-fast property.
- **JPQL operates on entity/field names, not table/column names** — `SELECT o FROM Order o` refers to the `Order` **class**, not necessarily a table literally named `Order` (which might be mapped to `orders` via `@Table(name="orders")`, Ch.2) — this is the key conceptual difference from raw SQL (Book 09), and exactly why native queries (Ch.9) are a distinct, separate mechanism when you genuinely need table/column-level SQL.
- **`@Modifying`** is required on any `@Query` performing `UPDATE`/`DELETE` — without it, Spring Data JPA assumes every `@Query` is a `SELECT` and would fail; combined with `@Transactional` (Ch.10, required since a modifying operation needs an active transaction), this is the correct, idiomatic pattern for bulk update/delete operations that don't need to load entities into the persistence context first (a genuine performance win over loading-then-modifying-then-saving for bulk operations).

### Real-World Example
Telugu: Real production repositories 90% derived query methods తోనే సరిపోతాయి — complex reporting queries (aggregate, multi-join) కి మాత్రమే `@Query` అవసరం అవుతుంది. Book 12, Ch.7 లో preview చేసిన `findByStockQuantityLessThan()` ఖచ్చితంగా ఈ derived-query mechanism.
English: Real production repositories are satisfied by derived query methods roughly 90% of the time — `@Query` is needed mainly for complex reporting queries (aggregates, multi-entity joins). Book 12, Ch.7's previewed `findByStockQuantityLessThan()` is exactly this derived-query mechanism, now understood in full depth.

### Interview Answer
"Derived query methods let Spring Data JPA generate a query purely from a method name's naming convention, parsed at application startup (failing fast on typos) — covering the vast majority of simple lookup needs with zero implementation code. `@Query` with JPQL handles more complex cases — JPQL operates on entity/field names rather than table/column names, distinguishing it from raw SQL. `@Modifying` is required for `UPDATE`/`DELETE` queries, combined with `@Transactional`, for efficient bulk operations without loading entities into the persistence context first."

### Cross Questions
- Q: When is a derived query method's correctness actually checked — at startup or at call time? → A: At application startup — Spring parses every derived query method's name when building the repository proxy, so a misspelled field name fails immediately with a clear startup error, not silently at runtime.
- Q: What's the key difference between JPQL and raw SQL? → A: JPQL operates on entity class names and field names (the Java object model), while raw SQL operates on actual table and column names — JPQL is translated to real SQL by Hibernate underneath.
- Q: Why is `@Modifying` required for an UPDATE/DELETE `@Query`? → A: Spring Data JPA assumes `@Query` methods are SELECT queries by default; `@Modifying` signals this one performs a data-modifying operation, which also requires an active transaction (`@Transactional`, Ch.10).

### Tricky Questions
- Q: Can a derived query method express an arbitrarily complex condition, like a query spanning 4 joined entities with conditional aggregation? → A: Not cleanly — derived query naming becomes unwieldy and impractical beyond a few conditions; this is exactly the practical threshold where `@Query` (or, for even more complex cases, the Criteria API, beyond this book's scope) becomes the better choice.
- Q: Does a `@Modifying` bulk UPDATE query automatically update the state of any already-loaded, managed entities matching that criteria within the current persistence context? → A: No — this is a genuine, documented gotcha: a bulk `@Modifying` query operates directly at the database level, bypassing the persistence context entirely, so already-loaded managed entities can become stale (out of sync with the database) unless the persistence context is explicitly cleared/refreshed afterward.

### Coding Exercise
**L1:** Write 5 derived query methods covering different naming patterns from the reference table.
**L2:** Write a `@Query` with JPQL performing a join across two entities with a `WHERE` and `ORDER BY`.
**L3:** Write a `@Modifying` `@Query` performing a bulk UPDATE, and research/explain the persistence-context staleness gotcha in a comment.
**L4 (Interview):** Explain how derived query methods are parsed and why misspellings fail fast at startup.
**L5 (Senior):** Design a repository interface for a reporting feature needing 3 different aggregate queries, deciding for each whether a derived method or `@Query` is more appropriate.
**L6 (Mastery):** Explain, from memory, the key conceptual difference between JPQL and raw SQL.

---

# CHAPTER 9 — Native Queries & Projections

### Telugu Explanation
కొన్నిసార్లు JPQL సరిపోదు — database-specific functions, complex window functions, performance-critical hand-tuned SQL అవసరం అయినప్పుడు, `@Query(nativeQuery = true)` వాడతారు — ఇది Book 09's raw SQL నే, JPA layer దాటి. **Projections** పూర్తి entity బదులు, కేవలం అవసరమైన fields మాత్రమే fetch చేయడానికి వాడతారు — performance optimization, N+1-adjacent over-fetching నివారించడానికి.

### Professional English Explanation
Sometimes JPQL isn't enough — database-specific functions, complex window functions, or performance-critical hand-tuned SQL require `@Query(nativeQuery = true)` — this is Book 09's raw SQL, bypassing the JPA abstraction layer. **Projections** fetch only needed fields instead of a full entity — a performance optimization avoiding N+1-adjacent over-fetching.

### Java Code

```java
import org.springframework.data.jpa.repository.*;
import org.springframework.beans.factory.annotation.Value;

interface OrderReportRepository extends JpaRepository<Order, Long> {

    // NATIVE QUERY: raw SQL (Book 09), for database-specific features JPQL can't express
    @Query(value = "SELECT * FROM orders WHERE total > :minTotal ORDER BY created_at DESC LIMIT :limit",
           nativeQuery = true)
    java.util.List<Order> findRecentHighValueOrdersNative(double minTotal, int limit);

    // INTERFACE-BASED PROJECTION: fetch ONLY these fields, not the full entity
    interface OrderSummary {
        Long getId();
        double getTotal();
        String getCustomerName();                                              // Spring Data can even follow nested paths!
    }

    @Query("SELECT o.id AS id, o.total AS total, o.customer.name AS customerName FROM Order o")
    java.util.List<OrderSummary> findAllSummaries();                              // Hibernate generates a MUCH narrower SELECT

    // RECORD-BASED (DTO) PROJECTION - Book 07, Ch.13's records, directly constructed by JPQL
    @Query("SELECT new com.example.OrderTotalDto(o.customer.name, SUM(o.total)) "
         + "FROM Order o GROUP BY o.customer.name")
    java.util.List<OrderTotalDto> getTotalPerCustomerAsDto();
}

record OrderTotalDto(String customerName, double totalSpent) {}                       // Book 07, Ch.13
```

### Internal Working
- **Interface-based projections** (`OrderSummary`) let Hibernate generate a SQL `SELECT` including **only the columns actually needed** — instead of `SELECT * FROM orders JOIN customer ...` (fetching every column of a full `Order` entity plus its associations), it generates something closer to `SELECT o.id, o.total, c.name FROM orders o JOIN customer c ...` — a genuine, measurable performance win for read-heavy reporting endpoints that don't need the full entity graph, directly avoiding unnecessary data transfer and object construction overhead.
- The **DTO/record constructor projection** (`new com.example.OrderTotalDto(...)` inside JPQL) requires the **fully-qualified class name** and a matching constructor — Hibernate literally invokes that constructor with the selected column values for each result row, producing genuinely immutable (Book 07, Ch.13's record) result objects directly from the query, without any intermediate entity materialization or manual mapping step at all.
- Native queries **lose JPQL's entity/field-name abstraction** entirely — they operate on real table/column names (`orders`, `total`, `created_at`) exactly like Book 09's raw JDBC SQL, and correspondingly lose some portability across different JPA providers/databases (a native query using MySQL-specific syntax won't work unchanged against PostgreSQL) — a genuine, deliberate trade-off made only when JPQL truly can't express what's needed.

### Real-World Example
Telugu: Reporting dashboards — "top 10 customers by spend," "products never ordered" వంటి complex analytical queries — తరచుగా projections లేదా native queries అవసరం అవుతాయి, full entity graph fetch చేయడం అనవసరమైన overhead అవుతుంది కాబట్టి.
English: Reporting dashboards — "top 10 customers by spend," "products never ordered" — often genuinely need projections or native queries, since fetching the full entity graph for a report that only needs 2-3 aggregated values would be unnecessary overhead.

### Interview Answer
"`@Query(nativeQuery = true)` uses raw SQL (Book 09) for cases JPQL can't express — database-specific functions, hand-tuned performance-critical queries — at the cost of reduced portability across databases/JPA providers. Projections (interface-based or DTO/record-constructor-based) fetch only needed fields instead of a full entity, letting Hibernate generate a narrower SELECT — a genuine performance optimization for read-heavy reporting endpoints that don't need the full entity graph."

### Cross Questions
- Q: What's the trade-off of using a native query instead of JPQL? → A: You gain access to database-specific SQL features and full control, but lose portability across different databases/JPA providers, and lose JPQL's entity-name abstraction.
- Q: Why would you use a projection instead of returning the full entity? → A: Performance — Hibernate generates a narrower SQL SELECT fetching only the projected fields, avoiding the overhead of loading and constructing full entity objects (and their associations) when only a few fields are actually needed.
- Q: What's required for a JPQL constructor expression (`new com.example.Dto(...)`) to work? → A: The fully-qualified class name and a constructor on that class matching the selected columns' types and order exactly.

### Tricky Questions
- Q: Does a native query benefit from JPA's entity lifecycle management (persistence context tracking, dirty checking, Ch.3-4) the same way a JPQL query does? → A: Only if the native query returns full entity objects (mapped correctly) — a native query returning a projection/scalar result (like a `List<Object[]>` or a DTO) produces plain, non-managed objects, with no persistence-context tracking or dirty checking applying to them at all.
- Q: Can interface-based projections include computed/derived values not directly stored as a single column? → A: Yes, via a default method on the projection interface computing a value from other declared getter methods, or via a `@Value` SpEL expression annotation on the getter — a genuinely useful, if slightly more advanced, projection feature.

### Coding Exercise
**L1:** Write a native query using a database-specific function (e.g., `LIMIT`, or a date-truncation function) not easily expressed in JPQL.
**L2:** Write an interface-based projection fetching only 3 fields from a 10-field entity, and verify (via SQL logging) the generated query is narrower.
**L3:** Write a record-based DTO constructor projection for an aggregate report query.
**L4 (Interview):** Explain when you'd reach for a native query instead of JPQL.
**L5 (Senior):** Design a reporting endpoint needing only 3 aggregated fields across a 15-field entity with 2 associations — justify using a projection over fetching full entities.
**L6 (Mastery):** Explain, from memory, why native queries lose portability, and what specifically JPQL abstracts away that native SQL doesn't.

---

# CHAPTER 10 — @Transactional Deep Dive

### Telugu Explanation
Book 09, Ch.5 లో manual `conn.setAutoCommit(false)`/`commit()`/`rollback()` నేర్చుకున్నాము. Spring `@Transactional` ఈ మొత్తం boilerplate ని Book 11, Ch.8's AOP proxy mechanism ద్వారా eliminate చేస్తుంది — method మొదలైనప్పుడు transaction start అవుతుంది, method exception లేకుండా return అయితే commit అవుతుంది, unchecked exception (default గా) propagate అయితే rollback అవుతుంది.

### Professional English Explanation
Book 09, Ch.5 taught manual `conn.setAutoCommit(false)`/`commit()`/`rollback()`. Spring's `@Transactional` eliminates that entire boilerplate via Book 11, Ch.8's AOP proxy mechanism — a transaction starts when the method begins, commits if the method returns without exception, and rolls back if an unchecked exception propagates (by default).

### Java Code

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.*;

@Service
class TransferService {
    private final AccountRepository accountRepository;                  // JpaRepository, Ch.8
    TransferService(AccountRepository accountRepository) { this.accountRepository = accountRepository; }

    @Transactional                                                          // Book 09, Ch.5's commit/rollback, AUTOMATED
    void transferMoney(Long fromId, Long toId, double amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        if (from.getBalance() < amount) {
            throw new InsufficientFundsException("Insufficient balance");        // UNCHECKED - triggers automatic ROLLBACK
        }

        from.setBalance(from.getBalance() - amount);                                // dirty checking (Ch.3) - no explicit save()!
        to.setBalance(to.getBalance() + amount);                                       // both auto-persisted at commit
        // If we reach the end of the method without exception - Spring commits automatically
    }

    @Transactional(rollbackFor = CustomCheckedException.class)                            // override: rollback on a CHECKED exception too
    void transferWithAudit(Long fromId, Long toId, double amount) throws CustomCheckedException {
        transferMoney(fromId, toId, amount);
        if (amount > 100_000) throw new CustomCheckedException("Requires manual audit");        // now DOES roll back
    }

    @Transactional(readOnly = true)                                                                 // OPTIMIZATION HINT for read-only ops
    java.util.List<Account> getAllAccounts() {
        return accountRepository.findAll();                                                            // Hibernate can skip dirty-checking overhead
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)                                              // ALWAYS a fresh, independent transaction
    void logAuditEntry(String message) {
        // even if the CALLING transaction rolls back, this audit log entry still commits independently
    }
}

class InsufficientFundsException extends RuntimeException { InsufficientFundsException(String m) { super(m); } }
class CustomCheckedException extends Exception { CustomCheckedException(String m) { super(m); } }
```

### Internal Working
- `@Transactional` is implemented via **exactly** Book 11, Ch.8's AOP proxy mechanism: Spring wraps the bean in a proxy that, before the real method executes, begins a transaction (opening/binding a `Connection` and `EntityManager`, Ch.3, to the current thread), and after the method returns, either commits (normal return) or rolls back (based on the exception rules below) — this is precisely why Book 11, Ch.8's **self-invocation limitation applies directly to `@Transactional` too**: calling a `@Transactional` method on `this` from within the same class bypasses the proxy, so no transaction boundary is actually applied — a very frequently-tested, genuinely important real gotcha.
- **Critical default rule**: `@Transactional` rolls back automatically **only for unchecked exceptions** (`RuntimeException` and its subclasses, Book 04, Ch.1) by default — a checked exception propagating out does **NOT** trigger rollback unless `rollbackFor` explicitly specifies it, a subtle, commonly-misunderstood distinction directly connecting to Book 04, Ch.3's checked-vs-unchecked design philosophy.
- `Propagation.REQUIRES_NEW` (used for the audit-logging example) suspends any existing transaction and starts a genuinely independent one — this is why an audit log entry written this way survives even if the calling business transaction later rolls back; the default propagation (`REQUIRED`) instead joins an existing transaction if one is already active, or starts a new one if not — the propagation setting controls exactly this "join vs. create independent" behavior across nested `@Transactional` calls.

### Real-World Example
Telugu: `@Transactional` లేకుండా, Book 09, Ch.5 లో మీరు manual గా రాసిన `transferMoney` method (`setAutoCommit(false)`, `try-catch-rollback-finally`) ఇప్పుడు ఒక్క annotation తో replace అవుతుంది — business logic identical గా ఉంటుంది, transaction management boilerplate పూర్తిగా పోతుంది.
English: Without `@Transactional`, the `transferMoney` method requires Book 09, Ch.5's manual boilerplate (`setAutoCommit(false)`, try-catch-rollback-finally) — with it, that entire transaction-management boilerplate disappears behind one annotation while the business logic stays identical, exactly the AOP payoff previewed in Book 11, Ch.9.

### Interview Answer
"`@Transactional` automates Book 09's manual commit/rollback via Spring's AOP proxy mechanism (Book 11, Ch.8) — a transaction begins when the method starts, commits on normal return, and rolls back automatically only for unchecked exceptions by default (checked exceptions need explicit `rollbackFor`). Critically, the same self-invocation limitation from AOP applies: calling a `@Transactional` method on `this` from within the same class bypasses the proxy entirely, meaning no transaction boundary is actually applied — a frequently-tested, genuinely important gotcha. `Propagation.REQUIRES_NEW` starts a genuinely independent transaction, useful for things like audit logs that should survive the calling transaction's rollback."

### Cross Questions
- Q: Does `@Transactional` roll back on a checked exception by default? → A: No — only unchecked (`RuntimeException`) exceptions trigger automatic rollback by default; checked exceptions need `@Transactional(rollbackFor = ...)` to also trigger it.
- Q: Why does calling a `@Transactional` method via `this.method()` from within the same class fail to apply the transaction? → A: Because `@Transactional` is implemented via an external AOP proxy (Book 11, Ch.8); an internal `this` call bypasses that proxy entirely, so no transaction boundary is established for that call.
- Q: What does `Propagation.REQUIRES_NEW` do differently from the default propagation? → A: It suspends any existing transaction and starts a completely independent new one, so its commit/rollback is decoupled from the calling transaction's outcome — versus the default `REQUIRED`, which joins an existing transaction if present.

### Tricky Questions
- Q: If a `@Transactional(readOnly = true)` method attempts a write operation, what happens? → A: Behavior is provider/database-dependent — some configurations throw an exception, others simply don't guarantee the write is properly persisted; `readOnly = true` is fundamentally a performance *hint* (allowing Hibernate to skip some tracking overhead), not a strictly enforced constraint by the JPA specification itself, though many production setups do configure it to be enforced.
- Q: If method A (no `@Transactional`) calls method B (`@Transactional`) on a *different* injected bean, is B's transaction boundary correctly applied? → A: Yes — since B is called through its own injected bean reference (going through B's own proxy, not `this`), the AOP interception correctly applies; the self-invocation limitation specifically only affects same-class, same-object (`this`) calls.

### Coding Exercise
**L1:** Build a `@Transactional` money-transfer method and verify a thrown unchecked exception correctly rolls back both balance changes.
**L2:** Reproduce the checked-exception "doesn't roll back by default" behavior, then fix it with `rollbackFor`.
**L3:** Reproduce the self-invocation gotcha: call a `@Transactional` method via `this.method()` from within the same class, and confirm no transaction boundary is actually applied.
**L4 (Interview):** Explain why @Transactional rolls back only for unchecked exceptions by default.
**L5 (Senior):** Design an audit-logging strategy using `Propagation.REQUIRES_NEW` so audit entries survive even when the surrounding business transaction rolls back.
**L6 (Mastery):** Explain, from memory, why @Transactional's self-invocation limitation is identical in cause to Book 11, Ch.8's general AOP self-invocation limitation.

---

# CHAPTER 11 — Pagination, Sorting & Performance

### Telugu Explanation
`Pageable`/`Sort` Spring Data JPA లో pagination, sorting handle చేయడానికి standard mechanism. `Page<T>` (total count కూడా తెలుసుకోవాలంటే) vs `Slice<T>` (total count అవసరం లేకపోతే, faster — COUNT query avoid చేస్తుంది). Performance techniques: projections (Ch.9), `JOIN FETCH` (Ch.6), `@BatchSize`, query result caching అవగాహన.

### Professional English Explanation
`Pageable`/`Sort` are Spring Data JPA's standard mechanism for pagination and sorting. `Page<T>` (when you need the total count) vs `Slice<T>` (when you don't — faster, avoiding the extra COUNT query). Performance techniques: projections (Ch.9), `JOIN FETCH` (Ch.6), `@BatchSize`, and awareness of query result caching.

### Java Code

```java
import org.springframework.data.domain.*;
import org.springframework.data.jpa.repository.JpaRepository;

interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByStatus(ProductStatus status, Pageable pageable);          // Pageable added automatically
    Slice<Product> findByPriceGreaterThan(double price, Pageable pageable);          // Slice - no COUNT query
}

public class PaginationDemo {
    void demo(ProductRepository repository) {
        // Basic pagination: page 0, 20 items per page
        Pageable firstPage = PageRequest.of(0, 20);
        Page<Product> page = repository.findByStatus(ProductStatus.ACTIVE, firstPage);

        System.out.println("Total elements: " + page.getTotalElements());          // extra COUNT query executed
        System.out.println("Total pages: " + page.getTotalPages());
        System.out.println("Current page content size: " + page.getContent().size());
        System.out.println("Has next page? " + page.hasNext());

        // Pagination WITH sorting
        Pageable sortedPage = PageRequest.of(0, 20, Sort.by("price").descending()
                .and(Sort.by("name").ascending()));                                   // multi-field sort (Book 05, Ch.10's Comparator chaining)
        Page<Product> sortedResults = repository.findByStatus(ProductStatus.ACTIVE, sortedPage);

        // Slice: when you DON'T need total count - avoids the extra COUNT(*) query entirely
        Slice<Product> slice = repository.findByPriceGreaterThan(1000.0, PageRequest.of(0, 20));
        System.out.println("Has next slice? " + slice.hasNext());                       // determined by fetching size+1 rows, no COUNT
    }
}
```

### Internal Working
- `Page<T>` internally executes **two** queries: the main paginated `SELECT ... LIMIT ... OFFSET ...` (or equivalent) query, AND a separate `SELECT COUNT(*) ...` query to compute `getTotalElements()`/`getTotalPages()` — this COUNT query can itself be genuinely expensive on very large tables (a full or near-full table scan in the worst case, Book 09, Ch.9), which is exactly why `Slice<T>` exists: it fetches one extra row beyond the requested page size internally, using whether that extra row exists to determine `hasNext()` **without ever running a separate COUNT query at all** — a real, meaningful performance optimization when "next page exists" is all you actually need, not an exact total count.
- **`OFFSET`-based pagination** (what `PageRequest.of(pageNumber, size)` generates by default) has a well-known scalability limitation: for very deep pages (page 10,000 of a huge table), the database still has to scan and discard all the preceding rows before returning the requested page — this is why very large-scale systems (Book 21's system design) often use **keyset/cursor-based pagination** instead (`WHERE id > lastSeenId ORDER BY id LIMIT n`) for deep pagination performance, a more advanced technique beyond Spring Data's built-in `Pageable` abstraction.
- Combining `Sort` with pagination is essential for **consistent** results — paginating without an explicit, stable sort order (relying on whatever "natural" order the database happens to return) can produce inconsistent results across page requests (the same row appearing on two different pages, or never appearing at all) if the underlying data changes between requests, or even without changes on some databases with no inherent ordering guarantee for unsorted queries.

### Real-World Example
Telugu: Product catalog listing page — వేల కొద్దీ products ఉంటే, `Page<Product>` వాడి 20-per-page గా serve చేస్తారు, UI కి "page 1 of 500" చూపించడానికి total count అవసరం. Infinite-scroll feeds (social media) కి మాత్రం `Slice`/cursor-based pagination better fit — total count ఎప్పుడూ చూపించాల్సిన అవసరం లేదు.
English: A product catalog listing page — with thousands of products — uses `Page<Product>` to serve 20 per page, needing the total count for a "page 1 of 500" UI indicator. Infinite-scroll feeds (social media) are a better fit for `Slice`/cursor-based pagination instead — there's never a need to show a total count, only "is there more to load."

### Interview Answer
"`Page<T>` provides pagination plus total count (via an extra COUNT query, which can be expensive on large tables), while `Slice<T>` avoids that COUNT query entirely by fetching one extra row to determine if a next page exists — a real performance optimization when you don't need an exact total. `Sort` combines with pagination for stable, consistent results across page requests. For very deep pagination on large datasets, offset-based pagination (Spring Data's default) has known scalability limits, motivating keyset/cursor-based pagination in high-scale system designs."

### Cross Questions
- Q: Why does `Page<T>` execute two queries instead of one? → A: One for the actual paginated data, and a separate COUNT query to compute total elements/pages — both are needed to support `getTotalElements()`/`getTotalPages()`.
- Q: How does `Slice<T>` determine `hasNext()` without a COUNT query? → A: It internally fetches one extra row beyond the requested page size; if that extra row exists, there's a next page — a clever technique avoiding the COUNT entirely.
- Q: Why is explicit sorting important when paginating? → A: Without a stable, explicit sort order, results across separate page requests can be inconsistent (the same row appearing twice, or a row being skipped entirely), especially if underlying data changes between requests.

### Tricky Questions
- Q: Is offset-based pagination (`LIMIT ... OFFSET ...`) equally performant for page 1 and page 10,000 of a huge table? → A: No — deep pages require the database to scan and discard all preceding rows before returning the requested page, making very deep offset-based pagination genuinely slow at scale; keyset/cursor-based pagination avoids this by directly seeking from a known position instead.
- Q: If two different requests paginate the same unsorted query and the underlying data hasn't changed between them, are the results guaranteed to be consistent across pages? → A: Not necessarily guaranteed by the SQL standard for a query with no explicit `ORDER BY` — some databases happen to return consistent order in practice for simple cases, but relying on unspecified behavior is a real correctness risk; explicit sorting removes this ambiguity entirely.

### Coding Exercise
**L1:** Build a paginated, sorted product listing endpoint using `Page<Product>` and `Pageable`/`Sort`.
**L2:** Convert it to use `Slice<Product>` instead, and explain (via SQL logging) the query-count difference.
**L3:** Research keyset/cursor-based pagination and sketch (in comments) how you'd implement it for a deep-pagination use case.
**L4 (Interview):** Explain the difference between Page and Slice, and when you'd choose each.
**L5 (Senior):** Design the pagination strategy for a high-traffic infinite-scroll social media feed versus an admin product-catalog table, justifying the different approaches for each.
**L6 (Mastery):** Explain, from memory, why offset-based pagination has scalability limits for very deep pages, and what alternative addresses it.

---

# CHAPTER 12 — Mini Project: E-commerce Data Model

### Goal
Combine every concept from this book into one complete, realistic JPA-based data model, integrated with a Spring Boot REST API (Book 12).

### Requirements
1. **Full entity model** (Ch.2, Ch.5): `Customer`, `CustomerProfile` (1:1), `Order`, `OrderItem` (composition, Ch.7), `Product`, and a `Product`↔`Category` many-to-many.
2. **Correct owning/inverse sides** (Ch.5) throughout, with helper methods on parent entities keeping both sides of bidirectional relationships in sync.
3. **Deliberate fetch strategy decisions** (Ch.6) for every relationship, justified in comments, with at least one `JOIN FETCH` query demonstrably fixing an N+1 problem you first reproduce and observe via SQL logging.
4. **Cascading and orphan removal** (Ch.7) correctly applied to the `Order`/`OrderItem` composition relationship, and explicitly NOT applied to `Product`/`Category` (an aggregation, not composition).
5. **A repository layer** (Ch.8-9) with at least 3 derived query methods, one `@Query` with JPQL including a join, and one projection (interface or DTO-based) for a reporting use case.
6. **`@Transactional` service methods** (Ch.10) for at least one multi-step operation (placing an order: validate stock, decrement inventory, create order + items, all atomic), correctly avoiding the self-invocation pitfall.
7. **Paginated, sorted listing endpoints** (Ch.11) for products and orders.
8. **Optimistic locking** (Ch.2's `@Version`) on `Product` to prevent lost updates during concurrent stock decrements (directly connecting to Book 09, Ch.6 and Book 08's race condition concepts).

### Concepts Reinforced
Every chapter in this book — the JPA/Hibernate/Spring Data layering (Ch.1), entity mapping (Ch.2), the persistence context (Ch.3), entity lifecycle (Ch.4), relationships (Ch.5), lazy/eager loading and N+1 (Ch.6), cascading (Ch.7), repositories (Ch.8-9), transactions (Ch.10), and pagination/performance (Ch.11) — applied together in one realistic e-commerce data model.

### Stretch Goal
Add a second-level cache configuration (Hibernate's L2 cache, beyond this book's core scope but worth researching) for the read-heavy, rarely-changing `Category` entity, and explain in comments how it differs from the first-level persistence-context cache (Ch.3).

---

# 📌 FINAL REVISION NOTES

- **Layering**: JPA (spec) → Hibernate (implementation) → Spring Data JPA (repository automation) → JDBC (Book 09) → database.
- **Entity mapping**: no-arg constructor mandatory (reflection); `EnumType.STRING` over `ORDINAL` (data-corruption risk); `BigDecimal` for money; `@Version` for optimistic locking.
- **Persistence context**: first-level cache (same ID → same object reference within a context); dirty checking auto-generates UPDATEs at commit, no explicit save() needed for managed entities.
- **Lifecycle**: Transient → Managed → Detached → (merge back to) Managed; Removed → deleted at commit. `merge()`'s return value, not the original reference, is what's managed afterward.
- **Relationships**: owning side (`@JoinColumn`) has the real FK and controls persistence; inverse side (`mappedBy`) is a Java-only view — must keep both sides in sync manually.
- **N+1**: accessing a lazy association in a loop triggers N extra queries; fix with `JOIN FETCH` for that specific query, not a global `FetchType.EAGER` change.
- **Cascading**: `CascadeType.ALL` + `orphanRemoval=true` = composition (Book 02, Ch.12); without orphan removal = aggregation.
- **Repositories**: derived queries parsed and validated at startup (fail-fast); JPQL operates on entity/field names; native queries operate on real table/column names, losing portability.
- **`@Transactional`**: AOP-proxy-based (Book 11, Ch.8) — same self-invocation limitation applies; rolls back only for unchecked exceptions by default; `REQUIRES_NEW` for independent sub-transactions.
- **Pagination**: `Page<T>` (extra COUNT query) vs `Slice<T>` (no COUNT, one extra fetched row); offset pagination has deep-page scalability limits; always pair with explicit `Sort` for consistency.

---

# 🗒️ CHEAT SHEET

```
Layers: JPA(spec) -> Hibernate(impl) -> Spring Data JPA(repo automation) -> JDBC(Book 09) -> DB
Entity: no-arg ctor REQUIRED | @Enumerated(STRING) not ORDINAL | BigDecimal for money | @Version = optimistic lock
Persistence context: 1st-level cache, SAME object ref for same ID within context | dirty checking = auto UPDATE at commit
Lifecycle: Transient -new()-> Managed -persist()/find()-> Detached -close()/detach()-> back to Managed via merge()
  merge() returns a NEW ref - use IT, not the original passed-in object
Relationships: owning side(@JoinColumn)=real FK, controls persistence | inverse side(mappedBy)=Java-only view
  MUST sync both sides manually (helper methods) or "added but not saved" bug
N+1: lazy assoc accessed IN A LOOP = N extra queries | FIX: JOIN FETCH for that query (not global EAGER)
Cascading: CascadeType.ALL+orphanRemoval=true = COMPOSITION (Book 02) | without orphanRemoval = aggregation
Repositories: derived query = parsed/validated at STARTUP (fail fast) | JPQL=entity/field names | native=real table/column, less portable
@Modifying required for UPDATE/DELETE @Query, needs @Transactional too
@Transactional: AOP proxy (Book 11 Ch.8) - SELF-INVOCATION BYPASSES IT | rollback DEFAULT = unchecked only, use rollbackFor for checked
  REQUIRES_NEW = independent sub-transaction (survives caller's rollback)
Pagination: Page<T>=+COUNT query(expensive at scale) | Slice<T>=no COUNT, fetch+1 row trick | always pair with Sort
  offset pagination has DEEP-PAGE scalability limits -> keyset/cursor pagination for huge datasets
```

---

# 🎤 INTERVIEW QUESTION BANK — Spring Data JPA / Hibernate

**Beginner**
1. What is the difference between JPA, Hibernate, and Spring Data JPA?
2. Why does every JPA entity need a no-arg constructor?
3. What is the persistence context?
4. What is the N+1 problem?
5. What is the difference between the owning side and the inverse side of a relationship?

**Intermediate**
6. Explain dirty checking and why it means you don't always need to call save().
7. Explain the four entity lifecycle states and how merge() works.
8. Why is EnumType.ORDINAL risky, and what should you use instead?
9. Explain cascading and orphan removal, and how they relate to Book 02's composition concept.
10. Explain how derived query methods work and why they fail fast on a typo.

**Advanced**
11. Explain the N+1 problem in depth and how JOIN FETCH fixes it without changing the entity's default fetch type.
12. Why does @Transactional's self-invocation limitation exist, and how does it connect to Book 11's AOP proxy mechanism?
13. Explain why @Transactional rolls back only for unchecked exceptions by default.
14. Explain the difference between Page and Slice, and the performance reasoning behind each.
15. Explain why JPQL is distinct from native SQL, and when you'd need to drop down to a native query.

**Senior/Architect**
16. Design a full entity relationship model for a real domain (e.g., a hospital system), specifying every relationship type, owning/inverse sides, fetch strategy, and cascade configuration.
17. Diagnose a production performance incident caused by N+1 queries — walk through your investigation and fix.
18. Design a concurrency-safe stock-decrement mechanism using optimistic locking (@Version), connecting to Book 08 and Book 09's concurrency concepts.
19. Explain end-to-end what happens from calling a @Transactional service method to the database being updated, including the persistence context and AOP proxy mechanics.
20. Design a deep-pagination strategy for a dataset with millions of rows, explaining why offset-based pagination is insufficient and what you'd use instead.

*(Full short/professional/deep-senior answer scaffolding for every question across the series lives in Book 23 — Java Interview Master Book.)*

---

# 🔁 CROSS-QUESTION ENGINE — Sample Chains

**Q: What is the N+1 problem?**
→ Q: Why does it happen specifically inside a loop? → Q: How does JOIN FETCH fix it? → Q: Why not just make the association EAGER everywhere instead? → Q: How would you detect this problem in a real production application?

**Q: What is the persistence context?**
→ Q: What does "managed" mean for an entity? → Q: How does dirty checking work without an explicit save() call? → Q: What happens to an entity when its persistence context closes? → Q: What is a common bug related to accessing a lazy field on a detached entity?

**Q: How does @Transactional work?**
→ Q: What mechanism implements it? → Q: What is the self-invocation limitation, and why does it apply here too? → Q: Does it roll back for a checked exception by default? → Q: What does Propagation.REQUIRES_NEW do differently?

---

# 🏋️ CONSOLIDATED EXERCISES (All Levels)

**L1 — Beginner:** Redo each chapter's L1 exercise unaided.
**L2 — Intermediate:** Redo each chapter's L2/L3 exercises, explaining every JPA/Hibernate mechanic out loud in Telugu.
**L3 — Advanced:** Build a 4-entity relationship model with correct owning/inverse sides and cascading, verified via SQL logging.
**L4 — Interview:** Answer all 20 Interview Bank questions from memory, under 90 seconds each.
**L5 — Senior:** Complete the Chapter 12 mini project fully, including the L2 cache stretch goal.
**L6 — Mastery:** Teach Chapters 3 (persistence context), 6 (N+1), and 10 (@Transactional) out loud, from memory, using fresh examples.

---

# 🗓️ ONE-DAY REVISION PLAN (≈6 hours)

| Time | Focus |
|---|---|
| 0:00–0:30 | Ch.1: JPA/Hibernate/Spring Data layering |
| 0:30–1:00 | Ch.2: Entity mapping — memorize the EnumType/BigDecimal gotchas |
| 1:00–1:30 | Ch.3: Persistence context — the highest-density conceptual block |
| 1:30–1:45 | Break |
| 1:45–2:15 | Ch.4: Entity lifecycle — the merge() gotcha |
| 2:15–2:45 | Ch.5: Relationships — owning vs inverse side |
| 2:45–3:30 | Ch.6: N+1 problem — the single most-tested topic in this book |
| 3:30–4:00 | Ch.7: Cascading/orphan removal |
| 4:00–4:30 | Ch.8–9: Repositories, native queries/projections |
| 4:30–5:15 | Ch.10: @Transactional deep dive — re-read the self-invocation section twice |
| 5:15–5:45 | Ch.11: Pagination/performance |
| 5:45–6:00 | Interview Bank — answer all questions from memory |

---

# 🗓️ ONE-WEEK MASTER REVISION PLAN

| Day | Focus |
|---|---|
| 1 | Ch.1–2 (layering, entity mapping) — set up a working JPA project and map 2 entities |
| 2 | Ch.3–4 (persistence context, lifecycle) — reproduce dirty checking and the merge() gotcha |
| 3 | Ch.5 (relationships) — build all 4 relationship types with correct owning/inverse sides |
| 4 | Ch.6–7 (N+1, cascading) — reproduce and fix N+1; build a composition relationship |
| 5 | Ch.8–9 (repositories, projections) — write derived queries, @Query, and a projection |
| 6 | Ch.10–11 (@Transactional, pagination) + Mini Project — build the full e-commerce data model |
| 7 | Full mock interview: all 20 bank questions + all cross-question chains, in Telugu and English, without notes |

---

# ✅ FINAL MASTERY CHECKLIST

- [ ] I can explain the JPA/Hibernate/Spring Data JPA layering.
- [ ] I can correctly map entities, including the no-arg constructor and EnumType requirements.
- [ ] I can explain the persistence context, first-level cache, and dirty checking.
- [ ] I can explain all 4 entity lifecycle states and the merge() gotcha.
- [ ] I can correctly map all 4 relationship types and identify owning vs inverse sides.
- [ ] I can diagnose and fix the N+1 problem using JOIN FETCH.
- [ ] I can apply cascading and orphan removal correctly, connecting to composition vs aggregation.
- [ ] I can write derived queries, @Query with JPQL, and projections.
- [ ] I can use @Transactional correctly, including its rollback rules and self-invocation limitation.
- [ ] I can implement pagination and sorting, choosing between Page and Slice appropriately.
- [ ] I built the E-commerce Data Model mini project, including the L2 cache stretch goal.
- [ ] I answered the full Interview Question Bank confidently in both Telugu and English.

**Once every box is checked, you are ready for `14_Spring_Security.md`.**
