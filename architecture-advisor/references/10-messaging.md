# 10 — Messaging

## What this category covers

Asynchronous, service-to-service (or component-to-component) communication: message brokers, event-driven vs request-response communication style, queue vs pub-sub delivery patterns, how events are produced (application-level vs Change Data Capture), broker properties including replayability, and event sourcing as an advanced persistence pattern that builds on this infrastructure. This is about how independently deployable units talk to each other without a direct synchronous call.

**Conditional category:** this file is only relevant once the system-level decision in `01-system-level.md` produced multiple deployable units (microservices, or a monolith plus separate workers/services) with a genuine need for async or event-driven communication between them. A single deployable monolith with no separate services to communicate with has nothing for this file to decide — skip it with a one-line reason in the generated table rather than working through the axes below. Note this is distinct from `11-background-jobs.md`, which covers a single service scheduling its own delayed/internal work — messaging here is specifically about communication *between* separate services or components.

**Scope note:** consistent with the skill's general avoidance of vendor/product selection (see `SKILL.md`'s excluded-categories list, and the same treatment `09-ai-integration.md` gives specific AI provider/model choice), this file does not choose between specific broker products. Kafka, RabbitMQ, AWS SNS/SQS, Google Pub/Sub, and similar are mentioned only as brief illustrative examples of the properties in Axis 4, never as options to weigh against each other in a table — product choice in this space moves faster than a project-agnostic reference file can track, and the decision that actually matters architecturally is which *properties* (delivery/ordering/durability guarantees, replayability, throughput characteristics) the project's communication needs require. Note also that **BullMQ** specifically is a Redis-backed job-queue library most commonly used for a single service's own internal background work rather than cross-service messaging — it fits more naturally as an illustrative example in `11-background-jobs.md` than here, per this file's distinction from that one above.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Communication style — request-response vs event-driven

### Request-response (synchronous or async-over-HTTP/RPC)

- **Pros:** simplest to reason about — the caller knows immediately (or via a direct async response) whether the call succeeded, and gets a direct result back; naturally fits the API paradigms already covered in `07-api-design.md`; easiest to debug, since the causal chain (A called B, B responded) is explicit and traceable
- **Cons:** couples the caller to the callee's availability — if the downstream service is slow or down, the caller is directly affected (this is exactly the resilience concern in `14-resilience.md`); doesn't scale well for one-to-many notification patterns, where many services need to react to the same event, since the caller would have to know and call each one individually

### Event-driven (services publish events; other services react independently)

- **Pros:** decouples producers from consumers — a service publishes "order created" without needing to know or care which services react to it, allowing new consumers to be added later without changing the producer; naturally fits one-to-many patterns (multiple services reacting to the same event) far better than request-response; producer isn't blocked waiting on consumers, and a consumer being slow or down doesn't directly affect the producer
- **Cons:** the causal chain is implicit rather than explicit — tracing "what happened as a result of this event" across multiple independently-reacting consumers is harder to debug and reason about than a direct call chain, and needs deliberate tracing/observability investment (`15-observability.md`) to stay manageable; eventual consistency becomes the norm rather than the exception, since consumers react on their own time; failure handling is less obvious — what happens if a consumer fails to process an event needs an explicit answer (retry, dead-letter queue) rather than the caller simply seeing an error

Request-response remains the right default for most direct service-to-service calls, especially where the caller genuinely needs an immediate result. Event-driven earns its complexity specifically for one-to-many notification patterns, workflows that should tolerate eventual consistency, or decoupling producers from a consumer set that's expected to grow over time — not as a wholesale replacement for direct calls.

---

## Axis 2: Delivery pattern — queue vs pub-sub

Relevant once some form of asynchronous messaging (rather than direct request-response) is in play.

### Queue (point-to-point, one consumer processes each message)

- **Pros:** natural fit for work distribution — multiple consumer instances can pull from the same queue, giving load-balanced, horizontally-scalable processing of a stream of work items (a queue is often the mechanism behind `11-background-jobs.md`'s async job processing too, though that file covers a service's own internal work rather than cross-service messaging); a message is processed exactly once (or at-least-once with idempotency handling), which fits workflows where duplicate processing would be a real problem; buffers and smooths bursty producer traffic so consumers can process at their own sustainable rate rather than being overwhelmed by a spike — a queue absorbs the burst and lets consumers drain it over time instead of the spike hitting consumers directly
- **Cons:** not a fit for one-to-many broadcast — once a message is consumed by one worker, it's gone, so this pattern doesn't naturally support multiple independent services all needing to react to the same message

