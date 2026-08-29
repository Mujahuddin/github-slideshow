# CHAPTER 6 — SPRING SECURITY BASICS

> **Scope note:** This chapter covers the filter chain, authentication vs
> authorization, and baseline configuration — enough to secure a real
> endpoint and explain the model correctly. JWT, OAuth2 flows, and
> DevSecOps-level concerns (CVE/CWE/Snyk, OWASP Top 10 in depth) are Books
> 5 and 15's dedicated territory, referenced here so the progression is clear.

---

## 6.1 CONCEPT: Authentication vs Authorization — The Distinction Everyone Blurs

### TELUGU EXPLANATION

ఈ రెండు పదాలు తరచుగా కలిపేసి వాడతారు, కానీ ఇవి **పూర్తిగా వేర్వేరు
ప్రశ్నలు**:

- **Authentication ("మీరు ఎవరు?"):** ఒక user identity ని **verify**
  చేయడం — ఉదా: username/password సరైనవేనా, JWT token valid గా,
  tamper చేయకుండా ఉందా.
- **Authorization ("మీకు ఇది చేయడానికి అనుమతి ఉందా?"):** ఒక **already
  authenticated** user కి, ఒక specific action/resource మీద **permission**
  ఉందా అని check చేయడం — ఉదా: ఈ user "ADMIN" role కలిగి ఉందా, ఈ order
  ఇదే customer దా.

**ఎందుకు ఈ order ముఖ్యం:** Authentication ఎప్పుడూ **మొదట** జరగాలి —
ఎవరో తెలియకుండా, వాళ్ళకి permission ఉందో లేదో చెప్పలేరు. HTTP status
codes కూడా దీన్ని reflect చేస్తాయి: **401 Unauthorized** (నిజానికి
"un**authenticated**" అనే అర్థం, historically confusing పేరు) —
identity verify కాలేదు. **403 Forbidden** — identity తెలుసు, కానీ ఈ
action కి permission లేదు.

### ENGLISH INTERVIEW ANSWER

"Authentication answers 'who are you' — verifying identity, via
credentials, tokens, certificates. Authorization answers 'are you allowed
to do this' — checking permissions for an already-verified identity. They
must happen in that order, and the HTTP status codes reflect it, if
confusingly named: `401 Unauthorized` actually means 'not authenticated' —
identity couldn't be established at all — while `403 Forbidden` means
'identity is known, but you don't have permission for this specific
action.' I've seen this distinction get blurred in code reviews, returning
403 for a missing/invalid token when it should be 401, which actively
misleads API consumers about what's actually wrong."

---

## 6.2 CONCEPT: The Security Filter Chain — Where Security Actually Happens

### TELUGU EXPLANATION

Spring Security **Servlet Filters** గా (Java EE యొక్క original filter
mechanism మీద build అయ్యింది) పని చేస్తుంది — ప్రతి HTTP request,
controller కి చేరుకునేముందు, **ఒక chain of filters** గుండా వెళ్తుంది.
ప్రతి filter ఒక specific responsibility కలిగి ఉంటుంది (ఉదా: ఒకటి JWT
token ని parse చేస్తుంది, ఇంకొకటి CSRF check చేస్తుంది, ఇంకొకటి
authorization rules apply చేస్తుంది).

**Modern configuration style (Spring Security 6+, `SecurityFilterChain`
bean — పాత `WebSecurityConfigurerAdapter` deprecated అయ్యింది):**

```java
@Configuration
@EnableWebSecurity
class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()      // authentication అవసరం లేదు
                .requestMatchers("/api/admin/**").hasRole("ADMIN")   // ADMIN role తప్పకుండా కావాలి
                .anyRequest().authenticated()                        // మిగతా అన్నిటికీ, authenticated అయితేనే
            )
            .csrf(csrf -> csrf.disable()) // stateless REST APIs కి సాధారణంగా disable చేస్తారు (సెషన్ కుకీ లేకపోతే CSRF risk వేరుగా ఉంటుంది)
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)); // JWT-based APIs కి — no server-side session
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // ఎప్పుడూ hashed passwords — plain text ఎప్పుడూ వద్దు
    }
}
```

