# CHAPTER 2 — RELATIONSHIPS & MAPPING

---

## 2.1 CONCEPT: The Owning Side — Who Actually Controls the Foreign Key

### TELUGU EXPLANATION

ఒక bidirectional `@OneToMany`/`@ManyToOne` relationship లో (ఉదా:
`Order` కి `List<OrderItem>`, ప్రతి `OrderItem` కి ఒక `Order`) —
**ఒక్కటే side "owning side"** గా ఉంటుంది, అంటే **అదే foreign key column
ని control చేస్తుంది**:

```java
@Entity
class Order {
    @Id @GeneratedValue Long id;

    @OneToMany(mappedBy = "order") // ❌ ఇది OWNING SIDE కాదు — "mappedBy" అనేది "నేను దీన్ని control చేయను" అని చెప్పడం
    List<OrderItem> items = new ArrayList<>();
}

@Entity
class OrderItem {
    @Id @GeneratedValue Long id;

    @ManyToOne // ✅ ఇదే OWNING SIDE — ఇక్కడే foreign key column (order_id) ఉంటుంది
    @JoinColumn(name = "order_id")
    Order order;
}
```

**సూత్రం:** **`@ManyToOne` side ఎప్పుడూ owning side** (ఎందుకంటే
foreign key column, "many" side table లోనే ఉంటుంది, relational
database design ప్రకారం). `mappedBy` ఉన్న side **inverse side** —
ఇది కేవలం **navigation కోసం మాత్రమే** (Java code లో
`order.getItems()` అని call చేయడానికి), కానీ **DB కి ఏం సేవ్ అవుతుందో
నియంత్రించదు**.

**⚠️ అత్యంత frequently-made mistake:**
```java
Order order = new Order();
OrderItem item = new OrderItem();
order.getItems().add(item); // ❌ ఇది inverse side ని మాత్రమే మారుస్తుంది!
entityManager.persist(order);
// item.order ఎప్పుడూ set చేయలేదు కాబట్టి, order_id column NULL గా save అవుతుంది!
```

**సరైన పద్ధతి — రెండు వైపులా consistent గా ఉంచడం (helper method
తో):**
```java
class Order {
    void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this); // ✅ OWNING SIDE ని కూడా తప్పకుండా set చేయండి
    }
}
```

### ENGLISH INTERVIEW ANSWER

"In a bidirectional relationship, exactly one side owns the foreign key —
by convention, the `@ManyToOne` side, since that's the side whose table
actually has the foreign key column in a normalized relational schema.
The `mappedBy` side is the inverse — purely for Java-side navigation, and
Hibernate ignores changes made only to that side when deciding what to
persist. The single most common bug I see here: adding an item to a
parent's collection (`order.getItems().add(item)`) without also setting
the child's reference back to the parent (`item.setOrder(order)`) —
since the collection side is inverse, that addition alone never gets
reflected in the database; the foreign key column silently saves as
NULL. My standard fix is a helper method on the parent — `addItem()` —
that updates both sides together in one call, so this inconsistency
becomes structurally impossible to introduce by accident."

---

## 2.2 CONCEPT: Cascade Types — Convenience with a Real Danger

### TELUGU EXPLANATION

