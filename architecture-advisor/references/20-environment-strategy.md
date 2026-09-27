# 20 — Environment & Release Strategy

## What this category covers

Infrastructure-as-Code vs manual infrastructure management, dev/staging/prod environment parity, deployment rollout strategy (how new code reaches production instances), feature flags and release decoupling (which users see which features, independent of deployment), and rollback plan for a bad release.

Distinct from `18-data-migration.md`'s rollback: that reverts a bad *data-model change*; this file's rollback reverts a bad *application release*. The two often need to work together — a bad deploy that included a migration may need both — but they're separate mechanisms for separate failure modes.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Infrastructure as Code (IaC) vs manual infrastructure management

### Manual infrastructure management (console/CLI configuration, no declarative source of truth)

- **Pros:** simplest to start — no tooling to learn, just directly configure infrastructure through a provider's console or CLI
- **Cons:** no record of what was configured or why, making infrastructure drift — production silently diverging from what anyone thinks is deployed — common; recreating an environment (a new stage, or disaster recovery per `03-backup-disaster-recovery.md`) means manually redoing every step, error-prone and slow; changes aren't code-reviewed or version-controlled the way application code is

### Infrastructure as Code (infrastructure defined in version-controlled, declarative configuration)

- **Pros:** infrastructure changes are code-reviewed, version-controlled, and repeatable — the same environment can be recreated reliably, which matters directly for `03-backup-disaster-recovery.md`'s DR-tier reasoning and for standing up new environments (Axis 2) consistently; provides the change-history audit trail `15-observability.md`'s configuration-change-tracking note already points to as this file's territory
- **Cons:** a real learning curve and upfront tooling investment; declarative IaC tools have their own failure modes — state drift between the tool's understanding and actual infrastructure — that need to be managed

IaC is close to the standard expectation for anything beyond a small, early-stage project — the reproducibility and audit-trail benefits compound in exactly the ways that matter for the rest of this file's axes: consistent environments, reliable rollback.

---

## Axis 2: Dev/staging/prod environment parity

### Low parity (environments differ meaningfully — a smaller database, mocked integrations, different scale)

- **Pros:** cheaper and simpler to run lower environments, since they don't need to replicate full production infrastructure or scale
- **Cons:** "works in staging" doesn't reliably predict "works in production" when the environments genuinely differ — environment-specific bugs surface only after a real production deploy, exactly the failure mode good staging is meant to prevent

### High parity (staging closely mirrors production configuration, integrations, and scale characteristics)

- **Pros:** staging becomes a genuinely reliable signal for whether a change will work in production, catching environment-specific issues before they reach real users — this is what makes `17-testing-strategy.md`'s acceptance testing (UAT) and this file's own canary/blue-green verification (Axis 3) actually trustworthy, since verification is only as good as how representative the environment being verified in actually is
- **Cons:** real cost to run a staging environment that mirrors production's infrastructure and scale, though not necessarily 1:1 traffic volume; more infrastructure to keep in sync as production evolves, itself requiring the IaC discipline from Axis 1 to manage without drift

High parity is worth the cost in rough proportion to how much a bad deploy would cost if it reached production undetected — a low-stakes internal tool can reasonably run a lighter, lower-parity staging setup; anything with real business or safety stakes benefits from closing that gap as much as practically affordable.

---

## Axis 3: Deployment rollout strategy

How a new version of the code actually reaches production traffic once a build is ready to deploy — controlling which *instances* run which *code*.

### All-at-once (deploy the new version everywhere simultaneously)

- **Pros:** simplest — one step, done; no intermediate state where old and new versions run side by side
- **Cons:** no gradual verification — if something's wrong, it's wrong for 100% of traffic immediately; the same risk profile as `18-data-migration.md`'s big-bang cutover strategy, applied to application deploys rather than schema changes

### Rolling deployment (instances updated gradually, a few at a time)

- **Pros:** no full-fleet downtime — some instances always serve traffic while others update; a problem surfaces on a subset of instances first rather than all at once, giving some warning before full exposure
- **Cons:** old and new versions run simultaneously during the rollout window, which requires the backward-compatibility discipline `18-data-migration.md`'s incremental/trickle pattern already covers for any data-model changes involved; no deliberate traffic-percentage control or built-in pause-and-observe step the way canary has — it mechanically works through instances rather than deliberately limiting exposure

### Blue-green deployment (two full parallel environments, traffic cut over at once after verification)

