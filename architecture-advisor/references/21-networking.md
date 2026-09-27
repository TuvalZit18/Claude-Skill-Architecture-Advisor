# 21 — Networking

## What this category covers

The mechanical and topological layer: network segmentation (VPCs/subnets), load balancing and traffic routing, API gateway vs direct exposure, service mesh, TLS termination, DNS, and firewalls/egress filtering. As `01-system-level.md` and `SKILL.md` both frame it: `16-security.md` is the *policy* layer (what should be encrypted, who should have access); this file is the *mechanical* layer that actually implements those policies at the network level.

This file is the concrete destination for more deferred cross-references than any other in this skill — `05-scalability.md`'s load balancer and gateway rate limiting, `06-auth.md`'s edge-level auth enforcement, `13-caching.md`'s DNS caching, `14-resilience.md`'s health-check failover and multi-zone routing, `16-security.md`'s TLS termination and SSRF egress filtering, and `20-environment-strategy.md`'s canary/blue-green traffic routing all land here. Each axis below explicitly closes one of these loops.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Network segmentation (VPCs, subnets, public/private separation)

### Flat network (no segmentation — everything in one network space)

- **Pros:** simplest to set up and reason about initially — no subnet planning, no routing rules between segments
- **Cons:** no containment if one component is compromised — an attacker or a misconfigured service that gains network access can reach everything else in it; doesn't distinguish between what genuinely needs to be internet-reachable (a public API) and what should never be directly reachable (a database), leaving broader exposure than necessary

### Segmented network (public/private subnets, or equivalent isolation)

- **Pros:** containment — a compromised or misconfigured component in a private subnet isn't directly reachable from the internet even if something goes wrong; enforces the same blast-radius-containment principle as `01-system-level.md`'s service boundaries and `16-security.md`'s least-privilege access, applied at the network layer — only what genuinely needs to be public (a load balancer, a gateway) sits in a public subnet, with databases and internal services in private subnets unreachable directly from outside
- **Cons:** real complexity in planning and maintaining subnet layout, routing rules, and which components go where; a mistake in the segmentation itself — an overly permissive routing rule — can quietly undermine the whole point

Segmented network topology is close to a non-negotiable baseline for any production system beyond a small internal tool — the containment benefit is one of the cheapest, highest-leverage security investments available, the same tier as least-privilege access in `16-security.md`.

---

## Axis 2: Load balancing & traffic routing

The concrete mechanical home for what `05-scalability.md`, `14-resilience.md`, and `20-environment-strategy.md` all deferred here.

### No load balancer (a single instance, or direct client-to-instance connections)

