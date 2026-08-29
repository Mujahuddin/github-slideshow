# CHAPTER 4 — OAUTH2 & OPENID CONNECT

---

## 4.1 CONCEPT: OAuth2 Is Authorization, Not Authentication — The Roles

### TELUGU EXPLANATION

**అత్యంత ముఖ్యమైన foundational fact:** OAuth2 **మొదట design అయ్యింది
authorization కోసం** ("ఈ app కి నా Google Contacts access ఇవ్వడానికి
నేను అంగీకరిస్తున్నాను") — **authentication కోసం కాదు** ("ఈ user
ఎవరో verify చేయడం"). ఇది Chapter 6 (Book 4) లో మనం చూసిన
authentication vs authorization distinction కి **ఖచ్చితంగా అనుగుణంగా**
ఉంటుంది.

**నాలుగు roles:**
- **Resource Owner:** User (వాళ్ళ డేటా కి యజమాని).
- **Client:** Access కావాల్సిన application (ఉదా: ఒక mobile app).
- **Authorization Server:** Authentication చేసి, tokens issue చేసేది
  (ఉదా: Google, Okta, Keycloak, మీ own custom auth server).
- **Resource Server:** Actual protected data/APIs కలిగినది (ఉదా: మీ
  backend API — Book 4 Chapter 6 లో మనం configure చేసిన SecurityFilterChain
  ఇక్కడ resource server side).

### ENGLISH INTERVIEW ANSWER

"OAuth2 was originally designed to solve delegated authorization — letting
a third-party app act on a user's behalf against a specific resource,
with the user's explicit consent — not to answer 'who is this user.' The
four roles matter for reasoning about any OAuth2 flow: the resource
owner is the user, the client is the application requesting access, the
authorization server authenticates the resource owner and issues tokens,
and the resource server holds the actual protected data and validates
incoming tokens — which, in Book 4's terms, is exactly the side of the
`SecurityFilterChain` we configured. This foundational distinction —
OAuth2 for authorization — is exactly why OpenID Connect had to be built
on top of it specifically to standardize authentication, which section
4.3 covers."

---

## 4.2 CONCEPT: Grant Types — Choosing the Right Flow for the Right Client

### TELUGU EXPLANATION

**Authorization Code Flow (+ PKCE) — web/mobile apps కి standard:**
1. Client, user ని Authorization Server కి redirect చేస్తుంది.
2. User login చేసి, consent ఇస్తారు.
3. Authorization Server, client కి ఒక **short-lived authorization code**
   తో redirect చేస్తుంది.
4. Client, ఆ code ని (backend channel ద్వారా, **browser కి కనిపించకుండా**)
   Authorization Server కి పంపి, **access token** (+ refresh token) పొందుతుంది.

**PKCE (Proof Key for Code Exchange) ఎందుకు అవసరం:** Mobile/SPA apps
కి **client secret ని safely store చేయడం సాధ్యం కాదు** (app binary
లేదా JavaScript లో ఏదైనా secret పెడితే, extract చేయగలరు). PKCE ఒక
**dynamic, per-request secret** (`code_verifier`/`code_challenge`) వాడి,
ఈ సమస్యని పరిష్కరిస్తుంది — ఇప్పుడు **అన్ని public clients కి** (mobile,
SPA) PKCE **తప్పనిసరి, best practice** గా పరిగణించబడుతుంది.

**Client Credentials Flow — service-to-service (Book 8 Microservices) కి:**
```
POST /oauth/token
grant_type=client_credentials&client_id=...&client_secret=...
```
ఇక్కడ **user అస్సలు involved కాదు** — ఒక service, తన own identity తో
(client_id/secret) directly token పొందుతుంది, మరో service ని call
చేయడానికి. ఇది Book 5 Chapter 3 లో మనం చూసిన "server-to-server
communication కి వేరే token strategy" కి ఖచ్చితమైన సమాధానం.

**⚠️ Deprecated/Discouraged flows (ఇంటర్వ్యూలో "ఎందుకు వాడకూడదు"
అని అడగడం common):**
- **Implicit Flow:** Access token ని నేరుగా browser URL fragment లో
  తిరిగి ఇచ్చేది — token browser history, logs లో exposed అయ్యే risk;
  PKCE-enabled Authorization Code Flow దీన్ని పూర్తిగా replace చేసింది.
- **Resource Owner Password Credentials (ROPC):** Client, user యొక్క
  username/password ని నేరుగా handle చేస్తుంది — ఇది **OAuth2 యొక్క
  మొత్తం purpose నే ఓడిస్తుంది** (client ఎప్పుడూ user credentials ని
  చూడకూడదు అనేదే core idea) — legacy migration cases తప్ప, ఎప్పుడూ వాడకూడదు.

### ENGLISH INTERVIEW ANSWER

"I choose the grant type based on the client type. Authorization Code
Flow with PKCE is the standard for anything with a user and a redirect
capability — web apps, mobile apps, SPAs. PKCE specifically solves the
problem that public clients — mobile apps, JavaScript SPAs — can't safely
store a client secret, since anyone can extract it from the app binary or
browser code; PKCE replaces a static secret with a dynamic, per-request
proof that closes this gap, and it's now considered mandatory best
practice for all public clients. Client Credentials Flow is what I use
for service-to-service authentication with no user involved at all — one
microservice authenticating directly to call another, using its own
identity. I actively avoid Implicit Flow, since returning access tokens
directly in a URL fragment exposes them to browser history and referrer
leakage, and Resource Owner Password Credentials, since having the client
handle the user's actual password directly defeats OAuth2's entire
purpose of keeping credentials away from third-party clients — both are
explicitly discouraged in modern OAuth2 guidance (RFC 9700's security best
practices) in favor of Authorization Code + PKCE."

---

## 4.3 CONCEPT: OpenID Connect (OIDC) — Adding Authentication on Top

### TELUGU EXPLANATION

Section 4.1 లో చెప్పినట్టు, OAuth2 **authentication కోసం design
అవ్వలేదు** — కానీ చాలామంది దాన్ని అలా వాడటానికి ప్రయత్నించారు (access
token ని "user login అయ్యారు" అని treat చేయడం), ఇది **inconsistent,
insecure patterns** కి దారితీసింది. **OpenID Connect (OIDC)** OAuth2
మీద ఒక **standardized identity layer** గా build అయ్యింది:

**కీలక addition: ID Token** — ఇది ఒక **JWT** (Chapter 3!), ఇందులో
user identity గురించి standardized claims ఉంటాయి (`sub`, `email`,
`name`, `iss`, `aud`). **Access token vs ID token — వీటిని కలిపేయడం
ఒక common mistake:**

| | Access Token | ID Token |
|---|---|---|
| **ప్రయోజనం** | Resource server APIs కి access ఇవ్వడానికి | "ఈ user ఎవరో" client కి తెలియజేయడానికి |
| **ఎవరు చదవాలి** | Resource server (verify చేస్తుంది) | Client (identity కోసం చదువుతుంది) |
| **Client దీన్ని API call లకి వాడాలా?** | ✅ అవును | ❌ కాదు — ఇది client కి మాత్రమే, API authorization కి కాదు |

### ENGLISH INTERVIEW ANSWER

"OAuth2 alone doesn't standardize 'who logged in' — different providers
built inconsistent, sometimes insecure workarounds trying to use access
tokens as a proxy for authentication. OpenID Connect standardizes this
properly by adding an ID Token — a JWT with well-defined identity claims
— on top of the OAuth2 flow. The distinction I make sure never gets
blurred: the access token is for the client to present to a resource
server to call APIs; the ID token is for the client itself to learn who
just authenticated. Using an access token as a stand-in for 'the user is
logged in,' or sending an ID token to a resource server as if it were an
access token, are both real anti-patterns I'd flag in review — they're
different tokens for different purposes, verified by different parties,
even though both happen to be JWTs."

---

## 4.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Building login for a mobile app | Uses Implicit Flow or ROPC because it "seems simpler" | Uses Authorization Code Flow + PKCE |
| Service-to-service auth | Reuses a user-facing flow awkwardly | Uses Client Credentials Flow |
| "Is the user logged in?" | Checks if an access token exists | Uses the ID Token's claims, understanding it's a distinct artifact from the access token |
| Choosing an OAuth2 provider/library | Rolls a custom auth server from scratch | Defaults to a standards-compliant provider (Keycloak, Okta, Auth0, or Spring Authorization Server) unless there's a strong reason not to |

---

## 4.5 COMMON MISTAKES

1. Using Implicit Flow or ROPC for new applications instead of
   Authorization Code + PKCE.
2. Treating an access token as proof of "who the user is" instead of
   using the ID Token for identity.
3. Sending an ID Token to a resource server as if it were an access
   token for API authorization.
4. Building a custom, non-standard OAuth2/OIDC implementation instead of
   using a well-vetted library or identity provider.
5. Confusing OAuth2 scopes (what the client can do) with OIDC claims
   (facts about the user) — they serve different purposes.

---

## 4.6 INTERVIEW QUESTION BANK — CHAPTER 4

**Basic:** 1. What are the four OAuth2 roles? 2. What does PKCE solve,
and for which client types?

**Intermediate:** 3. Why is Client Credentials Flow appropriate for
service-to-service auth but not Authorization Code Flow? 4. What's the
difference between an access token and an ID token?

**Senior:** 5. Design the authentication/authorization architecture for a
system with a web app (user-facing), a mobile app (user-facing), and 10
internal microservices (service-to-service) — which grant type for each,
and why? 6. Why is Resource Owner Password Credentials flow considered
an anti-pattern even though it's simpler to implement than Authorization
Code Flow?

**Architect:** 7. You're integrating with three different third-party
identity providers (for enterprise SSO customers) while also supporting
your own username/password login. How does OIDC's standardization help
you build one consistent internal authentication abstraction across all
of them?

**Scenario:** 8. A mobile app team implements login using the Implicit
Flow because "it's one less network round trip." A security review flags
this. Explain the actual risk and the recommended fix.

**Trick:** 9. "OAuth2 access tokens always prove who the user is." True
or false?

<details><summary>Key answers</summary>

- Q5: Web app and mobile app: Authorization Code Flow with PKCE (both
  are user-facing clients needing a browser redirect and unable to
  safely hold a static secret, mobile especially). Internal microservices
  calling each other: Client Credentials Flow, each service holding its
  own client credentials, no user context involved.
- Q6: ROPC requires the client application to directly collect and
  transmit the user's actual password to the authorization server —
  meaning the client sees and handles raw credentials, exactly what
  OAuth2 was designed to prevent third-party clients from needing to do;
  it also prevents the authorization server from using additional
  authentication factors (MFA, CAPTCHA, risk-based challenges) since the
  client-mediated flow bypasses the authorization server's own login UI entirely.
- Q7: OIDC standardizes the ID token shape and the discovery/metadata
  endpoints (`.well-known/openid-configuration`) across compliant
  providers — this lets you build one internal user model and
  authentication-handling code path that consumes standardized ID token
  claims regardless of which underlying identity provider issued them,
  rather than writing provider-specific integration code for each one.
- Q8: The risk is that Implicit Flow returns the access token directly in
  the redirect URL's fragment, which can be exposed via browser history,
  referrer headers, or logging of URLs, and provides no mechanism (like
  PKCE) to bind the token issuance to the specific request that initiated
  it, making token interception/injection attacks easier. The fix is
  migrating to Authorization Code Flow with PKCE, which keeps the actual
  token exchange in a back-channel request, not the browser-visible
  redirect.
- Q9: False — an access token proves the client is authorized to access
  specific resources/scopes; it doesn't reliably or consistently convey
  user identity across different providers, which is exactly the gap
  OpenID Connect's ID token was standardized to fill.

</details>

---

## 4.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why can't a mobile app safely use a static client secret, and what does PKCE substitute for it?
- **Coding Check:** Configure a Spring Boot resource server to validate incoming JWT access tokens issued by an external OAuth2/OIDC provider (using `spring-boot-starter-oauth2-resource-server`).
- **Explanation Check:** Explain in English why "OAuth2 is for authorization, OIDC adds authentication" is a precise, not just historical, distinction.
- **Real-World Check:** Your team's product needs "Login with Google" support. Map this feature onto the OAuth2/OIDC roles and flow covered in this chapter.
- **Senior Check:** When would you build a custom authorization server instead of using an off-the-shelf identity provider (Keycloak, Okta, Auth0)?
- **Master Check:** Design the complete token flow for a mobile banking app: initial login (Authorization Code + PKCE), calling the bank's API (access token to resource server), token refresh, and what happens differently on this app compared to a low-security app, given Chapter 3's revocation material.

<details><summary>Answers</summary>

- Real-World Check: Your application is the OAuth2/OIDC Client; Google is
  the Authorization Server (and OIDC Identity Provider); the user is the
  Resource Owner; the flow used is Authorization Code + PKCE, redirecting
  the user to Google to authenticate and consent, receiving an ID token
  (for "who logged in") and typically an access token (if your app also
  needs to call Google APIs on the user's behalf, like reading their
  calendar).
- Senior Check: When there's a genuine, well-justified requirement an
  off-the-shelf provider can't meet (extremely specific compliance/
  regulatory constraints, deep custom integration with a legacy identity
  system) — otherwise, building a custom authorization server means
  taking on the full security burden (correct PKCE implementation, token
  storage, revocation, key rotation) that mature, widely-audited
  providers have already solved; this is generally not a good use of
  engineering effort or an acceptable security risk for most organizations.
