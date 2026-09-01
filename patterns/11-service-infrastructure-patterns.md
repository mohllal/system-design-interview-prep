---
title: "Service infrastructure patterns"
concepts:
  - api-gateway
  - backend-for-frontend
  - service-discovery
  - service-registry
  - service-mesh
  - sidecar-proxy
  - client-side-vs-server-side-load-balancing
  - configuration-and-secret-distribution
related:
  - patterns/03-microservices-architecture.md
  - patterns/13-backend-for-frontend.md
  - fundamentals/12-proxies.md
  - fundamentals/13-load-balancing.md
  - fundamentals/28-leader-election.md
---

# Service infrastructure patterns

Service infrastructure is the layer that lets services find each other, be reached by clients, authenticate their peers, and be configured, without every service implementing all of that itself.

[Microservices architecture](./03-microservices-architecture.md) describes the decomposition. This document describes what has to exist underneath it for the decomposition to actually run.
The patterns here are not microservices-specific: any system with more than a couple of separately deployed processes hits the same questions, and answers them either deliberately with a gateway, a registry, and a config store, or accidentally with hardcoded IP addresses and a wiki page of environment variables.

The framing worth holding onto is that none of this is new functionality. A monolith already had routing, dependency lookup, and configuration; it got them from a URL router, an import statement, and a file on disk. Splitting the process moves each one onto the network, where it becomes a distributed systems problem with its own failure modes.

## Why this layer exists

Inside one process, calling another component is a method call against an address the linker resolved at startup. Across a fleet, the same call needs an address that changes every deploy, a health signal, a timeout, a retry policy, a credential, and a trace header. Every one of those is a small problem.
The reason they need a platform is that there are N services multiplied by M concerns, and solving each cell independently produces N slightly different, slightly wrong implementations.

| Concern             | Inside one process              | Across a service fleet                                                          |
| ------------------- | ------------------------------- | ------------------------------------------------------------------------------- |
| Finding the callee  | Imports resolved at build time  | Service discovery against a registry that changes constantly                    |
| Reaching the callee | A method call that cannot fail  | A network hop with timeouts, retries, and a load balancing decision             |
| Client entry point  | One URL router                  | An API gateway in front of many independently deployed services                 |
| Trust               | Same process, same memory       | Per-service identity, usually mTLS                                              |
| Configuration       | One file next to one deployable | A store every service reads, updatable without a redeploy                       |
| Observability       | One stack trace                 | Correlated traces propagated across every hop                                   |
| Failure isolation   | `try` / `except`                | Timeouts, [circuit breakers](./08-circuit-breaker.md), bulkheads per dependency |

There are only two places to solve any of these: inside each service, as a library, or outside it, as infrastructure.

```mermaid
graph TB
    subgraph "Per-service: every team reimplements"
        S1["Service A (Java)<br/>discovery + retries + mTLS + metrics"]
        S2["Service B (Go)<br/>discovery + retries + mTLS + metrics"]
        S3["Service C (Python)<br/>discovery + retries + mTLS + metrics"]
    end

    subgraph "Platform: implemented once, applied uniformly"
        P1["Service A"]
        P2["Service B"]
        P3["Service C"]
        INFRA["Gateway, registry, mesh, config store"]
        P1 --- INFRA
        P2 --- INFRA
        P3 --- INFRA
    end
```

The library approach is cheaper to start and gives the application full control, which is why Netflix's original stack (Eureka for discovery, Ribbon for client-side load balancing, Hystrix for breakers) was a set of Java libraries. Its limits show up in two places: a second language means a second implementation, and a policy change means recompiling and redeploying every service that embeds it.
The infrastructure approach pays a larger fixed cost for a uniform, out-of-process answer that a platform team can change on its own.

Most real systems run both. A gateway at the edge, discovery and load balancing wherever they are cheapest, and application-level fallbacks that no proxy could implement because only the application knows what a request means.

## API gateway

