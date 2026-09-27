# 11 — Background Jobs / Scheduling

## What this category covers

A single service's own internal "do this later" or "do this on a schedule" work: cron-style scheduled tasks and queued/delayed job processing. Distinct from `10-messaging.md`, which covers communication *between* separate services — this file covers a service handling its own deferred or recurring work, even in a single-deployable-unit monolith with no other services to talk to at all. A monolith with zero cross-service messaging needs can still have a very real background-jobs decision (e.g. sending emails asynchronously, generating a nightly report), which is exactly why this file isn't conditional on system-level shape the way `10-messaging.md` is.

**Scope note:** consistent with the skill's general avoidance of vendor/product selection (see `SKILL.md`'s excluded-categories list), this file does not choose between specific job-queue or scheduler products. BullMQ (a Redis-backed job-queue library, commonly used in Node.js services for exactly the single-service internal work this file covers), Sidekiq, Celery, and cloud-native schedulers (e.g. a provider's managed cron/scheduled-function offering) are mentioned only as brief illustrative examples of the properties in the axes below, not as options to weigh against each other in a table.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Scheduled (cron-style) vs on-demand (queued/triggered) jobs

### Scheduled / cron-style (runs on a fixed time interval or schedule)

- **Pros:** simplest mental model for genuinely time-based work — "run this every night at 2am," "run this every 15 minutes" — with no triggering logic needed beyond the schedule itself; predictable resource usage, since the timing is known in advance rather than driven by unpredictable demand
- **Cons:** a poor fit for work that should happen in response to a specific event (a user action, an upstream data change) — either the schedule has to run needlessly often to catch things promptly, or real events wait until the next scheduled run; doesn't naturally distribute or parallelize a variable amount of work the way a queue does, since a scheduled run typically processes whatever's due in one pass

### On-demand / queued (triggered by an application event, processed asynchronously)

- **Pros:** work happens close to when it's actually needed, rather than waiting for the next scheduled window — a natural fit for "do this later, but do it soon" work (send a welcome email, generate a thumbnail, process an upload) triggered directly by user or application activity; queue depth naturally reflects actual demand, and multiple workers can drain the queue in parallel, giving load-balanced processing that scales with volume (the same mechanism as `10-messaging.md`'s Axis 2 queue pattern, just scoped to one service's internal work rather than cross-service communication)
- **Cons:** needs a queue and worker infrastructure to exist at all, which is more moving parts than a simple scheduled task for genuinely time-based work; if not carefully monitored, a growing backlog (workers falling behind incoming volume) can go unnoticed without deliberate queue-depth observability

Most real systems need both — scheduled jobs for genuinely time-based, recurring work (nightly reports, periodic cleanup, batch reconciliation) and queued jobs for event-triggered, close-to-real-time async work (sending a confirmation email right after signup). This is rarely an either/or decision so much as recognizing which category a given piece of work falls into.

---

## Axis 2: Job execution guarantees

Once jobs are queued rather than purely scheduled, the same reliability questions from `10-messaging.md`'s broker-properties axis apply here, scoped to a single service's own job processing.

### At-most-once (fire and forget, no retry on failure)

- **Pros:** simplest to implement — no retry logic, no idempotency concerns; appropriate for genuinely low-stakes work where a missed execution is an acceptable, cheap-to-ignore outcome
- **Cons:** any transient failure (a brief downstream outage, a temporary resource constraint) permanently loses that job's work with no automatic recovery

### At-least-once (retried until success, may execute more than once)

- **Pros:** the standard default for most background work — a job that fails is retried rather than silently lost, giving real resilience against transient failures without manual intervention
- **Cons:** requires idempotent job handlers, since a retry after a partial success can mean the same job runs more than once (e.g. a job that failed after sending an email but before marking itself complete would resend the email on retry unless the handler accounts for this) — the same idempotency requirement noted in `10-messaging.md`'s dead-letter/idempotency section, scoped to this file's jobs

### Exactly-once (each job's effect happens exactly one time, no duplicates)

- **Pros:** removes the idempotency burden from job handlers — the strongest guarantee, appropriate when a job's side effect genuinely cannot tolerate any duplication (e.g. charging a payment method) and idempotency can't easily be built into the handler itself
- **Cons:** genuinely difficult and expensive to guarantee end-to-end in a distributed system (the same fundamental difficulty as `10-messaging.md`'s exactly-once note) — in practice, most systems achieve the equivalent outcome by combining at-least-once delivery with idempotent handlers (e.g. an idempotency key checked before applying the job's effect) rather than relying on a queue technology's literal exactly-once guarantee

At-least-once with idempotent handlers is the practical default for the large majority of background jobs — exactly-once is worth naming as a distinct tier for the narrow set of jobs where duplication would be a genuine correctness problem and idempotency is hard to retrofit, not a default target for every job.

---

## Axis 3: Failure handling — retries, backoff, and dead-letter handling

- **Retry with backoff:** rather than retrying a failed job immediately (which can hammer a downstream dependency that's already struggling), retrying with increasing delay between attempts gives a transient failure time to resolve — the same reasoning as `14-resilience.md`'s retry patterns, applied to job processing specifically
- **Maximum retry limits & dead-letter handling:** a job that keeps failing after a reasonable number of retries needs somewhere to go other than retrying forever or silently disappearing — a dead-letter queue or equivalent failed-job store, matching `10-messaging.md`'s dead-letter handling note, so failures are visible and recoverable rather than silent
- **Poison-message protection:** a job whose payload itself causes the failure (a malformed input that will never succeed no matter how many times it's retried) needs to be recognized and routed to dead-letter handling quickly rather than consuming retry budget indefinitely

These aren't independent options to choose between so much as required companions to any real queued-job system — worth naming explicitly in a generated table as a single combined consideration rather than three separate rows for most projects.

---

## What drives the recommendation (from upfront questions)

- **Domain coupling / nature of the work**: the direct driver for Axis 1 — genuinely time-based, recurring work (reports, cleanup, batch reconciliation) is a scheduling fit; work that should happen promptly in response to an event is a queuing fit; most real projects have some of both rather than needing to pick one exclusively
- **Consistency requirements**: work with irreversible or hard-to-undo side effects (payment processing, sending an email that can't be unsent) pushes toward at-least-once delivery with careful idempotency design (Axis 2), or toward treating exactly-once-equivalent behavior as a genuine requirement rather than a nice-to-have
- **Ops/deployment capacity**: low ops maturity weights toward a managed scheduling/queue offering (e.g. a cloud provider's managed cron or queue service) over self-hosting job-queue infrastructure, and toward simpler at-least-once-plus-idempotency handling over building more sophisticated exactly-once machinery
- **Time-to-ship pressure**: high pressure → a simple scheduled-task mechanism for recurring work plus a basic managed queue for event-triggered work, with straightforward retry-with-backoff and a dead-letter fallback, is a legitimate first version — not a gap
- **Expected scale**: low job volume rarely justifies much beyond a single worker process and a lightweight queue; high job volume is what actually justifies investing in queue-depth observability, multiple parallel workers, and more deliberate backpressure handling

---

## Recommendation table skeleton

| Category | Option A | Option B | Recommendation + why | Approve |
|---|---|---|---|---|
| Scheduled vs on-demand | *Scheduled/cron pros/cons* | *On-demand/queued pros/cons* | *most projects need both; state which categories of work fall into each rather than picking one exclusively* | ☐ |
| Execution guarantees | *At-least-once pros/cons* | *Exactly-once (or equivalent) pros/cons* | *at-least-once with idempotent handlers is the default; exactly-once only for jobs where duplication is a genuine correctness problem* | ☐ |
| Failure handling | *state retry/backoff, dead-letter, and poison-message approach* | — | *treat as one combined row rather than three separate ones for most projects; connects to `14-resilience.md`* | ☐ |

Note: for a simple project with modest job volume and no irreversible-side-effect jobs, this can reasonably collapse to a single consolidated row ("scheduled tasks for recurring work, a managed queue with at-least-once delivery and basic retry/dead-letter handling for event-triggered work") rather than three.

---

## Related files

- `01-system-level.md` — background-job infrastructure often lives inside the same deployable unit for a monolith, or as its own worker service in a microservices topology
- `10-messaging.md` — this file's queued-job pattern is the same underlying mechanism as that file's queue delivery pattern, just scoped to a single service's internal work rather than cross-service communication; that file's dead-letter/broker-property reasoning applies here too
- `14-resilience.md` — retry-with-backoff and failure-handling patterns (Axis 3) are covered in full there
- `15-observability.md` — queue depth and job failure rates are exactly the kind of operational signal that needs monitoring and alerting
