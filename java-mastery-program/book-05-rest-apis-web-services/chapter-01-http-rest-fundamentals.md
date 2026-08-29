# CHAPTER 1 — HTTP & REST FUNDAMENTALS DONE RIGHT

---

## 1.1 CONCEPT: Safe and Idempotent Methods — Not Just "GET Reads, POST Writes"

### TELUGU EXPLANATION

HTTP methods కి రెండు కీలక properties ఉన్నాయి, ఇవి "ఏ verb వాడాలి"
అనే నిర్ణయానికి మించి, **real production behavior** ని నిర్ణయిస్తాయి:

- **Safe:** Method server state ని **మార్చదు** (read-only). Client
  (browser, proxy, CDN) ఏ safe method ని అయినా, user కి తెలియకుండా,
  **retry/prefetch** చేయవచ్చు.
- **Idempotent:** Method ని **N సార్లు call చేసినా, ఫలితం ఒకసారి call
  చేసినట్టే** ఉంటుంది (state పరంగా — response ఒకేలా ఉండాలని కాదు).

| Method | Safe? | Idempotent? | ఎందుకు |
|---|---|---|---|
| `GET` | ✅ | ✅ | Read-only, repeat చేసినా ఏమీ మారదు |
| `HEAD` | ✅ | ✅ | GET లాంటిదే, body లేకుండా |
| `PUT` | ❌ | ✅ | "resource ని ఈ exact state కి సెట్ చేయి" — 10 సార్లు చేసినా ఫలితం ఒకటే |
| `DELETE` | ❌ | ✅ | మొదటిసారి delete అవుతుంది, తర్వాత సార్లు "already gone" — end state ఒకటే |
| `POST` | ❌ | ❌ | "కొత్తది create చేయి" — ప్రతిసారి **కొత్త resource** create అవుతుంది (ఉదా: duplicate orders!) |
| `PATCH` | ❌ | ❌ (సాధారణంగా) | Partial update — depends on semantics, కానీ సాధారణంగా idempotent కాదు (ఉదా: "increment by 1" అనేది idempotent కాదు) |

**ఎందుకు ఇది Book 2 Chapter 16 తో direct connection:** Retry logic
(exponential backoff) రాసేటప్పుడు, **safe గా retry చేయగలిగేది ఏ methods
మాత్రమే** అనేది ఖచ్చితంగా తెలియాలి. `POST` ని బ్లైండ్ గా retry చేస్తే
(network timeout అనుకుని), నిజానికి request server కి చేరి succeed
అయ్యి ఉండొచ్చు, retry **duplicate resource** create చేయవచ్చు — ఇదే
Book 2 Chapter 16 లో మనం idempotency keys ఎందుకు అవసరమో చెప్పిన కారణం,
ఇక్కడ HTTP semantics స్థాయిలో మళ్ళీ కనిపిస్తోంది.

### ENGLISH INTERVIEW ANSWER

"Safety and idempotency are precise, technical properties, not
approximations. Safe means no server state changes at all — GET and HEAD.
Idempotent means repeating the call any number of times leaves the server
in the same end state as calling it once — PUT and DELETE qualify, POST
does not, because POST's whole purpose is usually to create a new
resource each time. This distinction is exactly why my retry logic from
Book 2's production-coding chapter treats POST differently: it's never
safe to blindly retry a POST on a timeout, since the original request may
have already succeeded server-side, and a naive retry risks creating a
duplicate resource. That's precisely the gap idempotency keys close —
they let a client safely retry an inherently non-idempotent operation by
giving the server a way to recognize 'this is the same logical attempt,
not a new one.'"

---

## 1.2 CONCEPT: Status Codes — Precision Matters More Than Most Developers Think

### TELUGU EXPLANATION

**4xx family, ఖచ్చితంగా వేరు చేయాల్సినవి (చాలా devs వీటిని కలిపేస్తారు):**

| Code | ఖచ్చితమైన అర్థం | సాధారణ తప్పు వాడకం |
|---|---|---|
| **400 Bad Request** | Request **syntactically/structurally తప్పు** (malformed JSON, తప్పు type) | Business rule violations కి కూడా వాడేస్తారు (422 బదులు) |
| **401 Unauthorized** | **Authentication** విఫలమైంది (identity తెలియదు) | Authorization failures కి వాడేస్తారు (403 బదులు) |
| **403 Forbidden** | Identity తెలుసు, కానీ **permission లేదు** | — |
| **404 Not Found** | Resource **లేదు** | Sometimes security కోసం **intentionally** 403 బదులు వాడతారు (resource ఉందని కూడా reveal చేయకుండా) |
| **409 Conflict** | Request valid, కానీ **current server state** తో conflict అవుతుంది (ఉదా: duplicate email, concurrent edit conflict) | 400 బదులు వాడాలి, కానీ చాలామంది 400 వాడేస్తారు |
| **422 Unprocessable Entity** | Request **syntactically సరైనది, కానీ semantically/business-rule ప్రకారం invalid** (ఉదా: "endDate startDate కంటే ముందు ఉంది") | 400 తో confuse అవుతుంది |
| **429 Too Many Requests** | Rate limit దాటింది (Book 2 Chapter 16, Book 5 Chapter 6) | — |