A gateway is the single entry point clients talk to. Architecturally it is a reverse proxy (see [proxies](../fundamentals/12-proxies.md#reverse-proxy-server-side)) with routing intelligence and knowledge of the API surface: it terminates TLS, authenticates the caller, applies quotas, routes by path or host to whichever service owns that part of the API, and hides the service topology from clients entirely.

```mermaid
graph LR
    WEB[Web client] --> GW
    MOB[Mobile client] --> GW
    PARTNER[Partner API client] --> GW

    subgraph "Edge"
        GW["API gateway<br/>TLS, authn, quotas, routing, aggregation"]
    end

    GW -->|"/users/**"| USERS[User service]
    GW -->|"/orders/**"| ORDERS[Order service]
    GW -->|"/catalog/**"| CATALOG[Catalog service]
    GW -->|"/payments/**"| PAY[Payment service]
```

The value is that cross-cutting concerns get implemented once, at a point every request already passes through:

- **TLS termination**: One certificate lifecycle at the edge instead of one per service. Whether the internal hop is re-encrypted is a separate, explicit decision, and a [service mesh](#service-mesh-and-sidecar-proxies) is the usual way to say yes without touching application code.
- **Authentication**: Validate the token once, reject anonymous traffic at the edge, and forward a verified identity inward. Services then do *authorization* against an identity they can trust rather than re-parsing credentials.
- **Rate limiting and quotas**: The edge is where per-client identity is actually known, which is exactly the placement argument in [rate limiting](../fundamentals/32-rate-limiting.md#placement). Rejecting excess traffic here spends no backend capacity on requests that will fail anyway.
- **Routing and versioning**: Path, host, and header routing lets `/v2/orders` reach a new service while `/v1/orders` still reaches the old one, so client migration and service migration are decoupled.
- **Observability**: One place that sees every external request, assigns the request ID, and starts the trace that [observability](../fundamentals/15-observability.md#distributed-tracing) then propagates across hops.
- **Protocol translation**: External REST or GraphQL over HTTP/1.1, internal gRPC over HTTP/2, without every service exposing both.

### Aggregation and fan-out

The second job is composition. A product page needs the product, its price, its stock, and its reviews, which live in four services. Without a gateway the client makes four round trips over a mobile network; with one it makes a single request and the fan-out happens inside the data center where round trips cost a millisecond rather than a hundred.

```mermaid
sequenceDiagram
    participant C as Mobile client
    participant G as API gateway
    participant CAT as Catalog service
    participant PR as Pricing service
    participant RV as Review service

    C->>G: GET /product-page/1234
    par Fan-out inside the data center
        G->>CAT: GET /products/1234
        G->>PR: GET /prices/1234
        G->>RV: GET /reviews?product=1234
    end
    CAT-->>G: 200 product
    PR-->>G: 200 price
    RV-->>G: timeout after 300ms
    Note over G: Reviews are optional, so the page still renders
    G-->>C: 200 composed page, reviews omitted
```

Aggregation is where a gateway earns its place, and also where it becomes dangerous. Fan-out means the response is only as fast as the slowest upstream and only as available as the *product* of every upstream's availability, unless the gateway is explicit about which parts are required and which are optional.

```python
async def call(name, client, required):
    try:
        return await asyncio.wait_for(client.fetch(product_id), timeout=0.3)
    except (asyncio.TimeoutError, ConnectionError):
        if required:
            # Without this distinction the page's availability is the
            # product of four services' availability.
            raise GatewayError(f"{name} unavailable")
        return None  # Optional: degrade instead of failing the page

product, price, reviews = await asyncio.gather(
    call("catalog", catalog, required=True),
    call("pricing", pricing, required=True),
    call("reviews", reviews, required=False),
)
```

Note what the gateway does *not* do here: it does not decide what a price means, whether a discount applies, or whether the product is visible to this user. It calls, waits, and assembles. The moment it starts deciding, it has become a service that every team must change together, which is the [gateway as a business logic dumping ground](#the-gateway-as-a-business-logic-dumping-ground) anti-pattern.

### Backend for frontend

One gateway serving every client type ends up serving all of them badly: a mobile app wants few, small, pre-composed payloads, a web app can afford richer and chattier calls, and a partner API wants a stable contract above all. The backend-for-frontend pattern gives each client type its own gateway, owned by the team that owns that client, instead of one shared gateway trying to satisfy all three.

[Backend for frontend](./13-backend-for-frontend.md) is the full treatment: the ownership argument, the shared-edge-plus-BFFs topology, the pros and cons, and the discipline (presentation and composition only, never the only place a business rule lives) that keeps a BFF from turning into an unmaintainable pile of client-specific logic.

## Service discovery

An instance's address is not a stable fact. Autoscaling adds and removes instances, a rolling deploy replaces every one of them, a container scheduler moves a pod to a different node with a different IP, and a crash-and-restart cycle can change an address in seconds. Anything that writes an address down is wrong shortly afterward.

Service discovery replaces "where is the order service" as a configuration question with "where is the order service *right now*" as a runtime lookup.

```mermaid
graph LR
    subgraph "Registry"
        R["Service registry<br/>order-service:<br/>10.0.1.7:8080 healthy<br/>10.0.1.9:8080 healthy<br/>10.0.2.3:8080 draining"]
    end

    I1["order-service<br/>10.0.1.7"] -->|register + heartbeat| R
    I2["order-service<br/>10.0.1.9"] -->|register + heartbeat| R
    I3["order-service<br/>10.0.2.3"] -->|deregistering| R
    CALLER[Caller] -->|lookup| R
```

### The service registry

The registry is a database of live instances, and its whole job is three operations plus a failure story:

- **Register**: An instance announces its service name, address, port, and metadata (version, zone, protocol) when it becomes ready to serve.
- **Heartbeat**: The instance renews a lease on an interval well inside the time-to-live, so one dropped renewal does not evict a healthy instance.
- **Deregister**: On graceful shutdown, the instance removes itself *before* it stops accepting connections, so in-flight traffic drains instead of erroring.
- **Expire**: When heartbeats stop without a deregistration, the lease times out and the entry is removed. This is the backstop for every ungraceful exit, and it is why registration must be a lease rather than a permanent record.

```mermaid
stateDiagram-v2
    [*] --> Starting
    Starting --> Registered: Readiness check passes, register with TTL lease
    Registered --> Registered: Heartbeat renews the lease
    Registered --> Draining: Graceful shutdown, explicit deregister
    Registered --> Expired: Heartbeats stop, TTL elapses
    Draining --> [*]: In-flight requests complete
    Expired --> [*]: Entry removed, callers refresh
```

```python
def _heartbeat_loop(self):
    # Renew at a fraction of the TTL: losing one heartbeat to a blip
    # must not be enough to expire a healthy instance.
    interval = self.ttl_seconds / 3
    while not self._stop.wait(interval):
        try:
            self.registry.heartbeat(self.instance_id)
        except ConnectionError:
            pass  # If it stays unreachable, the lease expires on its own

def stop(self):
    """Graceful shutdown: leave the rotation before closing the listener."""
    self._stop.set()
    self.registry.deregister(self.instance_id)  # TTL is the backstop if this fails too
```

The registry is on the critical path of every lookup, so it has to be more available than the services it serves.
That makes it a replicated cluster with an elected leader, and the [leader election](../fundamentals/28-leader-election.md) trade-offs apply directly: strongly consistent registries (Consul, etcd, ZooKeeper) refuse writes during a partition rather than serve divergent membership, while AP-leaning ones (Eureka) keep serving a possibly stale list.

For a registry, stale usually beats unavailable. A caller holding a slightly outdated instance list will hit some dead addresses and retry elsewhere; a caller that cannot resolve anything at all fails every request.
This is the same reasoning that makes [lease-based election](../fundamentals/28-leader-election.md#lease-based-election) the practical registry mechanism: a lease expires on its own, so a partitioned registry node cannot keep a dead instance listed forever.

### Self-registration vs third-party registration

Who actually writes the entry is a real design split.

| Dimension     | Self-registration                                            | Third-party registration                                        |
| ------------- | ------------------------------------------------------------ | --------------------------------------------------------------- |
| Who registers | The service, from inside its own process                     | A registrar that watches the deployment platform                |
| Health signal | The instance's own heartbeat                                 | The platform's health checks and scheduler state                |
| Code impact   | The service links a registry client and knows the registry   | The service knows nothing about discovery at all                |
| Failure mode  | A wedged process can keep heartbeating while unable to serve | The registrar is another component that can fail                |
| Typical of    | Eureka, Consul agents driven by the application              | Kubernetes endpoints, AWS ECS service registration, Registrator |

Self-registration is simpler to reason about and gives the instance control over exactly when it declares itself ready. Its weakness is that a process can be healthy enough to send a heartbeat and too broken to serve, so the heartbeat should be gated on the same readiness check the load balancer uses, not on a bare thread being alive.

Third-party registration keeps discovery entirely out of application code, which is why every container platform does it. Kubernetes is the canonical example: the application does nothing at all, and the platform adds a pod to the Service's endpoint list when its readiness probe passes and removes it when the probe fails or the pod terminates.

### Client-side discovery

The caller asks the registry for the instance list, caches it, and picks one itself.

```mermaid
graph LR
    subgraph "Caller process"
        APP[Application code]
        DC["Discovery client<br/>cached instance list<br/>+ picks an instance"]
        APP --> DC
    end

    DC -.->|lookup + refresh| REG[Service registry]
    DC -->|direct call| I1["order-service 10.0.1.7"]
    DC -.->|not chosen this time| I2["order-service 10.0.1.9"]
```

```python
def instances(self, service_name):
    entry = self._cache.get(service_name)
    fresh = entry is not None and time.monotonic() - entry[0] < self.refresh_seconds
    if not fresh:
        try:
            self._cache[service_name] = (time.monotonic(), self.registry.lookup(service_name))
        except ConnectionError:
            # Serve the stale list. A registry outage must not become a total
            # outage: most of the addresses from a minute ago still answer.
            if entry is None:
                raise NoHealthyInstances(service_name)
    return list(self._cache[service_name][1])
```

The two properties that matter are in the comments. The list is **cached**, so the registry is not on the per-request path; and the cache is **served stale on failure**, so a registry outage degrades routing accuracy instead of stopping traffic.

### Server-side discovery

The caller sends every request to a stable address, and something in the path resolves the actual instance.

```mermaid
graph LR
    APP[Caller application] -->|"http://orders.internal"| LB["Router: internal load balancer,<br/>mesh sidecar, or Kubernetes Service"]
    LB -.->|watches| REG[Service registry]
    LB --> I1["order-service 10.0.1.7"]
    LB --> I2["order-service 10.0.1.9"]
```

The caller's code contains one hostname and no discovery logic at all. Kubernetes is the mainstream version of this: `orders.default.svc.cluster.local` resolves to a virtual IP, and kube-proxy or a mesh sidecar forwards to a ready pod.

| Dimension              | Client-side discovery                                  | Server-side discovery                                 |
| ---------------------- | ------------------------------------------------------ | ----------------------------------------------------- |
| Who picks the instance | The caller                                             | The router, load balancer, or sidecar                 |
| Network hops           | One, caller straight to instance                       | Two, unless the router is a local sidecar             |
| Language cost          | A discovery client per language in the fleet           | None, callers just use a hostname                     |
| Routing intelligence   | Full, the caller knows latency and errors per instance | Whatever the router exposes                           |
| Failure surface        | Registry outage degrades to a stale cache              | The router is on the path of every request            |
| Fits                   | A homogeneous stack that wants precise load balancing  | Polyglot fleets, and anything on a container platform |

### DNS-based discovery

The simplest real implementation of server-side discovery is the one that already exists everywhere: publish instances as DNS records and let clients resolve the name. Kubernetes headless services and Consul's DNS interface both work this way, and for many systems it is enough.

It is worth being explicit about the limits, because they are the reason dedicated registries exist:

- **TTLs make convergence slow**: A record cached for 60 seconds means callers keep dialing an instance that went away a minute ago. Very low TTLs shift the cost onto the resolvers.
- **Clients ignore TTLs**: Many runtimes and connection pools resolve once at startup and cache the result for the life of the process, so the record's TTL is advisory at best.
- **A DNS record carries almost no metadata**: No version, no zone, no weight, no health beyond present-or-absent. `SRV` records add port and priority and are still thin.
- **No health semantics**: DNS answers "what addresses exist", not "which of them are ready".

DNS is the right starting point precisely because it needs no new component. Outgrow it when convergence time starts showing up as errors during deploys.

## Service mesh and sidecar proxies

Everything above can be done with libraries. The reason meshes exist is what happens when the fleet is not homogeneous: discovery, retries with jitter, per-dependency timeouts, circuit breaking, mTLS, and trace propagation now have to be implemented and kept consistent in Java, Go, Python, Node.js, and the two Rust services someone wrote last quarter.
A policy change ("retry budget is now 10%", "mTLS is mandatory") becomes a fleet-wide code migration.

A service mesh moves that logic out of the process and into a proxy deployed next to every service instance.

### The sidecar pattern

A sidecar proxy runs alongside one service instance and intercepts all of its inbound and outbound traffic. The application makes a plain HTTP call to `localhost`; the sidecar handles discovery, instance selection, encryption, retries, and telemetry.

```mermaid
graph LR
    subgraph "Pod A"
        SA["Service A"] -->|"plain HTTP to localhost"| PA["Sidecar proxy (Envoy)"]
    end
    subgraph "Pod B"
        PB["Sidecar proxy (Envoy)"] -->|"plain HTTP to localhost"| SB["Service B"]
    end
    PA -->|"mTLS, retries, timeouts, metrics"| PB
```

This is the [sidecar proxy](../fundamentals/12-proxies.md#sidecar-proxy-per-instance) from the proxies document, applied fleet-wide. What it buys, concretely:

- **mTLS everywhere without touching application code**: Both proxies hold workload identities issued by the mesh, verify each other, and rotate certificates on a short cycle. The application never handles a key.
- **Uniform resilience policy**: Timeouts, retries with backoff, outlier detection, and connection pool limits configured once and applied identically to every language.
- **Consistent telemetry**: Every hop produces the same request metrics and trace spans, so a latency breakdown is complete rather than covering only the services whose teams instrumented properly.
- **Traffic shaping as a deploy tool**: Weighted routing for canaries, mirroring production traffic to a new version, and fault injection for resilience testing, all without a code change.

The limits are worth stating alongside. A sidecar sees bytes and status codes, not meaning, so it can fail fast but cannot serve a cached value or queue a write. Application-level fallbacks stay in the application, which is the same division of labour described under [where the breaker runs](./08-circuit-breaker.md#where-the-breaker-runs).

### Control plane vs data plane

A mesh is two distinct systems, and conflating them is the most common source of confusion about what a mesh actually is.

```mermaid
graph TB
    subgraph "Control plane"
        CP["Mesh control plane (Istio, Consul, Linkerd)"]
        CA["Certificate authority: workload identities"]
        CFG["Policy and route configuration"]
        CP --- CA
        CP --- CFG
    end

    subgraph "Data plane"
        P1["Sidecar A"]
        P2["Sidecar B"]
        P3["Sidecar C"]
    end

    CP -->|"config + certs, pushed on change"| P1
    CP -->|"config + certs, pushed on change"| P2
    CP -->|"config + certs, pushed on change"| P3
    P1 -->|request traffic| P2
    P2 -->|request traffic| P3
```

- **Data plane**: The sidecars. They carry every byte of application traffic, and their performance and stability are what users experience.
- **Control plane**: One logical service that knows the topology, computes the configuration each sidecar needs, issues workload certificates, and pushes updates. It is nowhere near the request path.

This is the same split [Kafka uses between its controller and its brokers](../advanced/05-kafka-architecture.md#control-plane-vs-data-plane), and the reasoning is identical: put the small, consistency-critical state behind a strongly consistent mechanism, and keep the high-volume path free of coordination. The practical consequence is the same too.
A control plane outage does not stop traffic, because the sidecars keep running their last-known-good configuration; it stops *changes*, so new instances may not get configuration and certificates cannot be rotated. That is a degraded state with a deadline attached, not an immediate outage.

### What a mesh costs

- **Latency per hop**: Two extra proxy traversals per call, one on each side. Each is small in isolation, on the order of a millisecond or less for a well-tuned proxy, but a request that crosses five services pays it ten times, and it lands hardest on tail latency.
- **Resources per instance**: A proxy process per service instance, with its own memory and CPU. Across a large fleet this is a real line item, and it is why sidecar-less (per-node or per-service-account) mesh designs exist.
- **Operational surface**: The mesh is now on the critical path of every request, so it needs its own upgrade strategy, dashboards, and on-call knowledge. A mesh upgrade is a fleet-wide change to the data path.
- **Debugging indirection**: Failures acquire a new class of cause. "Which proxy rejected this, and under which policy" becomes a question every engineer has to be able to answer, and proxy-generated status codes have to be distinguishable from application ones.

The honest summary is that a mesh trades N language-specific implementations for one uniform implementation plus a distributed system to operate. That trade is clearly good at fifty polyglot services and clearly bad at five services in one language, where a shared client library does the same job for a fraction of the cost.

## Client-side vs server-side load balancing

Discovery answers *which instances exist*. Load balancing answers *which one gets this request*. The algorithms themselves (round robin, least connections, weighted variants, latency-aware choices, and consistent hashing for affinity) are covered in [load balancing](../fundamentals/13-load-balancing.md#load-balancing-algorithms); this section is about where the decision is made, which is an architectural choice with different consequences.

```mermaid
graph TB
    subgraph "Client-side"
        C1["Caller<br/>holds instance list<br/>picks the instance"] --> T1[Instance 1]
        C1 -.-> T2[Instance 2]
        C1 -.-> T3[Instance 3]
    end

    subgraph "Server-side"
        C2["Caller<br/>knows one address"] --> LB["Load balancer<br/>picks the instance"]
        LB --> T4[Instance 1]
        LB -.-> T5[Instance 2]
        LB -.-> T6[Instance 3]
    end
```

| Dimension            | Client-side                                        | Server-side                                         |
| -------------------- | -------------------------------------------------- | --------------------------------------------------- |
| Decision location    | In the caller's process or its sidecar             | In a shared load balancer on the path               |
| Extra hop            | None                                               | One, unless the balancer is a local sidecar         |
| Bottleneck risk      | None, the decision scales with callers             | The balancer tier must be scaled and made redundant |
| View of the backends | Only this caller's own experience of them          | Aggregate view across all callers                   |
| Policy change        | Redeploy callers, unless a sidecar owns the policy | Change one component                                |
| Language cost        | A client per language, unless a sidecar owns it    | None                                                |

Three practical points decide most real arguments:

- **A mesh sidecar makes the distinction mostly disappear.** The sidecar is a client-side balancer by topology (it is local to the caller, so there is no extra network hop and no shared bottleneck) and a server-side one by ownership (the platform configures it, the application knows nothing). This is why "client-side vs server-side" is a less sharp question in 2020s architectures than it was in the Ribbon era.
- **Client-side balancers see a partial picture.** Each caller only knows how backends behaved *for it*, which is enough for least-connections or latency-based choices but produces uneven distribution when callers are few and unequal. A shared balancer sees everything and can distribute globally.
- **Health checking has to live wherever the decision lives.** A client-side balancer needs its own [passive health checks and outlier ejection](../fundamentals/13-load-balancing.md#health-checks), or it will keep confidently choosing a dead instance from a stale list.

The one case that genuinely still wants explicit client-side balancing is long-lived connections. gRPC and other HTTP/2 clients multiplex many requests over one connection, so a Layer 4 balancer pins every request to whichever backend the connection landed on, and load ends up distributed by connection rather than by request.
Either the client balances per request across a pool of connections, or the path needs a Layer 7 balancer that understands streams.

## Configuration and secret distribution

Configuration belongs in this document for one reason: in a fleet, changing a setting cannot require rebuilding and redeploying every service that reads it. Baking configuration into an image makes the deployment pipeline the only mechanism for change, which is fine for a value that changes twice a year and unacceptable for a feature flag, a rate limit, or a rotated credential.

The baseline is the twelve-factor position: configuration is whatever varies between environments, and it lives outside the artifact. The fleet-scale version adds two things that a `.env` file does not provide: one authoritative place for a value shared by many services, and the ability to change it while they are running.

```mermaid
graph LR
    subgraph "Config plane"
        STORE["Config store<br/>(Consul KV, etcd, AWS Parameter Store)"]
        VAULT["Secret store<br/>(Vault, cloud secret manager)"]
    end

    S1[Service A] -->|watch keys| STORE
    S2[Service B] -->|watch keys| STORE
    S1 -->|"short-lived credential, workload identity"| VAULT
    S2 -->|"short-lived credential, workload identity"| VAULT
```

### Centralized configuration and dynamic updates

A config client reads a namespace at startup, watches it for changes, and applies updates to a running process.

```python
def _on_change(self, key, new_value):
    # A curated set of keys is safe to apply live; the rest wait for a
    # restart by design. One bad listener must not block the others.
    if key not in self._hot_reloadable:
        return
    for listener in self._listeners:
        try:
            listener(key, self._values.get(key), new_value)
        except Exception:
            pass
    self._values[key] = new_value
```

Two disciplines make dynamic configuration safe rather than exciting:

- **Be explicit about what is hot-reloadable.** A page size or a feature flag can change under live traffic. A database connection string, a thread pool size, or a listening port generally cannot be swapped safely mid-flight; treat those as restart-only and say so, rather than discovering it during an incident.
- **Roll configuration changes out like deploys.** A bad value pushed to every instance at once is an outage with no rollback artifact. Stage it, watch the same metrics a deploy watches, and keep the previous value one command away.

The failure mode to design against is the config store becoming a hard dependency at startup. Local defaults plus a cached last-known-good snapshot mean a config store outage prevents *changes* rather than preventing services from booting, which is the same degraded-not-down property a mesh control plane has.

### Secrets and rotation without downtime

Secrets are configuration with three extra requirements: they must be encrypted at rest and in transit, access to them must be authorized per workload and audited, and they must be rotatable without a coordinated restart.

The mechanism that makes rotation non-disruptive is overlapping validity. There is never a single instant when the old secret stops working and the new one starts; instead, both are valid for a window long enough for every consumer to pick up the new one.

```mermaid
sequenceDiagram
    participant OP as Operator or scheduler
    participant SS as Secret store
    participant A as Signing service
    participant B as Verifying service

    Note over A,B: Only v1 exists, everything signs and verifies with v1
    OP->>SS: Create v2, keep v1 valid
    SS-->>B: v2 available, v1 still accepted
    Note over B: Verifies signatures from both versions
    SS-->>A: v2 available
    Note over A: Now signs with v2
    Note over OP: Wait longer than the longest token lifetime
    OP->>SS: Revoke v1
    Note over A,B: Only v2 remains valid
```

The ordering is the whole trick: **every verifier must accept the new version before any signer starts producing it**, and the old version is only revoked after nothing can still be holding it.

```python
def verify(self, key_id, payload, signature):
    # Accepting every unexpired version (newest first, in self._versions) is
    # what makes rotation a window rather than a cutover.
    for version in self._versions:
        if version["id"] == key_id:
            return hmac.compare_digest(signature, self._mac(version["material"], payload))
    return False
```

Beyond rotation mechanics, three properties separate a real secret story from a folder of environment variables:

- **Workload identity, not a bootstrap password**: The service proves what it is using a platform-issued identity (a Kubernetes service account token, an instance role, a mesh certificate) and exchanges it for secrets. Otherwise the secret that fetches secrets has to be distributed somehow, and the problem has only moved.
- **Short-lived, dynamically issued credentials**: Rather than a static database password, the service asks for a credential with a lifetime measured in hours. A leaked credential then expires on its own, and rotation is the normal path rather than an emergency procedure.
- **Audit and revocation**: Every read is attributable to a workload, and any version can be revoked immediately. Neither is possible when a secret lives in an image layer or a repository.

## Putting it together

The value of the whole layer is easiest to see in one request's path through it. Nothing below is exotic; this is what a request to a mainstream container-platform deployment actually does.

```mermaid
sequenceDiagram
    participant U as User
    participant DNS as Global DNS / anycast
    participant EDGE as Edge proxy / CDN
    participant GW as API gateway
    participant SC as Order sidecar
    participant OS as Order service
    participant SP as Payment sidecar
    participant PS as Payment service

    U->>DNS: Resolve api.example.com
    DNS-->>U: Nearest region
    U->>EDGE: TLS handshake, HTTPS request
    EDGE->>GW: Forward, coarse limits applied
    GW->>GW: Validate token, apply per-client quota, assign request ID
    GW->>SC: POST /orders with identity and trace headers
    Note over SC: Resolves order-service instances from the control plane's view,<br/>picks one, terminates mTLS
    SC->>OS: Plain HTTP on localhost
    OS->>SC: Needs a payment: POST payment-service/charges
    Note over SC: Client-side pick, timeout and retry budget applied
    SC->>SP: mTLS call to a chosen payment instance
    SP->>PS: Plain HTTP on localhost
    PS-->>SP: 201 Created
    SP-->>SC: Response, span recorded
    SC-->>OS: Response
    OS-->>GW: 201 Created
    GW-->>U: 201 Created
```

Read it as a sequence of narrowing decisions. DNS picks a *region*. The edge and gateway handle everything that is true of *every* request regardless of which service serves it: TLS, identity, quotas, request ID. The sidecar handles everything that is true of *this hop*: which instance, with what timeout, over what encrypted channel, recorded how. The service handles only what is true of *this request's meaning*.

Each layer also has an independent failure story worth being able to state:

| Component fails    | Immediate effect                                                   | What keeps working                                           |
| ------------------ | ------------------------------------------------------------------ | ------------------------------------------------------------ |
| Service registry   | New instances are not discovered, dead ones stay listed            | Existing callers serve from cached instance lists            |
| Mesh control plane | No new config, no certificate rotation, new instances may not join | Sidecars keep routing on last-known-good config              |
| Config store       | Configuration changes do not propagate                             | Services run on cached values and local defaults             |
| API gateway        | External traffic stops                                             | Nothing external, which is why the gateway tier is redundant |
| A single sidecar   | That one instance is unreachable                                   | Every other instance of that service                         |

The gateway row is the important one: a single entry point is also a single point of failure, so it is always deployed as a redundant tier behind its own load balancer.

## When to use this layer

The components are independent, and the right answer is almost always to adopt them one at a time, in the order the pain arrives.

```mermaid
graph TD
    A["How many services?"] -->|One or two| B["A reverse proxy for TLS and auth.<br/>Static addresses are fine."]
    A -->|A handful| C["Gateway plus DNS-based discovery.<br/>Skip the mesh."]
    A -->|Dozens| D{"Polyglot, or fleet-wide<br/>policy changes needed?"}
    D -->|No| E["Gateway plus a shared client library<br/>for discovery, retries, and breakers."]
    D -->|Yes| F["Gateway plus a service mesh.<br/>Budget for operating it."]
    C --> G["Add a config and secret store<br/>as soon as a value is shared by two services."]
    E --> G
    F --> G
```

**Adopt a gateway when:**

- More than one service is exposed to external clients, so routing and auth need one home.
- Multiple client types want meaningfully different API shapes, which is the point at which a BFF per client beats one gateway with conditionals.
- Auth, TLS, or quotas are currently implemented per service, which means they are implemented inconsistently.

**Adopt real service discovery when:**

- Instances are created and destroyed by an autoscaler or a scheduler, so no address is stable.
- Deploys cause errors because callers still hold addresses of instances that are shutting down.
- The count of instances per service is large enough that maintaining a list by hand is already failing.

**Adopt a mesh when:**

- The fleet is polyglot and resilience or security policy has to be identical across languages.
- mTLS between services is a requirement, and modifying every service to do it is not realistic.
- Fleet-wide policy changes are frequent enough that "redeploy every service" is a genuine obstacle.
- There is a team that can own it. A mesh nobody operates is worse than no mesh.

### When it is overkill

- **A single service, or two.** Discovery for two services is a hostname. A mesh here adds two proxies, a control plane, and a new class of outage in exchange for nothing. A plain reverse proxy for TLS termination and auth is still worth having, because certificate handling and token validation are genuinely better done once at the edge.
- **A homogeneous fleet in one language.** A shared client library gives the same discovery, retry, and breaker behaviour with none of the data-plane cost. Revisit when the second language arrives, not before.
- **A platform that already provides the piece.** Kubernetes Services plus an ingress controller already cover discovery, server-side load balancing, and edge routing. Adding a separate registry and gateway on top of that duplicates a working mechanism.
- **A mesh adopted for observability alone.** Distributed tracing is worth having, but instrumenting services with OpenTelemetry is a much smaller commitment than running a data plane. Take the mesh for mTLS, policy, and polyglot uniformity; treat the telemetry as a bonus.

The general rule: adopt the piece whose absence is currently causing incidents, and only that piece. Every component here is on the critical path of every request once installed, which is a serious enough property that it should be paid for by a problem you can name.

## Common anti-patterns

### The gateway as a business logic dumping ground

The gateway is the one component every request passes through, which makes it the most tempting place to put "just one small rule". Rules accumulate, and eventually the gateway knows the pricing model, the entitlement rules, and the order state machine. It is now a service that every team must coordinate to change, deployed by a platform team that understands none of the rules, and it cannot be tested without the whole system.

```python
# Anti-pattern: the gateway decides what things mean
class OrderGateway:
    async def create_order(self, request):
        user = await self.users.get(request["user_id"])
        items = await self.catalog.get_many(request["item_ids"])

        # Pricing rules living at the edge. The pricing team cannot change
        # this, cannot test it, and does not know it exists.
        subtotal = sum(item["price"] * request["quantities"][item["id"]] for item in items)
        if user["tier"] == "gold":
            subtotal *= 0.9
        if subtotal > 100:
            shipping = 0
        else:
            shipping = 5.99

        return await self.orders.create({"user_id": user["id"], "total": subtotal + shipping})


# Better: the gateway routes, authenticates, and shapes. The service decides.
class OrderGateway:
    async def create_order(self, request, caller_identity):
        order = await self.orders.create({
            "user_id": caller_identity.user_id,   # From the validated token
            "item_ids": request["item_ids"],
            "quantities": request["quantities"],
        })
        # Reshaping the response for this client is presentation, not a rule.
        return {"id": order["id"], "total": order["total"], "eta": order["estimated_delivery"]}
```

The test to apply: if deleting the gateway would change what the system *means* rather than only how clients reach it, logic has leaked into the wrong layer.

### A mesh without the problem that justifies it

A mesh is adopted because it is what large companies run, on a fleet of six services all written in the same language. The team now operates a control plane, debugs proxy configuration, and pays two extra hops per call, to get retries and mTLS that a shared HTTP client would have provided in an afternoon. The first mesh upgrade becomes a fleet-wide incident.

The mesh is justified by a specific, statable problem: multiple languages that cannot share a client library, or a policy surface that changes often enough that redeploying every service is a real obstacle, or a hard mTLS requirement across a large fleet. Without one of those, the cost is real and the benefit is aspirational. The reasonable order is a shared client library first, and a mesh when the library stops being able to reach everything.

### Hardcoded service addresses

Discovery exists and the code bypasses it, usually because someone needed one call working quickly and an IP address was right there in a terminal.

```python
import requests

# Anti-pattern: addresses as constants
ORDER_SERVICE_URL = "http://10.0.3.14:8080"
PAYMENT_SERVICE_URL = "http://payment-prod-3.internal:9000"


def get_order(order_id):
    # Breaks the next time that instance is rescheduled, and the failure is
    # a connection refused with no clue about why the address is wrong.
    return requests.get(f"{ORDER_SERVICE_URL}/orders/{order_id}", timeout=2).json()


# Better: resolve a logical name at call time
def get_order(order_id, discovery):
    instance = discovery.pick("order-service")  # Or a stable virtual hostname
    url = f"http://{instance['address']}:{instance['port']}/orders/{order_id}"
    return requests.get(url, timeout=2).json()
```

The variant that hides longer is a service that resolves a hostname once at startup and caches the IP for the life of the process. It looks like it is using discovery, and it fails the same way the constant does, just one deploy later. Resolve per call, or per short-lived connection, and let the pool re-resolve.

### Secrets in configuration files

A database password in a YAML file, a repository, or an environment variable printed by a crash handler. The specific harm is not only exposure: a secret stored this way cannot be rotated without a redeploy, cannot be audited, and cannot be revoked for one service without breaking others that share the file.

```python
# Anti-pattern: the secret is the config
DATABASE_URL = "postgres://app:hunter2@db.internal:5432/orders"  # In git, forever
STRIPE_KEY = "sk_live_xxxx"


# Better: a reference in config, resolved at runtime against a workload identity
class DatabaseConfig:
    def __init__(self, config, secret_store):
        self.host = config.get("database.host")       # Not sensitive
        self.name = config.get("database.name")       # Not sensitive
        self.secret_ref = config.get("database.credential_ref")  # A pointer
        self.secret_store = secret_store

    def connection_url(self):
        # Short-lived credential, issued to this workload, expires on its own.
        credential = self.secret_store.issue(self.secret_ref)
        return (f"postgres://{credential['username']}:{credential['password']}"
                f"@{self.host}:5432/{self.name}")
```

The distinction that keeps this clean is that configuration says *which* secret, never *what* the secret is. A config store holding pointers can be readable by everyone who needs to debug configuration, which is a large part of why the two systems stay separate.

## Interview talking points

- **Name the layer explicitly.** Microservices assume a platform: an entry point, a way to find instances, a way to call them safely, and a way to configure them. Saying "and here is the infrastructure that makes it work" is what separates a design from a box diagram.
- **A gateway is a reverse proxy that knows the API.** It exists so TLS, auth, quotas, and request IDs are implemented once. A BFF exists because one gateway cannot serve mobile, web, and partner clients well at the same time.
- **Discovery is about churn, not lookup.** Instances come and go constantly, so the question is how fast the system converges and what happens to callers during a registry outage. Stale-but-serving beats correct-but-unavailable.
- **Say where the load balancing decision is made, then cite the algorithm separately.** Client-side avoids a hop and scales with callers but needs a client per language; server-side is language-free but is on the path of every request. A mesh sidecar is deliberately both.
- **A mesh is a control plane plus a data plane**, the same split Kafka makes between its controller and its brokers. The control plane going down stops changes, not traffic.
- **Cost the mesh out loud.** Two hops per call, a proxy per instance, and a fleet-wide upgrade path. Justify it with polyglot services or a hard mTLS requirement, or propose a shared client library instead.
- **Configuration and secrets belong in the same conversation.** A fleet needs values it can change without a redeploy, and rotation works by overlapping validity: every verifier accepts the new version before any signer emits it.

## Reference materials

- [Pattern: API Gateway / Backends for Frontends](https://microservices.io/patterns/apigateway.html)
- [Backends For Frontends](https://samnewman.io/patterns/architectural/bff/)
- [Pattern: Service registry](https://microservices.io/patterns/service-registry.html)
- [Pattern: Client-side service discovery](https://microservices.io/patterns/client-side-discovery.html)
- [Pattern: Server-side service discovery](https://microservices.io/patterns/server-side-discovery.html)
- [Pattern: Sidecar](https://microservices.io/patterns/deployment/sidecar.html)
- [Envoy architecture overview](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro)
- [Istio architecture](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Consul service discovery](https://developer.hashicorp.com/consul/docs/concepts/service-discovery)
- [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [The Service Mesh: What Every Software Engineer Needs to Know](https://buoyant.io/service-mesh-manifesto)
- [The Twelve-Factor App: Config](https://12factor.net/config)