- **Pros:** simplest — nothing extra to configure or run
- **Cons:** no redundancy at the network layer — a single instance is a single point of failure regardless of what `14-resilience.md`'s Axis 6 says about wanting redundancy, since nothing actually routes traffic around a failed instance; doesn't support horizontal scaling (`05-scalability.md`'s Axis 1) at all, since nothing distributes traffic across multiple instances

### Load balancer with health-check-based routing

- **Pros:** the mechanical implementation of `14-resilience.md`'s Axis 6 redundancy — detects an unhealthy instance via a health-check endpoint and stops routing traffic to it automatically, which is what actually makes running multiple instances protective rather than just idle extra capacity; enables horizontal scaling (`05-scalability.md`) by distributing traffic across however many instances are running; supports the percentage-based traffic routing `20-environment-strategy.md`'s canary release and blue-green cutover both depend on
- **Cons:** the load balancer itself becomes a component that needs to be reliable, though managed load balancer offerings largely handle this; adds a small amount of latency and a configuration surface — health-check tuning, routing rules — to get right

A load balancer with health checks is close to a required companion to any horizontally-scaled or redundant system — without it, the redundancy `14-resilience.md` and `05-scalability.md` call for doesn't actually protect availability in practice.

---

## Axis 3: API gateway vs direct exposure

The concrete mechanical home for what `05-scalability.md` and `06-auth.md` both deferred here.

### Direct exposure (clients connect straight to application services, no gateway layer)

- **Pros:** simplest — one less component to run and maintain; lowest latency, since there's no extra hop through a gateway
- **Cons:** coarse-grained concerns like authentication enforcement (`06-auth.md`'s Axis 3) and rate limiting (`05-scalability.md`'s Axis 4) have to be implemented in every service individually rather than centrally, risking inconsistency across services; harder to present a single, coherent external API surface when the backend is actually multiple services

### API gateway (a dedicated layer in front of services, handling cross-cutting concerns centrally)

- **Pros:** centralizes exactly the coarse-grained, cross-cutting concerns `06-auth.md` and `05-scalability.md` flagged — authentication/authorization checks, rate limiting, and request routing all happen in one place rather than being reimplemented per service; presents a single, coherent API surface to external clients regardless of how many services sit behind it
- **Cons:** another component to run, and a potential single point of failure or bottleneck if not itself made redundant (Axis 2); still needs fine-grained, resource-aware authorization at the service layer for anything the gateway can't reasonably know about — `06-auth.md`'s layered-enforcement point

An API gateway earns its cost specifically once there's more than a single deployable service (`01-system-level.md`) needing a coherent external-facing surface, or once centralizing auth/rate-limiting enforcement is worth the added component. A monolith exposing its own API directly has comparatively little need for a separate gateway layer.

---

## Axis 4: Service mesh (advanced/rare)

Relevant specifically in a microservices topology — the internal, service-to-service counterpart to Axis 3's external-facing gateway.

### No service mesh (services call each other directly)

- **Pros:** simplest — no additional infrastructure layer between services
- **Cons:** service-to-service concerns (mutual TLS, retries, circuit breaking, service-to-service auth) each have to be implemented individually per service or per client library, the same consistency risk direct exposure has at the external layer (Axis 3)

### Service mesh (a dedicated infrastructure layer handling service-to-service communication)

- **Pros:** centralizes service-to-service traffic management — mutual TLS between services (extending Axis 5's end-to-end encryption option), consistent retry/circuit-breaker behavior (reinforcing `14-resilience.md`'s patterns at the infrastructure layer rather than requiring every service to implement them individually), and fine-grained traffic routing and observability for internal calls
- **Cons:** significant operational complexity — genuinely heavyweight infrastructure to run and understand, in the same advanced/situational tier as `01-system-level.md`'s microkernel or `05-scalability.md`'s space-based architecture; only pays off at real microservices scale, with enough services that consistent cross-service policy enforcement is worth the operational cost

A service mesh should almost always be excluded from a project's shown table unless the system already has a substantial number of services and the team has real ops maturity to run one — a narrow, advanced option, not a default recommendation for "we have microservices."

---

## Axis 5: TLS termination

The concrete mechanical home for what `16-security.md` deferred here.

### TLS terminated at the load balancer / edge

- **Pros:** centralizes certificate management in one place rather than every service needing its own certificate; offloads the computational cost of TLS handshakes from application servers to the load balancer, typically optimized for it
- **Cons:** traffic between the load balancer and internal services is unencrypted by default unless deliberately re-encrypted for that internal hop too — a real gap if internal network segments (Axis 1) aren't otherwise trusted

### TLS terminated at each service (end-to-end encryption)

- **Pros:** encrypts traffic all the way to the service handling the request, not just to the edge — closes the internal-hop gap edge termination leaves open, relevant for genuinely zero-trust internal network postures or compliance requirements that specifically mandate encryption in transit at every hop, not just the perimeter
- **Cons:** every service needs its own certificate and TLS configuration to manage, multiplying the certificate-management burden; more computational overhead spread across every service rather than centralized at the edge

Edge termination is the standard default, sufficient for most projects, especially combined with a genuinely segmented, trusted internal network (Axis 1). End-to-end — often implemented via a service mesh, Axis 4 — is worth the added complexity specifically for compliance requirements mandating encryption in transit everywhere, or a security posture that doesn't trust the internal network by default.

---

## Axis 6: DNS

The concrete mechanical home for what `13-caching.md` deferred here.

### Standard DNS (provider-managed, default TTL)

- **Pros:** simplest — works out of the box with whatever a domain registrar or cloud provider offers by default
- **Cons:** default TTL settings may not be tuned for the project's actual failover/rollout needs — `13-caching.md` and `20-environment-strategy.md` both already flag that DNS TTL directly affects failover and rollout propagation speed

### Deliberately-tuned DNS (TTL and routing policy set intentionally)

- **Pros:** a short TTL enables fast failover/rollout propagation when needed (`20-environment-strategy.md`'s rollout strategies, `03-backup-disaster-recovery.md`'s failover posture) at the cost of more frequent DNS lookups; some providers also support health-check-aware or geo-based routing policies, extending load-balancing-like behavior to the DNS layer for multi-region setups
- **Cons:** more DNS configuration to get right and maintain; overly short TTLs can add meaningful lookup volume and cost at high traffic without a corresponding benefit if failover speed isn't actually a priority

Most projects can reasonably use standard DNS defaults; deliberately tuning TTL and routing policy earns its attention specifically once fast failover or multi-region routing (`14-resilience.md`'s Axis 6) is a real requirement.

---

## Axis 7: Firewalls & egress filtering

The concrete mechanical home for what `16-security.md` deferred here, specifically its SSRF mitigation note.

### Default-open (minimal firewall rules, broad outbound access)

- **Pros:** simplest — nothing blocks legitimate traffic by accident, no rules to maintain
- **Cons:** no containment against a compromised service reaching out to somewhere it shouldn't — directly relevant to `16-security.md`'s SSRF note, where a compromised or abused URL-fetching feature could reach internal metadata endpoints or arbitrary external destinations with no network-layer check stopping it

### Default-deny with explicit allowlisting (inbound and outbound rules both restrictive by default)

- **Pros:** the concrete enforcement mechanism for `16-security.md`'s SSRF mitigation — blocking requests to internal/private IP ranges and restricting egress to only known-necessary destinations closes the SSRF attack surface at the network layer, rather than relying solely on application-level input validation; also limits what a compromised service can reach or exfiltrate to, containing the blast radius of any single compromise
- **Cons:** real ongoing maintenance to keep allowlists accurate as legitimate integrations and destinations change; a too-restrictive rule can break a legitimate feature in a way that's often only caught in testing, or worse, production

Default-deny is worth the maintenance overhead specifically once a system has real security stakes, or handles a URL-fetching feature or agentic tool access (`09-ai-integration.md`, `16-security.md`'s SSRF note) that makes egress filtering a genuine mitigation rather than a theoretical one.

---

## What drives the recommendation (from upfront questions)

- **System-level shape** (from `01-system-level.md`): the primary driver for Axis 3 (API gateway) and Axis 4 (service mesh) — a monolith has little need for either; microservices strengthens the case for a gateway, with a service mesh only once there's real scale in the number of services
- **Consistency requirements / uptime expectations**: strict uptime needs push toward Axis 2's load balancer with health checks — non-negotiable for the redundancy `14-resilience.md` calls for — and toward Axis 6's deliberately-tuned DNS for fast failover
- **Ops/deployment capacity**: low ops maturity weights away from a service mesh (Axis 4), toward edge TLS termination (Axis 5) over end-to-end, toward standard DNS defaults (Axis 6) over deliberate tuning, and toward a managed load balancer offering over self-managed
- **Domain / compliance / security posture**: SSRF exposure — any URL-fetching feature or agentic tool per `09-ai-integration.md` — is the direct driver for default-deny egress filtering (Axis 7); compliance requirements mandating encryption in transit everywhere push toward end-to-end TLS (Axis 5) and toward treating network segmentation (Axis 1) as strict rather than a baseline
- **Time-to-ship pressure**: high pressure → segmented network as a baseline (cheap, high-leverage, non-negotiable per Axis 1's own guidance), a managed load balancer with health checks, direct exposure or a minimal gateway if multiple services exist, edge TLS termination, standard DNS, and default-open firewall rules unless there's a specific URL-fetching/SSRF exposure, is a legitimate first version — deferring a service mesh and end-to-end TLS until real scale or compliance demands them

---

## Recommendation table skeleton

| Category | Option A | Option B | Recommendation + why | Approve |
|---|---|---|---|---|
| Network segmentation | *Flat network pros/cons* | *Segmented (public/private subnets) pros/cons* | *segmented is close to non-negotiable beyond a small internal tool* | ☐ |
| Load balancing & traffic routing | *No load balancer pros/cons* | *Load balancer with health checks pros/cons* | *close to required for any redundant or horizontally-scaled system (`05`, `14`)* | ☐ |
| API gateway vs direct exposure | *Direct exposure pros/cons* | *API gateway pros/cons* | *justify against number of services and need to centralize auth/rate-limiting (`05`, `06`)* | ☐ |
| Service mesh | *pros/cons, flagged advanced/rare* | — | *only include with real microservices scale and ops maturity; otherwise omit this row entirely* | ☐ |
| TLS termination | *Edge termination pros/cons* | *End-to-end pros/cons* | *edge is the default; end-to-end for zero-trust or compliance-mandated encryption everywhere* | ☐ |
| DNS | *Standard defaults pros/cons* | *Deliberately-tuned TTL/routing pros/cons* | *standard is fine for most; tune once fast failover or multi-region routing is a real need* | ☐ |
| Firewalls & egress filtering | *Default-open pros/cons* | *Default-deny with allowlisting pros/cons* | *default-deny once real security stakes or SSRF exposure (URL-fetching features, `09`) exist* | ☐ |

Note: for a small, early-stage monolith under time pressure, this can reasonably collapse to three rows ("segmented network with a managed load balancer," "edge TLS termination, standard DNS," and "default-open firewall unless a URL-fetching feature exists") rather than seven — drop the API gateway and service mesh rows entirely for a monolith, and the service mesh row for anything short of real microservices scale.

---

## Related files

- `01-system-level.md` — service topology decided there shapes whether Axis 3 (API gateway) and Axis 4 (service mesh) are relevant at all
- `04-file-storage.md` — CDN/edge routing touches this file's topology, though the storage-and-delivery decision itself stays in that file
- `05-scalability.md` — horizontal scaling and gateway-level rate limiting depend on Axis 2 (load balancer) and Axis 3 (gateway) respectively
- `06-auth.md` — edge-level auth enforcement decided there is implemented via Axis 3's API gateway
- `09-ai-integration.md` — an agentic tool with URL-fetching access is the direct driver for Axis 7's egress filtering
- `13-caching.md` — DNS caching's actual treatment (Axis 6) lives here
- `14-resilience.md` — redundancy's health-check failover mechanism (Axis 2) and multi-zone traffic routing live here
- `16-security.md` — TLS termination (Axis 5), network segmentation (Axis 1), and SSRF egress filtering (Axis 7) are the mechanical enforcement of that file's encryption and access policy
- `19-deployment-infra.md` — the compute model chosen there shapes what this file's topology needs to route to and secure
- `20-environment-strategy.md` — canary's percentage-based routing and blue-green's cutover (Axis 2) both depend on this file's load balancer capability