### Pub-sub (broadcast, every subscriber receives each message)

- **Pros:** the natural fit for the event-driven, one-to-many pattern from Axis 1 — a publisher emits an event once, and every interested subscriber receives its own copy independently; new subscribers can be added later without the publisher or existing subscribers needing to change
- **Cons:** doesn't inherently give work-distribution/load-balancing across multiple instances of the *same* consumer the way a queue does (though many pub-sub systems support consumer groups that combine both patterns — worth flagging as a hybrid rather than a strict either/or); message ordering and delivery guarantees vary meaningfully by specific broker/technology, which matters for anything sensitive to processing order

Many real messaging systems (Kafka, SNS+SQS combinations, RabbitMQ with the right exchange type) support both patterns, or a hybrid (pub-sub with consumer groups that load-balance within each subscribing group) — the decision is less "pick a technology" and more "which delivery semantics does the actual communication pattern need," which the specific broker is then chosen to support.

---

## Axis 3: Event production mechanism — application-level events vs Change Data Capture (CDC)

Once event-driven communication (Axis 1) is in play, a separate question is *how* events actually get produced in the first place.

### Application-level events (the service explicitly publishes an event when something happens)

- **Pros:** the event's meaning and shape are deliberately designed by the application — an event like "OrderShipped" carries exactly the business-meaningful fields the producer intends, not raw table columns; the producer controls exactly when and what gets published, so business logic (e.g. only publish once a transaction fully commits) is naturally enforced; no dependency on database internals or a CDC connector's compatibility with a specific database engine
- **Cons:** requires deliberate application code to publish events — easy to forget for a given code path, or to publish inconsistently across services without shared discipline or a common library; if a write bypasses the application code entirely (a manual fix, a batch job, another system writing to the same database), no event gets published and downstream consumers silently miss it

### Change Data Capture (CDC) — capturing changes at the database level and streaming them out automatically

- **Pros:** requires no application code changes — reads the database's write-ahead log or binlog directly (e.g. via Debezium or a cloud provider's native CDC offering), so every change is captured regardless of what wrote it, including direct writes, batch jobs, or legacy code paths that application-level events would miss entirely; decouples event production from the application, so producer services carry no event-publishing responsibility or risk of inconsistent implementation
- **Cons:** captured changes are raw database-level row changes, not business-meaningful domain events — consumers often have to interpret or reconstruct business meaning from column-level diffs rather than receiving an intentionally-designed event; tightly coupled to the specific database engine and its replication/logging mechanism, and to the CDC tool's support for it; a database schema change can break downstream consumers in ways a deliberately-designed application-level event contract wouldn't

Most systems building genuinely new event-driven functionality default to application-level events, since the event shape can be intentional and business-meaningful. CDC earns its place either as a bridge for legacy systems or databases that can't easily be modified to publish events themselves, or for data-platform use cases (search index sync, analytics ingestion, cache invalidation, data pipelines feeding a warehouse) where capturing every change accurately matters more than a hand-designed event contract.

---

## Axis 4: Message broker considerations

Not a set of mutually exclusive options so much as the properties that actually differentiate broker choices once Axes 1-3 have settled on the communication style, delivery pattern, and event production mechanism needed. Worth naming explicitly rather than leaving as an implicit assumption:

- **Delivery guarantees:** at-most-once (fastest, can lose messages), at-least-once (most common, requires idempotent consumers since duplicates are possible), exactly-once (strongest guarantee, hardest and most expensive to actually achieve end-to-end)
- **Ordering guarantees:** whether messages are guaranteed to be processed in the order they were published (often only within a partition/shard, not globally) — matters for event sequences where order changes meaning (e.g. "created" then "deleted" vs the reverse)
- **Durability/persistence:** whether messages survive a broker restart or are held only in memory — affects what happens to in-flight messages during an outage
- **Log retention & replayability:** whether the broker retains a durable, replayable history of past messages that consumers can rewind and reprocess from any retained offset (a log-based broker like Kafka), versus only holding a message until it's consumed and then discarding it (a traditional queue). Distinct from the durability point above — that's about not losing in-flight messages during an outage; this is about deliberately retaining history for reuse. This is the property that enables recovering from a consumer bug by reprocessing, rebuilding a downstream system's state from scratch, validating new business logic against real historical events, or backing event sourcing (below) — a traditional queue that discards messages after consumption can't support any of this
- **Throughput and latency characteristics:** brokers differ meaningfully here (e.g. a log-based broker built for high-throughput event streaming vs a traditional message queue optimized for lower-latency task distribution)