**కీలక design decisions ఇంటర్వ్యూలో explicitly అడిగేవి:**
1. **`csrf().disable()` ఎప్పుడు సరైనది:** CSRF attacks **cookie-based
   session authentication** మీద ఆధారపడతాయి — browser automatic గా
   cookies పంపుతుంది కాబట్టి. **Stateless, token-based (JWT, Authorization
   header)** APIs కి CSRF risk వర్తించదు (browser automatic గా
   Authorization header పంపదు) — కాబట్టి ఇక్కడ disable చేయడం సరైనది.
   **Session-cookie-based** web apps కి మాత్రం CSRF protection తప్పకుండా
   ఉండాలి.
2. **`SessionCreationPolicy.STATELESS`:** Server ఏ session state ని
   maintain చేయదు — ప్రతి request తనతో పాటు authentication సమాచారం
   (JWT token) తీసుకురావాలి. ఇది **horizontal scalability** కి ముఖ్యం
   (Book 8 Microservices లో వివరంగా — session affinity అవసరం లేకుండా,
   ఏ server instance అయినా ఏ request నైనా handle చేయగలదు).
3. **`BCryptPasswordEncoder`:** Passwords **ఎప్పుడూ plain text లో store
   చేయకూడదు** — BCrypt ఒక **one-way, salted hashing algorithm**, ఇది
   intentionally **slow** గా design చేయబడింది (brute-force attacks ని
   నిరోధించడానికి) — ఇది Book 15 (Security) లో వివరంగా చూసే OWASP
   సూత్రాల్లో ఒకటి.

### ENGLISH INTERVIEW ANSWER

"Spring Security operates as a chain of Servlet filters that every
request passes through before reaching a controller — each filter has one
job, like extracting and validating a JWT, or enforcing authorization
rules. The modern configuration style uses a `SecurityFilterChain` bean
with a lambda-based DSL, replacing the deprecated
`WebSecurityConfigurerAdapter` inheritance style. Three decisions I always
make deliberately, not by copy-pasting: whether CSRF protection is needed
— only relevant for cookie-based session authentication, not for
stateless token-based APIs, since CSRF exploits the browser's automatic
cookie-sending behavior, which doesn't apply to an `Authorization` header
a browser won't attach automatically; whether sessions should be stateless
— essential for horizontal scalability, since a stateless service doesn't
need session affinity and any instance can handle any request; and never
storing plain-text passwords — `BCryptPasswordEncoder` is a deliberately
slow, salted hashing algorithm specifically chosen to resist brute-force
attacks, which is foundational OWASP-level security hygiene."

---

## 6.3 CONCEPT: Method-Level Security — `@PreAuthorize`

### TELUGU EXPLANATION

URL-level rules (`requestMatchers`) కాకుండా, **method-level** గా కూడా
authorization rules apply చేయవచ్చు — ఇది **fine-grained** control ఇస్తుంది,
service layer కి దగ్గరగా:

```java
@Service
class OrderService {
    @PreAuthorize("hasRole('ADMIN') or #customerId == authentication.name") // SpEL expression
    Order getOrder(String customerId, String orderId) {
        // customer తన own orders మాత్రమే చూడగలరు, ADMIN ఎవరివైనా చూడగలరు
    }
}
```

**ఇది ఎందుకు ముఖ్యం:** URL-level rules "ఈ endpoint కి ఎవరు access
చేయగలరు" మాత్రమే చెప్తాయి — **data-level** authorization (ఉదా: "ఈ
specific customer, తన own data మాత్రమే చూడాలి, వేరే customer యొక్క
కాదు") controller/URL layer లో express చేయడం కష్టం. `@PreAuthorize`
ఇది service method స్థాయిలో, actual business data తో (parameters
ద్వారా) చేయడానికి వీలు కల్పిస్తుంది.

**Book 3 Chapter 4 connection:** `@PreAuthorize` కూడా **AOP-based** —
కాబట్టి **self-invocation problem ఇక్కడ కూడా వర్తిస్తుంది**! ఒక
`@PreAuthorize` method ని అదే class లో `this.method()` గా call చేస్తే,
security check **silently బైపాస్ అవుతుంది** — ఇది ఒక genuine,
critical security vulnerability కి దారితీయవచ్చు, కేవలం "aspect పని
చేయలేదు" అనే inconvenience కాదు.

