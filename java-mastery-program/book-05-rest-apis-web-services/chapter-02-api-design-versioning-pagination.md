# CHAPTER 2 — API DESIGN: VERSIONING, PAGINATION, FILTERING, SORTING

---

## 2.1 CONCEPT: API Versioning Strategies — Trade-offs, Not a Universal Answer

### TELUGU EXPLANATION

APIs మారుతూ ఉంటాయి, కానీ existing consumers ని break చేయకూడదు. **మూడు
ప్రధాన versioning strategies:**

| Strategy | ఉదాహరణ | Pros | Cons |
|---|---|---|---|
| **URI path versioning** | `/api/v1/orders`, `/api/v2/orders` | సులభం, స్పష్టం, cache-friendly (URL మారితేనే cache invalidate) | "REST resources కి version ఉండకూడదు" అని purists వాదన; URL proliferation |
| **Header-based versioning** | `Accept: application/vnd.company.v2+json` | URL clean గా ఉంటుంది, resource identity మారదు | Less discoverable (browser URL bar లో కనపడదు), tooling support తక్కువ |
| **Query parameter versioning** | `/api/orders?version=2` | సులభం | Caching కి awkward, "optional" గా అనిపిస్తుంది (దీన్ని విస్మరిస్తే ఏమవుతుంది?) |

**Senior గా, practical answer:** **URI path versioning** ఇండస్ట్రీ లో
అత్యంత widely-used, ఎందుకంటే ఇది **simple, discoverable, debugging-friendly**
(browser లో direct గా hit చేయవచ్చు, logs లో స్పష్టంగా కనిపిస్తుంది).
Purist arguments (headers "more RESTful") ఆచరణలో **cost-benefit** లో
తక్కువ win ఇస్తాయి, ఎక్కువ complexity తో.

**అత్యంత ముఖ్యమైన senior-level సూత్రం:** **Versioning అనేది చివరి
resort గా ఉండాలి, మొదటి response కాదు.** చాలా changes **backward-compatible**
గా చేయవచ్చు (Chapter 7 లో వివరంగా) — కొత్త optional field add చేయడం,
కొత్త endpoint add చేయడం — ఇవి కొత్త version అవసరం లేకుండా చేయవచ్చు.
**Breaking change** (existing field తీసేయడం, type మార్చడం, required
field add చేయడం) కి మాత్రమే కొత్త version అవసరం.

### ENGLISH INTERVIEW ANSWER

"I default to URI path versioning — `/api/v1/`, `/api/v2/` — not because
it's the most theoretically 'pure' REST approach, but because it's the
most practical: discoverable, debuggable directly in a browser, obvious
in logs, and well-supported by every piece of tooling. Header-based
versioning is more elegant in principle but adds real friction with
limited practical benefit for most teams. The more important principle,
though, is that versioning should be a last resort, not a first
instinct — most API changes can be made backward-compatible: adding an
optional field, adding a new endpoint, are non-breaking and don't need a
version bump at all. I reserve introducing a new version specifically for
genuine breaking changes — removing or renaming a field, changing a
field's type, adding a new required field to a request — which is a much
rarer event than teams sometimes assume."

---

## 2.2 CONCEPT: Pagination — Why Offset-Based Breaks at Scale

### TELUGU EXPLANATION

**Offset-based pagination** (సులభమైనది, మొదట అందరూ వాడేది):
```
GET /api/orders?page=3&size=20   →  SQL: OFFSET 40 LIMIT 20
```

**రెండు నిజమైన సమస్యలు, scale వద్ద:**

1. **Performance:** `OFFSET 100000` అంటే, database **మొదటి 100,000
   rows ని scan చేసి, తర్వాత discard చేయాలి** — page number పెరిగేకొద్దీ,
   query **నెమ్మదిగా** అవుతుంది (Book 6 Database indexing/query
   optimization కి direct connection — index ఉన్నా, OFFSET తో skip
   చేసిన rows ని ఇప్పటికీ traverse చేయాలి).
