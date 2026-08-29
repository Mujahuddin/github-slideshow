# CHAPTER 2 — EXTERNALIZED CONFIGURATION & PROFILES

---

## 2.1 CONCEPT: `@Value` vs `@ConfigurationProperties` — Why Type-Safety Wins

### TELUGU EXPLANATION

Configuration values (`application.properties`/`application.yml` నుండి)
ని code లోకి తీసుకురావడానికి **రెండు మార్గాలు**:

**1. `@Value` (single value, ఒక్కో property కి ఒక్కటి):**
```java
@Value("${app.notification.retry-count}")
private int retryCount; // ❌ field injection కూడా — Book 3 Chapter 1 సూత్రం ఇక్కడ కూడా వర్తిస్తుంది
```

**2. `@ConfigurationProperties` (group of related properties, type-safe,
బలంగా recommended for anything beyond 1-2 values):**
```java
@ConfigurationProperties(prefix = "app.notification")
public class NotificationProperties {
    private final int retryCount;
    private final Duration timeout;
    private final List<String> enabledChannels;

    // Constructor binding (Spring Boot 2.2+) — immutable, constructor-injected config!
    public NotificationProperties(int retryCount, Duration timeout, List<String> enabledChannels) {
        this.retryCount = retryCount;
        this.timeout = timeout;
        this.enabledChannels = enabledChannels;
    }
    // getters only — no setters, immutable (Book 1 Chapter 3 సూత్రం ఇక్కడ కూడా)
}
```

```java
@Configuration
@EnableConfigurationProperties(NotificationProperties.class) // లేదా @ConfigurationPropertiesScan
class ConfigPropertiesConfig { }
```

**ఎందుకు `@ConfigurationProperties` బాగుంటుంది (senior-level reasons):**
1. **Type safety** — `Duration`, `List<String>`, nested objects కూడా
   automatic గా bind అవుతాయి (`app.notification.timeout=30s` — Spring
   ఇది automatic గా `Duration` గా parse చేస్తుంది).
2. **Grouping** — సంబంధిత properties ఒకే class లో, IDE autocomplete తో.
3. **Validation** — `@Validated` + Bean Validation annotations (`@Min`,
   `@NotBlank`) వాడి, **startup వద్దే** invalid config ని catch చేయవచ్చు
   (Book 4 Chapter 1 సూత్రం: "fail fast at startup" — misconfigured
   `retryCount=-5` production లో run అవ్వడానికి బదులు, startup వద్దే
   fail అవ్వాలి).
4. **Immutability** (constructor binding తో) — Book 1 Chapter 3 సూత్రం
   direct application: config values ఎప్పుడూ మారవు, runtime లో
   accidental mutation సాధ్యం కాదు.

**`@Value` ఎప్పుడు ఇప్పటికీ సరిపోతుంది:** నిజంగా ఒక్క, isolated
value కి, ఒకే చోట వాడేటప్పుడు — కానీ 3+ సంబంధిత properties ఉంటే,
`@ConfigurationProperties` కి మారడం ఎప్పుడూ better.

### ENGLISH INTERVIEW ANSWER

"`@Value` works for a single, isolated property, but I default to
`@ConfigurationProperties` for anything with more than one or two related
settings. It gives me real type safety — Spring automatically converts
`30s` into a `Duration`, comma-separated strings into a `List`, and
supports nested property groups — plus, critically, Bean Validation
integration, so I can annotate the properties class with `@Min`,
`@NotBlank`, etc., and have Spring fail at startup if the configuration is
invalid, rather than discovering a bad config value mid-request in
production. I use constructor binding specifically, not setter-based
binding, so the resulting properties object is immutable once
constructed — the same reasoning as favoring constructor injection and
immutable value objects throughout this program, just applied to
configuration data specifically."

---

## 2.2 CONCEPT: Property Source Precedence — Who Wins?

### TELUGU EXPLANATION

