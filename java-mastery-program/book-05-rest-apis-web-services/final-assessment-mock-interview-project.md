# BOOK 5 — FINAL ASSESSMENT, REST API MOCK INTERVIEW ROUND, AND CAPSTONE PROJECT

---

## PART A — FINAL ASSESSMENT (CUMULATIVE, ALL 7 CHAPTERS)

1. A mobile client retries a `PUT /api/orders/{id}/address` three times
   after a timeout. Is this safe without an idempotency key? Justify
   using Chapter 1's method semantics. *(Ch. 1)*
2. Design cursor-based pagination for an endpoint returning audit log
   entries, where entries are append-only (never updated/deleted).
   Would offset pagination's "page drift" problem even apply here? *(Ch. 2)*
3. Explain why a JWT's payload being readable by anyone is not itself a
   security flaw, but putting a password hash in that payload would be. *(Ch. 3)*
4. A mobile app team wants to implement "Login with Company SSO" using
   OAuth2. Which grant type, and why not the alternatives? *(Ch. 4)*
5. Your team's OpenAPI spec says a field is optional, but the actual
   implementation treats it as required. Which is the "real" contract,
   and how would you find and fix this discrepancy systematically? *(Ch. 5)*
6. Explain why rate limiting "1000 requests/minute per API key" requires
   different infrastructure at 1 gateway instance vs 20 gateway
   instances. *(Ch. 6)*
7. Is narrowing an enum field's allowed values (removing one previously
   valid option) a breaking change? Compare this to widening it (adding
   a new value). *(Ch. 7)*
8. Design the complete request flow for a client calling a payment API
   with an idempotency key, where the client's connection drops after
   sending the request but before receiving the response — what should
   happen when the client retries?
9. Why does OpenID Connect exist as a separate specification instead of
   OAuth2 simply being extended to include authentication semantics directly?
10. Trace how a "safe" architectural decision in one chapter enables a
    later chapter's guarantee: connect statelessness (Ch. 1) to
    horizontal scaling of rate limiting (Ch. 6).

<details>
<summary>Answer Key</summary>

1. Yes, safe — PUT is idempotent, so retrying the same address update
   any number of times leaves the resource in the same end state; no
   idempotency key is needed here, unlike a POST creating a new resource.
2. Cursor-based pagination anchored on a monotonically increasing ID or
   timestamp; since entries are append-only, offset-based pagination's
   "page drift" from mid-list insertions/deletions genuinely cannot occur
   here — new entries only ever append at the end, which offset
   pagination handles correctly for already-fetched earlier pages. Cursor
   pagination would still be preferred for its performance advantage at
   scale, even though the specific drift bug doesn't apply.
3. Readability alone isn't a flaw — JWTs are designed to be
   Base64url-encoded, not encrypted, and non-sensitive identity claims
   are meant to be read by the client. A password hash, however, is
   sensitive data that should never be exposed to the client or anyone
   who intercepts the token — putting it in the payload violates the
   "don't put sensitive data in JWT claims" principle regardless of the
   signature's integrity guarantee.
4. Authorization Code Flow with PKCE — the mobile app is a public
   client unable to safely store a static secret, ruling out flows
   assuming a confidential client; Implicit Flow and ROPC are both
   explicitly deprecated/discouraged for the reasons in Chapter 4.
5. The *actual runtime behavior* is the real contract that matters to
   consumers, regardless of what the spec claims — but this discrepancy
   itself is a serious problem (the spec is lying to consumers). Fix
   systematically with automated spec-vs-implementation validation in CI
   (Chapter 5) that would catch exactly this class of drift, rather than
   relying on manual spec maintenance.
6. At 1 instance, a simple in-memory counter correctly enforces the
   limit. At 20 instances behind a load balancer, each with its own
   independent in-memory counter, the effective aggregate limit becomes
   up to 20x the intended value unless a shared, atomic, distributed
   counter (Redis-backed `INCR`) is used instead.