2. **Consistency (concurrent inserts):** Page 1 చూశాక, ఎవరో కొత్త row
   insert చేస్తే, page 2 request చేసినప్పుడు, **items shift అవుతాయి** —
   ఒక item **రెండుసార్లు కనిపించొచ్చు, లేదా పూర్తిగా skip అవ్వొచ్చు**
   ("page drift" అనే well-known bug).

**Cursor-based pagination (production-grade పరిష్కారం):**
```
GET /api/orders?after=eyJpZCI6MTIzfQ==&size=20
```
ఇక్కడ `after` ఒక **opaque cursor** (సాధారణంగా last-seen item యొక్క
sortable field, encoded) — query ఇలా మారుతుంది:
```sql
SELECT * FROM orders WHERE id > :lastSeenId ORDER BY id LIMIT 20
```
**ఇది ఎందుకు fast:** `id > :lastSeenId` ఒక **indexed range scan** —
మునుపటి rows ని scan చేయాల్సిన అవసరం లేదు, ఎంత deep page అయినా **O(log n
+ page size)** (B-tree index లో, Book 6/7 లో వివరంగా). **ఇది "page
drift" సమస్యని కూడా పరిష్కరిస్తుంది** — cursor ఎప్పుడూ ఒక **specific
point** ని సూచిస్తుంది, "page number" ని కాదు.

### ENGLISH INTERVIEW ANSWER

