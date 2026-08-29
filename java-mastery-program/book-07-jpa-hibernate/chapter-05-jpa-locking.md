# CHAPTER 5 — LOCKING IN JPA: `@VERSION` AND PESSIMISTIC ANNOTATIONS

> This chapter is the JPA/Hibernate-specific implementation of Book 6
> Chapter 6's locking concepts. If Chapter 6's `SELECT ... FOR UPDATE`
> and version-column patterns aren't fresh, revisit them first — this
> chapter assumes that reasoning and shows how Hibernate automates it.

---

## 5.1 CONCEPT: `@Version` — Optimistic Locking, Fully Automated

### TELUGU EXPLANATION

Book 6 Chapter 6 లో మనం optimistic locking ని **manual SQL** తో చూశాం
(`UPDATE ... WHERE version = ?`). JPA దీన్ని **పూర్తిగా automate**
చేస్తుంది:

```java
@Entity
class Account {
    @Id @GeneratedValue Long id;
    BigDecimal balance;

    @Version // ఇదొక్క annotation — మిగతా అంతా Hibernate చూసుకుంటుంది
    Long version;
}
```

**Hibernate automatic గా ఏం చేస్తుంది:**
1. ప్రతి `UPDATE` కి, `WHERE id = ? AND version = ?` జోడిస్తుంది
   (మీరు చదివిన version value తో).
2. Update **successful** అయితే, `version` ని **automatic గా +1**
   చేస్తుంది.
3. Update **0 rows affect** చేస్తే (మరొకరు మధ్యలో మార్చారు అని అర్థం),
   Hibernate **`OptimisticLockException`** throw చేస్తుంది —
   మీరు ఇక్కడ manual గా "affected rows == 0" చెక్ చేయాల్సిన అవసరం లేదు
   (Book 6 Chapter 6 లో మనం manual SQL తో చేసినట్టు).

```java
@Service
class AccountService {
    @Retryable(retryFor = OptimisticLockException.class, maxAttempts = 3) // Book 2 Chapter 16 retry సూత్రం
    @Transactional
    void withdraw(Long accountId, BigDecimal amount) {
        Account account = entityManager.find(Account.class, accountId);
        if (account.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException();
        }
        account.setBalance(account.getBalance().subtract(amount)); // dirty checking, Chapter 1
        // commit అయినప్పుడు, version mismatch అయితే OptimisticLockException — @Retryable దాన్ని catch చేసి retry చేస్తుంది
    }
}
```

### ENGLISH INTERVIEW ANSWER

"`@Version` fully automates the manual optimistic-locking pattern from
Book 6 — Hibernate appends the version check to every UPDATE's WHERE
clause automatically, increments it on success, and throws
`OptimisticLockException` if zero rows were affected because someone
else's update won the race first. I never manually check affected-row
counts myself with JPA; I just handle the exception. My standard pattern
wraps the transactional method with `@Retryable` targeting
`OptimisticLockException` specifically, following the same retry
discipline from Book 2's production coding chapter — retry a bounded
number of times, since the conflict is expected to be transient and
retrying with fresh data is the correct recovery, not a sign of a genuine bug."

---

## 5.2 CONCEPT: `@Lock` — Pessimistic Locking via Repository Methods

### TELUGU EXPLANATION

Book 6 Chapter 6's `SELECT ... FOR UPDATE` ని, Spring Data repository
methods మీద **declarative గా** apply చేయవచ్చు:

```java
interface AccountRepository extends JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE) // ఇది SELECT ... FOR UPDATE generate చేస్తుంది
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);
}
```

**`LockModeType` options:**
| Type | అర్థం |
|---|---|
| `PESSIMISTIC_READ` | Shared lock — ఇతరులు చదవగలరు, కానీ modify చేయలేరు |
| `PESSIMISTIC_WRITE` | Exclusive lock — `FOR UPDATE`, ఇతరులు చదవలేరు/modify చేయలేరు |
| `PESSIMISTIC_FORCE_INCREMENT` | Pessimistic lock + version ని కూడా force గా increment చేయడం (mixed strategies కి) |
| `OPTIMISTIC` | `@Version` behavior ని explicit గా ఒక specific query కి force చేయడం |

### ENGLISH INTERVIEW ANSWER

"`@Lock(LockModeType.PESSIMISTIC_WRITE)` on a repository query method is
the declarative JPA equivalent of Book 6's `SELECT ... FOR UPDATE` —
Hibernate generates exactly that SQL. `PESSIMISTIC_READ` gives a shared
lock, useful when you need to prevent modification while still allowing
concurrent reads. I choose between `@Version` and `@Lock` using the same
decision framework from Book 6 Chapter 6 — frequent contention or
expensive-to-retry operations favor pessimistic locking with `@Lock`;
rare contention favors optimistic locking with `@Version` — JPA doesn't
change that underlying decision, it just gives me clean, declarative
syntax for whichever I choose."

---

## 5.3 CONCEPT: The Detached-Entity Version Trap

### TELUGU EXPLANATION

