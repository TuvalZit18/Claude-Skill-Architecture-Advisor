# 05 — Scalability

## What this category covers

Techniques for handling growth in load — more users, more data, more traffic — once the system-level shape (`01-system-level.md`) and data storage model (`02-data-storage.md`) are already decided. This file covers the *mechanisms*: vertical vs horizontal scaling, statelessness, database read/write scaling (replicas, sharding), rate limiting/throttling, and — as an advanced/rare option — space-based architecture.

Distinct from `13-caching.md`: caching reduces the *work* a system has to do per request; scalability here is about handling more *concurrent* requests/data volume even when each one still does its full work. They compound (caching often reduces how much raw scaling is needed) but are separate decisions.

Distinct from `01-system-level.md`: that file decides the deployable-unit shape (monolith vs microservices, etc.); this file assumes that shape is settled and asks how *each* unit (or the system as a whole) scales under load.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Vertical vs horizontal scaling

### Vertical scaling (scale up)

- **Pros:** simplest to reason about — no distributed-systems complexity added; no code changes required in most cases; works for both compute and database tiers
- **Cons:** hard ceiling — a single machine's capacity is finite regardless of budget; no redundancy (one instance is a single point of failure); typically requires downtime or a failover step to resize; cost grows faster than linearly at the high end

### Horizontal scaling (scale out)

- **Pros:** effectively unbounded capacity — add more instances rather than a bigger one; redundancy comes largely for free (multiple instances survive one failing); cost tends to scale more linearly with load
- **Cons:** requires the application layer to support running multiple instances concurrently (see Axis 2 — statelessness is usually a prerequisite); needs a load balancer or equivalent traffic-distribution mechanism (`21-networking.md`); database tier doesn't horizontally scale as easily as the app tier without additional work (see Axis 3)

Most real systems do both to some degree — vertical scaling for the database tier up to a point, horizontal scaling for the stateless app tier — rather than picking one exclusively.

---

## Axis 2: Statelessness & session affinity

Whether app-tier instances can be freely added, removed, or routed to interchangeably.

### Stateless app tier

- **Pros:** prerequisite for horizontal scaling to actually work cleanly — any instance can handle any request; instances can be added/removed/replaced without losing user state; simplifies deployment (rolling deploys, autoscaling) since no instance is "special"
- **Cons:** session/user state has to live somewhere external (shared cache or DB — see `13-caching.md` and `02-data-storage.md`) rather than in-process, which is an explicit design decision, not automatic

### Stateful app tier / sticky sessions

- **Pros:** simpler initial implementation — session data lives in-process, no external session store needed; can be lower-latency for session reads (no network hop)
- **Cons:** requires session affinity at the load balancer (same user routed to the same instance), which undermines horizontal scaling's redundancy benefit — losing that instance loses in-flight session state; complicates rolling deploys and autoscaling; doesn't fit serverless/FaaS (`01-system-level.md`) at all, since function instances aren't durable

For most projects planning any horizontal scaling, stateless app tier plus an external session/cache store (Redis or equivalent) is the default rather than a genuinely close call — sticky sessions are usually a legacy pattern or a narrow fit for specific real-time/stateful-connection use cases (e.g. WebSocket servers holding open connections).

---

## Axis 3: Database scaling

The app tier scaling horizontally doesn't automatically solve database-tier scaling — the database is usually the actual bottleneck once the app tier is stateless and horizontally scaled.

### Read replicas

- **Pros:** offloads read traffic from the primary — cheap to add for read-heavy workloads (the common case); relatively simple to set up on most managed relational/document databases; primary still handles all writes, so no write-consistency complexity added
- **Cons:** replication lag means replicas can serve slightly stale data (eventual consistency for reads) — a real constraint for any read-your-own-write requirement; doesn't help write-heavy workloads at all; adds a routing decision (which queries go to replica vs primary) at the app or ORM layer

### Sharding / partitioning

- **Pros:** scales both reads and writes horizontally by splitting data across multiple database instances by some key (customer ID, region, hash range); the real answer once a single database instance's write throughput or storage is the bottleneck
- **Cons:** significant complexity — cross-shard queries/joins are hard or impossible without application-level stitching; rebalancing shards as data grows unevenly is an ongoing operational burden; choosing the shard key wrong early is expensive to undo later, similar in spirit to the service-boundary risk in `01-system-level.md`; many managed databases don't support this natively, pushing it into custom infrastructure territory

### Connection pooling

Not a horizontal-scaling technique on its own, but worth a one-line note: as app-tier instances multiply, naive per-instance database connections can exhaust the database's connection limit well before compute or query throughput is the actual bottleneck. A connection pooler (PgBouncer or equivalent, or a managed database's built-in pooling) is close to a required companion to any horizontally-scaled app tier talking to a relational database, not an optional extra.

Most projects should reach for read replicas well before sharding — sharding is justified specifically once write throughput or data volume on a single instance is the demonstrated bottleneck, not a default response to "we expect scale."

---

## Axis 4: Rate limiting & throttling

Protecting the system from being overwhelmed — whether by legitimate traffic spikes, a runaway client, or abuse — rather than scaling to absorb unlimited load.

### No rate limiting

- **Pros:** simplest — nothing to configure or tune
- **Cons:** a single misbehaving client or traffic spike can degrade service for everyone; no protection against abuse or accidental self-inflicted load (e.g. a buggy retry loop)

### Application-level rate limiting