Spring Boot **అనేక చోట్ల నుండి** configuration values చదవగలదు —
command-line args, environment variables, `application.yml`, profile-specific
files, మొదలైనవి. అవి **conflict** అయితే, ఎవరు గెలుస్తారు? Spring Boot
కి ఒక **strict precedence order** ఉంది (high to low priority, ఇక్కడ
అత్యంత frequently-tested/relevant వాటిని మాత్రమే):

1. **Command-line arguments** (`--app.notification.retry-count=5`) — **అత్యధిక priority**
2. **`SPRING_APPLICATION_JSON`** environment variable
3. **JVM System properties** (`-Dapp.notification.retry-count=5`)
4. **OS Environment variables** (`APP_NOTIFICATION_RETRY_COUNT=5`)
5. **Profile-specific properties file** (`application-prod.yml`, active profile అయితే)
6. **`application.yml`/`application.properties`** (default, base file)
7. **`@PropertySource`-annotated values**
8. **Default values** (code లో hardcoded defaults)

**ఎందుకు ఇది ముఖ్యం (real production reason):** ఇది **12-Factor App**
సూత్రం (config ని environment నుండి externalize చేయడం) కి Spring Boot
యొక్క implementation — production లో, మీరు `application.yml` లో ఒక
default DB URL పెట్టవచ్చు, కానీ **Kubernetes environment variable** ద్వారా
దాన్ని override చేయవచ్చు, code/artifact మారకుండానే, **ఒకే built JAR**
ని dev/staging/prod — మూడు చోట్లా వేర్వేరు config తో deploy చేయవచ్చు.

**Relaxed binding:** `APP_NOTIFICATION_RETRY_COUNT` (env var, uppercase,
underscores) automatic గా `app.notification.retry-count` (property,
lowercase, hyphens) కి map అవుతుంది — ఎందుకంటే environment variables
లో lowercase/hyphens/periods తరచుగా అనుమతించబడవు, Spring ఈ **relaxed
binding rules** తో దీన్ని handle చేస్తుంది.

### ENGLISH INTERVIEW ANSWER

"Spring Boot reads configuration from many sources, and precedence
matters a lot in practice — command-line arguments and environment
variables outrank the `application.yml` file, which is exactly what
enables the 12-Factor App principle of building one artifact once and
configuring it differently per environment purely through external
configuration, without rebuilding. In Kubernetes specifically, this is why
you set environment variables or mount a ConfigMap rather than baking
environment-specific values into the JAR. Relaxed binding is the detail
that makes this practical — an environment variable like
`APP_NOTIFICATION_RETRY_COUNT` automatically maps to the property
`app.notification.retry-count`, since environment variable naming
conventions (uppercase, underscores) don't match property file
conventions (lowercase, hyphens), and Spring bridges that gap
automatically."

---

## 2.3 CONCEPT: Profiles in Spring Boot — Building on Book 3

### TELUGU EXPLANATION

Book 3 Chapter 5 లో `@Profile` bean-level గా చూశాం. Spring Boot దీన్ని
**configuration file level** కి extend చేస్తుంది:

- `application.yml` — **అన్ని profiles కి common** గా apply అయ్యే base config.
- `application-dev.yml`, `application-prod.yml` — **profile-specific**
  overrides, active profile బట్టి **additionally** load అవుతాయి (base
  ని replace చేయవు, దానిపైన **overlay** అవుతాయి).
- Active profile set చేయడం: `spring.profiles.active=prod` (environment
  variable, command-line arg, లేదా — అరుదుగా — `application.yml` లోనే).

```yaml
# application.yml (base — అన్ని profiles కి apply)
app:
  notification:
    retry-count: 3

---
# application-prod.yml (ఇది active అయినప్పుడు మాత్రమే, పైదాన్ని override చేస్తుంది)
app:
  notification:
    retry-count: 5 # production లో ఎక్కువ retries కావాలి అనుకుంటే
```