**Cascade** అంటే, ఒక entity మీద చేసిన operation (persist, remove,
మొదలైనవి) ని, **సంబంధిత entities కి కూడా automatic గా apply చేయడం**:

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
List<OrderItem> items = new ArrayList<>();
```

| Cascade Type | అర్థం |
|---|---|
| `PERSIST` | Parent save చేస్తే, children కూడా automatic గా save అవుతాయి |
| `MERGE` | Parent merge చేస్తే, children కూడా merge అవుతాయి |
| `REMOVE` | Parent delete చేస్తే, children కూడా **delete అవుతాయి** |
| `ALL` | పైవన్నీ కలిపి |

**⚠️ `CascadeType.REMOVE` యొక్క నిజమైన ప్రమాదం:** ఇది **"ownership"**
సూచిస్తుంది — parent delete అయితే, children **తప్పకుండా** delete
అవ్వాలి అనే **strong అర్థం**. దీన్ని **తప్పు relationship** మీద వాడితే
(ఉదా: `Customer` నుండి `Order` కి `CascadeType.REMOVE` — customer
delete అయితే, వాళ్ళ **అన్ని order history** కూడా poతాయి!) — ఇది ఒక
**catastrophic, silent data loss బగ్**.

**`orphanRemoval = true`:** ఇది వేరే, కానీ సంబంధిత concept — child ని
parent యొక్క collection నుండి **తీసేస్తే** (`order.getItems().remove(item)`),
అది database నుండి కూడా delete అవుతుంది (కేవలం foreign key ని NULL
చేయడం కాదు) — ఇది **strict "child cannot exist without this specific
parent"** అనే ownership ని express చేస్తుంది.

**Senior rule:** Cascade **నిజమైన "part-of" (composition) relationships**
కి మాత్రమే వాడాలి (`Order` → `OrderItem` — item, order లేకుండా అర్థం
లేనిది). **"reference" relationships** కి (`Order` → `Customer`) cascade
**ఎప్పుడూ వాడకూడదు** — ఒక Order delete అయినంత మాత్రాన, Customer delete
అవ్వకూడదు.

### ENGLISH INTERVIEW ANSWER

"Cascade types propagate an operation from a parent entity to its
related entities automatically. `CascadeType.REMOVE` is the one I'm most
careful with in code review — it means deleting the parent deletes the
children too, which is exactly correct for a true composition
relationship like `Order` to `OrderItem`, where an item genuinely has no
independent existence without its order. Applied to the wrong
relationship — like `Customer` to `Order` — it becomes a catastrophic,
silent data-loss bug: deleting a customer record would delete their
entire order history. My rule is that cascade, especially `REMOVE`,
belongs only on genuine part-of/composition relationships, never on
reference relationships between independently-meaningful entities.
`orphanRemoval = true` is a related but distinct concept — it deletes a
child specifically when it's *removed from the parent's collection*, even
without deleting the parent itself, expressing 'this child cannot exist
without belonging to some specific instance of this parent.'"

---

## 2.3 CONCEPT: `@ManyToMany` — Why an Explicit Join Entity Is Often Better

### TELUGU EXPలanaTION

```java
// Simple @ManyToMany — join table automatic గా create అవుతుంది, కానీ...
@ManyToMany
@JoinTable(name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
List<Course> courses;
```

**సమస్య:** ఈ join table కి **extra columns** (ఉదా: "enrollment_date,"
"grade") add చేయాల్సి వస్తే, plain `@ManyToMany` దీన్ని **support
చేయదు** — join table ని కేవలం రెండు foreign keys మాత్రమే కలిగిన,
"invisible" structure గానే treat చేస్తుంది.

**Senior పరిష్కారం — Join table ని ఒక explicit entity గా మార్చడం:**
```java
@Entity
class Enrollment { // ఇదే మునుపటి "invisible" join table, ఇప్పుడు ఒక first-class entity
    @Id @GeneratedValue Long id;
    @ManyToOne Student student;
    @ManyToOne Course course;
    LocalDate enrollmentDate; // ఇప్పుడు extra columns సాధ్యం!
    String grade;
}
```
ఇది రెండు `@OneToMany`/`@ManyToOne` relationships గా మారుతుంది
(`Student` → `Enrollment` → `Course`), plain `@ManyToMany` కంటే
**ఎక్కువ verbose**, కానీ **నిజ-ప్రపంచ requirements కి చాలా flexible**.

### ENGLISH INTERVIEW ANSWER

"A plain `@ManyToMany` works fine for a genuinely simple association with
no additional data about the relationship itself. The moment you need to
store something *about* the relationship — an enrollment date, a grade, a
quantity, a role — a plain `@ManyToMany`'s implicit join table can't hold
it, since JPA treats that table as an invisible implementation detail
with only the two foreign keys. My standard fix is to promote the join
table to an explicit entity — `Enrollment` between `Student` and
`Course` — which becomes two `@ManyToOne` relationships instead of one
`@ManyToMany`. It's more verbose to set up, but it's the only way to
attach real data to the relationship itself, and in my experience, real
requirements almost always end up needing this eventually, so I often
default to the explicit join entity from the start for any
many-to-many relationship that seems likely to grow additional attributes."

---

## 2.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Adding a child to a bidirectional relationship | Only updates the collection side, unaware of owning-side rules | Uses a helper method updating both sides together |
| Choosing cascade types | Uses `CascadeType.ALL` everywhere "to be safe" | Applies cascade (especially REMOVE) only to genuine composition relationships |
| `@ManyToMany` with future requirements uncertain | Uses plain `@ManyToMany` and gets stuck when extra data is needed later | Considers an explicit join entity upfront if the relationship might need attributes |
| Deleting a parent entity | Surprised when unrelated data disappears too | Understands exactly which cascade settings apply and why |

---

## 2.5 COMMON MISTAKES

1. Updating only the inverse side of a bidirectional relationship,
   silently leaving the foreign key unset.
2. Applying `CascadeType.REMOVE` to a reference relationship instead of
   a true composition relationship, causing unintended data loss.
3. Using plain `@ManyToMany` when the relationship needs additional
   attributes, then having to awkwardly refactor later.
4. Confusing `orphanRemoval` with cascade `REMOVE` — they're related but
   trigger under different conditions.
5. Not writing a helper method to keep both sides of a bidirectional
   relationship consistent, leaving it to every call site to remember.

---

## 2.6 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. Which side of a `@OneToMany`/`@ManyToOne` relationship owns
the foreign key? 2. What does `mappedBy` indicate?

**Intermediate:** 3. Why does `order.getItems().add(item)` alone not
persist the relationship correctly? 4. What's the difference between
`orphanRemoval` and `CascadeType.REMOVE`?

**Senior:** 5. Design the cascade strategy for an e-commerce domain:
`Order`→`OrderItem`, `Customer`→`Order`, `Order`→`Coupon` (a coupon can
be reused across orders). Justify each decision. 6. When would you choose
an explicit join entity over plain `@ManyToMany` from the very start of a design?

**Architect:** 7. You're reviewing a legacy codebase where
`CascadeType.ALL` is applied to nearly every relationship "to keep things
simple." What risks would you flag, and how would you propose fixing it
incrementally without a risky big-bang refactor?

**Scenario:** 8. A production incident: deleting a single `Customer`
record unexpectedly deleted 200 associated `Order` records that were
needed for financial record-keeping. Diagnose the root cause.

**Trick:** 9. "`mappedBy` determines which table's schema has the
foreign key column." True or false?

<details><summary>Key answers</summary>

- Q5: `Order`→`OrderItem`: `CascadeType.ALL` + `orphanRemoval = true` —
  true composition, items have no meaning without their order.
  `Customer`→`Order`: NO cascade `REMOVE` — orders are independently
  meaningful records (financial history) that must survive even if
  hypothetically a customer record were removed; at most `PERSIST` for
  convenience when creating a new order under a known customer.
  `Order`→`Coupon`: NO cascade at all — a coupon is a shared, independent
  entity reused across many orders; cascading any operation from Order to
  Coupon would be incorrect since Coupon's lifecycle is entirely
  independent of any single Order.
- Q6: When there's any realistic chance the relationship will need to
  carry its own data (a timestamp, a status, a quantity, a role) — since
  retrofitting a plain `@ManyToMany` into an explicit join entity later
  is a real migration (new entity, changed relationship types, data
  migration if already in production), it's often worth the extra
  upfront verbosity to default to an explicit join entity whenever that
  future need seems plausible.
- Q7: Risks: unintended cascading deletes on relationships that aren't
  true compositions (Q8's exact scenario), unexpected cascading persists
  creating implicit, hard-to-trace inserts. Incremental fix: audit each
  relationship's actual semantic type (composition vs reference) one at a
  time, starting with the highest-risk ones (anything cascading REMOVE),
  narrowing `CascadeType.ALL` down to only the specific types actually
  needed and correct for each relationship, verified with tests before
  each change ships.
- Q8: This is precisely the `CascadeType.REMOVE` misapplication from
  section 2.2 — `Customer`→`Order` was incorrectly configured with
  cascade REMOVE (likely as part of a blanket `CascadeType.ALL`),
  treating orders as if they were composition-owned by the customer, when
  they're actually independently meaningful records that must survive
  customer deletion (or, in a well-designed system, customers likely
  shouldn't be hard-deleted at all, but that's a separate schema design consideration).
- Q9: False — `mappedBy` is a Java/JPA-level annotation attribute
  indicating which side is the *inverse* (non-owning) side for
  object-navigation purposes; the actual database schema (which table
  has the foreign key column) is determined by the `@ManyToOne`/
  `@JoinColumn` side, independent of where `mappedBy` appears in the Java code.

</details>

---

## 2.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is `@ManyToOne` always the owning side, by relational schema necessity, not just JPA convention?
- **Coding Check:** Write a `Post`/`Comment` bidirectional one-to-many relationship with a correct helper method (`addComment()`) keeping both sides consistent, plus appropriate cascade settings.
- **Explanation Check:** Explain in English why "just use CascadeType.ALL everywhere" is a dangerous default rather than a helpful simplification.
- **Real-World Check:** Your team's `Author`↔`Book` relationship is many-to-many, and a new requirement asks to track which edition/print-run each book was published under, per author collaboration. Redesign the mapping.
- **Senior Check:** When would a unidirectional (not bidirectional) relationship be the better design choice, avoiding this chapter's owning-side complexity entirely?
- **Master Check:** Design the complete entity relationship model for a blogging platform: `User`, `Post`, `Comment`, `Tag` (many-to-many with posts), and `Like` (a user liking a post, needs a timestamp). Specify cascade settings and owning sides for every relationship, justifying each.

<details><summary>Answers</summary>

- Real-World Check: Convert the plain `@ManyToMany` between `Author` and
  `Book` into an explicit `Authorship` (or similarly named) join entity
  with `@ManyToOne` to both `Author` and `Book`, plus the new
  `editionOrPrintRun` attribute — exactly the pattern from section 2.3.
- Senior Check: When the Java-side navigation is genuinely only ever
  needed in one direction — e.g., an `Order` needs to know its
  `ShippingAddress`, but nothing ever needs to query "which orders use
  this address" — a unidirectional `@ManyToOne` (or `@OneToOne`) avoids
  the owning-side bookkeeping entirely, since there's no inverse
  collection to keep in sync; adding bidirectionality only when an actual
  navigation need arises, not preemptively, is often the simpler and less error-prone default.
- Master Check: `User`→`Post`: `@OneToMany`/`@ManyToOne`, cascade
  `PERSIST` only (creating a post under a known user), no `REMOVE`
  cascade (a user's posts likely shouldn't vanish if the user is
  deleted/deactivated in most real platforms — a business decision worth
  confirming). `Post`→`Comment`: `@OneToMany`/`@ManyToOne`, cascade
  `ALL` + `orphanRemoval = true` (true composition — a comment has no
  meaning without its post). `Post`↔`Tag`: plain `@ManyToMany` is
  reasonable if tags carry no relationship-specific data (a tag is just a
  label). `User`+`Post`→`Like`: an explicit `Like` entity (not plain
  `@ManyToMany`) specifically because it needs a timestamp attribute,
  with `@ManyToOne` to both `User` and `Post`, no cascade `REMOVE` from
  either side onto the other (deleting a like shouldn't delete the post
  or user, and vice versa is a business decision, but likely also no cascade).

</details>

---

## 2.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Owning side | Always `@ManyToOne` — holds the actual foreign key column |
| `mappedBy` side | Inverse — navigation only, ignored for persistence unless the owning side is also updated |
| Keeping both sides in sync | Use a helper method (`addX()`/`removeX()`) on the parent |
| `CascadeType.REMOVE` | Only for true composition ("part-of") relationships — never reference relationships |
| `orphanRemoval` | Deletes a child when removed from the parent's collection, even without deleting the parent |
| Plain `@ManyToMany` | Fine for simple associations with no relationship-specific data |
| Relationship needs its own attributes | Use an explicit join entity instead of plain `@ManyToMany` |

---

*(Continues to Chapter 3 — Fetching Strategies & The N+1 Problem.)*
