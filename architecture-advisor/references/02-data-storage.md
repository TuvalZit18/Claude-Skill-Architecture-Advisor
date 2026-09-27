# 02 — Data Storage

Covers the primary data storage decision for a system: what storage *model* to use, whether to build on a managed backend-as-a-service platform or a custom backend, and whether CQRS is warranted as a data-access pattern.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Storage model

The core question: what shape does the data naturally take, and how strict are the consistency/relationship requirements?

### Relational (SQL)

- **Pros:**
  - Strong consistency guarantees (ACID transactions) out of the box
  - Mature tooling: migrations, ORMs, query optimizers, decades of operational knowledge
  - Enforces data integrity at the schema level (foreign keys, constraints, types)
  - Joins make relational queries across entities cheap to express and reason about
  - Widely known — lowest onboarding cost for most teams
- **Cons:**
  - Schema changes require migrations; can be friction-heavy on fast-iterating projects
  - Horizontal scaling (sharding) is harder than document/KV stores; vertical scaling has a ceiling
  - Rigid schema is a poor fit for highly variable or nested data shapes
  - Can become a bottleneck under very high write throughput without additional patterns (see Scalability, `05-scalability.md`)

### Document (NoSQL document stores — e.g. MongoDB, Firestore)

- **Pros:**
  - Flexible schema — good fit for evolving or heterogeneous data shapes
  - Natural fit for nested/hierarchical data without join overhead
  - Generally easier horizontal scaling than relational
  - Fast to iterate on early in a project when the data model isn't settled
- **Cons:**
  - Weaker consistency guarantees by default (though many now offer tunable consistency)
  - Cross-entity queries (the equivalent of joins) are often denormalized-data or application-level, pushing complexity into the app layer
  - Lack of enforced schema can lead to data-integrity drift over time without discipline
  - Less mature tooling for complex reporting/analytics compared to SQL

### Key-value (KV — e.g. Redis, DynamoDB)

- **Pros:**
  - Extremely fast reads/writes for simple lookup-by-key access patterns
  - Scales horizontally very well; often the backbone of high-throughput systems
  - Simple mental model — minimal schema design needed
- **Cons:**
  - Poor fit as a *primary* store when the access pattern isn't lookup-by-key (no native support for complex queries, filtering, or relationships)
  - Usually paired with another store rather than replacing one entirely (e.g. cache + relational, not KV alone)
  - Data modeling has to anticipate every access pattern in advance — changing access patterns later can require redesigning keys

### Graph (e.g. Neo4j)

- **Pros:**
  - Purpose-built for traversing relationships — queries like "friends of friends" or multi-hop connections are cheap and natural, where relational joins get expensive fast
  - Relationship data (edges) is a first-class citizen with its own properties, not just a foreign key
  - Strong fit for domains where the *connections* between entities are the primary thing being queried (social graphs, recommendation engines, fraud/anomaly detection, knowledge graphs)
- **Cons:**
  - Poor fit outside relationship-heavy access patterns — overkill and unfamiliar for typical CRUD data
  - Smaller talent pool and tooling ecosystem than relational/document
  - Horizontal scaling is harder than document/KV stores due to the interconnected nature of the data

### Wide-column / columnar (e.g. Cassandra, BigQuery)

- **Pros:**
  - Built for very high write throughput and horizontal scale across many nodes
  - Columnar layout makes large analytical scans (aggregations over huge datasets) fast
  - Good fit for write-heavy, append-mostly workloads (logs, metrics, large-scale event data) or analytics-at-scale
- **Cons:**
  - Weak or no support for ad-hoc relational queries/joins — access patterns need to be known upfront and modeled into the schema
  - Operationally heavier to run and tune than relational/document at small-to-medium scale
  - Overkill below a certain data volume — the complexity isn't justified until scale actually demands it

### Time-series (e.g. InfluxDB, TimescaleDB)

- **Pros:**
  - Purpose-built for timestamped data — efficient storage, compression, and querying of metrics/events over time
  - Native support for time-bucketed aggregations, downsampling, and retention policies that a general-purpose store would need to hand-roll
  - Strong fit for monitoring, IoT sensor data, analytics dashboards, or any domain dominated by "value over time" queries
- **Cons:**
  - Narrow fit — poor choice as a general-purpose primary store for non-time-series data
  - Usually a secondary/specialized store alongside a relational or document primary store, not a replacement for one
  - Smaller ecosystem and talent pool than relational/document

