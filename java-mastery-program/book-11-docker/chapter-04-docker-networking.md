# CHAPTER 4 — DOCKER NETWORKING

---

## 4.1 CONCEPT: Network Drivers — Bridge, Host, and None

### TELUGU EXPLANATION

**Docker, ప్రతి container కి, ఒక network driver ఎంచుకునే option
ఇస్తుంది:**

- **Bridge (default):** Docker, ఒక **virtual network** create చేస్తుంది
  (host మీద), ప్రతి container కి, ఆ virtual network లో ఒక **own,
  isolated IP** ఇస్తుంది. Containers, ఈ bridge ద్వారా ఒకదానితో ఒకటి
  communicate చేయగలవు; host నుండి access చేయాలంటే, **port mapping**
  (section 4.2) అవసరం.
- **Host:** Container, **host యొక్క network stack నే నేరుగా share**
  చేస్తుంది — port mapping అవసరం లేదు (container లోని port, host
  మీద నేరుగా అదే port గా అందుబాటులో ఉంటుంది), కానీ **isolation
  తగ్గుతుంది** (Chapter 1 యొక్క namespace isolation యొక్క network
  భాగం, ఇక్కడ **disable** అవుతుంది).
- **None:** Container కి **ఎలాంటి network access ఉండదు** — పూర్తిగా
  isolated (rare, specific security-sensitive use cases కోసం).

**Senior-level default:** Production Java services కి, **bridge**
(లేదా Compose/Kubernetes యొక్క తమ own equivalent networking) సరైనది
— `host` mode, isolation ని weaken చేస్తుంది కాబట్టి, సాధారణంగా
avoid చేస్తారు, performance-critical, specific scenarios తప్ప.

### ENGLISH INTERVIEW ANSWER

"Docker offers a few network drivers per container. Bridge, the
default, creates a virtual network on the host where each container
gets its own isolated IP; containers can talk to each other over this
bridge, but reaching them from the host requires explicit port mapping,
which we cover next. Host networking makes the container share the
host's network stack directly — no port mapping needed since a
container's port is directly the host's same port — but this weakens
isolation by disabling the network portion of the namespace isolation
from Chapter 1. None gives a container no network access at all, used
rarely, for specific security-sensitive isolation needs. My default for
production Java services is bridge networking — host mode trades away
real isolation for a convenience that's rarely worth it outside
specific performance-critical edge cases, so I'd only reach for it with
a clear, deliberate justification."

---

## 4.2 CONCEPT: Port Mapping — Bridging Container-Internal and Host-External Ports

### TELUGU EXPLANATION

**`docker run -p 8080:8080 myimage`:** ఇది **host port : container
port** mapping ని define చేస్తుంది — host యొక్క port 8080 కి వచ్చే
traffic, container లోపలి port 8080 కి route అవుతుంది. **ఇవి రెండూ
ఒకే number గా ఉండాల్సిన అవసరం లేదు** — `docker run -p 9090:8080`,
host port 9090 ని, container port 8080 కి map చేస్తుంది (ఉదా: ఒకే
host మీద, అదే image నుండి **అనేక containers** run చేయాలంటే, ప్రతి
దానికి **వేరే host port** అవసరం, container internal port ఒకటే
అయినా).

**Chapter 2.2 యొక్క `EXPOSE` తో సంబంధం:** `EXPOSE`, కేవలం
documentation; `-p` flag మాత్రమే, actual traffic ని route చేస్తుంది
— ఒక image, `EXPOSE` లేకుండా కూడా, `docker run -p` తో publish
చేయొచ్చు (అది just less self-documenting).

### ENGLISH INTERVIEW ANSWER

"`docker run -p 8080:8080 myimage` defines a host-port-to-container-port
mapping — traffic hitting the host's port 8080 routes to the
container's internal port 8080. These don't have to match:
`-p 9090:8080` maps host port 9090 to the container's port 8080, which
is exactly what you'd need to run multiple containers from the same
image on one host, each needing a distinct host port even though their
internal port is identical. This connects back to Chapter 2's `EXPOSE`
instruction — `EXPOSE` is purely documentation, and it's the `-p` flag
at `docker run` time that actually does the real work of routing
traffic; you can publish a port with `-p` even on an image that never
declared `EXPOSE` for it, it's just less self-documenting for anyone
reading the Dockerfile."

---

## 4.3 CONCEPT: Container-to-Container Communication — Docker's Built-In DNS

### TELUGU EXPLANATION