- **Pros:** fine-grained control — per-user, per-endpoint, or per-API-key limits; can express business logic (different tiers get different limits)
- **Cons:** adds load to the app tier itself to enforce (though typically minor); needs a shared store (same cache/session store as Axis 2) to enforce consistently across horizontally-scaled instances — an in-process counter doesn't work once there's more than one instance

### Gateway/edge-level rate limiting

- **Pros:** enforced before traffic even reaches the app tier — offloads the app tier entirely and protects it from the load, not just from the consequence of the load; often available as a built-in feature of API gateways, CDNs, or load balancers
- **Cons:** coarser-grained than application-level in some implementations (less business-logic awareness); adds a dependency on whatever gateway/edge product is chosen (`21-networking.md`)

The two aren't mutually exclusive — coarse edge-level limiting to blunt large-scale abuse, plus finer application-level limiting for business-logic-aware quotas, is a common combination rather than an either/or.

---

## Axis 5: Space-based architecture (advanced/rare)

Flagged explicitly as a narrow, situational option rather than a default candidate — most projects should not reach for this.

- **What it is:** an architecture pattern built around in-memory data grids ("processing units" holding both application logic and data together) that replicate and partition state across nodes, explicitly avoiding a central database as the bottleneck — sometimes implemented via a tuple-space model.
- **Pros:** removes the database as a single point of contention entirely for extreme-scale, high-throughput scenarios (the pattern was popularized for use cases like flash-sale traffic spikes); linear scalability by adding processing units
- **Cons:** major architectural complexity and a steep departure from conventional request/database patterns; small ecosystem and talent pool; data durability requires deliberate design (in-memory-first inverts the usual database-as-source-of-truth model); overkill and likely actively harmful (unjustified complexity) below genuinely extreme scale requirements

This should almost always be excluded from a project's shown table rather than forced in — include it only when the upfront questions surface a genuinely extreme, spiky-scale requirement (e.g. flash-sale-style traffic) that read replicas/sharding/horizontal scaling alone don't comfortably address.

---

## What drives the recommendation (from upfront questions)

- **Expected scale**: the primary driver for this entire file — low/moderate expected scale means most of these axes collapse to "not needed yet" (vertical scaling, stateless-by-default, no rate limiting beyond a basic default) rather than forcing every axis into the table; high or rapidly growing expected scale pushes toward horizontal scaling, read replicas, and deliberate rate limiting
- **Traffic pattern** (steady vs spiky): spiky traffic patterns strengthen the case for horizontal autoscaling and rate limiting/throttling specifically, and are the one condition under which space-based architecture (Axis 5) might even be worth naming
- **Ops/deployment capacity**: low ops maturity weights against sharding and space-based architecture regardless of scale need, since both carry high operational burden — read replicas and vertical scaling are the lower-ops-cost lever to pull first
- **Time-to-ship pressure**: high pressure → vertical scaling plus a stateless app tier (so horizontal scaling stays *possible* later without a rewrite) is a legitimate answer; sharding and space-based architecture are almost never justified under time pressure
- **Team size / ownership boundaries**: connects back to `01-system-level.md` — a single small team on a monolith rarely needs sharding or space-based architecture; these become more plausible alongside a genuinely large-scale, multi-team system
- **Consistency requirements**: strict consistency needs push against relying heavily on read replicas for read-your-own-write paths (Axis 3's replication-lag con becomes a real constraint, not just a note) and may require routing certain reads to the primary explicitly

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| Scaling direction | *Vertical pros/cons* | *Horizontal pros/cons* | — | *pick one or a combination, justify against expected scale / ops capacity / time pressure* | ☐ |
| App-tier statelessness | *Stateless pros/cons* | *Stateful/sticky pros/cons* | — | *justify against horizontal-scaling plans; note external session store choice if stateless* | ☐ |
| Database scaling | *Read replicas pros/cons* | *Sharding pros/cons* | — | *justify against read/write ratio and demonstrated bottleneck; note connection pooling if horizontally-scaled app tier* | ☐ |
| Rate limiting | *App-level pros/cons* | *Gateway/edge-level pros/cons* | *Combination* | *justify against abuse/traffic-spike risk and available infra (gateway/CDN)* | ☐ |
| Space-based architecture | *pros/cons, flagged advanced/rare* | — | — | *only include if expected scale is genuinely extreme/spiky; otherwise omit this row entirely* | ☐ |

Note: for most projects — especially those with moderate expected scale and time-to-ship pressure — this can reasonably collapse to two rows ("vertical scaling now, stateless app tier to keep horizontal scaling available later" plus "no rate limiting beyond a basic default") rather than five. Don't force all five rows when the project's answers make most of them a non-decision, and drop the space-based-architecture row by default per Axis 5's guidance above.

---

## Related files

- `01-system-level.md` — scaling techniques here assume the deployable-unit shape decided there; a single-instance monolith vs a horizontally-scaled microservices fleet changes which of these axes are even live
- `02-data-storage.md` — database scaling (Axis 3) builds on the storage model chosen there; sharding strategy differs meaningfully between relational and document stores
- `13-caching.md` — caching reduces work per request rather than scaling capacity for more requests; the two strategies compound but are separate decisions
- `21-networking.md` — load balancers and API gateways are the mechanical layer that horizontal scaling (Axis 1) and gateway-level rate limiting (Axis 4) depend on
- `14-resilience.md` — rate limiting protects against overload; resilience patterns (retries, circuit breakers) handle failure once overload or a downstream failure has already occurred