**ఇది Chapter 1 (detached entities) + ఈ chapter (versioning) కలిపిన,
తరచుగా tricky అయ్యే scenario:** ఒక entity ని fetch చేసి (version=5
తో), దాన్ని **detach అయ్యేలా వదిలేసి** (ఉదా: HTTP response గా పంపి,
తర్వాత మళ్ళీ receive చేసి), తర్వాత **merge** చేసినప్పుడు —
Hibernate, ఈ detached entity యొక్క version ని, **database లో ప్రస్తుత
version** తో compare చేస్తుంది:

```java
Account detachedAccount = /* client నుండి వచ్చిన, version=5 తో ఉన్న entity */;
Account merged = entityManager.merge(detachedAccount);
// database లో ప్రస్తుత version 7 అయితే (మధ్యలో వేరే update జరిగింది),
// merge() ఇక్కడే OptimisticLockException throw చేస్తుంది
```

**ఇదే "long conversation" optimistic locking యొక్క real-world use case**
— ఒక user, ఒక form ని open చేసి, కొన్ని నిమిషాలు edit చేసి, submit
చేసేసరికి, మరొక user అదే record ని మార్చేసి ఉంటే — version mismatch
ద్వారా ఈ **"lost update across a human-timescale gap"** ని కూడా
Hibernate పట్టుకుంటుంది, కేవలం database-transaction-timescale
conflicts మాత్రమే కాదు.

### ENGLISH INTERVIEW ANSWER

"This is where Chapter 1's detached-entity concept and this chapter's
versioning meet in a genuinely important real-world scenario: a user
opens an edit form, which detaches the loaded entity (it travels to the
client and back, well outside any single transaction), edits it over
several minutes, then submits. When the update finally happens via
`merge()`, Hibernate compares the version the client is submitting
against the current database version — if someone else modified the
record in between, `merge()` throws `OptimisticLockException` right
there. This is actually optimistic locking's most valuable real use
case: not just protecting against a race between two nearly-simultaneous
transactions, but catching a 'lost update' across a human-timescale gap —
two people editing the same record minutes apart, which no
database-transaction-level locking mechanism could ever catch, since
there's no long-held database transaction spanning the user's think time
at all."

---

## 5.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Handling concurrent updates in JPA | Doesn't add `@Version`, relies on "it probably won't happen" | Adds `@Version` by default on any frequently-updated entity |
| `OptimisticLockException` | Treats it as an unhandled bug/500 error | Catches it, retries with `@Retryable`, or surfaces a clear "someone else edited this" message to the user |
| Edit-form scenarios (detach, edit, merge) | Doesn't realize a lost-update risk exists across the human editing gap | Recognizes `@Version` protects exactly this scenario via `merge()` |
| High-contention hot row | Uses `@Version` uniformly everywhere | Switches to `@Lock(PESSIMISTIC_WRITE)` for the specific high-contention case |

---

## 5.5 COMMON MISTAKES

1. Not adding `@Version` to entities that are frequently updated
   concurrently, missing lost-update protection entirely.
2. Not catching `OptimisticLockException`, letting it surface as an
   unhandled 500 error instead of a meaningful retry or user-facing message.
3. Using `@Version` uniformly even for genuinely high-contention hot
   rows where pessimistic locking would perform better.
4. Manually managing a version field instead of letting `@Version`
   automate the increment and check.
5. Not considering the human-timescale "edit form" lost-update scenario,
   assuming locking only matters for near-simultaneous database transactions.

---

## 5.6 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. What does `@Version` automate? 2. What exception does a
version conflict throw?

**Intermediate:** 3. How does `@Lock(LockModeType.PESSIMISTIC_WRITE)`
relate to Book 6's `SELECT ... FOR UPDATE`? 4. What's the difference
between `PESSIMISTIC_READ` and `PESSIMISTIC_WRITE`?

**Senior:** 5. Design the concurrency-handling strategy for an
"edit profile" form (long user think-time between load and submit) using
`@Version` — what happens on conflict, and what should the user see? 6.
Why is `OptimisticLockException` on a `merge()` call a fundamentally
different scenario from one on a same-transaction `flush()`?

**Architect:** 7. You're designing a collaborative document-editing
feature (multiple users can view/edit the same record) where losing a
user's edit due to optimistic lock conflicts is a poor user experience.
What alternatives or enhancements to plain `@Version` would you consider
(field-level merging, operational transforms, last-write-wins with warning)?

**Scenario:** 8. A `@Retryable(retryFor = OptimisticLockException.class)`
method keeps failing after all retry attempts under moderate load. What
does this suggest about the actual contention level, and what's the
better fix?

**Trick:** 9. "`@Version` prevents lost updates in every scenario where
two users might edit the same record." True or false?

<details><summary>Key answers</summary>