### Note: search-oriented stores (e.g. Elasticsearch)

Not typically a primary data store, but worth flagging when full-text search or relevance ranking is a dominant access pattern — occasionally used as a secondary store synced from the primary, and in rarer cases as the primary store itself when search is genuinely the core of the product. Distinct from the options above in that its main strength (text relevance/ranking) doesn't overlap with any of them.

---

## Axis 2: Managed backend-as-a-service vs custom backend

Independent of storage model — this is about *who owns the backend*, not what shape the data takes. (Supabase, for instance, is a managed platform sitting on top of a relational/Postgres model; Firebase's Firestore is a managed platform on a document model.)

### Managed backend-as-a-service (e.g. Supabase, Firebase)

- **Pros:**
  - Drastically faster time-to-ship — auth, storage, realtime, and DB come bundled
  - Lower ops burden — no infra to provision or maintain for the core data layer
  - Often generous free tiers, good for early-stage or small-team projects
  - Realtime subscriptions and row-level security (in Supabase's case) available out of the box
- **Cons:**
  - Vendor lock-in — migrating off later can be costly, especially with platform-specific features (RLS policies, realtime channels, generated APIs)
  - Less control over performance tuning, indexing strategy, and infra-level optimization
  - Cost can scale unpredictably at high usage compared to self-managed infra
  - Team ownership boundaries (Conway's Law) can get blurry — a BaaS often becomes a shared dependency across otherwise-independent teams

### Custom backend

- **Pros:**
  - Full control over data layer, performance tuning, and infra choices
  - No vendor lock-in — portable across cloud providers
  - Scales predictably with deliberate infra investment
  - Fits naturally into a microservices/team-boundary setup where each service owns its own store
- **Cons:**
  - Significantly higher build and ops burden — auth, realtime, backups, scaling are all DIY or require assembling separate tools
  - Slower time-to-ship, especially early on
  - Requires more in-house ops/infra expertise

---

## Axis 3: CQRS (Command Query Responsibility Segregation)

A data-*access* pattern, orthogonal to storage model choice — CQRS can be applied on top of relational, document, or a mix.

- **What it is:** separating the model (and often the datastore) used for writes (commands) from the model used for reads (queries), rather than using one unified model for both.
- **Pros:**
  - Read and write sides can be scaled and optimized independently (e.g. denormalized, heavily-indexed read store vs a normalized write store)
  - Read models can be tailored precisely to the queries the app actually needs, avoiding complex joins at query time
  - Pairs naturally with event sourcing for full audit history and rebuildable read models
  - Well suited to systems with a large asymmetry between read and write volume
- **Cons:**
  - Adds real architectural complexity — two models to maintain, plus a sync mechanism between them
  - Eventual consistency between write and read sides is common, which not every domain can tolerate
  - Overkill for most CRUD-shaped applications — the complexity cost usually isn't justified unless read/write patterns are genuinely asymmetric or the domain is event-sourced already
- **Cross-reference:** CQRS often pairs with event sourcing and affects API shape (queries and commands may be exposed as distinct API surfaces). See `07-api-design.md` for how this interacts with REST/GraphQL/RPC choice.

---

## Summary of decision shape for the skill

At generation time, the skill should surface up to three related-but-separate decisions from this file, only as relevant to the project:

1. **Storage model** — relational vs document vs KV vs graph vs wide-column vs time-series (or a mix); search-oriented stores flagged as a secondary-store note rather than a default candidate
2. **Platform** — managed BaaS vs custom backend
3. **Access pattern** — whether CQRS is warranted, or standard CRUD access is sufficient

These are not mutually exclusive combinations — e.g. "Supabase (managed, relational) with no CQRS" is a coherent choice, as is "custom backend, document store, with CQRS for a high-read-asymmetry domain."

---

## Related files

- `01-system-level.md` — backend shape (monolith/microservices/etc.) informs BaaS-vs-custom trade-offs
- `03-backup-disaster-recovery.md` — backup/PITR posture narrows if this file chose a managed BaaS
- `04-file-storage.md` — structured data here, blob/media assets there
- `05-scalability.md` — read replicas/sharding build on the storage model chosen here
- `07-api-design.md` — CQRS often pairs with event sourcing and shapes the API surface