7. Narrowing (removing a previously valid enum value) IS breaking —
   any consumer currently sending or expecting that value breaks.
   Widening (adding a new value) is generally non-breaking UNLESS a
   consumer has exhaustive switch/if-else logic with no default case
   over the previously-known values (Chapter 7's enum-widening nuance).
8. The client should retry the SAME request with the SAME idempotency
   key. The server, having already processed and stored the result from
   the first (successful, but response-lost) attempt, returns the
   already-computed result rather than re-executing the payment — this
   is exactly the scenario idempotency keys are designed for: the
   request succeeded, but the client never learned that, and a naive
   retry-without-a-key would have caused a duplicate charge.
9. Because OAuth2's core design and threat model are specifically about
   delegated *authorization* (a client acting on a resource owner's
   behalf against a resource server) — retrofitting rigorous,
   standardized authentication semantics (a verifiable "who is this
   user" identity token, with defined claims and discovery metadata)
   directly into OAuth2 would have either compromised its focused
   design or required the same amount of new specification work anyway;
   building OIDC as a layer on top let both specifications stay focused
   and interoperable independently.
10. Because REST/Book 4 statelessness means no server-side session
    state ties a client to a specific instance, any gateway instance can
    handle any request — which is exactly the precondition that makes a
    shared, Redis-backed distributed rate limit counter (Ch. 6) the
    correct solution: if requests were sticky to specific instances
    (stateful), you could almost get away with per-instance counters,
    but statelessness (by design, for scalability) is precisely what
    makes per-instance counters incorrect and a shared counter necessary.

</details>

---

## PART B — MOCK INTERVIEW: REST API ROUND

**Interviewer:** "Design the API for a hotel booking system's search
endpoint: `GET /api/hotels/search`. Walk me through query parameters,
pagination, and how you'd version this API as requirements evolve."

**Model answer:** "Query parameters would include filters like
`city`, `checkIn`, `checkOut`, `minPrice`/`maxPrice`, `guests`, plus
`sort=price,asc` following the Chapter 2 convention. For pagination, I'd
lean toward cursor-based, since a hotel inventory changes frequently
(availability updates) and search result sets can be large — though I'd
also note that if the product genuinely needs 'jump to page 5 of results'
UX, offset-based might be the pragmatic choice here despite its
trade-offs, since search results aren't as deep or as write-heavy as,
say, a social feed. For versioning, I'd start at `/api/v1/hotels/search`
and treat it as a last resort — most evolving requirements, like adding a
new filter parameter or a new field in the response, are backward-
compatible and don't need a version bump at all; I'd only introduce v2 if
a genuinely breaking change becomes necessary, like restructuring the
response shape entirely."

**Follow-up:** "How would you handle 'total results found' in a
cursor-paginated response?" (Optionally provide an approximate count, or
omit it — computing an exact count on a large, frequently-changing
dataset can be expensive, exactly Chapter 2's `totalCount` trade-off.)

---

**Interviewer:** "A junior engineer on your team wants to store the JWT
access token in `localStorage` for simplicity. Walk me through how you'd
guide this decision."

**Model answer:** "I wouldn't just say 'use httpOnly cookies instead'
without explaining the actual trade-off, since it's not a
universally-solved question. I'd walk through that `localStorage` is
vulnerable to XSS — any successful script injection can read and
exfiltrate the token directly — while `httpOnly` cookies eliminate that
specific risk but introduce CSRF exposure, which needs `SameSite`
attributes and CSRF tokens to mitigate properly. I'd ask about our
existing XSS defenses — do we have a solid Content Security Policy, are
we consistently escaping output — because if those are weak,
`localStorage` is a bigger risk than if they're strong. My default
recommendation leans toward `httpOnly` cookies with proper CSRF
protection, since in practice, guaranteeing zero XSS across an entire
application's codebase over its lifetime is harder than correctly
configuring CSRF defenses once, but I'd want the junior engineer to
understand *why*, not just follow the recommendation."

---

**Interviewer:** "Your team needs to add a required field to an existing,
widely-used `CreateUserRequest`. How do you do this without breaking
existing API consumers on day one?"

**Model answer:** "I wouldn't add it as required immediately — that's an
instant breaking change per Chapter 7's taxonomy. I'd add it as optional
first, with a sensible default applied server-side when it's absent, and
communicate to consumers that it will become mandatory by a specific
date — using `Deprecation`-style signals if there's an existing field
being replaced, or just clear proactive communication if it's genuinely
new. I'd monitor how many requests are still omitting the field as the
deadline approaches, ideally via consumer-driven contract tests if we
have significant external consumers, and only enforce it as truly
required — potentially in a new API version if it needs to be a hard
break — once adoption data shows it's safe. This is the same expand-
contract discipline applied to a request field instead of a response field."

---

## PART C — CAPSTONE PROJECT: "PARTNER-FACING BOOKING API PLATFORM"

**Goal:** Design and document (contract-first) a REST API for external
partner consumption, demonstrating every chapter of Book 5.

**Requirements:**

1. Write a complete OpenAPI 3.0 spec (Ch. 5) for a booking API:
   `POST /api/v1/bookings` (create), `GET /api/v1/bookings/{id}` (read),
   `GET /api/v1/bookings` (cursor-paginated list with filtering/sorting,
   Ch. 2), using precise status codes throughout (Ch. 1).
2. Document the authentication approach: OAuth2 Client Credentials Flow
   for the partner's server-to-server integration (Ch. 4), with JWT
   access tokens using RS256 (Ch. 3) verified by your resource server.
3. Design and document the `Idempotency-Key` header contract for the
   create-booking endpoint, including the exact behavior on a retried
   request (Ch. 7).
4. Design rate limiting for the partner tier: per-API-key limits,
   distributed across gateway instances (Ch. 6), with a documented
   fail-open/fail-closed decision and justification.
5. Write a versioning and deprecation policy document: what counts as
   breaking (using Chapter 7's full taxonomy), how expand-contract will
   be applied, and what `Sunset` header commitments you're making to partners.
6. (Implementation, optional but recommended) Build the
   `POST /api/v1/bookings` endpoint in Spring Boot implementing the
   idempotency behavior with a DB unique constraint, and one consumer-
   driven-contract-style test verifying the response matches the OpenAPI spec.

**Self-assessment rubric:**

| Criterion | Signal of mastery |
|---|---|
| Status code precision | 400 vs 422, 401 vs 403 used correctly throughout the spec |
| Pagination choice | Justified with a real trade-off analysis, not just "cursor is better" |
| Auth flow correctness | Client Credentials justified specifically for server-to-server, not copy-pasted from a user-facing example |
| Idempotency correctness | Retry behavior is specified precisely, including the DB-constraint-level guarantee |
| Evolution policy | References the full breaking/non-breaking taxonomy, not just "don't remove fields" |

---

*(This completes BOOK 5 — REST APIs + WEB SERVICES. Book 6 — SQL + JDBC +
Databases — shifts focus to the data layer these APIs sit on top of.)*