**Senior distinction, తరచుగా ఇంటర్వ్యూలో అడిగేది — 400 vs 422:** 400
అంటే "నేను ఈ request ని **parse చేయలేను**" (JSON malformed, required
field missing entirely). 422 అంటే "నేను ఈ request ని parse చేశాను,
అర్థం చేసుకున్నాను, కానీ **business rules ప్రకారం ఇది invalid**" (ఉదా:
`quantity: -5` — syntactically valid number, కానీ semantically
invalid).

**5xx family:** ఇవి ఎప్పుడూ **server-side సమస్య** ని సూచించాలి —
client కి తప్పు లేదని అర్థం. **client input వల్ల వచ్చిన ఏ error ని
అయినా 5xx గా return చేయడం ఎప్పుడూ తప్పు** — అది client కి "మీరు ఏం
తప్పు చేశారో నాకు తెలియదు" అని చెప్పడం లాంటిది, debugging కష్టతరం
చేస్తుంది.

### ENGLISH INTERVIEW ANSWER

"I treat status codes as part of the API's actual contract, not
decoration. The distinction I see violated most often is 400 vs 422: 400
means the request couldn't even be parsed or is structurally malformed;
422 means it was parsed successfully but violates a business rule — a
negative quantity, an end date before a start date. Conflating them means
a client can't programmatically distinguish 'fix your JSON' from 'fix
your business logic,' which matters for automated retry/correction logic
on the client side. Similarly, 401 vs 403 — a request with no or an
invalid token should never be 403, since that implies identity was
established but permission was denied, which is a different, misleading
signal. And 5xx should be reserved strictly for genuine server-side
failures — returning a 500 for bad client input is a real anti-pattern
that actively obstructs debugging for API consumers."

---

## 1.3 CONCEPT: Statelessness and Caching Headers

### TELUGU EXPలanaTION

**REST యొక్క core constraint: Statelessness** — ప్రతి request, server
కి అవసరమైన **మొత్తం సమాచారం తనతో పాటే తీసుకురావాలి** (auth token,
context) — server **ఏ per-client session state ని మధ్యలో గుర్తుపెట్టుకోకూడదు**.
ఇది Book 4 Chapter 6 లో మనం చూసిన `SessionCreationPolicy.STATELESS`
కి **theoretical పునాది**, మరియు horizontal scalability కి (Book 8)
నేరుగా దారితీస్తుంది — ఏ server instance అయినా ఏ request నైనా handle
చేయగలదు, session affinity అవసరం లేకుండా.

**Caching headers (frequently underused, real performance win):**
- **`ETag`** — resource యొక్క current version కి ఒక hash/identifier.
- **`If-None-Match`** — client "నా దగ్గర ఉన్న ETag ఇదే, ఇంకా valid
  గానే ఉందా?" అని అడగడం — valid అయితే, server **304 Not Modified**
  (body లేకుండా) return చేస్తుంది — bandwidth ఆదా.
- **`Cache-Control`** — ఎంతసేపు cache చేయవచ్చో (`max-age`), ఎలా
  (`private`/`public`, `no-store`).

### ENGLISH INTERVIEW ANSWER

"Statelessness means every request must be self-contained — the server
holds no per-client session state between requests. This is exactly the
theoretical basis for Book 4's `SessionCreationPolicy.STATELESS`
decision, and it's what makes horizontal scaling straightforward: any
instance behind a load balancer can serve any request without needing
session affinity. On caching, `ETag` plus conditional requests via
`If-None-Match` is an underused but genuinely valuable pattern — the
server can respond with a lightweight `304 Not Modified` instead of
re-sending the full payload when nothing has changed, which meaningfully
reduces bandwidth for frequently-polled, infrequently-changing resources."

---

## 1.4 CONCEPT: The Richardson Maturity Model — How "RESTful" Is Your API, Really?

### TELUGU EXPLANATION