### ENGLISH INTERVIEW ANSWER

"URL-based rules answer 'who can hit this endpoint,' but data-level
authorization — 'this customer can only see their own orders, not
anyone else's' — needs to reason about the actual data, which
`@PreAuthorize` enables at the method level using SpEL expressions that
can reference method parameters and the current authentication. The
critical thing I always flag in reviews: `@PreAuthorize` is AOP-based,
exactly like `@Transactional`, which means Book 3's self-invocation
problem applies here with much higher stakes — a security check
silently not applying isn't just a bug, it's a real vulnerability. I make
sure security-critical methods are never called via `this` internally
without independently verifying the security boundary still holds."

---

## 6.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Missing/invalid auth token | Returns 403 | Returns 401 — identity couldn't be established at all |
| Storing passwords | Considers using plain text or simple hashing (MD5/SHA1) "for now" | Always `BCryptPasswordEncoder` (or Argon2) — deliberately slow, salted |
| CSRF configuration | Copy-pastes `csrf().disable()` everywhere without understanding why | Disables it only for genuinely stateless, token-based APIs; keeps it for session-cookie-based apps |
| Authorization scope | Only checks "can this role hit this URL" | Also enforces data-level authorization ("can THIS user access THIS specific resource") |

---

## 6.5 COMMON MISTAKES

1. Confusing 401 and 403, or using them inconsistently.
2. Storing passwords with fast/reversible hashing instead of a
   deliberately slow, salted algorithm like BCrypt.
3. Disabling CSRF protection reflexively without understanding whether
   the API is actually stateless/token-based.
4. Relying only on URL-level authorization and forgetting data-level
   checks — "any authenticated user can hit `/api/orders/{id}`" without
   verifying the order actually belongs to that user.
5. Assuming `@PreAuthorize` always applies, without considering the
   self-invocation problem from Book 3 Chapter 4.

---

## 6.6 INTERVIEW QUESTION BANK — CHAPTER 6

**Basic:** 1. What's the difference between authentication and
authorization? 2. Why is `BCryptPasswordEncoder` preferred over a plain
hash function like SHA-256 for passwords?

**Intermediate:** 3. When is it appropriate to disable CSRF protection?
4. Explain `SessionCreationPolicy.STATELESS` and why it matters for scalability.

**Senior:** 5. Design the authorization rules (URL-level and
method-level) for an API where regular users can only view/modify their
own orders, but admins can view/modify any order. 6. Why does the
self-invocation problem make `@PreAuthorize` bugs more dangerous than a
typical AOP bug?

**Architect:** 7. Your organization is moving from session-cookie-based
authentication to a token-based (JWT) approach for a new set of
microservices, while legacy services still use sessions. How do you
architect security configuration to support both during the transition,
and what CSRF/statelessness decisions differ between the two?

**Scenario:** 8. A security audit finds an endpoint that correctly checks
`hasRole('ADMIN')` at the URL level but doesn't verify that a non-admin
user requesting `/api/orders/{id}` actually owns that order. What class of
vulnerability is this, and how do you fix it using this chapter's tools?

**Trick:** 9. "Once a request passes Spring Security's filter chain and
reaches the controller, no further authorization checks are needed."
True or false?

<details><summary>Key answers</summary>

- Q5: URL-level: `anyRequest().authenticated()` (both regular users and
  admins must be authenticated to reach any order endpoint). Method-level:
  `@PreAuthorize("hasRole('ADMIN') or #order.customerId ==
  authentication.name")` on the service methods handling individual order
  access/modification, enforcing the data-level ownership check that URL
  patterns alone cannot express.
- Q6: A typical AOP bug (e.g., a caching aspect not applying) causes a
  performance or correctness inconvenience — annoying but rarely
  catastrophic. A `@PreAuthorize` bug via self-invocation means a security
  boundary silently doesn't exist for that call path — an attacker (or
  even an innocent internal caller) can access something that was
  intended to be restricted, with no error or warning that anything is
  wrong, making it both more severe and harder to detect than most AOP gotchas.