- **Pros:** the new version ("green") is fully deployed and verified in an environment identical to production before any real traffic touches it; cutover is close to instant, and rollback is just as fast — point traffic back at the still-running old environment ("blue"); the same underlying pattern as `18-data-migration.md`'s parallel-run and version-upgrade blue-green options, applied here to application deploys
- **Cons:** requires double the infrastructure running simultaneously, at least temporarily — a real cost; database/data-layer state shared between blue and green environments needs careful handling, since both typically hit the same underlying data store

### Canary release (a small percentage of traffic routed to the new version first, gradually increased as it's verified healthy)

- **Pros:** limits the blast radius of a bad deploy to a small, deliberately-chosen fraction of traffic before it ever reaches everyone; pairs naturally with `15-observability.md`'s metrics and alerting — the canary's health is actively monitored against the baseline before traffic is increased, making this the most observation-driven of the rollout strategies
- **Cons:** requires the traffic-routing layer (`21-networking.md`'s load balancer or service mesh) to actually support percentage-based routing; takes longer to reach full rollout than blue-green's near-instant cutover, since the whole point is a gradual, monitored ramp

These aren't fully mutually exclusive — canary and rolling are often combined (a canary phase followed by a broader rolling rollout once the canary looks healthy), and blue-green can incorporate a gradual, canary-style traffic shift during cutover rather than an instant one. All-at-once is reasonable only when brief downtime or inconsistency is genuinely tolerable and the deploy is low-risk; the other three are all meaningfully safer — canary is the most gradual and observation-heavy, blue-green offers the fastest safe cutover and fastest rollback, and rolling is the standard default most container orchestrators and PaaS platforms already support with minimal extra configuration.

---

## Axis 4: Feature flags & release decoupling

Distinct from Axis 3: that axis controls which server instances run which *code*; this axis controls which *users* see which *features*, independent of what code is actually deployed and running.

### No feature flags (deploying code and releasing a feature are the same event)

- **Pros:** simplest — nothing extra to build or manage; what's deployed is what users see, with no additional layer of indirection
- **Cons:** every deploy is also a release — there's no way to get new code running in production without immediately exposing it to all users, which removes several safer patterns below (dark launches, gradual feature rollout, A/B testing) as options entirely

### Feature-flag-based release (code ships behind a flag; the flag controls actual user exposure separately from deployment)

- **Pros:** decouples "is this code running in production" from "can users see or use this" — a feature can be deployed and even exercised in production (a dark launch, below) well before it's turned on for real users; enables gradual, controlled feature rollout — a percentage of users, a specific cohort — independent of the deployment rollout strategy in Axis 3; provides an instant kill switch for a bad feature without needing a full redeploy or rollback
- **Cons:** real additional complexity — flag management itself becomes a piece of infrastructure with its own failure modes; stale flags never cleaned up after a feature is fully rolled out accumulate as technical debt and conditional-logic clutter in the codebase, a common and easy-to-neglect maintenance burden

**Dark launches:** a specific application of feature-flag-based release — deploying and running new code in production, fully exercising it under real production load and conditions, while keeping it invisible or inert to actual users (the flag stays off, or the feature's output is computed but discarded rather than surfaced). Used to validate a feature's real-world performance and behavior before ever exposing it. Distinct from canary release (Axis 3), which exposes a new *deployment* to a fraction of *traffic* — a dark launch can run at 100% of traffic while remaining 0% visible to users.

**A/B testing:** another specific application of feature-flag-based release — routing different user segments to different variants of a feature, not just on/off, to measure the impact of each on some outcome, typically via the same flag-targeting/bucketing mechanism used for gradual feature rollout. Distinct in *purpose* from the deployment-safety motivations behind the rest of this axis: A/B testing is product and business experimentation, not deployment risk reduction, even though it reuses the same underlying flag infrastructure. Worth flagging as a different use of the same mechanism rather than a separate piece of infrastructure to build.

Feature flags are worth the investment specifically once a project needs to decouple release timing from deploy timing — gradual or targeted feature rollout, dark-launch validation, A/B testing, or an instant kill switch on risky features. A project shipping infrequently, to a small trusted user base, with low-risk changes has comparatively less need for this added layer.

---

## Axis 5: Rollback plan

Distinct from `18-data-migration.md`'s migration rollback (reverting a bad *data-model change*): this axis covers reverting a bad *application release* specifically.

### No formal rollback plan (fix forward)

- **Pros:** no separate rollback tooling or process to build and maintain
- **Cons:** recovering from a bad release under pressure means writing and shipping a fix while the incident is still live — slower and higher-pressure than simply reverting to the last known-good version

### Formal rollback capability (revert to the previous known-good release)

- **Pros:** a fast, well-understood path back to a working state when a new release causes problems — meaningfully faster than fixing forward under pressure; pairs naturally with blue-green (Axis 3), where rollback is close to instant by construction, and with feature flags (Axis 4), which can serve as an even faster rollback for a specific bad feature without touching the deploy at all
- **Cons:** rollback isn't always clean if the bad release included a database migration — reverting the application code without also reverting the schema (`18-data-migration.md`'s territory) can leave the system in a broken, mismatched state, so a genuine rollback plan has to account for everything a specific release actually changed, not just the application code

A formal rollback capability is close to a non-negotiable baseline for anything beyond a small, low-stakes project — it's cheap to have "redeploy the last known-good build" as an option, and expensive not to have it during an actual incident.

---

## What drives the recommendation (from upfront questions)

- **Consistency requirements / uptime expectations**: strict uptime pushes toward blue-green or canary (Axis 3) over all-at-once, and toward feature flags (Axis 4) as an instant kill switch alongside a formal rollback plan (Axis 5)
- **System-level shape / statelessness** (from `01-system-level.md`, `05-scalability.md`): a horizontally-scaled, stateless app tier is close to a prerequisite for rolling, blue-green, or canary to work cleanly at all — a single-instance deployment has less to gain from these over all-at-once
- **Domain / product needs**: a project that wants to run experiments or measure feature impact is a direct driver for feature flags (Axis 4) regardless of deployment-safety needs alone
- **Ops/deployment capacity**: low ops maturity weights toward IaC via a managed platform's built-in templates rather than bespoke tooling, toward rolling deployment (usually a built-in default) over blue-green/canary's added infrastructure cost, and toward a lighter, lower-parity staging setup if full parity isn't affordable
- **Time-to-ship pressure**: high pressure → IaC via simple provider templates, rolling deployment (often free/default on most platforms), no feature flags until a real need for decoupled release timing emerges, and at minimum a basic "redeploy the last known-good build" rollback capability, is a legitimate first version

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Option D | Recommendation + why | Approve |
|---|---|---|---|---|---|---|
| IaC vs manual | *Manual pros/cons* | *IaC pros/cons* | — | — | *IaC is close to the standard default beyond a small/early project* | ☐ |
| Environment parity | *Low parity pros/cons* | *High parity pros/cons* | — | — | *justify against the stakes of an undetected bad deploy* | ☐ |
| Deployment rollout strategy | *All-at-once pros/cons* | *Rolling pros/cons* | *Blue-green pros/cons* | *Canary pros/cons* | *rolling is the common default; canary/blue-green for higher-stakes uptime needs; all-at-once only when downtime is genuinely tolerable* | ☐ |
| Feature flags | *No feature flags pros/cons* | *Feature-flag-based release pros/cons* | — | — | *justify against need for decoupled release timing — dark launches, A/B testing, gradual rollout, or a kill switch* | ☐ |
| Rollback plan | *No formal plan pros/cons* | *Formal rollback capability pros/cons* | — | — | *close to non-negotiable; note that a release including a migration needs `18-data-migration.md`'s rollback too* | ☐ |

Note: for a small, early-stage project under time pressure, this can reasonably collapse to three rows ("IaC via simple templates, low-parity staging," "rolling deployment, no feature flags yet," and "basic redeploy-last-build rollback") rather than five.

---

## Related files

- `01-system-level.md` / `05-scalability.md` — rolling, canary, and blue-green deployment (Axis 3) all assume a horizontally-scaled, stateless app tier
- `03-backup-disaster-recovery.md` — IaC's reproducibility (Axis 1) directly supports that file's DR-tier reasoning
- `09-ai-integration.md` — that file's prompt-as-artifact management note ties to this file's IaC/config-management reasoning
- `13-caching.md` — DNS TTL affects rollout and failover propagation speed, decided here
- `15-observability.md` — canary release (Axis 3) depends on that file's metrics and alerting to actually judge health before widening rollout; configuration-change tracking there is a byproduct of this file's IaC
- `17-testing-strategy.md` — acceptance testing (UAT) typically happens in the staging environment whose parity is decided here (Axis 2)
- `18-data-migration.md` — migration rollback is distinct from this file's deploy rollback (Axis 5); this file's rollout strategy (Axis 3) interacts directly with whether that file's incremental/trickle migration pattern is needed
- `19-deployment-infra.md` — a full CI/CD pipeline there is what actually executes this file's rollout strategies (Axis 3)
- `21-networking.md` — canary's percentage-based traffic routing and blue-green's cutover both depend on the load balancer/traffic-routing capability decided there
