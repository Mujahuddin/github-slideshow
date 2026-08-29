# CHAPTER 7 — API EVOLUTION: BACKWARD COMPATIBILITY & IDEMPOTENCY

---

## 7.1 CONCEPT: The Robustness Principle Applied to APIs

### TELUGU EXPLANATION

**Postel's Law ("Robustness Principle"):** "**Be conservative in what you
send, be liberal in what you accept.**" API design కి ఇది ఇలా apply
అవుతుంది:

- **Server (API provider):** Response లో **ఖచ్చితంగా అవసరమైనవే**
  పంపాలి, కానీ **ఇప్పటికే ఉన్న fields ని ఎప్పుడూ తీసేయకూడదు, రకం
  మార్చకూడదు**.
- **Client (API consumer):** Response లో **తనకి తెలియని కొత్త fields**
  వచ్చినా, **fail అవ్వకూడదు** — వాటిని ignore చేయాలి. Client ఈ
  discipline పాటించకపోతే (ఉదా: strict schema validation, "extra
  fields ఉంటే reject చేయి"), **server ఏ non-breaking addition చేసినా,
  client break అవుతుంది** — ఇది server fault కాదు, **client యొక్క
  brittle parsing** fault.

### ENGLISH INTERVIEW ANSWER

"Postel's Law — be conservative in what you send, liberal in what you
accept — is the philosophical foundation of safe API evolution. As a
provider, I never remove or change the type of an existing field. As a
consumer, my client code should tolerate unknown new fields in a response
without failing — using a JSON parser configured to ignore unrecognized
properties rather than strict schema validation that rejects anything not
explicitly expected. This matters because if consumers write brittle
parsers that break on any new field, then providers can never add fields
safely — every addition becomes a breaking change in practice, even
though it shouldn't be in principle. Robust client-side parsing is what
actually makes non-breaking API evolution possible in the real world."

---

## 7.2 CONCEPT: The Complete Breaking vs Non-Breaking Change Taxonomy

### TELUGU EXPLANATION

| మార్పు | Breaking? | ఎందుకు |
|---|---|---|
| కొత్త **optional** response field add చేయడం | ❌ కాదు | Postel's Law — clients ignore చేయాలి |
| కొత్త **endpoint** add చేయడం | ❌ కాదు | Existing endpoints ప్రభావితం కావు |
| కొత్త **optional** request field add చేయడం | ❌ కాదు | పాత clients దాన్ని పంపకపోయినా పని చేస్తుంది |
| Response నుండి ఒక field **తీసేయడం** | ✅ **అవును** | Clients దాన్ని expect చేస్తుండొచ్చు |
| ఒక field యొక్క **type మార్చడం** (string→number) | ✅ **అవును** | Client parsing fail అవుతుంది |
| కొత్త **required** request field add చేయడం | ✅ **అవును** | పాత clients దాన్ని పంపరు — validation fail అవుతుంది |
| ఒక field **rename** చేయడం | ✅ **అవును** | ఇది "తీసేయడం + కొత్తది add చేయడం" గా ఉంటుంది |
| **Validation rules కఠినతరం చేయడం** (ఉదా: max length తగ్గించడం) | ✅ **అవును** (subtle!) | ఇప్పటికే valid గా ఉన్న data ఇప్పుడు invalid అవుతుంది |
| **Error response structure మార్చడం** | ✅ **అవును** (frequently ignored!) | Clients error handling logic విరిగిపోతుంది |

**అత్యంత frequently missed కేసు:** **Validation rules కఠినతరం చేయడం**
— చాలామంది దీన్ని "non-breaking" అని అనుకుంటారు (structure మారలేదు
కదా), కానీ ఇది **ఇప్పటికే valid గా accept అవుతున్న requests ని ఇప్పుడు
reject** చేస్తుంది — ఇది కూడా ఒక breaking change యే.

### ENGLISH INTERVIEW ANSWER

"The taxonomy I use goes beyond the obvious 'don't remove fields' rule.
Tightening validation is the case most teams miss — narrowing a max
length, adding a new required field, making a previously-optional field
mandatory — none of these change the response *structure*, but they
change what was previously *accepted* as valid, which breaks any existing
client sending data that used to work. I also flag error response
structure changes specifically, since client error-handling logic often
depends on it just as much as success-path parsing, and it's easy to
overlook when reviewing a 'just the happy path' diff."

---

## 7.3 CONCEPT: The Expand-Contract Pattern for Genuinely Breaking Changes

### TELUGU EXPLANATION

ఒక genuine breaking change (ఉదా: field rename) చేయాల్సి వస్తే, **ఒక్క
step లో చేయకూడదు** — **Expand-Contract** (ఇది Book 6/Database migration
strategies కి కూడా వర్తించే సూత్రమే) పద్ధతి వాడాలి:

1. **Expand:** కొత్త field **add చేయండి**, పాత దాన్ని అలాగే ఉంచండి —
   server **రెండింటినీ** populate చేస్తుంది (ఉదా: `customerName` కొత్తది,
   `custName` పాతది, రెండూ ఒకే value తో).
2. **Migrate:** Consumers ని కొత్త field కి **మారమని communicate**
   చేయండి (deprecation notice, section 7.4), వాళ్ళు మారేవరకు **సమయం
   ఇవ్వండి**.
3. **Contract:** అన్ని (లేదా దాదాపు అన్ని) consumers మారాక, పాత field
   ని **తీసేయండి**.

ఇది ఒక **single breaking change** ని, **మూడు non-breaking steps** గా
విభజిస్తుంది — ఏ దశలోనూ existing consumers ఆకస్మికంగా break అవ్వరు.

### ENGLISH INTERVIEW ANSWER

"For a genuinely necessary breaking change like renaming a field, I never
do it in one step — I use expand-contract. First, expand: add the new
field alongside the old one, with the server populating both from the
same underlying data, so nothing breaks yet. Then, migrate: communicate
the change to consumers with a clear deprecation timeline, giving them
time to switch to the new field. Finally, contract: once consumers have
migrated — verified via usage monitoring or consumer-driven contract
tests from Chapter 5 — remove the old field. This turns one breaking
change into three individually non-breaking steps, and it's the same
underlying pattern used for backward-compatible database schema
migrations, just applied to an API contract instead."

---

## 7.4 CONCEPT: Deprecation Strategy — the `Sunset` Header and Communication

### TELUGU EXPLANATION

Ended field/endpoint ని deprecate చేసేటప్పుడు, **explicit, machine-readable
signal** ఇవ్వాలి — కేవలం "docs లో ఒక line" సరిపోదు:

```
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Link: <https://api.example.com/docs/migration-v2>; rel="deprecation"
```

`Sunset` header (RFC 8594) — "ఈ endpoint/field **ఎప్పుడు నిజంగా
తీసేయబడుతుందో**" ఖచ్చితంగా చెప్పడం. Monitoring/alerting ఈ header ఉన్న
responses ని track చేసి, "ఇంకా ఎవరు deprecated endpoint వాడుతున్నారు"
అని గుర్తించవచ్చు — Chapter 5 consumer-driven contracts తో కలిపి,
**ఎప్పుడు safely contract చేయవచ్చో** data-driven గా నిర్ణయించడానికి.

### ENGLISH INTERVIEW ANSWER

"Deprecation needs to be a machine-readable signal, not just a
documentation note easy to miss. I use the `Deprecation` and `Sunset`
headers (RFC 8594) on responses from deprecated endpoints/fields,
specifying exactly when they'll actually be removed, plus a `Link` header
pointing to migration documentation. This lets us — and, ideally, the
consumers themselves — programmatically monitor which deprecated
functionality is still actually being used and by whom, which combined
with Chapter 5's consumer-driven contract tests gives a genuinely
data-driven answer to 'is it safe to complete the contract phase yet,' rather
than guessing or just picking an arbitrary date."

---

## 7.5 CONCEPT: Idempotency in Practice — The `Idempotency-Key` Header Standard

### TELUGU EXPLANATION

Book 2 Chapter 16 లో మనం idempotency keys concept నేర్చుకున్నాం. ఇక్కడ,
దాన్ని **REST API స్థాయిలో, standard header గా** ఎలా వాడాలో చూద్దాం
(Stripe వంటి major APIs ఇదే pattern వాడతాయి):

```http
POST /api/payments
Idempotency-Key: 7da8f9e2-4b3c-4a1d-9f5e-1234567890ab
Content-Type: application/json

{"amount": 1000, "currency": "USD"}
```

**Server-side handling (Book 2 Chapter 16 పద్ధతిని, DB-backed గా
extend చేయడం):**
```java
@PostMapping("/payments")
ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @Valid @RequestBody PaymentRequest request) {

    // Unique constraint ఉన్న DB column మీద ఆధారపడి, atomic గా check+insert
    Optional<PaymentResponse> existing = idempotencyStore.findResponse(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get()); // అదే response, మళ్ళీ charge చేయకుండా
    }

    PaymentResponse response = paymentService.charge(request);
    idempotencyStore.save(idempotencyKey, response); // అదే transaction లో (Book 4 Chapter 5)
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

**Production-grade detail (DB unique constraint, race-condition-proof):**
Book 2 Chapter 16 లో `ConcurrentHashMap.computeIfAbsent` వాడాము (single
JVM). **Multiple gateway/service instances** (Chapter 6) ఉన్నప్పుడు,
in-memory map సరిపోదు — **DB level `UNIQUE` constraint** మీద
`idempotency_key` column ఉండాలి, రెండు concurrent requests (వేర్వేరు
instances నుండి వచ్చినా) ఒకేసారి insert చేయడానికి ప్రయత్నిస్తే,
**database itself** ఒకదాన్ని accept చేసి, రెండోదాన్ని constraint
violation తో reject చేస్తుంది — ఇదే **distributed** correctness
guarantee ఇస్తుంది, ఒక్క JVM లోపలి `ConcurrentHashMap` ఇవ్వలేనిది.

### ENGLISH INTERVIEW ANSWER

"I extend Book 2's idempotency key pattern to the standard `Idempotency-Key`
HTTP header convention used by APIs like Stripe. The important upgrade at
the real API level is that `ConcurrentHashMap.computeIfAbsent` only
guarantees atomicity within a single JVM — the moment you have multiple
service or gateway instances, which Chapter 6 established is the norm,
you need the atomicity guarantee to come from the database itself: a
`UNIQUE` constraint on the idempotency key column means that even if two
concurrent requests with the same key hit two different instances
simultaneously, the database allows exactly one insert to succeed and
rejects the other with a constraint violation, which the service then
handles by returning the already-stored result. This combines Book 4's
`@Transactional` material — the charge and the idempotency record write
happen in one atomic transaction — with Book 1's concurrency principles,
just enforced at the database layer instead of in-process."

---

## 7.6 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Tightening a validation rule | Assumes it's safe since "the structure didn't change" | Recognizes it as breaking — it rejects previously-valid requests |
| Renaming a field | Renames it directly in one release | Uses expand-contract: add new, deprecate old, remove old later |
| Deprecating an endpoint | Mentions it in a changelog/docs page | Uses `Sunset`/`Deprecation` headers, monitors actual usage before removal |
| Idempotency at scale | Uses an in-memory map, assumes it's enough | Uses a DB unique constraint for correctness across multiple instances |

---

## 7.7 COMMON MISTAKES

1. Treating validation-tightening as automatically non-breaking.
2. Renaming or removing fields in a single step instead of using
   expand-contract.
3. Deprecating functionality with only a documentation note, no
   machine-readable signal or usage monitoring.
4. Relying on an in-memory idempotency store when multiple service
   instances exist.
5. Client code using strict schema validation that rejects unknown
   fields, making even correctly non-breaking server changes feel
   breaking in practice.

---

## 7.8 INTERVIEW QUESTION BANK — CHAPTER 7

**Basic:** 1. What is Postel's Law, applied to APIs? 2. Why is adding a
new required request field a breaking change?

**Intermediate:** 3. Explain the expand-contract pattern with a concrete
field-rename example. 4. Why does an in-memory idempotency store fail
once there are multiple service instances?

**Senior:** 5. Design the complete deprecation timeline for removing a
legacy `/api/v1/orders` endpoint in favor of `/api/v2/orders`, including
headers, monitoring, and decision criteria for when it's safe to remove
v1. 6. Why is tightening a validation rule a commonly-missed breaking change?

**Architect:** 7. You're setting API evolution policy for an organization
with dozens of teams. What automated checks (in CI, or via a shared
library/gateway policy) would you implement to prevent teams from
accidentally shipping breaking changes?

**Scenario:** 8. A team ships a change that reduces a `description` field's
max length from 1000 to 200 characters, considering it a minor
tweak. Several partner integrations start failing. Explain what happened and how it could have been prevented.

**Trick:** 9. "As long as you don't remove any fields, your API changes are always backward-compatible." True or false?

<details><summary>Key answers</summary>

- Q5: Add `Deprecation`/`Sunset` headers to v1 responses, with a sunset
  date 6-12 months out (context-dependent); document the migration path
  to v2; monitor actual v1 traffic volume and, ideally, which specific
  consumers still use it (via API keys/auth identity in logs); set up
  consumer-driven contract tests (Chapter 5) for any known remaining
  consumers; only remove v1 once traffic has dropped to zero or all known
  consumers have explicitly confirmed migration — data-driven, not
  calendar-driven alone.
- Q6: It's missed because the response/request *shape* (fields, types)
  doesn't change at all — teams checking "did I change the schema"
  naturally overlook it, even though it silently rejects requests that
  used to succeed, which is precisely a breaking change from the
  consumer's perspective, just a subtler one than a structural change.
- Q7: Automated OpenAPI diff tooling in CI that flags exactly the
  taxonomy from section 7.2 (removed fields, type changes, new required
  fields, tightened constraints) and fails the build without an explicit
  override/sign-off; mandatory consumer-driven contract tests for any API
  with external/cross-team consumers (Chapter 5); a shared linting policy
  for API design consistency (status codes, versioning) enforced at PR review time.
- Q8: This is exactly the "tightened validation is breaking" case from
  section 7.2 — partner integrations that were sending longer, previously-
  valid descriptions started getting validation failures on a field whose
  *structure* never changed, which is why the team didn't flag it as
  risky. Prevention: treat validation-tightening explicitly as a breaking
  change in the team's change-review checklist, and use consumer-driven
  contract tests that would have caught real partner data patterns
  hitting the new, stricter limit before the change shipped.
- Q9: False — as the full taxonomy in section 7.2 shows, type changes,
  new required fields, tightened validation, and error-structure changes
  can all be breaking even without removing a single field.

</details>

---

## 7.9 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is tightening a validation constraint a breaking change even though it doesn't alter the response/request schema shape?
- **Coding Check:** Implement server-side idempotency key handling backed by a database unique constraint (not just an in-memory map), following section 7.5's pattern.
- **Explanation Check:** Explain in English why a client that uses strict schema validation (rejecting unknown fields) undermines the whole premise of backward-compatible API evolution.
- **Real-World Check:** Your team needs to change a `status` field's possible values from `["ACTIVE", "INACTIVE"]` to include a new `"PENDING"` state. Is this breaking? Under what client-side assumption would it become breaking?
- **Senior Check:** When would you skip the full expand-contract process and just make a breaking change directly, accepting the coordination cost?
- **Master Check:** Design the complete safe-evolution process for changing a core `Order` API's `total` field from representing pre-tax to post-tax amount (same field name and type, completely different meaning) — explain why this is arguably worse than a structural breaking change, and how you'd handle it.

<details><summary>Answers</summary>

- Real-World Check: Adding a new enum value is generally non-breaking IF
  clients handle unknown/unexpected enum values gracefully (e.g., a
  default/fallback case) — it becomes breaking if any client code has
  an exhaustive switch/if-else over the exact known values with no
  default case, which would either crash or silently misbehave on
  encountering `"PENDING"` — worth flagging to consumers even for an
  addition, given how commonly this specific assumption is violated in practice.
- Senior Check: When there are few enough known consumers that direct,
  coordinated communication and a synchronized release is genuinely more
  efficient than the multi-phase expand-contract process — e.g., an
  API with exactly one internal consumer team that can coordinate a
  same-day joint deployment; expand-contract is a tool for managing an
  unknown or large number of independent consumers, not a rule to apply
  unconditionally regardless of actual consumer count.
- Master Check: This is a *semantic* breaking change with no structural
  signal at all — same field name, same type — making it far more
  dangerous than a structural change, since neither schema validation
  nor most contract tests checking only shape/type would catch it; the
  correct approach is to treat this exactly like a rename despite the
  name staying the same: introduce a NEW, distinctly-named field
  (`totalPostTax` or similar) alongside the existing `total` (left
  unchanged in meaning), communicate the semantic distinction explicitly,
  migrate consumers to the new field, and only then consider changing or
  removing the ambiguous original — silent semantic changes to
  unchanged-looking fields are exactly the class of bug that "just diff
  the schema" tooling cannot catch, requiring human-communicated,
  deliberate handling.

</details>

---

## 7.10 CHEAT SHEET

| Concept | Rule |
|---|---|
| Postel's Law | Server: conservative in what you send. Client: liberal in what you accept (ignore unknown fields) |
| Non-breaking | New optional fields, new endpoints, new optional request fields |
| Breaking | Removed/renamed fields, type changes, new required fields, tightened validation, error structure changes |
| Expand-Contract | Add new → migrate consumers → remove old, never in one step |
| Deprecation signal | `Deprecation`/`Sunset` HTTP headers (RFC 8594), not just docs |
| Idempotency at scale | DB `UNIQUE` constraint on idempotency key, not just an in-memory map |
| Silent semantic change | Same field/type, different meaning — the most dangerous, least detectable breaking change |

---

## BOOK 5 — CHAPTER 7 COMPLETE

*(All 7 chapters of Book 5's chapter content are now complete. See
`final-assessment-mock-interview-project.md` in this directory for the
cumulative Final Assessment, the REST API Mock Interview round, and the
Book 5 capstone Project Assignment.)*
