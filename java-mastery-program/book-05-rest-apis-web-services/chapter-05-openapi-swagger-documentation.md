# CHAPTER 5 — OPENAPI/SWAGGER & API DOCUMENTATION AS CONTRACT

---

## 5.1 CONCEPT: OpenAPI Is a Contract, Not "Documentation That Might Go Stale"

### TELUGU EXPLANATION

చాలామంది "API documentation" ని ఒక **afterthought** గా చూస్తారు —
code రాశాక, ఎప్పుడో "వీలున్నప్పుడు" docs రాయడం. ఇది తప్పు mental
model. **OpenAPI Specification (గతంలో Swagger Spec)** ని ఒక
**machine-readable contract** గా చూడాలి — ఇది కేవలం humans చదవడానికే
కాదు, **tooling** కూడా దీన్ని వాడుతుంది:

- **Client SDK generation** — వేర్వేరు languages కి, spec నుండి automatic గా.
- **Server stub generation** — contract-first workflow లో (5.2 చూడండి).
- **Contract testing** — API అసలు spec కి match అవుతుందో లేదో automated గా verify చేయడం.
- **API Gateway configuration** — Chapter 6 లో చూసే gateways, routing/
  validation rules కి OpenAPI spec నే వాడతాయి.

```yaml
openapi: 3.0.3
paths:
  /api/orders/{orderId}:
    get:
      summary: Get an order by ID
      parameters:
        - name: orderId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Order found
          content:
            application/json:
              schema: { $ref: '#/components/schemas/OrderResponse' }
        '404':
          description: Order not found
```

### ENGLISH INTERVIEW ANSWER

"I treat OpenAPI as a contract that tooling depends on, not documentation
written after the fact for humans to read. It drives client SDK
generation across languages, server stub generation in a contract-first
workflow, automated contract testing that verifies the running API
actually matches what's promised, and API gateway configuration. Treating
it as 'just docs' is exactly how it goes stale — treating it as a
contract that tooling actively consumes creates real pressure to keep it
accurate, since a wrong spec breaks generated clients and failing
contract tests, not just confuses a human reader."

---

## 5.2 CONCEPT: Code-First vs Contract-First — A Real Team-Structure Decision

### TELUGU EXPLANATION

**Code-First (springdoc-openapi, annotations ఆధారంగా):**
```java
@Operation(summary = "Get an order by ID")
@ApiResponse(responseCode = "200", description = "Order found")
@GetMapping("/{orderId}")
OrderResponse getOrder(@PathVariable String orderId) { ... }
```
Code రాశాక, OpenAPI spec **automatic గా generate** అవుతుంది annotations
నుండి. **సులభం, fast** — కానీ **risk:** implementation details (Java
type shapes, internal naming) contract లోకి **leak** అయ్యే అవకాశం
ఉంటుంది, మరియు backend team పూర్తి చేసేవరకు, frontend/consumer teams
**wait** చేయాలి.

**Contract-First:** ముందు OpenAPI spec (YAML) ని **design చేసి, team
లందరూ (backend, frontend, partner consumers) దానిమీద agree** అవుతారు
— **అప్పుడు మాత్రమే** implementation మొదలవుతుంది, **parallel గా**
(backend real logic రాస్తుంది, frontend generated mock server తో
పని చేస్తుంది, రెండూ ఏకకాలంలో).

**Senior గా, ఎప్పుడు ఏది వాడాలి:**
- **Code-first:** చిన్న team, internal-only APIs, iteration speed
  ముఖ్యం అయినప్పుడు.
- **Contract-first:** **బహుళ teams** (ముఖ్యంగా వేర్వేరు organizations
  — partner APIs) ఒకే API మీద depend అయినప్పుడు — ఇక్కడ **ముందుగా
  agreement** critical, ఎందుకంటే **contract మారడం costly** (అనేక
  consumers ని ప్రభావితం చేస్తుంది).

### ENGLISH INTERVIEW ANSWER

"Code-first — generating the OpenAPI spec from annotations via something
like springdoc-openapi — is fast and convenient for a single team
iterating quickly, especially on internal-only APIs. Contract-first —
designing and agreeing on the spec before any implementation starts — is
what I push for whenever multiple teams, especially across organizational
boundaries like partner APIs, depend on the same contract. The value is
that frontend and backend teams can work in parallel against an agreed
contract, using a generated mock server on the frontend side while the
backend implements the real logic, and it forces the API's public shape
to be designed deliberately rather than accidentally leaking whatever
internal Java structure happens to fall out of annotation-based
generation. The decision genuinely depends on team structure and how many
independent consumers depend on getting the contract right upfront."