- Q5: On load, capture the version. On submit, attempt the update/merge;
  if `OptimisticLockException` is thrown, do NOT blindly retry with the
  user's stale data — that would silently overwrite whatever the other
  user saved. Instead, surface a clear message ("this record was changed
  by someone else since you started editing — please review the current
  version and reapply your changes"), ideally showing what changed, so
  the user can make an informed decision rather than the system guessing.
- Q6: A conflict within one transaction's `flush()` reflects a genuine
  near-simultaneous database-level race, typically resolved by retrying
  quickly with fresh data (Book 2's retry principles apply cleanly). A
  conflict on `merge()` after a detached entity's human-timescale edit
  reflects that someone else's *entire* edit already completed and
  committed — blindly retrying with the original stale data would
  silently discard that other completed work, which is why this scenario
  needs human-facing conflict resolution (Q5), not an automatic retry.
- Q7: Consider field-level merge strategies (only reject/merge the
  specific fields that actually conflict, not the whole record), showing
  a diff/merge UI to the user when a conflict occurs, or accepting a
  deliberate last-write-wins policy with an audit trail if the business
  context tolerates it — plain `@Version`'s all-or-nothing "reject the
  entire update" behavior is a reasonable default but not always the best
  UX for genuinely collaborative editing scenarios.
- Q8: Persistent retry failures under only moderate load suggest the
  actual contention on this record/table is higher than "rare" — the
  premise optimistic locking depends on for good performance isn't
  holding. The better fix is likely switching this specific hot path to
  pessimistic locking (`@Lock(PESSIMISTIC_WRITE)`), which serializes
  access directly rather than repeatedly failing and retrying under real contention.
- Q9: False — as Q7 discusses, `@Version` prevents a lost update in the
  sense of *detecting* a conflict and rejecting the second write, but it
  doesn't merge or reconcile the two users' changes — the second user's
  entire submission is rejected, which is a form of conflict *detection*,
  not conflict *resolution*; whether this counts as "preventing" the lost
  update depends on how the rejection is then handled (Q5), and it
  certainly doesn't handle every collaborative-editing scenario gracefully
  without additional UX work.

</details>

---

## 5.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does `@Version` need to be checked at `merge()` time for a detached entity, not just at `flush()` time for a managed one?
- **Coding Check:** Implement an account withdrawal method using `@Version` and `@Retryable`, and write a test simulating two concurrent withdrawal attempts where one must fail with `OptimisticLockException`.
- **Explanation Check:** Explain in English why blindly retrying an `OptimisticLockException` from a `merge()` call (detached entity, human-timescale edit) is dangerous, unlike retrying one from a same-request `flush()`.
- **Real-World Check:** Your team's inventory system uses `@Version` on the `Product` entity, but under flash-sale load, retry storms are degrading performance. Diagnose and propose the fix using this chapter's material.
- **Senior Check:** When would you use BOTH `@Version` and `@Lock` on the same entity, for different operations?
- **Master Check:** Design the complete concurrency strategy for a seat-booking system (this book's callback to Book 6 Chapter 5's Master Check) using JPA specifically: which entities get `@Version`, which repository methods get `@Lock`, and how `OptimisticLockException` vs pessimistic-lock waits are handled differently in the booking flow.

<details><summary>Answers</summary>

- Real-World Check: Exactly the Q8 scenario — high contention under
  flash-sale conditions defeats optimistic locking's core assumption;
  switch the inventory-decrement operation specifically to
  `@Lock(LockModeType.PESSIMISTIC_WRITE)` for that hot path, potentially
  keeping `@Version` for other, lower-contention `Product` updates (price
  changes, description edits) elsewhere in the same entity.
- Senior Check: A `Product` entity might use `@Version` for general
  updates (description, price — low contention, human-editing-gap
  scenarios where detecting-and-informing is appropriate) while a
  dedicated repository method for the specific "reserve stock" operation
  uses `@Lock(PESSIMISTIC_WRITE)` for the high-contention, correctness-critical path — both applied to the same entity type for different operations.
- Master Check: `Seat`/`Booking` entities get `@Version` for general
  updates. The specific "reserve this seat" repository method uses
  `@Lock(LockModeType.PESSIMISTIC_WRITE)`, since seat reservation during
  a popular on-sale event is exactly the high-contention scenario
  favoring pessimistic locking (Book 6 Chapter 6's reasoning, applied
  here). `OptimisticLockException` elsewhere (e.g., a user editing their
  booking's contact details) is handled by informing the user and asking
  them to retry with current data; pessimistic lock waits on the seat
  reservation itself simply block briefly (acceptable given the brief
  hold duration) rather than failing outright, since the whole point of
  choosing pessimistic locking there was to serialize access rather than
  detect-and-reject conflicts.

</details>

---

## 5.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| `@Version` | Automates optimistic locking — auto-increment + `OptimisticLockException` on conflict |
| `@Lock(PESSIMISTIC_WRITE)` | JPA equivalent of `SELECT ... FOR UPDATE` (Book 6 Ch. 6) |
| `PESSIMISTIC_READ` vs `WRITE` | Shared (blocks writes) vs exclusive (blocks reads and writes) |
| `OptimisticLockException` on `flush()` | Near-simultaneous DB conflict — safe to retry with fresh data |
| `OptimisticLockException` on `merge()` | Human-timescale lost update — needs user-facing conflict resolution, NOT blind retry |
| High contention + `@Version` | Retry storms signal a mismatch — switch to `@Lock` for that hot path |

---

*(Continues to Chapter 6 — Second-Level Cache & Performance Optimization.)*