చాలా "REST APIs" నిజానికి REST యొక్క original constraints పూర్తిగా
follow చేయవు — ఇది తప్పు కాదు, కానీ **అర్థం చేసుకోవడం ముఖ్యం**:

- **Level 0:** ఒకే URL, ఒకే HTTP method (సాధారణంగా POST) — SOAP-లాంటి
  RPC style.
- **Level 1:** వేర్వేరు resources కి వేర్వేరు URLs (`/orders`,
  `/customers`) — కానీ HTTP methods సరిగ్గా వాడరు (అన్నీ POST).
- **Level 2:** HTTP methods సరిగ్గా వాడతారు (GET/POST/PUT/DELETE) +
  status codes సరిగ్గా — **చాలా production "REST APIs" ఇక్కడే ఆగిపోతాయి**,
  ఇదే practical, widely-accepted level.
- **Level 3 (HATEOAS):** Response లో **next possible actions కి links**
  ఉంటాయి (ఉదా: ఒక order response లో, "cancel" link, ఆ order cancel
  చేయగలిగే స్థితిలో ఉంటేనే ఉంటుంది) — client కి URL structure
  hardcode చేయాల్సిన అవసరం లేకుండా.

**Senior గా realistic అభిప్రాయం:** Level 2 **industry standard** —
దాదాపు ప్రతి "REST API" గా పిలువబడేది ఇక్కడే ఉంటుంది. HATEOAS
(Level 3) సైద్ధాంతికంగా ఆకర్షణీయం, కానీ **ఆచరణలో అరుదు** — client
libraries, tooling ecosystem దీన్ని పూర్తిగా support చేయవు, complexity
దాని benefit కంటే ఎక్కువ చాలా cases లో.

### ENGLISH INTERVIEW ANSWER

"The Richardson Maturity Model gives useful vocabulary for how RESTful an
API actually is. Level 0 is basically RPC over HTTP through one endpoint.
Level 1 introduces resource-based URLs but still misuses HTTP methods.
Level 2 — correct use of HTTP methods and status codes per resource — is
where the overwhelming majority of real-world 'REST APIs' actually sit,
and I consider that a legitimate, practical target, not a compromise.
Level 3, HATEOAS, adds discoverable links to related actions directly in
responses — theoretically elegant, since clients don't need to hardcode
URL structures, but in practice it's rarely adopted because tooling and
client expectations haven't caught up, and the added complexity often
isn't worth it for typical internal or partner APIs. I'd mention HATEOAS
in an interview to show awareness of the full model, while being honest
that Level 2 is where I actually design for in most real systems."

---

## 1.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Retrying a failed request | Retries any method blindly on timeout | Only safely retries idempotent methods; uses idempotency keys otherwise (Book 2 Ch. 16) |
| Invalid business rule in request | Returns 400 | Returns 422, reserving 400 for structurally malformed requests |
| Missing/invalid auth token | Returns 403 | Returns 401 |
| "Is our API RESTful?" | Doesn't know what the question means precisely | Can place it on the Richardson Maturity Model and explain the trade-off of going further |

---

## 1.6 COMMON MISTAKES

1. Retrying POST requests blindly without idempotency protection.
2. Using 400 for both malformed requests and business rule violations
   instead of distinguishing 400 from 422.
3. Returning 403 for missing/invalid authentication instead of 401.
4. Returning 500 for client input errors.
5. Treating PATCH as automatically idempotent without checking the
   specific semantics of the update being performed.

---

## 1.7 INTERVIEW QUESTION BANK — CHAPTER 1

**Basic:** 1. What does "idempotent" mean precisely? 2. Difference
between 401 and 403?

**Intermediate:** 3. Why is PUT idempotent but POST is not? 4. Explain
400 vs 422 with an example of each.

**Senior:** 5. Design the retry policy for a payment API client,
specifying exactly which HTTP methods can be retried safely and which
need additional protection (tie to Book 2 Chapter 16). 6. Is PATCH
idempotent? Justify with both a case where it is and a case where it isn't.

**Architect:** 7. You're designing API standards for 50 microservices.
What status code usage rules would you mandate, and how would you enforce
consistency (contract tests, API gateway validation, linting)?

**Scenario:** 8. A mobile client on a flaky network retries a `POST
/api/payments` three times after not receiving a response, and the
customer is charged three times. Diagnose and fix using this chapter's material.

**Trick:** 9. "Idempotent means the response is always identical." True
or false?

<details><summary>Key answers</summary>

- Q5: GET/HEAD retryable freely (safe). PUT/DELETE retryable safely
  since idempotent. POST requires an idempotency key (Book 2 Ch. 16)
  before any retry is safe — without one, retry only after confirming
  via a status-check call whether the original attempt actually succeeded.