---

## 5.3 CONCEPT: Consumer-Driven Contract Testing — Keeping Docs Honest at Scale

### TELUGU EXPLANATION

Spec accurate గా ఉండాలంటే, కేవలం "బాగా maintain చేయండి" అనే discipline
సరిపోదు — **automated verification** అవసరం. **Consumer-Driven Contract
Testing** (Pact వంటి tools, Book 8/22 లో microservices context లో
వివరంగా) ఈ idea మీద పని చేస్తుంది:

1. **Consumer** (ఉదా: frontend, లేదా ఒక downstream microservice) తను
   API నుండి **ఏం expect చేస్తున్నానో** ఒక contract గా define చేస్తుంది.
2. ఈ contract, **Provider (API owner) యొక్క CI pipeline** లో run అవుతుంది
   — provider యొక్క actual API, ప్రతి consumer యొక్క expectations ని
   satisfy చేస్తుందో లేదో verify చేస్తుంది.
3. Provider ఏదైనా breaking change చేయాలనుకుంటే, **CI immediately fail
   అవుతుంది** — ఏ consumer break అవుతుందో స్పష్టంగా చూపిస్తుంది,
   production లో discover చేసేబదులు.

**ఇది Chapter 7 (Backward Compatibility) కి direct పునాది:** Consumer-driven
contracts, "ఈ API change ఎవర్ని break చేస్తుంది" అనే ప్రశ్నకి **తగినంత
ముందుగానే, automated గా** జవాబు ఇస్తాయి.

### ENGLISH INTERVIEW ANSWER

"Documentation staying accurate isn't a discipline problem you solve by
asking people to remember to update it — it's a verification problem you
solve with automated tests. Consumer-driven contract testing has each
consumer of an API define what they actually expect from it as a
contract, which then runs against the provider's own CI pipeline. If the
provider makes a change that breaks a consumer's expectations, CI fails
immediately, naming exactly which consumer would be affected — instead of
discovering the breakage when a partner's integration mysteriously stops
working in production. This is exactly the kind of tooling that makes
Chapter 7's backward compatibility discipline enforceable rather than
aspirational, and it's a technique I'd specifically bring up when asked
how to manage API evolution safely across many independent consumers."

---

## 5.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Writing API docs | Treats it as an optional, after-the-fact task | Treats OpenAPI spec as a contract tooling and consumers depend on |
| Multiple teams building against one API | Lets the backend team design the contract unilaterally, code-first | Pushes for contract-first design with all stakeholders before implementation |
| "Did this change break any consumers?" | Finds out from a support ticket after deployment | Has consumer-driven contract tests fail in CI before merge |
| Generating a spec | Doesn't consider what leaks into the public contract from internal code shapes | Deliberately designs the public contract shape, independent of internal implementation details |

---

## 5.5 COMMON MISTAKES

1. Treating OpenAPI documentation as optional or write-once-forget.
2. Using code-first generation for a widely-consumed, multi-team/partner
   API, letting internal implementation details leak into the public contract.
3. Having no automated way to detect when an API change breaks an
   existing consumer's expectations.
4. Designing contract-first but never actually validating the running
   implementation against the agreed spec.
5. Assuming a nicely-rendered Swagger UI page automatically means the
   underlying spec is accurate and complete.

---

## 5.6 INTERVIEW QUESTION BANK — CHAPTER 5

**Basic:** 1. What is OpenAPI used for beyond generating a documentation
page? 2. What's the difference between code-first and contract-first API design?

**Intermediate:** 3. When would you choose contract-first over
code-first? 4. What problem does consumer-driven contract testing solve?

**Senior:** 5. Design the contract-first workflow for a new partner-facing
API: who's involved, what artifacts get created and reviewed, and how do
frontend/backend teams work in parallel? 6. How does an accurate OpenAPI
spec reduce the risk of an accidental breaking change?

**Architect:** 7. You're standardizing API documentation practices across
50 microservices with varying team sizes and consumer bases. Would you
mandate contract-first universally, or allow teams to choose based on
context? Justify.

**Scenario:** 8. A partner integration breaks after a routine deployment,
even though the OpenAPI spec's documented endpoints didn't change. What
are two possible root causes given this chapter's material?

**Trick:** 9. "Once you set up code-first OpenAPI generation, your
documentation is guaranteed to always be accurate." True or false?

