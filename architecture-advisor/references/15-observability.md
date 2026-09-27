# 15 — Observability

## What this category covers

Understanding what a running system is actually doing: logging, metrics/monitoring, error tracking, and distributed tracing. This is the file that `10-messaging.md` defers to for log aggregation and metrics/monitoring pipelines (noted there as downstream use cases of event streaming), and that `14-resilience.md` defers to for the health-check monitoring that makes redundancy and failover actually observable in practice.

Distinct from `16-security.md`: security's audit logging is about who did what for compliance/forensic purposes, and security monitoring (intrusion detection, anomalous access-pattern detection, failed-auth tracking) is about detecting malicious or unauthorized activity specifically — both belong to that file's threat-focused lens. This file's logging, metrics, and alerting are about understanding general system behavior and diagnosing operational problems, not security incidents. The two often share underlying infrastructure — the same log pipeline can feed both operational dashboards and security alerting — but serve different purposes and are covered separately.

Distinct from `14-resilience.md`: resilience is about automatically *handling* in-flight failure (retries, circuit breakers); this file is about *seeing* what's happening well enough to know a failure occurred, understand its scope, and diagnose its cause — resilience patterns act on failures, observability makes failures visible in the first place.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Logging

### Unstructured (plain text) logging

- **Pros:** simplest to write — a log line is just a string, with no schema or format to design upfront; trivially human-readable when viewed directly
- **Cons:** difficult to query, filter, or aggregate programmatically at any real volume — extracting a specific field means parsing free-form text with regex or similar, which is brittle and breaks whenever the log message wording changes

### Structured logging (JSON or similar, consistent fields)

- **Pros:** machine-parseable by construction — fields like `user_id`, `request_id`, `duration_ms` can be filtered, aggregated, and queried directly by a log-analysis tool without fragile text parsing; scales to real production log volume in a way unstructured text doesn't
- **Cons:** more upfront discipline required — establishing and maintaining a consistent schema across the codebase (and across services, in a multi-service system) takes deliberate effort; less immediately human-readable when viewed raw, though tooling typically renders it readably

Structured logging is the standard default for anything beyond a small early-stage project or local development — the aggregation and query benefits compound as log volume and system complexity grow, and retrofitting structure onto years of unstructured logs later is far more painful than establishing it from the start.

**Log levels & sampling:** worth a one-line note rather than a full option — using log levels (debug/info/warn/error) lets verbosity be tuned per environment (verbose in development, quieter in production) without code changes, and sampling high-volume, low-value log lines (rather than logging every single occurrence) controls both storage cost and noise at scale. Close to a required companion to any real logging setup rather than an optional refinement.

---

## Axis 2: Metrics & monitoring

### Ad hoc / no dedicated metrics (logs as the only signal)