These are the properties that should drive which broker technology gets recommended, per the scope note above — not brand-name preference.

---

## Axis 5: Event sourcing (advanced/rare, persistence pattern)

Distinct from Axis 1's event-driven *communication* style: event sourcing is about *persistence* — storing every state change to an entity as an immutable sequence of events, and deriving the entity's current state by replaying or folding those events, rather than storing only the current state as a mutable row. It depends on the log-retention property from Axis 4, which is why it's treated here rather than only in `02-data-storage.md` — that file already flags that CQRS "pairs naturally with event sourcing for full audit history and rebuildable read models" without going deep on the pattern itself.

- **Pros:** gives a complete, immutable audit trail of every change by construction — nothing is ever overwritten or lost, valuable for compliance-heavy or audit-sensitive domains; enables reconstructing state at any point in time, or building an entirely new read-model projection from the event log if requirements change later, without having lost any historical information; pairs naturally with CQRS (`02-data-storage.md`) — the event log becomes the write side's source of truth, with one or more read-optimized projections derived from it
- **Cons:** significant architectural complexity and a real departure from familiar CRUD persistence — reading "current state" always requires either replaying events or maintaining a derived read-model projection, never a direct row read; schema evolution of the events themselves is a genuinely hard problem, since old events already in the log must remain interpretable as the event shape evolves over time; overkill for the large majority of CRUD-shaped domains where a full change history isn't an actual business requirement — the complexity cost should be justified by an explicit need (audit/compliance, complex domain logic that benefits from replay, a demonstrated need to rebuild projections), not adopted by default just because the system already uses events for communication

Should be flagged as advanced/rare and generally omitted from a project's shown table by default, the same treatment as multi-agent orchestration (`09-ai-integration.md`), space-based architecture (`05-scalability.md`), and microkernel (`01-system-level.md`) — include it only when a real, demonstrated need for full state history or event-driven state reconstruction exists.

**Stream processing (advanced consumption pattern):** worth a one-line note rather than a full axis — beyond a simple consumer reacting to individual messages one at a time, some workloads need continuous processing *across* a stream (filtering, enriching, joining multiple streams, windowed aggregation) using a dedicated stream-processing framework (e.g. Kafka Streams, Flink, Spark Streaming) rather than plain consumer code. This is justified specifically by genuine streaming-analytics or real-time-aggregation needs (e.g. computing rolling metrics from a high-volume event stream); most consumers processing one event at a time with ordinary application code don't need a dedicated stream-processing framework, and reaching for one prematurely adds real operational and conceptual complexity for no benefit.

**Common downstream use cases, briefly noted rather than expanded into their own axes:** several recognizable patterns are applications of the axes above rather than distinct architectural decisions — user activity tracking (clicks, searches, page views) is a straightforward pub-sub use case (Axis 2); centralized log aggregation and metrics/monitoring pipelines are more properly `15-observability.md`'s domain, though they often reuse the same broker/streaming infrastructure decided here as their transport layer; data pipelines feeding a data lake or warehouse are typically CDC or application-level events (Axis 3) flowing into an analytics platform, an application of this file's primitives rather than a new one. None of these need their own table row — they're worth naming as concrete motivating examples for why a project needs event-driven messaging at all, not additional decisions to make once the axes above are settled.

---

## Dead-letter handling & idempotency