**Senior గమనిక:** Profile files **completely replace కాదు, override
చేస్తాయి మాత్రమే** — `application-prod.yml` లో `retry-count` ప్రస్తావించకపోతే,
base file లోని value (3) అలాగే వర్తిస్తుంది.

### ENGLISH INTERVIEW ANSWER

"Spring Boot profiles extend Book 3's bean-level `@Profile` concept to
the configuration-file level: a base `application.yml` holds settings
common to every environment, and profile-specific files like
`application-prod.yml` only need to specify the values that actually
differ for that environment — they overlay on top of the base, they don't
replace it wholesale. This keeps environment-specific config minimal and
DRY, since you're not repeating every setting in every profile file, only
the deltas. The active profile is typically set via an environment
variable in the deployment environment itself — `SPRING_PROFILES_ACTIVE=prod`
— continuing the same externalized-configuration principle from section 2.2."

---

## 2.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| 5 related configuration values | Five separate `@Value` fields | One `@ConfigurationProperties` class, constructor-bound |
| Invalid config value | Discovered when the feature using it breaks in production | Caught at startup via `@Validated` on the properties class |
| Environment-specific DB URL | Hardcodes different URLs in different branches/builds | One build artifact, externalized config per environment (env vars/profiles) |
| Config field mutability | Uses setters, mutable properties class | Constructor binding, immutable properties object |

---

## 2.5 COMMON MISTAKES

1. Using many scattered `@Value` fields instead of a grouped, type-safe
   `@ConfigurationProperties` class once there are several related settings.
2. Not validating configuration properties, letting bad config surface as
   a runtime bug instead of a startup failure.
3. Baking environment-specific values into the build artifact instead of
   externalizing them — breaking the "build once, deploy anywhere" principle.
4. Forgetting relaxed binding rules and assuming environment variable
   names must exactly match property key casing/punctuation.
5. Repeating every property in every profile file instead of only
   specifying the deltas from the base `application.yml`.

---

## 2.6 INTERVIEW QUESTION BANK — CHAPTER 2

**Basic:** 1. When would you use `@ConfigurationProperties` over
`@Value`? 2. What is "relaxed binding"?

**Intermediate:** 3. List the property source precedence order and
explain why command-line arguments outrank the properties file. 4. How
do profile-specific YAML files relate to the base `application.yml`?

**Senior:** 5. Design a `@ConfigurationProperties` class for a
rate-limiter configuration (capacity, refill rate, enabled flag), with
validation ensuring capacity is positive — show the class and explain
where the validation failure surfaces. 6. Why does constructor binding
matter for configuration properties beyond just "it's a good practice" —
what real bug class does it prevent?

**Architect:** 7. You're standardizing configuration management across 50
microservices, some needing secrets (DB passwords, API keys) that must
NOT live in `application.yml` even in an externalized form. How does this
change your configuration strategy (hint: think toward a secrets manager/
config server, previewing Book 13/14 territory)?

**Scenario:** 8. A service works correctly in staging but fails in
production with a clearly wrong configuration value, even though the
`application-prod.yml` file looks correct in the repository. What are the
first two things you check, given this chapter's precedence rules?

**Trick:** 9. "Profile-specific YAML files completely replace the base
`application.yml` when that profile is active." True or false?

<details><summary>Key answers</summary>

- Q5: `@ConfigurationProperties(prefix = "app.ratelimiter")` with
  constructor-bound `int capacity`, `double refillRate`, `boolean
  enabled`, and `@Validated` on the class plus `@Min(1)` on `capacity` —
  the validation failure surfaces at application startup as a
  `ConfigurationPropertiesBindException`/constraint violation, before any
  request-handling code ever runs, per the "fail fast" principle from
  Chapter 1.
- Q6: Without constructor binding (i.e., using mutable setter-based
  binding), nothing prevents some other code elsewhere in the application
  from calling a setter and mutating configuration values at runtime,
  potentially causing configuration to silently drift from what was
  actually deployed/intended — constructor binding makes the properties
  object genuinely immutable after startup, eliminating that entire bug class.