"Offset-based pagination is simple but has two real production problems.
Performance-wise, a large `OFFSET` forces the database to scan and
discard that many rows before returning results — deep pages get
progressively slower even with an index, since the index still has to
traverse past the skipped rows. Consistency-wise, if rows are inserted or
deleted between page requests, offset-based pagination can show duplicate
items or silently skip items entirely — a real, well-known bug class
called page drift. Cursor-based pagination fixes both: instead of 'give
me page 3,' the client says 'give me everything after this specific
item,' which translates to an indexed range scan — `WHERE id >
:lastSeenId` — that's fast regardless of how deep into the dataset you
are, and it's immune to page drift since it's anchored to a specific
point in the data, not a shifting numeric offset. I default to cursor-based
pagination for any API where the underlying data changes frequently or
the dataset is large enough that deep pages are a realistic access pattern."

**Interviewer follow-up:** "What's the downside of cursor-based
pagination?" — You lose the ability to jump directly to an arbitrary page
number ("go to page 47") — cursors are inherently sequential,
next/previous-oriented. For UIs that genuinely need page-number jumping
(rare, but real — e.g., some admin dashboards), offset-based pagination,
accepted with its performance caveat, may still be the pragmatic choice.

---

## 2.3 CONCEPT: Filtering and Sorting — Consistent Query Parameter Design

### TELUGU EXPLANATION

**సాధారణ, consistent conventions** (వేర్వేరు teams వేర్వేరు ways
కనిపెట్టడం confusion కి దారితీస్తుంది):

```
GET /api/orders?status=PLACED&customerId=CUST-123&sort=createdAt,desc&page=0&size=20
```

- **Filtering:** field పేరు నే query parameter గా వాడటం
  (`status=PLACED`) — సూటిగా, REST-idiomatic.
- **Sorting:** `sort=field,direction` (Spring Data ఇదే convention
  native గా support చేస్తుంది, `Sort`/`Pageable` ద్వారా).
- **Range/comparison filters** (ఉదా: "price > 100"): `price[gte]=100`
  లేదా `minPrice=100&maxPrice=500` — రెండూ common, team-wide **consistency**
  ఏదో ఒకటి ఎంచుకోవడం కంటే ముఖ్యం.

**Senior గమనిక — response envelope:** Paginated response లో, కేవలం
data array మాత్రమే కాకుండా, **metadata** ఇవ్వడం మంచిది:
```json
{
  "data": [ ... ],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ==",
    "hasMore": true,
    "totalCount": 5000
  }
}
```
`totalCount` ఇవ్వడం **expensive** కావొచ్చు పెద్ద tables కి (`COUNT(*)`
కూడా full scan కావొచ్చు) — దీన్ని **optional/approximate** గా ఇవ్వడం,
లేదా పూర్తిగా వదిలేయడం, ఒక legitimate senior-level trade-off.

### ENGLISH INTERVIEW ANSWER

"I keep filtering and sorting conventions consistent across every
endpoint in an API — field name as the query parameter for equality
filters, a `sort=field,direction` convention (which Spring Data's
`Pageable`/`Sort` abstraction supports natively), and a clearly documented
convention for range filters, whichever specific syntax the team picks.
For paginated responses, I wrap results in an envelope with pagination
metadata — cursor for the next page, a `hasMore` flag — rather than just
returning a bare array, since clients need that metadata to paginate
correctly. I'm deliberately cautious about including an exact
`totalCount`, since computing it can require a full table scan on large
datasets, which is expensive precisely for the large-data-volume APIs
where accurate pagination metadata matters most — I'd rather provide an
approximate count or omit it than silently make every paginated request expensive."

---

## 2.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Any API change | Reaches for a new version immediately | Checks if the change can be backward-compatible first |
| Building pagination | Defaults to `page`/`size` (offset) without considering scale | Chooses cursor-based pagination for large/frequently-changing datasets |
| Paginated response | Returns a bare JSON array | Returns an envelope with pagination metadata |
| `totalCount` in a paginated response | Always computes and returns it | Considers the cost of `COUNT(*)` on large tables before committing to it |

---

## 2.5 COMMON MISTAKES

1. Versioning an API for every change instead of preferring
   backward-compatible evolution.
2. Using offset-based pagination for large, frequently-changing datasets
   without understanding the performance and page-drift risks.
3. Inconsistent filter/sort query parameter conventions across different
   endpoints in the same API.
4. Always computing an exact `totalCount` without considering its cost on
   large tables.
5. Returning a bare array from a paginated endpoint with no metadata
   about how to get the next page.

---

## 2.6 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. Name the three common API versioning strategies. 2. What's
the main advantage of cursor-based over offset-based pagination?

**Intermediate:** 3. Explain "page drift" with a concrete example. 4.
Why does `OFFSET 100000` get slower as the offset increases, even with an index?

**Senior:** 5. Design the pagination strategy for a social media feed
(constantly changing, users scroll deep) versus an admin reporting
dashboard (needs "jump to page 50"). Are they the same? 6. What
determines whether an API change is backward-compatible or requires a
new version — give three examples of each.

**Architect:** 7. You're designing pagination for an API that serves both
a mobile app (infinite scroll) and a partner's batch export process
(needs to reliably fetch ALL records without missing any due to
concurrent writes). Design an approach that serves both needs correctly.

**Scenario:** 8. A partner integration reports that their nightly batch
job, paginating through `/api/orders?page=N`, sometimes misses orders
that were clearly created before the job ran. Diagnose.

**Trick:** 9. "Cursor-based pagination is strictly better than
offset-based pagination in every situation." True or false?

<details><summary>Key answers</summary>

- Q5: Different needs justify different approaches — the social feed
  wants cursor-based pagination (large, constantly changing dataset,
  sequential scrolling, no need to jump to an arbitrary page); the admin
  dashboard's explicit "jump to page 50" requirement is fundamentally an
  offset-based access pattern, which cursor pagination cannot support
  directly — accepting offset pagination's performance/consistency
  trade-offs there is the pragmatic choice given the actual UX requirement.
- Q6: Backward-compatible: adding a new optional response field, adding
  a new endpoint, adding a new optional query parameter. Breaking:
  removing/renaming an existing field, changing a field's data type,
  adding a new *required* request field, changing an endpoint's existing behavior/semantics.
- Q7: Cursor-based pagination for the mobile infinite-scroll experience;
  for the partner's reliable full-export need, provide either a
  cursor-based export with a stable ordering guarantee (e.g., ordered by
  an immutable, monotonically increasing ID or a "last modified" watermark
  the batch job tracks across runs) so it can reliably resume and
  guarantee completeness even with concurrent writes — this is a real,
  common dual-audience API design problem, and the batch export use case
  specifically motivates cursor pagination even more strongly than the
  mobile case, since correctness (not missing records) matters even more
  than UX smoothness.
- Q8: This is page drift from offset-based pagination — new orders
  inserted between the batch job's page requests shift the offset-based
  result set, causing some orders to be skipped entirely (or duplicated).
  The fix is switching the export endpoint to cursor-based pagination
  anchored on a stable, monotonically increasing field (like order ID or
  creation timestamp).
- Q9: False — cursor-based pagination cannot support jumping to an
  arbitrary page number, which some genuine UI requirements need; offset-
  based pagination remains the pragmatic (if imperfect) choice when that
  specific capability is a hard requirement and the dataset size/change
  frequency make its downsides tolerable.

</details>

---

## 2.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does `WHERE id > :lastSeenId LIMIT 20` scale better than `OFFSET 40000 LIMIT 20`, in terms of what the database actually has to do?
- **Coding Check:** Design the request/response contract (query parameters, response envelope) for a cursor-based paginated endpoint in a Spring Boot API.
- **Explanation Check:** Explain in English why "just add a v2 endpoint" is not always the right first response to an API change request.
- **Real-World Check:** Your team's `/api/products` endpoint needs to support filtering by category, price range, and in-stock status, plus sorting by price or popularity. Design the consistent query parameter contract.
- **Senior Check:** When would URI path versioning's "URL proliferation" downside actually become a real operational problem worth addressing?
- **Master Check:** Design the complete API evolution plan for adding a mandatory new field to an existing `CreateOrderRequest` (previously all fields were optional-with-defaults) without breaking any existing API consumers immediately — what interim steps would you take before the field becomes truly mandatory?

<details><summary>Answers</summary>

- Real-World Check: `GET /api/products?category=electronics&minPrice=100&maxPrice=500&inStock=true&sort=popularity,desc&after=<cursor>&size=20` —
  consistent field-name-as-parameter filtering, a documented range-filter
  convention (`minX`/`maxX`), the standard `sort=field,direction`
  convention, and cursor-based pagination given products catalogs are
  typically large and frequently updated.
- Senior Check: When maintaining many old, still-in-use versions (v1
  through v6, say) becomes a genuine maintenance and testing burden —
  each version needing its own bug fixes, security patches, and
  regression testing — at which point a deliberate deprecation and
  sunset policy (Chapter 7) becomes necessary, not just versioning itself
  being wrong.
- Master Check: (1) Add the field as optional with a sensible default in
  the current version, announce the upcoming requirement and a deadline
  to consumers; (2) monitor/log which consumers are still omitting the
  field as the deadline approaches; (3) once usage data shows adoption or
  the deadline passes, either enforce it as required in the current
  version (a breaking change, now properly communicated and expected) or
  introduce a new API version where it's mandatory, keeping the old
  version available for a defined sunset period — turning what could have
  been an abrupt breaking change into a managed, communicated evolution.

</details>

---

## 2.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Default versioning strategy | URI path (`/api/v1/`) — simple, discoverable |
| Versioning philosophy | Last resort — prefer backward-compatible changes first |
| Offset pagination | Simple, but slow at depth + prone to "page drift" |
| Cursor pagination | Indexed range scan, immune to page drift — the default for large/changing datasets |
| When offset is still OK | Small/static datasets, or a genuine "jump to page N" UI requirement |
| Filtering/sorting | Consistent, field-name-based query params across the whole API |
| `totalCount` | Consider its `COUNT(*)` cost before always including it |

---

*(Continues to Chapter 3 — JWT Deep Dive.)*