Worth a one-line note rather than a full axis: any asynchronous messaging system needs an answer for what happens when a message can't be successfully processed after retries (a dead-letter queue or equivalent, so failed messages are visible and recoverable rather than silently dropped or endlessly retried), and consumers need to handle at-least-once delivery's possibility of duplicate messages (idempotent processing — the same message arriving twice shouldn't have a different effect than arriving once). Both are close to required companions to any real async messaging setup rather than optional refinements, and connect directly to `14-resilience.md`'s retry/failure-handling patterns.

---

## What drives the recommendation (from upfront questions)

- **System-level shape** (from `01-system-level.md`): the primary gate for whether this file is relevant at all — a single deployable monolith has no messaging decision to make; multiple services is the prerequisite condition, and the more of them there are with genuine cross-service communication needs, the more this category matters
- **Domain coupling**: workflows that are naturally one-to-many (many services need to react to the same business event) push toward event-driven/pub-sub; workflows that are naturally point-to-point with an immediate-result need push toward request-response
- **Consistency requirements**: strict consistency needs push against event-driven's inherent eventual consistency — a workflow that truly cannot tolerate consumers being out of sync even briefly should lean toward request-response or, if async is unavoidable, toward patterns that make the eventual-consistency window explicit and bounded
- **Ops/deployment capacity**: low ops maturity weights against standing up and operating a message broker at all — request-response (reusing whatever's already needed for the API layer) is the lower-ops-cost default, and a simpler managed queue (e.g. a cloud provider's managed queue service) is a lower-ops-cost step up from self-hosting a broker like Kafka
- **Time-to-ship pressure**: high pressure → request-response by default, introducing messaging only for the specific interactions that genuinely need decoupling or one-to-many notification, is a legitimate answer, not a gap
- **Expected scale**: high-volume event streams (analytics, activity feeds, high-throughput data pipelines) are the case where a log-based, high-throughput broker's added operational complexity actually pays for itself; moderate scale rarely justifies that complexity over a simpler managed queue or pub-sub service
- **Legacy/existing-system constraints**: an existing database or legacy service that can't easily be modified to publish its own events is the direct driver for CDC (Axis 3) over application-level events; a greenfield service with no such constraint should default to application-level events for the intentional, business-meaningful event shape they provide
- **Domain/compliance context and audit needs**: a hard requirement for a complete, immutable audit trail of every state change (common in finance, healthcare, or other regulated domains) is close to the only strong driver toward event sourcing (Axis 5) — absent that, standard CRUD persistence (`02-data-storage.md`) remains the default; per the skill's excluded-categories note, don't assume a compliance driver applies without asking

---

## Recommendation table skeleton

| Category | Option A | Option B | Recommendation + why | Approve |
|---|---|---|---|---|
| Communication style | *Request-response pros/cons* | *Event-driven pros/cons* | *pick one per interaction type if they differ; justify against one-to-many needs, consistency requirements, and coupling* | ☐ |
| Delivery pattern | *Queue pros/cons* | *Pub-sub pros/cons (note hybrid/consumer-group support)* | *justify against whether the interaction is work-distribution or broadcast; only relevant if Axis 1 chose event-driven/async* | ☐ |
| Event production mechanism | *Application-level events pros/cons* | *CDC pros/cons* | *justify against legacy/existing-system constraints; application-level events are the default for new event-driven work* | ☐ |
| Broker properties | *state required delivery/ordering/durability/replayability guarantees* | — | *name the properties needed (not a specific product); note dead-letter handling and idempotency as required companions* | ☐ |
| Event sourcing | *pros/cons, flagged advanced/rare* | — | *only include if a real audit/compliance or state-reconstruction need is demonstrated; otherwise omit this row entirely* | ☐ |

Note: if `01-system-level.md` settled on a single deployable monolith, this entire file collapses to a one-line note ("no cross-service messaging need — single deployable unit") rather than a table. For projects with multiple services but only occasional, simple async needs, this can also reasonably collapse to two rows ("request-response as the default, a managed queue with application-level events for the specific interactions that need decoupling") rather than five — drop the event-sourcing row by default per its own guidance above, and collapse event production mechanism into a stated assumption ("application-level events") when no legacy-system driver is present.

---

## Related files

- `01-system-level.md` — the gating decision for whether this file applies at all; number and boundaries of services shape messaging needs
- `02-data-storage.md` — event sourcing (Axis 5) is an alternative persistence pattern to that file's storage-model axis, and pairs with its CQRS option
- `07-api-design.md` — request-response style here reuses the same API paradigm reasoning covered there
- `11-background-jobs.md` — internal, single-service delayed/scheduled work, distinct from this file's cross-service messaging; often built on the same underlying queue technology
- `14-resilience.md` — retry, dead-letter, and idempotency patterns needed for reliable async messaging are covered in full there
- `15-observability.md` — event-driven systems need deliberate tracing investment to keep the implicit causal chain debuggable; log aggregation and metrics pipelines belong there, though they often reuse this file's broker infrastructure as transport