**ఇది Book 8 Chapter 2 (Service Discovery) కి direct సారూప్యం,
local Docker environment లో:** ఒకే custom bridge network లో ఉన్న
containers, ఒకదానికొకటి, **IP address కి బదులుగా, container/service
name ద్వారా** reach అవ్వగలవు — Docker, ఆ network కోసం ఒక **internal
DNS** run చేస్తుంది. ఉదాహరణకి, ఒక `order-service` container, ఒక
`inventory-service` container ని, `http://inventory-service:8080`
అనే URL తో reach అవ్వగలదు — actual IP address తెలుసుకోవాల్సిన
అవసరం లేదు (IP, container restart అయినప్పుడు కూడా మారొచ్చు).

**Senior-level connection, explicit గా:** ఇది, Book 8 Chapter 2 లో
చూసిన Eureka యొక్క **service discovery** concept యొక్క, చాలా
సరళమైన, local-development-scale రూపం — production/Kubernetes లో,
ఇదే idea, DNS-based service discovery (Kubernetes Services) గా
మరింత robust గా extend అవుతుంది (Book 12 లో చూస్తాం).

**Default `bridge` network యొక్క limitation:** Docker యొక్క
**default** bridge network (`docker run` ఏ `--network` flag లేకుండా
వాడితే create అయ్యేది), ఈ DNS-based name resolution ని support
చేయదు — ఇది **custom, user-defined bridge networks** కి మాత్రమే
పనిచేస్తుంది (`docker network create`, లేదా Compose, Chapter 6 లో
చూసినట్టు, automatic గా ఒకటి create చేస్తుంది).

### ENGLISH INTERVIEW ANSWER

"This is a direct, local-scale analog of Book 8 Chapter 2's service
discovery concept. Containers on the same custom bridge network can
reach each other by container or service name instead of IP address,
because Docker runs an internal DNS for that network. An
`order-service` container can reach `inventory-service` at
`http://inventory-service:8080` without ever needing to know its actual
IP, which is convenient since that IP can change across restarts. I'd
explicitly connect this to Eureka from Book 8: this is genuinely the
same underlying idea — resolving a logical service name to an actual
network location — just implemented as simple DNS at local development
scale rather than a full service registry; in production on Kubernetes,
which we cover in Book 12, this same idea extends into a more robust
DNS-based service discovery mechanism. One practical gotcha worth
knowing: Docker's *default* bridge network, the one you get without
specifying `--network`, doesn't actually support this name-based DNS
resolution — it only works on custom, user-defined bridge networks,
which is exactly what `docker network create` gives you, and what
Docker Compose sets up automatically, as we'll see in Chapter 6."

---

## 4.4 SENIOR VS JUNIOR THINKING

| Situation | Junior | Senior |
|---|---|---|
| Choosing a network mode | Uses host mode for convenience | Uses bridge by default, reserving host mode for justified exceptions |
| Container-to-container communication | Hardcodes container IP addresses | Uses container/service names on a custom bridge network's DNS |
| Running multiple instances on one host | Doesn't realize port conflicts will occur | Maps distinct host ports to the same internal container port |
| Understanding `EXPOSE` vs `-p` | Conflates the two | Knows `EXPOSE` documents; `-p` actually routes traffic |

---

## 4.5 COMMON MISTAKES

1. Using host networking by default, unnecessarily weakening
   container isolation.
2. Hardcoding container IP addresses instead of using service names on
   a custom bridge network.
3. Relying on Docker's default bridge network and being confused when
   name-based resolution doesn't work.
4. Trying to map the same host port for multiple containers running
   the same image on one host.
5. Assuming `EXPOSE` in the Dockerfile is sufficient without an
   explicit `-p` flag at runtime.

---

## 4.6 INTERVIEW QUESTION BANK — CHAPTER 4

**Basic:** 1. What's the difference between bridge and host network
modes? 2. What does `-p 8080:9090` do?

**Intermediate:** 3. Why can't two containers using host networking
both bind to port 8080? 4. Why doesn't Docker's default bridge network
support container name resolution?

**Senior:** 5. Design the networking setup for three containers (an
API service, a database, and a cache) that need to communicate with
each other but should only expose the API service to the host.

**Architect:** 6. Explain to a team why they shouldn't default to host
networking "for simplicity" across all their containerized services.

**Scenario:** 7. Two containers on the same custom bridge network can't
reach each other by name, but `docker network inspect` shows they're
both attached to it. What would you check next?

**Trick:** 8. "Since containers on the same bridge network can resolve
each other by name, they must also be reachable from the host machine
using that same name." True or false?

<details><summary>Key answers</summary>