- Q7: Secrets should never live in plain-text externalized files at all,
  even outside the repo — the standard approach is a dedicated secrets
  manager (HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets) or a
  config server that injects secrets as environment variables or mounted
  files at deploy/runtime, keeping them out of both the build artifact and
  any plain configuration file, with access control and rotation handled
  by the secrets system rather than application config.
- Q8: First check: is an environment variable or command-line argument
  (which outranks the YAML file) overriding the value unexpectedly in the
  production environment specifically? Second check: is the correct Spring
  profile (`SPRING_PROFILES_ACTIVE`) actually active in production — a
  typo or missing profile activation would silently fall back to base
  `application.yml` values instead of the intended `application-prod.yml` overlay.
- Q9: False — profile-specific files overlay on top of the base file,
  only overriding the specific keys they define; any key not
  present in the profile-specific file still comes from the base
  `application.yml`.

</details>

---

## 2.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Why does constructor binding for `@ConfigurationProperties` matter for the same underlying reason constructor injection matters for regular beans?
- **Coding Check:** Write a validated, immutable `@ConfigurationProperties` class for a third-party API client's configuration (base URL, API key, timeout, max retries) with appropriate Bean Validation annotations.
- **Explanation Check:** Explain in English why "build once, configure per environment via external sources" is more reliable than "build a different artifact per environment."
- **Real-World Check:** Your team needs a feature flag that can be toggled instantly in production without a redeploy. Does this chapter's static-at-startup configuration model support that, or what additional piece would you need (hint: this foreshadows a dynamic config/feature-flag service, beyond static `application.yml`)?
- **Senior Check:** When would `@Value` still be the right, sufficient choice over `@ConfigurationProperties`?
- **Master Check:** Design the full precedence-aware configuration strategy for a service that needs: a safe default retry count baked into the JAR, an environment-specific override via Kubernetes ConfigMap (environment variable), and an emergency, no-redeploy override via a JVM system property passed at container start. Map each to the correct property source and explain why each layer exists.

<details><summary>Answers</summary>

- Real-World Check: Static `application.yml`/environment-variable
  configuration is fixed at startup — it cannot change without a restart.
  Instant, no-redeploy toggling requires a dynamic mechanism layered on
  top: a feature-flag service (LaunchDarkly-style), or polling a config
  server/database value periodically — this is a genuinely different
  problem from "externalize static config," worth explicitly
  distinguishing in an interview rather than conflating the two.
- Senior Check: A single, truly standalone value used in exactly one
  place, where creating an entire properties class would be needless
  ceremony — e.g., a one-off feature toggle read in a single method with
  no related settings.
- Master Check: Default retry count → hardcoded default value in the
  `@ConfigurationProperties` class itself (lowest precedence, safety net);
  Kubernetes ConfigMap override → OS environment variable (mid precedence,
  the normal per-environment configuration path); emergency override → JVM
  system property (`-Dapp.retry.count=...`, which per the precedence list
  in 2.2 outranks OS environment variables) passed at container start
  without touching the ConfigMap — used specifically for an urgent,
  temporary override that shouldn't require a full config-management
  change process.

</details>

---

## 2.8 CHEAT SHEET

| Need | Mechanism |
|---|---|
| One isolated config value | `@Value("${key}")` |
| Multiple related config values | `@ConfigurationProperties`, constructor-bound |
| Startup-time config validation | `@Validated` + Bean Validation annotations on the properties class |
| Environment-specific override without rebuild | Environment variables / command-line args (outrank file-based config) |
| Environment-specific base settings | `application-{profile}.yml`, overlaying `application.yml` |
| Env var ↔ property key mismatch | Relaxed binding (`APP_FOO_BAR` ↔ `app.foo-bar`) |
| Secrets | Never in plain config files — use a secrets manager |

---

*(Continues to Chapter 3 — Building REST Controllers.)*