- Q6: `PATCH {"status": "SHIPPED"}` (setting an absolute value) is
  idempotent — applying it repeatedly leaves the same end state. `PATCH
  {"quantity": "+1"}` (a relative/incremental update) is NOT idempotent —
  each repeated call changes the state further. This is why PATCH's
  idempotency must be evaluated per-endpoint based on actual semantics,
  not assumed from the method name alone.
- Q7: Mandate: 422 for business rule violations vs 400 for malformed
  requests, 401 vs 403 correctly distinguished, no 5xx for client-input
  errors, idempotency-key requirements for POST endpoints performing
  financial/critical operations — enforce via automated contract tests in
  CI (Book 14) checking response codes against expected scenarios, and/or
  API gateway-level validation rejecting responses that violate the
  organization's status code policy.
- Q8: The client's retry-on-timeout logic treated POST as safely
  retryable, which it is not — each retry created a genuinely new payment
  charge server-side (the original requests likely all succeeded; only
  the *responses* were lost to the flaky network). Fix: require an
  idempotency key generated once per logical payment attempt (Book 2
  Chapter 16), so the server recognizes and safely no-ops repeated
  attempts with the same key instead of charging multiple times.
- Q9: False — idempotent describes the *server-side end state* after N
  calls, not that the response body is identical each time (e.g., a
  `DELETE` might return 200 with a body the first time and 404 on
  subsequent calls — different responses, but the same idempotent
  end state: the resource is gone).

</details>

---

## 1.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is DELETE idempotent even though calling it a second time typically returns a different status code (404) than the first call (200/204)?
- **Coding Check:** Design the exact status codes for a `PATCH /api/orders/{id}` endpoint covering: success, order not found, invalid state transition attempted, and malformed request body.
- **Explanation Check:** Explain in English why "REST" and "uses JSON over HTTP" are not the same thing, using the Richardson Maturity Model.
- **Real-World Check:** Your team's API returns 400 for "coupon code doesn't exist" and 400 for "request body is missing the required 'items' field." A partner team asks if these can be told apart programmatically. Redesign the status codes.
- **Senior Check:** When would returning 404 instead of 403 be a deliberate security decision rather than a mistake?
- **Master Check:** Design the full safe/idempotent classification and retry policy for a hypothetical `POST /api/transfers` (money transfer) endpoint, including how idempotency keys interact with the underlying database transaction from Book 4 Chapter 5.

<details><summary>Answers</summary>

- Real-World Check: "Coupon code doesn't exist" is a business-rule
  violation on an otherwise well-formed request → 422. "Missing required
  field" is a structurally malformed request → 400. Distinguishing these
  lets the partner team's client branch on status code alone instead of
  parsing message text.
- Senior Check: When revealing "this resource exists but you don't have
  permission" would itself leak sensitive information (e.g., confirming
  another user's private resource ID is valid) — returning 404 instead of
  403 avoids confirming existence to an unauthorized caller, a deliberate
  security-through-non-disclosure choice used in some sensitive-data APIs.
- Master Check: `POST /api/transfers` is inherently non-idempotent (each
  call intends to create a new transfer) and must require a client-
  generated idempotency key; server-side, the key is checked and stored
  atomically within the same database transaction that performs the
  transfer (via `ConcurrentHashMap.computeIfAbsent`-equivalent atomic
  check-and-store at the DB level, e.g., a unique constraint on the
  idempotency key column) so that a retry with the same key returns the
  original result without re-executing the transfer, and the whole
  operation — transfer plus idempotency record — commits or rolls back
  together as one atomic unit (Book 4 Chapter 5's `@Transactional`).

</details>

---

## 1.9 CHEAT SHEET

| Concept | Rule |
|---|---|
| Safe methods | GET, HEAD — no state change, freely retryable |
| Idempotent methods | GET, HEAD, PUT, DELETE — safe to retry |
| Non-idempotent | POST, (usually) PATCH — need idempotency keys to retry safely |
| 400 vs 422 | 400 = malformed/unparseable; 422 = valid structure, business rule violation |
| 401 vs 403 | 401 = not authenticated; 403 = authenticated, not authorized |
| 5xx | Reserved for genuine server-side failures — never for client input errors |
| Richardson Level 2 | The practical, industry-standard target for "RESTful" |
| ETag / `If-None-Match` | Conditional requests → 304 Not Modified, saves bandwidth |

---

*(Continues to Chapter 2 — API Design: Versioning, Pagination, Filtering, Sorting.)*
