# CHAPTER 3 — JWT DEEP DIVE

---

## 3.1 CONCEPT: JWT Structure — Encoded, Not Encrypted

### TELUGU EXPLANATION

**ఇది అత్యంత frequently misunderstood JWT fact:** JWT (JSON Web Token)
**mostly base64url-encoded, encrypted కాదు** (unless మీరు specifically
JWE — JSON Web Encryption — వాడితే, ఇది సాధారణంగా వాడేది JWS — JSON
Web Signature). ఒక JWT మూడు భాగాలు, `.` తో వేరు చేయబడి:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4ifQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
      ↑ Header (base64url)                    ↑ Payload (base64url)                          ↑ Signature
```

1. **Header:** `{"alg": "HS256", "typ": "JWT"}` — signing algorithm.
2. **Payload (Claims):** `{"sub": "1234567890", "name": "John", "exp":
   1735689600}` — **ఎవరైనా ఇది base64 decode చేసి చదవగలరు** (encryption
   కాదు!) — **ఎప్పుడూ sensitive data (passwords, PII) ని JWT payload
   లో పెట్టకూడదు**.
3. **Signature:** `HMACSHA256(base64UrlEncode(header) + "." +
   base64UrlEncode(payload), secret)` — ఇదే **tamper-proof** చేసేది —
   ఎవరైనా payload మార్చితే, signature verify అవ్వదు (secret లేకుండా
   correct signature generate చేయలేరు).

**కీలక distinction:** JWT **confidentiality ఇవ్వదు** (ఎవరైనా చదవగలరు),
**integrity ఇస్తుంది** (ఎవరైనా మార్చలేరు, పట్టుబడకుండా).

### ENGLISH INTERVIEW ANSWER

"The single most important thing to understand about JWTs is that the
header and payload are base64url-*encoded*, not encrypted — anyone who
intercepts a token can trivially decode and read its claims. What the
signature provides is *integrity*, not confidentiality — it proves the
token hasn't been tampered with since it was signed, since recomputing a
valid signature requires the signing secret or private key. This is
exactly why sensitive data — passwords, full PII, anything genuinely
confidential — should never go in a JWT payload; only non-sensitive
identity/claims data belongs there. If you need actual confidentiality,
you need JWE (encrypted JWTs) or, more commonly in practice, just don't
put sensitive data in a token at all and look it up server-side using the
token's subject claim instead."

---

## 3.2 CONCEPT: Signing Algorithms — Symmetric vs Asymmetric, and the "alg: none" Attack

### TELUGU EXPLANATION

**HS256 (HMAC, symmetric):** ఒకే **shared secret** signing మరియు
verification రెండింటికీ వాడతారు. **Limitation:** ఏ service verify
చేయాలన్నా, అదే secret కావాలి — microservices architecture లో (Book 8),
ప్రతి service కి secret share చేయడం **security risk పెంచుతుంది**
(ఎక్కువ places లో secret ఉంటే, leak అయ్యే అవకాశం ఎక్కువ).

**RS256 (RSA, asymmetric):** **Private key** signing కి, **Public key**
verification కి. **Advantage:** Token issue చేసే service (Authorization
Server) మాత్రమే private key కలిగి ఉంటుంది; verify చేసే services
(Resource Servers, Book 8 లో అనేక microservices) కేవలం **public key**
మాత్రమే కలిగి ఉంటే సరిపోతుంది — public key leak అయినా, ఎవరూ కొత్త
valid tokens **forge** చేయలేరు (private key అవసరం). ఇది **microservices
architecture కి బాగా సరిపోతుంది**.

**⚠️ "alg: none" / Algorithm Confusion Attack (real, historical
vulnerability, ఇంటర్వ్యూలో genuinely అడిగేది):** కొన్ని పాత/పేలవమైన
JWT libraries, token యొక్క **header లో ఉన్న `alg` field ని blindly
trust చేసేవి**. Attacker ఒక token తీసుకుని, దాని `alg` ని `"none"` కి
మార్చి, signature తీసేసి పంపితే, కొన్ని libraries దాన్ని **valid గా
accept చేసేవి** — ఇది ఒక **critical vulnerability** (CVE-level, Book
15 Security లో వివరంగా). **పరిష్కారం:** Server-side verification code
**ఎప్పుడూ expected algorithm ని explicitly specify చేయాలి** (`alg`
field ని token నుండి నమ్మకూడదు) — modern libraries (`jjwt`,
`nimbus-jose-jwt`) ఇది సరిగ్గా handle చేస్తాయి, కానీ **మీరు దీన్ని
ఎందుకు జాగ్రత్తగా configure చేయాలో అర్థం చేసుకోవడం ముఖ్యం**.

### ENGLISH INTERVIEW ANSWER

"HS256 uses one shared secret for both signing and verification — simple,
but every service that needs to verify tokens needs that same secret,
which is a real liability in a microservices architecture with many
services. RS256 uses a private key to sign and a public key to verify —
only the authorization server holds the private key, and every resource
server can safely hold just the public key, since possessing it doesn't
let you forge new valid tokens. This asymmetry is exactly why RS256 is
the more common choice for distributed systems. On the algorithm
confusion attack: historically, some JWT libraries trusted the `alg`
field inside the token itself to decide how to verify it — an attacker
could set `alg: none`, strip the signature, and some implementations would
accept it as valid. The fix, which modern well-maintained libraries
enforce, is that the verifying code must explicitly specify which
algorithm it expects and reject anything else, never trusting the
token's own header to dictate verification behavior. I'd flag this
specifically in a security review of any custom or older JWT handling code."

---

## 3.3 CONCEPT: Statelessness vs. Revocation — The Fundamental JWT Tension

### TELUGU EXPLANATION

**JWT యొక్క ప్రధాన advantage — statelessness** (Chapter 1 సూత్రం): a
resource server, token ని **verify చేయడానికి database call అవసరం లేదు**
— signature check చేస్తే సరిపోతుంది, ఇది fast, scalable.

**కానీ ఇదే దాని ప్రధాన limitation కి కారణం:** ఒక user logout అయితే,
లేదా account compromise అయితే, **ఆ token ని "invalidate" చేయడం ఎలా?**
Token stateless కాబట్టి, server కి "ఈ token ఇంకా valid గా ఉందా" అని
అడగడానికి ఏ central place లేదు — token, దాని `exp` (expiration) వరకు
**technically valid గానే ఉంటుంది**, revoke చేయాలన్నా చేయలేరు (నేరుగా).

**Production పరిష్కారాలు (ప్రతి దానికి ఒక trade-off):**
1. **Short-lived access tokens (5-15 నిమిషాలు) + Refresh tokens:** Access
   token expire అయ్యేలోపు damage window తక్కువగా ఉంటుంది. Refresh
   token (దీర్ఘకాలం valid, కానీ **server-side, revocable గా** store
   అవుతుంది — database/Redis లో) కొత్త access token పొందడానికి వాడతారు.
   Logout/compromise అయితే, **refresh token ని revoke చేస్తే సరిపోతుంది**
   — ఇప్పటికే issue అయిన access tokens కొద్ది నిమిషాల్లో ఎలాగూ
   expire అవుతాయి.
2. **Token blocklist (Redis-backed):** Revoke చేసిన tokens ని ఒక
   fast-lookup store లో పెట్టడం — కానీ ఇది **statelessness ని పాక్షికంగా
   వదులుకోవడమే** (ప్రతి verification కి ఇప్పుడు ఒక lookup అవసరం) —
   pure JWT యొక్క benefit ని తగ్గిస్తుంది.

**Senior గా, సరైన mental model:** "**Access tokens ని ఎప్పుడూ short-lived
గా treat చేయండి, revocation control refresh token layer లో ఉంచండి**"
— ఇది statelessness (fast verification) మరియు revocability (security
control) రెండింటినీ balance చేస్తుంది.

### ENGLISH INTERVIEW ANSWER

"JWT's core advantage — no database lookup needed to verify a token — is
also exactly why revocation is hard: there's no central place to ask 'is
this token still valid' without reintroducing state. The standard
production pattern resolves this by splitting responsibilities: short-lived
access tokens, expiring in minutes, limit the damage window if one is
compromised, while a longer-lived refresh token is stored server-side —
in a database or Redis — specifically so it CAN be revoked. Logging a
user out, or responding to a compromised account, means revoking the
refresh token; the currently-outstanding access tokens simply expire
naturally within minutes rather than needing active revocation. A token
blocklist is the alternative some teams use, but it's worth being honest
that it partially sacrifices JWT's main advantage — you're back to a
lookup on every verification, just against a smaller, revocation-specific
store rather than the full user database."

---

## 3.4 CONCEPT: Where to Store JWTs on the Client — XSS vs CSRF Trade-off

### TELUGU EXPLANATION

| Storage | XSS risk | CSRF risk | ఎప్పుడు వాడాలి |
|---|---|---|---|
| **`localStorage`** | ❌ **High** — JavaScript ఏదైనా (attacker's injected script సహా) దీన్ని చదవగలదు | ✅ లేదు — browser దీన్ని automatic గా పంపదు | SPA APIs, XSS defenses (CSP, sanitization) strong గా ఉంటే |
| **`httpOnly` cookie** | ✅ లేదు — JavaScript దీన్ని access చేయలేదు | ❌ **ఉంది** — browser automatic గా cookie పంపుతుంది (Chapter... Book 4 Chapter 6 CSRF సూత్రం గుర్తుంచుకోండి) | CSRF protection (SameSite, CSRF tokens) సరిగ్గా configure చేస్తే |

**Senior గా correct answer, "ఏది better" కాదు — "ఏ threat model ని మీరు
prioritize చేస్తున్నారు":** `httpOnly` cookie + `SameSite=Strict` +
CSRF token combination సాధారణంగా **more defense-in-depth** ఇస్తుంది
(modern recommendation), ఎందుకంటే XSS defense (CSP) ని 100% guarantee
చేయడం కష్టం, కానీ CSRF ని `SameSite` + token తో సమర్థవంతంగా
neutralize చేయవచ్చు.

### ENGLISH INTERVIEW ANSWER

"This isn't a question with one universally correct answer — it's a
threat-model trade-off. `localStorage` is immune to CSRF since the browser
never sends it automatically, but any successful XSS attack gets full,
trivial access to the token via JavaScript. An `httpOnly` cookie is
immune to XSS-based token theft since JavaScript can't read it, but is
exposed to CSRF, since browsers attach cookies automatically — which is
exactly Book 4's CSRF material. My default recommendation is `httpOnly`
cookies with `SameSite=Strict` plus CSRF token protection, because in
practice, fully eliminating XSS risk across an entire application's
codebase is harder to guarantee than properly configuring CSRF defenses,
which are more contained and verifiable. But I'd want to know the
specific application's architecture and existing defenses before treating
this as settled — it's a genuine trade-off, not a solved problem."

---

## 3.5 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Putting data in a JWT | Includes user's email, phone, or other PII "for convenience" | Only non-sensitive identity claims — anyone can decode and read the payload |
| Verifying algorithm | Trusts the `alg` field in the incoming token | Hardcodes the expected algorithm server-side, never trusts the token's own header |
| Handling logout | Assumes "the token is gone from the client, so it's revoked" | Understands the token is still technically valid until expiry; relies on short expiry + refresh token revocation |
| Choosing HS256 vs RS256 | Picks whichever tutorial used | Chooses RS256 for multi-service verification scenarios specifically to avoid shared-secret distribution |

---

## 3.6 COMMON MISTAKES

1. Storing sensitive/PII data in JWT claims, assuming it's protected
   because it's "encoded."
2. Trusting the `alg` field from an incoming token instead of hardcoding
   the expected algorithm server-side.
3. Believing "logout" truly invalidates a JWT immediately, without a
   refresh-token-revocation or blocklist strategy.
4. Using long-lived access tokens (hours/days) without a refresh token
   pattern, maximizing the damage window of a leaked token.
5. Sharing an HS256 secret across many microservices, increasing the
   attack surface for secret leakage.

---

## 3.7 INTERVIEW QUESTION BANK — CHAPTER 3

**Basic:** 1. Is a JWT encrypted? 2. What do the three parts of a JWT
represent?

**Intermediate:** 3. Why is RS256 often preferred over HS256 in a
microservices architecture? 4. What is the "alg: none" attack, and how do
modern libraries prevent it?

**Senior:** 5. Design the full access-token/refresh-token lifecycle for a
web application, including what happens on logout and on suspected
account compromise. 6. Compare `localStorage` and `httpOnly` cookie
storage for JWTs — what threat model does each protect against, and
which fails against which attack?

**Architect:** 7. You're designing token-based authentication for an API
consumed by both a web SPA and a set of server-to-server microservices
(Book 8). Would you use the same token strategy for both, or different
ones? Justify.

**Scenario:** 8. A security review finds that your application's JWT
verification code doesn't specify an expected algorithm and instead reads
it from the incoming token. What's the risk, and what's the fix?

**Trick:** 9. "Because JWTs are signed, their contents are protected from
being read by anyone without the signing key." True or false?

<details><summary>Key answers</summary>

- Q5: Access token: short-lived (5-15 min), returned on login, sent on
  every API request, verified statelessly. Refresh token: longer-lived,
  stored server-side (DB/Redis) with a revocation flag, used only to
  obtain new access tokens. Logout: revoke the refresh token server-side;
  outstanding access tokens simply expire shortly after. Suspected
  compromise: revoke all refresh tokens for that user immediately,
  forcing re-authentication once current access tokens expire — an
  immediate full lockout would require a blocklist for currently-valid
  access tokens too, an additional, heavier measure for severe cases.
- Q6: `localStorage` fails against XSS (any injected script can read and
  exfiltrate the token) but is immune to CSRF. `httpOnly` cookies are
  immune to XSS-based token theft (JavaScript can't read them) but are
  exposed to CSRF unless `SameSite` and CSRF tokens are properly
  configured — the "which is better" answer depends on which threat the
  application's existing defenses handle better.
- Q7: Likely different: the web SPA benefits from short-lived access
  tokens plus a refresh-token flow (user-facing, browser-based threat
  model, section 3.4's storage trade-offs apply). Server-to-server
  microservice communication (Book 8) more commonly uses OAuth2 client
  credentials flow (Chapter 4) issuing tokens with a machine-appropriate
  lifecycle — no "user," no browser storage concerns, and RS256 for
  cross-service verification without shared secrets, exactly per section 3.2.
- Q8: The risk is the algorithm confusion / "alg: none" attack — an
  attacker could craft a token with `alg: none` (or downgrade RS256 to
  HS256 using the known public key as an HMAC secret, another real
  variant of this attack) and potentially bypass verification. Fix:
  explicitly configure the verification library with the expected
  algorithm, rejecting any token that doesn't match, rather than deriving
  the algorithm from the token itself.
- Q9: False — signing provides integrity (tamper detection), not
  confidentiality; the payload remains base64url-*encoded*, which anyone
  can decode and read without any key at all — this is precisely the
  section 3.1 misconception this chapter opens with.

</details>

---

## 3.8 MASTERY CHECKPOINTS

- **Knowledge Check:** Why can anyone read a JWT's payload without the signing key, but not modify it undetected?
- **Coding Check:** Implement JWT generation and verification in Java (using a library like `jjwt`), explicitly configuring the expected algorithm on the verification side rather than trusting the token's header.
- **Explanation Check:** Explain in English why "the user logged out" doesn't mean "the JWT is now invalid" without additional infrastructure.
- **Real-World Check:** Your team's application currently stores JWTs in `localStorage` and has had no XSS incidents, but a new security hire recommends migrating to `httpOnly` cookies. How do you evaluate whether this migration is worth the effort?
- **Senior Check:** When might HS256 still be the right choice despite RS256's advantages in a distributed system?
- **Master Check:** Design a complete token revocation strategy for a banking application requiring near-immediate session termination on suspected fraud (i.e., short-lived access tokens alone are not fast enough) — what additional mechanism would you add, and what's its cost to the statelessness benefit?

<details><summary>Answers</summary>

- Real-World Check: Weigh the actual risk reduction (does the
  application have other strong XSS defenses already — CSP, output
  encoding, dependency scanning — making the marginal benefit of cookie
  storage smaller?) against the migration cost and the CSRF protections
  that must be correctly implemented as part of the switch — "no
  incidents so far" isn't proof of safety, but the migration should be
  justified by an actual risk assessment, not just following a
  recommendation reflexively.
- Senior Check: When all token issuers and verifiers are within a single
  trusted, tightly-controlled boundary (e.g., a monolith or a very small,
  tightly-coupled set of services all deployed and secured together by
  the same team) where secret distribution isn't a meaningful additional
  risk — the simplicity of symmetric signing can be a reasonable trade-off there.
- Master Check: For near-immediate revocation needs, a token blocklist
  (Redis-backed, checked on every verification) is necessary despite its
  cost — this explicitly sacrifices pure statelessness, adding a fast
  lookup to every request, but for a banking application, the security
  requirement (immediate termination capability) outweighs the
  performance/architectural purity cost; the blocklist only needs to
  retain entries until the token's natural expiry, keeping it bounded in size.

</details>

---

## 3.9 CHEAT SHEET

| Concept | Rule |
|---|---|
| JWT payload | Base64url-ENCODED, not encrypted — never put sensitive data in it |
| Signature | Provides integrity (tamper-proofing), not confidentiality |
| HS256 | Symmetric — simple, but shared-secret risk across services |
| RS256 | Asymmetric — private key signs, public key verifies — better for microservices |
| `alg` field | Never trust it from the token — hardcode the expected algorithm server-side |
| Revocation | Not natively possible — use short-lived access tokens + revocable refresh tokens |
| Storage trade-off | `localStorage` (XSS risk) vs `httpOnly` cookie (CSRF risk, mitigated by SameSite+CSRF token) |

---

*(Continues to Chapter 4 — OAuth2 & OpenID Connect.)*
