# 14 — Resilience

## What this category covers

Handling in-flight failure — a downstream dependency being slow or down, a network blip, an instance crashing — while the rest of the system is otherwise intact: timeouts, retries with backoff, circuit breakers, bulkhead isolation, graceful degradation, and redundancy across failure domains. This is the canonical treatment of the patterns referenced throughout this skill wherever another file mentions calling something that could fail: `08-integrations.md` (third-party calls), `09-ai-integration.md` (AI provider calls), `10-messaging.md` and `11-background-jobs.md` (retry and dead-letter mechanics), and `13-caching.md` (a cache as an external dependency in its own right).

Distinct from `03-backup-disaster-recovery.md`: that file covers already-occurred, catastrophic data loss; this file covers a request or instance failing right now while the system and its data are otherwise intact — recovering from a live, in-flight failure, not restoring lost data after the fact. Axis 6 (redundancy) does mirror that file's DR-tier reasoning, just applied to compute rather than data.

**Conditional category, with a nuance:** Axes 1-5 (handling an *external dependency's* failure) are thin or largely skippable for a simple monolith with no external calls at all — no third-party integrations, no other internal services, no distributed cache or AI provider. Flag with a one-line reason if that's genuinely the case. Axis 6 (redundancy) is different — it's about the service's *own* availability, and applies even to a dependency-free monolith as long as it has a real uptime requirement, so it shouldn't be skipped along with the rest just because there are no outbound calls to protect against.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Timeouts

### No explicit timeout (rely on the underlying library/OS default)

- **Pros:** nothing extra to configure
- **Cons:** default timeouts, where they exist at all, are often far too long — or effectively unbounded — for the caller's actual tolerance, meaning a slow dependency can hold a request (and the resources serving it: a thread, a connection) open far longer than acceptable, which can cascade into resource exhaustion under load

### Explicit, deliberately-set timeouts

- **Pros:** bounds how long a call to a dependency can hold up the caller, freeing resources promptly rather than letting them accumulate against a slow dependency; forces an explicit decision about what "too slow" means for each specific call rather than accepting a library default that may not fit
- **Cons:** setting the right value is a genuine tuning problem — too aggressive and normal-but-slightly-slow responses get needlessly cut off, a real availability cost of its own; too generous and the timeout doesn't actually protect against the failure mode it's meant to catch; different calls often need different timeouts (a fast internal cache lookup vs a slower third-party API call), so a single global value rarely fits every case well

Timeouts are close to a non-negotiable baseline for any call to an external dependency — the question is less "should we have timeouts" and more "what should each one be set to," informed by the dependency's normal latency profile plus a deliberate margin.

---

## Axis 2: Retries (with backoff and jitter)

### No retries (fail immediately on first error)

- **Pros:** simplest — fails fast, so the caller (or its own caller further up the chain) learns of the failure immediately rather than waiting through retry attempts
- **Cons:** any transient failure — a brief network blip, a momentary dependency hiccup — that would have succeeded on a second attempt is treated as a hard failure with no automatic recovery

### Retry with fixed delay

- **Pros:** recovers automatically from transient failures without the complexity of a backoff schedule
- **Cons:** a short, fixed delay retried by many callers simultaneously right after a dependency recovers can itself overwhelm the just-recovering dependency again — a "thundering herd" retrying in lockstep, not much better than no delay at all relative to how many callers are retrying together

### Retry with exponential backoff (and jitter)

- **Pros:** each successive retry waits longer than the last, giving a struggling dependency progressively more room to recover rather than facing the same retry pressure repeatedly; adding jitter — randomizing the exact delay within a range — spreads retries from many simultaneous callers apart so they don't all retry at the exact same moments, which is what actually prevents the thundering-herd problem that backoff alone doesn't fully solve
- **Cons:** more complex to implement than a fixed delay; a request retried multiple times with increasing delays means the caller's own caller waits longer before learning of an eventual failure, which needs to be bounded (a maximum retry count or total time budget) rather than backing off indefinitely

**Idempotency requirement:** any retry strategy beyond "no retries" requires the operation being retried to be idempotent — the same request executed twice, because the first attempt's response was lost rather than because it actually failed, must not have a different effect than executing it once. This is not optional once retries are in play; without it, retries can cause real harm (double-charging a payment, sending a duplicate email) rather than just extra load. The same requirement noted in `10-messaging.md` and `11-background-jobs.md` applies here on the calling side too, and is usually implemented via an idempotency key the receiver can use to detect and ignore a duplicate.

Exponential backoff with jitter and a bounded maximum is the standard default for any retried operation; fixed-delay retries are reasonable only for very low-volume, low-concurrency scenarios where thundering-herd risk is genuinely negligible.

---

## Axis 3: Circuit breakers

### No circuit breaker (always attempt the call, rely on timeouts/retries alone)

- **Pros:** simplest — no additional state to track
- **Cons:** a dependency that's confirmed down still gets hit by every caller's full timeout-and-retry cycle on every single request, wasting caller resources and adding load to a dependency that's already struggling to recover — the caller learns nothing from repeated identical failures and keeps paying the same cost for each one

### Circuit breaker (stops attempting calls after repeated failures, periodically tests recovery)

- **Pros:** once a failure threshold is crossed, the breaker "opens" and further calls fail fast locally without even attempting the dependency, protecting both the caller's own resources and the struggling dependency from continued load; periodically allows a small number of test requests through (a "half-open" state) to detect recovery automatically, closing the breaker again once the dependency is confirmed healthy
- **Cons:** real additional complexity — failure-threshold and recovery-test tuning, and a miscalibrated breaker (too sensitive) can open on transient blips that a plain retry would have handled fine, unnecessarily degrading availability; adds a piece of shared state (the breaker's current status) that needs to be visible and consistent across a horizontally-scaled caller fleet (`05-scalability.md`) to be fully effective, echoing that file's shared-store requirement for rate limiting

Circuit breakers earn their complexity specifically once a dependency's failures are frequent or sustained enough that blind retry-on-every-request is itself a meaningful cost, to either the caller's resources or the struggling dependency. For infrequent, genuinely transient failures, timeouts plus backoff-retry alone are often sufficient without a full circuit breaker.

---

## Axis 4: Bulkhead isolation

### Shared resource pool across all dependencies

- **Pros:** simplest — one pool of threads, connections, or capacity serves calls to every dependency
- **Cons:** a single slow or failing dependency can consume the entire shared pool — every available thread or connection tied up waiting on the struggling dependency — starving calls to *other*, perfectly healthy dependencies of any capacity at all. A failure in one place cascades into an outage for unrelated functionality that had nothing to do with the actual failure

### Bulkheads (isolated resource pools per dependency or dependency class)

- **Pros:** caps how much of the caller's total capacity any single dependency can consume, containing a failure to just the calls that actually depend on the failing thing rather than letting it starve unrelated functionality — the same blast-radius-containment principle as separate deployable services in `01-system-level.md`, applied to resource pools within a single service instead
- **Cons:** real additional complexity in sizing and managing multiple pools instead of one; a pool sized too small for its dependency's actual healthy-state demand becomes an artificial bottleneck even when nothing is failing

Bulkheads matter most when a service depends on multiple external things with meaningfully different reliability profiles or criticality — a service with only one external dependency has less need for isolation, since there's nothing else left to protect from that dependency's failure.

---

## Axis 5: Graceful degradation & fallback

### Fail the whole request when a dependency is unavailable

- **Pros:** simplest — a dependency failure directly and honestly becomes a request failure, with no additional fallback logic to design or maintain
- **Cons:** often fails more of the request than strictly necessary — a page that draws on data from three dependencies to render fully doesn't necessarily need to fail completely just because one, non-essential dependency is down

### Graceful degradation (serve a reduced but still-useful response)

- **Pros:** a request can still largely succeed even when a non-critical dependency is unavailable — render a page without personalized recommendations rather than failing the whole page load, or serve a cached/stale value (`13-caching.md`) rather than no value at all; meaningfully better perceived availability from the user's perspective than an all-or-nothing failure
- **Cons:** requires deliberately identifying which dependencies are essential (must fail the request) versus non-essential (can be degraded around) for every feature, and building distinct fallback behavior for each — not automatic, and easy to under-invest in for features where "just fail" was the path of least resistance

Whether a given dependency is essential or degradable is a genuine per-feature judgment call, not a system-wide default — a payment dependency failing should probably fail the checkout request outright; a recommendations dependency failing on a product page probably shouldn't take the whole page down with it.

---

## Axis 6: Redundancy & failure domains

Distinct from Axis 4's bulkheads, which isolate different *dependencies* from each other within a service: this axis is about running enough *instances* of a thing, spread across enough *failure domains*, that the failure of any single instance or domain doesn't take the whole service down. Unlike Axes 1-5, which are about handling an *external dependency's* failure, this axis is about the service's *own* availability — it applies even to a monolith with zero outbound calls, as long as that monolith has a real uptime requirement.

### Single instance (no redundancy)

- **Pros:** simplest — one thing to run, deploy, and reason about, at the lowest cost
- **Cons:** a single point of failure — that instance failing (a crash, a host failure, a deploy gone wrong) takes the entire service down with it; no spare capacity to absorb even routine maintenance (a restart, a rolling deploy) without a service interruption

### Multiple instances, single failure domain (e.g. same availability zone)

- **Pros:** survives a single instance failure — if one instance crashes, the others continue serving traffic; enables rolling deploys and routine maintenance without full downtime
- **Cons:** doesn't survive failure of the shared failure domain itself (an AZ-level outage, a rack failure) — every instance goes down together despite being individually redundant, the same single-AZ-vs-multi-AZ distinction already drawn in `03-backup-disaster-recovery.md`'s Axis 4 for data, applied here to compute

### Multiple instances spread across failure domains (multi-AZ, or multi-region for the most critical services)

- **Pros:** survives not just an individual instance failure but the failure of an entire zone or region — the strongest protection against infrastructure-level failure short of full active-active multi-region, the same DR-tier spectrum `03-backup-disaster-recovery.md` covers for data, applied here to compute
- **Cons:** cross-zone and especially cross-region traffic and coordination add real latency and cost; requires the traffic-routing layer to actually be aware of and route around a failed zone, which is a networking-layer capability (`21-networking.md`) this axis assumes rather than implements

**Health-check-based failover:** the mechanism that actually makes redundancy useful in practice — a load balancer or service mesh needs to detect an unhealthy instance (via a health-check endpoint) and stop routing traffic to it, automatically and quickly, or redundant instances sitting idle don't actually protect availability during a live failure. This is the resilience-relevant *use* of load balancing; the load balancer's own configuration and topology is `21-networking.md`'s decision, not this file's — this file only flags that redundancy is only as effective as the failover mechanism that detects and routes around a failed instance.

Redundancy is close to a baseline expectation for any service with a real uptime requirement, in the same way timeouts are a baseline for any external call (Axis 1) — the open questions are how many instances, and across how many failure domains, which scales with the criticality and consistency requirements of the specific service rather than having a one-size-fits-all answer.

---

## What drives the recommendation (from upfront questions)

- **System-level shape / domain coupling**: services with more external dependencies — third-party integrations (`08-integrations.md`), AI providers (`09-ai-integration.md`), other internal services (`10-messaging.md`), a distributed cache (`13-caching.md`) — have proportionally more to gain from investing in this file's patterns; a monolith with no outbound calls has essentially nothing to decide here
- **Consistency requirements**: operations with irreversible side effects (payments, anything that can't be safely duplicated) raise the stakes on Axis 2's idempotency requirement specifically, since a retry-without-idempotency bug here is a correctness bug, not just a resilience gap
- **Ops/deployment capacity**: low ops maturity weights toward the simpler end of each axis — timeouts and basic retry-with-backoff are cheap enough to always be worth doing, while circuit breakers and bulkheads are the more expensive tier, best reserved for dependencies whose failure patterns have actually demonstrated the need
- **Time-to-ship pressure**: high pressure → timeouts and retry-with-backoff on every external call (both cheap, high-value defaults), deferring circuit breakers, bulkheads, and deliberate graceful-degradation design until a specific dependency's failure history justifies the added investment, is a legitimate first version
- **Domain/criticality**: which specific dependencies are essential versus degradable (Axis 5) is inherently domain-specific and needs to be asked about per feature rather than assumed uniformly across the system
- **Consistency/uptime requirements and expected scale**: the direct drivers for Axis 6 — a service with a real uptime commitment needs redundancy regardless of whether it has any external dependencies at all; low-stakes internal tools or early-stage projects can reasonably start with a single instance and add redundancy once uptime actually matters, while services spanning multiple availability zones or regions should do so proportional to how costly downtime for that specific service actually is

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| Timeouts | *No explicit timeout pros/cons* | *Explicit timeouts pros/cons* | — | *close to non-negotiable; state per-dependency values informed by normal latency profile* | ☐ |
| Retries | *No retries pros/cons* | *Fixed-delay retry pros/cons* | *Exponential backoff + jitter pros/cons* | *exponential backoff + jitter is the default; note the idempotency requirement explicitly* | ☐ |
| Circuit breakers | *No circuit breaker pros/cons* | *Circuit breaker pros/cons* | — | *justify against failure frequency/duration per dependency; only where blind retry is itself costly* | ☐ |
| Bulkhead isolation | *Shared pool pros/cons* | *Isolated pools pros/cons* | — | *justify against number and criticality-variance of dependencies; only relevant with multiple dependencies of differing reliability* | ☐ |
| Graceful degradation | *Fail whole request pros/cons* | *Degrade gracefully pros/cons* | — | *state which dependencies are essential vs degradable per feature, not system-wide* | ☐ |
| Redundancy & failure domains | *Single instance pros/cons* | *Multi-instance, single failure domain pros/cons* | *Multi-instance, multi-failure-domain pros/cons* | *scale with the service's actual uptime requirement; note health-check failover as a required companion, mechanics live in `21-networking.md`* | ☐ |

Note: for a monolith with no outbound calls, Axes 1-5 collapse to a one-line note ("no external dependencies — resilience patterns not applicable"), but Axis 6 (redundancy) still needs its own answer based on the service's uptime requirement, independent of that collapse. For a project with a small number of external dependencies under time pressure, this can reasonably collapse to three rows ("timeouts plus exponential-backoff retry on every external call," "fail the request for essential dependencies, degrade gracefully for the rest," and "single instance for now, redundancy once uptime requirements demand it") rather than six — defer circuit breakers and bulkheads until a specific dependency's failure pattern demonstrates the need.

---

## Related files

- `01-system-level.md` — bulkhead isolation (Axis 4) applies the same blast-radius-containment principle within a single service that separate deployable services apply at the system level
- `05-scalability.md` — a circuit breaker's shared state (Axis 3) needs the same kind of shared store as that file's rate limiting, to work correctly across a horizontally-scaled caller fleet
- `08-integrations.md` — flags which third-party integrations warrant this file's patterns; this file holds the full mechanics
- `09-ai-integration.md` — AI provider calls are a third-party integration and benefit from the same timeout/retry/circuit-breaker treatment
- `10-messaging.md` — retry-with-backoff and dead-letter handling for async messaging build on this file's retry patterns
- `11-background-jobs.md` — job retry-with-backoff reuses this file's Axis 2 reasoning, scoped to a single service's own job processing
- `13-caching.md` — a distributed cache is itself an external dependency subject to this file's timeout/circuit-breaker reasoning if it becomes slow or unavailable
- `03-backup-disaster-recovery.md` — Axis 6's redundancy/failure-domain reasoning mirrors that file's DR-tier spectrum and single-AZ-vs-multi-AZ distinction, applied to compute instead of data
- `21-networking.md` — load balancer configuration, health-check routing, and multi-zone traffic topology are the mechanical implementation of Axis 6's redundancy; this file only covers why and how much redundancy a service needs