- Q7: Maintain two separate `SecurityFilterChain` beans, each scoped to
  different URL patterns (`securityMatcher()`) — one configured with
  session-based authentication and CSRF protection enabled for legacy
  paths, another configured stateless with CSRF disabled for new
  JWT-secured paths — Spring Security explicitly supports multiple filter
  chains scoped by request matcher for exactly this kind of coexistence
  during a migration.
- Q8: This is a broken object-level authorization vulnerability — a very
  real, commonly cited category (related to OWASP's "Broken Access
  Control," which Book 15 covers in depth) — where a user can access data
  belonging to another user simply by changing an ID in the URL. Fix: add
  a `@PreAuthorize` (or equivalent explicit check in the service) verifying
  the requested order's owner matches the current authenticated user,
  unless the user has an admin role.
- Q9: False — the filter chain handles authentication and URL-level
  authorization, but data-level/object-level authorization (Q8) generally
  still needs explicit checks at the service or method level; reaching the
  controller only means the request passed whatever rules were actually
  configured, not that every necessary check has been performed.

</details>

---

## 6.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why is returning 403 for a missing authentication token technically incorrect, and what should be returned instead?
- **Coding Check:** Configure a `SecurityFilterChain` for a stateless REST API with public health-check endpoints, authenticated-only order endpoints, and admin-only management endpoints.
- **Explanation Check:** Explain in English why CSRF protection matters for session-cookie-based apps but not for stateless, Authorization-header-based APIs.
- **Real-World Check:** Your team's API currently checks roles only at the URL level (`hasRole('ADMIN')` for `/api/admin/**`). A penetration test finds a regular user can view another user's private data via a non-admin endpoint. Diagnose and fix using this chapter's material.
- **Senior Check:** When would you choose method-level (`@PreAuthorize`) security over URL-level (`requestMatchers`) security, or use both together?
- **Master Check:** Design the complete security boundary for a multi-tenant SaaS API where users belong to organizations, and must only ever access data within their own organization, with an additional "org admin" role having elevated permissions within (but never across) their organization. Specify both URL and method-level rules needed.

<details><summary>Answers</summary>

- Real-World Check: This is the broken-object-level-authorization pattern
  from Q8 above — the fix is adding explicit ownership verification (via
  `@PreAuthorize` with a SpEL expression referencing the resource owner, or
  an explicit check in the service layer) at the method level, since the
  URL-level role check alone cannot express "and only THEIR OWN data."
- Senior Check: URL-level rules are best for coarse-grained,
  role-based access to entire endpoint groups (only admins can reach
  `/api/admin/**` at all); method-level rules are necessary the moment
  authorization depends on the specific data being accessed, not just the
  caller's role — real systems typically need both together, URL-level as
  a first coarse filter and method-level for the fine-grained,
  data-dependent checks URL patterns can't express.
- Master Check: URL-level: `anyRequest().authenticated()` for all
  tenant-facing endpoints. Method-level: every service method accessing
  org-scoped data checks `@PreAuthorize("#orgId ==
  authentication.principal.organizationId")` (tenant isolation — the
  single most critical check in a multi-tenant system, since a mistake
  here leaks one customer's data to another), with an additional `hasRole('ORG_ADMIN')`
  check layered on top (via `and`) for org-admin-only operations — crucially,
  the org-admin check must always be combined with, never replace, the
  tenant-isolation check, since an org admin's elevated permissions still
  must not cross organization boundaries.

</details>

---

## 6.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Authentication | "Who are you" — verify identity — 401 if it fails |
| Authorization | "Are you allowed" — check permission — 403 if it fails |
| `SecurityFilterChain` | Modern config style (replaces deprecated `WebSecurityConfigurerAdapter`) |
| CSRF | Needed for cookie-session auth; safely disabled for stateless token-based APIs |
| `SessionCreationPolicy.STATELESS` | No server-side session — required for horizontal scalability |
| Passwords | Always `BCryptPasswordEncoder` (or Argon2) — never plain text or fast hashes |
| `@PreAuthorize` | Method-level, data-aware authorization — AOP-based, subject to self-invocation (Book 3 Ch. 4) |
| Broken object-level authorization | URL/role checks alone don't verify data ownership — needs explicit method-level checks |

---

*(Continues to Chapter 7 — Actuator, Logging, Production Configuration.)*