<details><summary>Key answers</summary>

- Q5: Backend, frontend, and (for a partner API) the partner's technical
  team collaboratively design the OpenAPI YAML spec; once agreed, a
  generated mock server lets frontend/partner development proceed against
  realistic responses while the backend team implements the real
  service, and consumer-driven contract tests (5.3) are set up early to
  continuously verify the eventual real implementation against the
  agreed contract as it's built.
- Q6: An accurate spec is what automated contract tests and API gateway
  validation actually check against — if the spec correctly and
  completely describes the API's real behavior, any deviation (a
  breaking change) is caught by tooling automatically, rather than
  relying on developers manually remembering every consumer's
  expectations before making a change.
- Q7: Likely context-dependent rather than a blanket mandate — small
  internal-only services with a single consuming team can reasonably use
  code-first for speed; any API with external partners, multiple
  independent consuming teams, or high change-coordination cost should be
  required to use contract-first, since the coordination cost of a
  breaking surprise is far higher there — a tiered policy based on
  "external/multi-team" vs "internal/single-team" is a practical
  compromise between rigor and team autonomy.
- Q8: (1) The response *behavior* changed in a way not reflected in the
  documented schema (a field's actual values changed meaning, or a
  previously-always-present optional field started being omitted more
  often) even though the endpoint/method/status codes look unchanged; (2)
  a change to something not typically captured in basic OpenAPI docs —
  rate limiting behavior, authentication requirements, or response
  timing/pagination behavior — broke an assumption the partner's
  integration relied on that was never formally part of the "documented
  contract" to begin with.
- Q9: False — code-first generation accurately reflects what the
  annotations say, but if a developer's annotations are incomplete,
  outdated, or don't capture actual runtime behavior (e.g., an
  undocumented error response actually returned in some code path), the
  generated spec will be just as inaccurate — code-first reduces one
  class of staleness (spec never touched after writing) but doesn't
  guarantee correctness without additional verification like contract testing.

</details>

---

## 5.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Name three things an OpenAPI spec is used for beyond rendering a documentation webpage.
- **Coding Check:** Write an OpenAPI YAML spec for a simple `POST /api/reviews` endpoint including request schema, validation constraints, and 201/400/422 responses.
- **Explanation Check:** Explain in English why "we have a nice Swagger UI page" doesn't guarantee an API's contract is trustworthy.
- **Real-World Check:** Your team is about to expose an API to an external partner for the first time. Design the process (contract-first steps, review, testing) you'd follow before writing any implementation code.
- **Senior Check:** When would investing in consumer-driven contract testing NOT be worth the setup cost?
- **Master Check:** Design a complete API governance workflow combining contract-first design, automated spec validation in CI, and consumer-driven contract testing — describe what blocks a merge and what doesn't, for a team maintaining an API with 5 external partner consumers.

<details><summary>Answers</summary>

- Real-World Check: Draft the OpenAPI spec collaboratively with the
  partner's technical contacts, review and get explicit sign-off before
  any implementation, set up consumer-driven contract tests representing
  the partner's expectations, implement against the agreed spec, and
  validate the final implementation against both the spec and the
  contract tests before the integration goes live — avoiding building
  something the partner then has to renegotiate.
- Senior Check: For a small, single-consumer internal API where the cost
  of the occasional breaking change is low (one team, easy internal
  coordination, quick to fix) — the setup and maintenance overhead of
  consumer-driven contract testing may not be justified relative to just
  communicating directly with the one internal consuming team.
- Master Check: A merge is blocked if: the implementation diverges from
  the agreed OpenAPI spec (automated spec-vs-implementation validation
  fails), or any of the 5 partners' consumer-driven contract tests fail
  against the new code. A merge is NOT blocked by internal refactoring
  that doesn't change the public contract, or by adding genuinely
  backward-compatible additions (new optional fields, new endpoints) that
  don't violate any existing partner's contract — this distinction is
  exactly what makes the governance process enforce real safety without
  freezing all further API development.

</details>

---

## 5.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| OpenAPI spec | A contract tooling depends on — not just human-readable docs |
| Code-first | Fast, good for small/internal teams — risk of leaking implementation details |
| Contract-first | Required for multi-team/partner APIs — enables parallel work, deliberate design |
| Consumer-driven contract testing | Automated, CI-enforced detection of consumer-breaking changes |
| Documentation staleness | Solved by verification (contract tests), not discipline alone |

---

*(Continues to Chapter 6 — API Gateway Patterns & Rate Limiting.)*