- **Pros:** nothing extra to instrument or run — logs alone can answer some operational questions if queried carefully
- **Cons:** logs are fundamentally event-oriented (this happened, at this time), not naturally suited to answering aggregate, trend, or threshold questions (what's the p99 latency over the last hour, is error rate climbing) without significant post-hoc processing; no natural mechanism for real-time alerting on a threshold being crossed

### Dedicated metrics & dashboards

- **Pros:** purpose-built for exactly the aggregate/trend/threshold questions logs answer poorly — request rate, latency percentiles, error rate, resource utilization, tracked continuously and visualized on dashboards; the natural feed for alerting (Axis 3) and for the health-check-based failover reasoning `14-resilience.md`'s Axis 6 depends on, since a dashboard makes a redundant fleet's actual health visible rather than assumed
- **Cons:** requires deliberate instrumentation — deciding what to measure and adding the code to emit it — plus infrastructure to collect, store, and visualize the resulting time-series data; another system to run and maintain, or another vendor relationship to manage if using a managed offering

Metrics and structured logs serve different, complementary purposes rather than substituting for each other — logs answer "what exactly happened in this specific case," metrics answer "how is the system behaving in aggregate, right now and over time." A project with any real production traffic benefits from having both rather than relying on logs alone for questions metrics answer far better.

**Cost/spend as a metric:** worth naming explicitly rather than treating as a separate discipline — infrastructure cost and resource utilization (is the system over-provisioned, is spend trending up unexpectedly) is a legitimate category of metric to track alongside performance and error metrics, using the same dashboard/alerting mechanisms as any other metric. For AI-call cost specifically, see `09-ai-integration.md`'s dedicated cost-control axis; this note is the general infrastructure-cost case.

**Capacity forecasting:** worth flagging as an advanced, optional layer rather than a default — trend-based prediction on top of already-collected metrics (projecting when disk usage or connection-pool capacity will be exhausted at the current growth rate) lets a team act before a threshold is actually crossed rather than purely reactively. Valuable once a project has enough metrics history and scale for a trend to be meaningful; premature for an early-stage project without enough historical data to forecast from.

---

## Axis 3: Alerting

Relevant once metrics (Axis 2) exist to alert on — alerting is the active-notification layer built on top of the passive dashboards metrics alone provide.

### No alerting (dashboards checked manually / reactively)

- **Pros:** nothing extra to configure
- **Cons:** problems are only discovered when someone happens to look, or when a user reports them — for anything with a real uptime expectation, this means meaningfully slower detection than the system could otherwise achieve

### Threshold-based alerting

- **Pros:** the team is notified automatically when a metric crosses a defined threshold (error rate above X%, latency above Y ms, a health check failing), rather than relying on someone noticing; directly operationalizes the health-check-based failover concept from `14-resilience.md`'s Axis 6 — an alert firing is what actually gets a human involved when automated failover alone isn't enough
- **Cons:** threshold tuning is a genuine and ongoing problem — too sensitive produces alert fatigue (real signal gets lost in noise, and the team starts ignoring alerts), too loose misses real problems until they're already serious; static thresholds don't naturally account for expected variation (e.g. normal traffic patterns by time of day)

### Anomaly detection (statistical/ML-based)

- **Pros:** catches deviations from normal behavior without needing a human to define a specific threshold upfront — useful for metrics with natural variation (traffic that's cyclical by time of day or day of week) where a single static threshold would either be too tight during quiet periods or too loose during busy ones; can surface genuinely novel failure modes a predefined threshold wouldn't have anticipated
- **Cons:** less predictable and harder to reason about than "alert if X crosses threshold Y" — false positives and negatives depend on a model or statistical baseline that isn't always transparent to the team relying on it; typically requires either a managed observability platform with this built in, or real investment to build and tune, putting it a tier above threshold-based alerting in cost and complexity

Threshold-based alerting remains the right default for most projects — clear, predictable, and cheap to reason about. Anomaly detection earns its added complexity specifically once traffic patterns are variable enough (strong daily/weekly cycles, unpredictable spikes) that static thresholds are genuinely producing too many false positives or missing real problems — more often a concern at real production scale than for an early-stage project.

**Automated remediation:** worth a one-line note rather than a full option — once an alert fires (threshold-based or anomaly-based), the response can be a human being paged (the default) or, for well-understood, low-risk failure modes, an automated action triggered directly: auto-restarting a crashed instance, auto-scaling in response to load, running a known remediation script, without waiting on a human at all. This is a further tier of investment beyond alerting itself, appropriate for specific, well-understood failure patterns with a known-safe automated fix — not a default to reach for broadly, since an automated action taken on a misdiagnosed problem can make things worse rather than better. Ties directly into `14-resilience.md`'s redundancy/failover reasoning — an unhealthy instance being automatically replaced is exactly this pattern applied to compute redundancy.

Alert routing and on-call process (who gets paged, escalation policy) is an operational/organizational decision downstream of the technical alerting setup — worth a one-line acknowledgment in a generated table rather than its own row, since it's more a team-process question than an architecture one.

---

## Axis 4: Error tracking

### Errors surfaced only through logs

- **Pros:** no additional tooling — errors are just another thing that gets logged (Axis 1)
- **Cons:** no automatic grouping of the same underlying error occurring repeatedly (each occurrence is a separate log line to notice and correlate manually); no built-in trend view of whether a given error is increasing, decreasing, or newly introduced by a recent deploy; harder to see error impact (how many users/requests affected) without custom log analysis

### Dedicated error-tracking tool

- **Pros:** automatically groups repeated occurrences of the same underlying error into one trackable issue, with a count and trend over time; typically captures rich context per error (stack trace, request details, user/session info, the specific deploy/release it appeared in), directly speeding up diagnosis; makes it easy to see whether a specific deploy introduced a new error or regression
- **Cons:** another tool/vendor relationship to integrate and maintain; captured error context can include sensitive data if not deliberately filtered, a real consideration alongside `16-security.md`'s data-handling policies

A dedicated error-tracking tool is a comparatively low-cost, high-leverage addition once a project has any real user traffic — the diagnostic speed-up it provides over sifting through logs manually tends to justify its cost well before other, heavier observability investments do.

---

## Axis 5: Distributed tracing

Relevant specifically once a request's handling spans multiple services or components — the direct counterpart to `10-messaging.md`'s point that event-driven systems need deliberate tracing investment to keep an otherwise-implicit causal chain debuggable.

### No tracing (logs/metrics per service only)

- **Pros:** nothing extra to instrument; sufficient for a monolith where a request's entire handling happens in one process and one set of logs already shows the full picture
- **Cons:** once a request spans multiple services (a microservices topology, `01-system-level.md`, or an event-driven chain, `10-messaging.md`), reconstructing the full path of a single request — which services it touched, where time was spent, where it failed — requires manually correlating logs across services by timestamp and any shared identifiers, which is slow and error-prone

### Distributed tracing (a trace ID follows a request across every service it touches)

- **Pros:** gives a single, unified view of a request's full path across every service and hop it touched, with per-hop timing — turns "which of these five services caused the slow response" from a manual log-correlation exercise into a direct visual answer; the concrete tool that makes an event-driven system's implicit causal chain (`10-messaging.md`) debuggable in practice
- **Cons:** requires trace-context propagation to be wired through every service boundary (each service needs to pass the trace ID along and emit its own spans) — a real, deliberate integration effort across the whole system, not a single tool installed once; adds a small amount of overhead per request for the tracing data itself

Distributed tracing earns its integration cost specifically once `01-system-level.md`'s system-level shape involves multiple services a single request can span — for a monolith, this axis collapses to "not applicable" rather than a real choice to make.

**Service dependency mapping:** worth naming as a benefit of Axis 5's investment rather than a separate mechanism — many tracing platforms can aggregate trace data across many requests into a service map, a visual graph of which services call which others and how often. This is a byproduct of already having distributed tracing in place, not a separate integration, and is particularly useful in a microservices topology for understanding blast radius before a change or onboarding someone to an unfamiliar system's actual call graph.

**Root cause analysis:** not a separate axis — it's the practice this file's tooling collectively enables, rather than a distinct architectural decision requiring its own tool. Structured logs (Axis 1) answer what exactly happened in a specific case, error tracking (Axis 4) surfaces and groups the problem across occurrences, and distributed tracing (Axis 5) shows where in a multi-service request it originated — together these are what make root cause analysis practical, rather than something requiring a dedicated fifth capability on top of them.

---

**Configuration change tracking:** worth a one-line note rather than a full axis — knowing what configuration changed, when, and by whom is a real operational need, but it's better handled as a byproduct of how configuration is managed in the first place than as a new mechanism this file needs to build. Version-controlled Infrastructure-as-Code naturally provides this via its own commit history (`19-deployment-infra.md`, `20-environment-strategy.md`); who changed what for compliance/forensic purposes is a specific case of `16-security.md`'s audit logging. Flagged here so it isn't mistaken for an oversight.

---

## What drives the recommendation (from upfront questions)

- **System-level shape** (from `01-system-level.md`): the primary driver for Axis 5 — a monolith has little need for distributed tracing since a request's full handling is visible in one place already; a microservices or event-driven topology makes tracing considerably more valuable, in direct proportion to how many services a typical request actually spans
- **Expected scale**: low-traffic, early-stage projects get comparatively little benefit from dedicated metrics/dashboards (Axis 2) or alerting (Axis 3) beyond structured logging and a dedicated error tracker; real production traffic is what makes the aggregate/trend questions metrics answer, and the noise-reduction alerting provides, actually valuable
- **Ops/deployment capacity**: low ops maturity weights toward managed observability offerings (a hosted logging/metrics/error-tracking platform) over self-hosting the equivalent open-source stack, and toward simpler alerting (a handful of well-chosen thresholds) over a sophisticated alerting setup
- **Time-to-ship pressure**: high pressure → structured logging plus a dedicated error-tracking tool (both comparatively cheap, high-leverage) is a legitimate first version, deferring dedicated metrics dashboards, alerting, and distributed tracing until the system's scale or topology actually demands them
- **Consistency requirements / domain criticality**: systems where failures need to be caught quickly (anything with a real uptime commitment, or where a slow-to-detect failure has real cost) push harder toward investing in alerting (Axis 3) specifically, rather than relying on dashboards being checked manually

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| Logging | *Unstructured pros/cons* | *Structured pros/cons* | — | *structured is the default beyond a small/early project; note log levels and sampling as required companions* | ☐ |
| Metrics & monitoring | *Ad hoc/logs-only pros/cons* | *Dedicated metrics & dashboards pros/cons* | — | *justify against expected traffic and whether aggregate/trend questions matter; logs and metrics are complementary, not substitutes; note cost/spend as a tracked metric category* | ☐ |
| Alerting | *No alerting pros/cons* | *Threshold-based pros/cons* | *Anomaly detection pros/cons* | *threshold-based is the default; anomaly detection only for genuinely variable traffic patterns; note automated remediation as a further response-tier option* | ☐ |
| Error tracking | *Logs-only pros/cons* | *Dedicated error-tracking tool pros/cons* | — | *dedicated tooling is comparatively low-cost, high-leverage once there's real user traffic* | ☐ |
| Distributed tracing | *No tracing pros/cons* | *Distributed tracing pros/cons* | — | *only relevant once a request spans multiple services (`01-system-level.md`); collapses to not-applicable for a monolith* | ☐ |

Note: for a monolith with modest traffic under time pressure, this can reasonably collapse to two rows ("structured logging plus a dedicated error-tracking tool" and "dashboards and alerting deferred until real production traffic demands them") rather than five — drop the distributed-tracing row entirely for a monolith per its own guidance above.

---

## Related files

- `01-system-level.md` — the primary driver for whether distributed tracing (Axis 5) is relevant at all
- `09-ai-integration.md` — AI-call cost control is that file's dedicated axis; this file's cost-as-metric note (Axis 2) is the general infrastructure-cost case
- `10-messaging.md` — that file's log-aggregation and metrics/monitoring downstream-use-case note lands here; event-driven systems specifically need this file's Axis 5 (tracing) to stay debuggable
- `14-resilience.md` — health-check-based failover (that file's Axis 6) depends on this file's Axis 2 (metrics) and Axis 3 (alerting) to actually detect and surface an unhealthy instance; automated remediation (Axis 3 note) is that reasoning applied beyond just traffic routing
- `16-security.md` — security monitoring (intrusion/anomalous-access detection) and audit logging are that file's concern, distinct from this file's operational diagnostics; config-change audit trails also land there
- `19-deployment-infra.md` / `20-environment-strategy.md` — error tracking (Axis 4) tying errors to a specific deploy depends on release tracking there; configuration change tracking is a natural byproduct of IaC's version history there rather than a mechanism this file builds