- Q5: Create a custom bridge network and attach all three containers to
  it. The API service, database, and cache communicate with each other
  by container name over that network (e.g., API connects to
  `db:5432` and `cache:6379`). Only the API service gets a `-p` port
  mapping to the host (e.g., `-p 8080:8080`) — the database and cache
  containers have no host port mapping at all, so they're reachable
  from other containers on the network but not from outside the Docker
  host's container network.
- Q6: Host networking disables the network isolation namespaces provide
  (Chapter 1), meaning containers share the host's actual network
  stack directly — this creates port conflicts when running multiple
  containers needing the same port, removes the ability to run
  multiple instances of the same service on one host without manual
  port coordination, and increases the blast radius of any
  network-level compromise since there's no network boundary between
  the container and host. The "simplicity" it offers (no port mapping
  needed) isn't worth losing this isolation for typical application
  services.
- Q7: Check whether both containers are actually using a *custom*
  user-defined bridge network, not each independently attached to the
  *default* bridge network (which doesn't support name resolution) —
  `docker network inspect` showing them "attached" needs to specifically
  confirm they're on the same custom network, not that they each have
  some bridge network membership. Also verify there's no typo in the
  service name being used to connect, and that both containers are
  actually running and healthy.
- Q8: False — a custom bridge network's internal DNS only resolves
  names for other containers *on that same network*, not for the host
  machine. Reaching a container from the host machine requires an
  explicit port mapping via `-p`, regardless of whether that container
  is also reachable by name from its network peers; the two mechanisms
  (internal DNS between containers, and host-to-container port
  publishing) are separate and serve different communication paths.

</details>

---

## 4.7 MASTERY CHECKPOINTS

- **Knowledge Check:** Explain why Docker's default bridge network behaves differently from a custom, user-defined bridge network for name resolution.
- **Coding Check:** Write the `docker network create` and `docker run --network` commands needed to put two containers (a web app and a database) on the same custom network, with only the web app's port published to the host.
- **Explanation Check:** Explain to a teammate why host networking mode is generally discouraged for typical application containers.
- **Real-World Check:** Your team wants to run 3 instances of the same API container on one host for local load-testing. How would you configure port mappings to make this work?
- **Senior Check:** Why is Docker's container-name DNS resolution described as "a local-scale analog" of Eureka rather than a full replacement for service discovery in production?
- **Master Check:** Design the full local networking setup for a Book 8-style microservices stack (order-service, inventory-service, payment-service, and a shared PostgreSQL container) running via plain `docker run` commands (not yet Compose), ensuring correct isolation and inter-service communication.

<details><summary>Answers</summary>

- Real-World Check: Run each instance with a distinct host port mapped
  to the same internal container port — e.g.,
  `-p 8081:8080`, `-p 8082:8080`, `-p 8083:8080` for three instances all
  internally listening on 8080 — allowing a load-testing tool to
  distribute requests across `localhost:8081`, `:8082`, and `:8083`.
- Senior Check: Docker's built-in DNS only provides basic name-to-IP
  resolution within a single Docker host's network — it has no concept
  of health checks, load balancing across multiple healthy instances of
  a service, cross-host service registration, or the richer metadata
  Eureka provides; it solves the same fundamental "resolve a name to a
  location" problem but at a much smaller scale and with far fewer
  production-grade capabilities, which is why Kubernetes's more robust
  DNS-based service discovery (Book 12) is what's actually used in production.
- Master Check: Create one custom bridge network (e.g.,
  `microservices-net`). Run the PostgreSQL container attached to it
  with no host port mapping (only accessible to other containers on
  the network, by name `postgres`). Run inventory-service and
  payment-service attached to the same network, also without host port
  mappings, reachable by name from order-service. Run order-service
  attached to the same network, with a `-p 8080:8080` mapping being the
  *only* host-exposed port — replicating the same "only the entry point
  is externally reachable" pattern from Q5, scaled to a realistic
  microservices topology.

</details>

---

## 4.8 CHEAT SHEET

| Concept | Rule |
|---|---|
| Bridge network | Default, isolated per-container IP — the right default for most services |
| Host network | Shares host's network stack directly — weakens isolation, avoid by default |
| None network | No network access at all — rare, security-sensitive use cases |
| Port mapping (`-p host:container`) | Only mechanism that actually routes host traffic into a container |
| `EXPOSE` vs `-p` | `EXPOSE` documents; `-p` actually publishes |
| Container-to-container DNS | Works by name, but ONLY on custom (not default) bridge networks |
| Local service discovery | A small-scale analog of Eureka — not a production replacement |
| Multiple instances, one host | Map each to a distinct host port; internal ports can stay identical |

---

*(Continues to Chapter 5 — Docker Volumes & Data Persistence.)*