- Master Check: Login: Authorization Code + PKCE against the bank's
  authorization server. API calls: short-lived access token (very short —
  banking apps often use even shorter expiry than typical apps, given the
  stakes) presented as a Bearer token to resource server APIs. Refresh:
  refresh token stored securely on-device (platform secure storage, not
  plain storage) used to obtain new access tokens. Given Chapter 3's
  revocation material, a banking app would very likely also implement an
  active token blocklist/immediate revocation capability for
  suspected-fraud scenarios, accepting the statelessness cost, unlike a
  lower-security app that might rely purely on short expiry.

</details>

---

## 4.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| OAuth2 purpose | Delegated authorization — NOT originally for authentication |
| OIDC | Standardized identity/authentication layer built on OAuth2 |
| Authorization Code + PKCE | Standard flow for user-facing apps (web, mobile, SPA) |
| Client Credentials | Standard flow for service-to-service, no user involved |
| Avoid | Implicit Flow, Resource Owner Password Credentials (ROPC) |
| Access Token vs ID Token | Access token → resource server API calls; ID token → client-side identity only |
| Custom auth server | Avoid unless there's a genuine, well-justified reason — use a vetted provider |

---

*(Continues to Chapter 5 — OpenAPI/Swagger & API Documentation as Contract.)*
