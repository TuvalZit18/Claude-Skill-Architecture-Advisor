# 03 — Backup & Disaster Recovery

## What this category covers

Protecting against **catastrophic, permanent data loss** — a deleted database, a corrupted volume, a region-wide outage, ransomware, or a human error like a bad migration — and being able to recover the *data itself* after the fact.

Distinct from `14-resilience.md`: resilience is about *in-flight* failure handling (a request failing right now, a downstream service being unavailable) while the system is otherwise healthy and no data has actually been lost. Backup & DR is about what happens when data loss has already occurred.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

**Scope note:** this file treats "the data" as the structured/transactional store decided in `02-data-storage.md`. Backing up files, media, and other blob-type assets is covered separately in `04-file-storage.md`, since object storage typically has its own (often provider-native) durability and backup story rather than sharing a pipeline with the primary database. A real project may apply different backup rigor to each — that split is expected, not an oversight.

---

## Axis 1: Backup type

### Full backup

- **Pros:** simplest to restore from — a single artifact; no chain to replay
- **Cons:** most expensive in storage and time; rarely run every cycle at real scale

### Incremental backup

- **Pros:** cheapest and fastest to take — only changes since the last backup (full or incremental)
- **Cons:** restore requires replaying the full chain in order; slower and has more failure points than a single artifact

### Differential backup

- **Pros:** restore only needs the full plus one differential — simpler than an incremental chain
- **Cons:** grows larger between fulls than an incremental would, since each differential captures all changes since the last full, not just since the last differential

### Continuous / log-based backup

- **Pros:** streams write-ahead-log or change-data-capture continuously rather than on a schedule; enables near-zero RPO (see Axis 3); required for point-in-time recovery (Axis 2)
- **Cons:** depends on the storage engine supporting log shipping (e.g. Postgres WAL); more moving parts to operate than scheduled backups

### Snapshot-based backup

- **Pros:** often built directly into the platform (EBS snapshots, RDS automated snapshots, VM snapshots) — minimal setup, fast to take, copy-on-write so overhead is low; frequently the default mechanism for cloud-hosted databases
- **Cons:** usually tied to the specific cloud provider/storage layer — less portable than an exported backup file; restoring typically means provisioning new infrastructure from the snapshot rather than importing into existing infra, which can complicate cross-environment or cross-provider restores

Most real systems combine these — a periodic full plus incremental or log shipping in between — rather than picking one exclusively.

**Retention/rotation policy:** however backups are taken, how long they're kept and how old ones are rotated out (e.g. grandfather-father-son rotation, or a simple "keep 30 daily + 12 monthly") is part of this axis's mechanics rather than a separate decision. Retention length is itself often driven by compliance requirements or storage cost, not just RPO.

A bad migration is one of the main real-world triggers for needing this whole category — a backup taken before the migration ran is the actual safety net, distinct from any schema-versioning/rollback tooling covered in `18-data-migration.md`.

---

## Axis 2: Point-in-time recovery (PITR)

Whether the system can restore to an arbitrary timestamp ("restore to 2:14pm yesterday, right before the bad migration ran") rather than only to the moment of the last discrete backup.

### Without PITR

- **Pros:** simpler to implement and reason about; matches a straightforward scheduled-backup setup
- **Cons:** restore points are only as granular as the backup schedule — a daily backup means losing up to 24h of data even in a clean restore

### With PITR

- **Pros:** restores to any point covered by the retained log, not just backup checkpoints — turns "restore to yesterday's backup" into "restore to 30 seconds before the incident"; the meaningful difference for human-error scenarios (bad delete, bad migration) rather than pure infrastructure loss
- **Cons:** needs the underlying engine/service to support it — native in most managed relational databases, not universal for NoSQL/document stores and varies by provider; requires continuous log shipping (Axis 1) to be running

---

## Axis 3: RTO and RPO

The two targets that actually define what backup strategy is needed — everything else is implementation detail in service of these numbers.

- **RPO (Recovery Point Objective):** how much data can be lost, measured in time. An RPO of one hour means worst case, up to an hour of writes is lost. Driven by backup/log-shipping frequency.
- **RTO (Recovery Time Objective):** how long the system can be down while restoring. An RTO of four hours means locating the backup, provisioning, replaying logs, and validating must all complete within four hours. Driven by restore tooling, data volume, and how automated the process is.

These are normally captured as an upfront constraint — tighter RTO/RPO costs more (more frequent backups, warm standbys, automated failover) — rather than derived from the storage model itself. Loose RTO/RPO ("we can lose a day, be down half a day") can get away with a nightly full backup and a manual restore runbook. Tight RTO/RPO (seconds of data loss, minutes of downtime) needs continuous replication and often a hot standby, which shades into Axis 5.

---

## Axis 4: Backup storage & redundancy

Where the backups themselves live — a backup stored on the same infrastructure as the primary data doesn't protect against infrastructure-level loss.

### Same region, different storage

- **Pros:** cheapest, fastest to restore from
- **Cons:** doesn't protect against a regional outage or account-level incident

Worth distinguishing two sub-cases here rather than treating "same region" as one thing: a **single-AZ** backup (same availability zone as primary) doesn't survive an AZ-level failure, while **multi-AZ within the region** does — and multi-AZ is the default posture for most managed databases out of the box (e.g. RDS Multi-AZ), effectively free protection against the most common infrastructure failure mode (a single data center going down) without taking on cross-region latency/cost. Cross-region below is the step up from that for regional-outage protection specifically, not the first step up from single-AZ.

### Cross-region replication

- **Pros:** protects against regional outages; standard practice for anything with a real RTO/RPO commitment
- **Cons:** adds latency and cost to the backup process

### Cross-provider / offline (cold) copy

- **Pros:** protects against provider-level incidents (account compromise, provider outage, accidental account deletion) and ransomware that targets connected backups
- **Cons:** slowest and most operationally heavy to restore from; typically reserved for compliance-driven or high-stakes cases rather than default practice

### Immutable / WORM (write-once-read-many)

- **Pros:** specifically protects against ransomware and credential-compromise scenarios — a backup that can't be altered or deleted even by someone with valid write access, for a set retention window (e.g. S3 Object Lock); increasingly treated as standard practice rather than exotic, since a replicated-but-mutable backup can still be encrypted or deleted by an attacker holding valid credentials
- **Cons:** retention window has to be decided upfront and can't be shortened without provider-level intervention; typically adds cost; doesn't substitute for cross-region redundancy — it's an orthogonal protection (immutability vs. geographic redundancy), often combined with one of the two options above rather than replacing them

**Backup encryption:** encryption at rest and in transit is `16-security.md`'s general policy line, but backups raise a question specific to this category — key custody. A backup encrypted with the same key/account as the primary data doesn't add protection against credential compromise (an attacker with valid access can decrypt both); a cross-provider or offline copy only delivers on its promise if the keys are held separately too. Worth a one-line flag in the generated table rather than a full axis of its own — treat it as a qualifier on whichever storage/redundancy option is chosen, not a separate row.

The general principle (sometimes called 3-2-1: 3 copies, 2 media types, 1 offsite) scales down for smaller projects to "at least one backup that isn't in the same place as the thing it's backing up."

---

## Axis 5: Restore testing & failover posture

A backup that has never been restored is unverified. This is a process decision rather than a technology choice, but it carries a real cost/rigor trade-off.

### Untested backups

- **Pros:** cheapest — no drill process to maintain
- **Cons:** actual RTO/RPO are unknown until an incident forces a first restore attempt

### Periodic restore drills

- **Pros:** confirms backups are actually usable; measures real restore time against the RTO target
- **Cons:** ongoing process overhead — scheduled exercises restoring into a scratch environment and validating the data

### Warm/hot standby

- **Pros:** a continuously-updated replica ready to take over, effectively driving RTO toward zero
- **Cons:** runs duplicate infrastructure continuously; differs from a backup in that it's already running rather than something restored from — this is where backup/DR starts to overlap with high-availability architecture rather than pure disaster recovery

Not to be confused with a deploy rollback plan (`20-environment-strategy.md`) — that reverts a bad *release*; failover here recovers from a bad *data* state or infra loss.

**Backup monitoring & alerting:** a related but distinct concern from restore testing — knowing whether a backup job even *ran successfully* (vs. silently failing for weeks, one of the most common real-world backup failures) rather than knowing whether it can be *restored from*. A backup can succeed on schedule and still be worthless if no one is alerted when it stops running. Cheap to add (a job-completion check + alert) regardless of which restore-testing posture is chosen, so it's less a fourth option than a baseline that should accompany any of the three above.

**DR tier:** Axis 4 (where backups live) and Axis 5 (how ready the system is to fail over) jointly determine which point a project sits at on the standard backup/DR spectrum — **backup-and-restore** (untested or lightly-tested backups, same-region or cross-region, restore on demand) → **pilot light** (core infra kept minimally running elsewhere, scaled up on failover) → **warm standby** (a scaled-down but live replica, per the option above) → **multi-site active-active** (multiple fully live sites, no real "failover" moment at all). Most projects land on backup-and-restore or, at most, pilot light; warm/hot standby as described above is already a meaningfully expensive step up, and active-active is rare enough it doesn't warrant its own bullet — worth naming in the generated table as context for where the chosen combination sits, rather than presenting Axis 4 and 5 as fully independent choices.

---

## Managed vs. self-managed backups

If `02-data-storage.md` resulted in a managed backend-as-a-service or managed database, backup/PITR is often built in (e.g. automated daily backups plus a PITR retention window as a plan feature) — the decision shrinks to "which tier/retention window" rather than building the pipeline. Self-managed or self-hosted data stores require building Axes 1–5 explicitly. Worth flagging in the generated table, since it can make this whole category close to a non-decision for some projects.

---

## What drives the recommendation (from upfront questions)

- **Consistency requirements**: domains where losing recent writes is unacceptable (financial transactions, anything with a strict audit trail) → push toward PITR and continuous log-based backup regardless of other pressures
- **Ops/deployment capacity**: low ops maturity → weights toward managed-provider defaults (see Managed vs. self-managed above) and away from building a custom log-shipping/PITR pipeline
- **Time-to-ship pressure**: high pressure → nightly full backup via the managed provider's default, no PITR, no cross-region copy, is a legitimate answer, not a gap
- **Budget/cost constraints**: tight budget → same-region storage and untested/lightly-tested backups over cross-region redundancy and restore drills
- **Domain/compliance context**: compliance-driven projects (finance, health) typically push toward stricter RPO/RTO and cross-region/offline copies by default — but per the skill's excluded-categories note, don't assume this; ask if it's relevant to the specific project rather than forcing it

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Option D | Option E | Recommendation + why | Approve |
|---|---|---|---|---|---|---|---|
| Backup type | *Full pros/cons* | *Incremental pros/cons* | *Differential pros/cons* | *Continuous/log-based pros/cons* | *Snapshot-based pros/cons* | *pick one or a combination, justify against consistency needs / ops capacity / time pressure; note retention policy* | ☐ |
| Point-in-time recovery | *Without PITR pros/cons* | *With PITR pros/cons* | — | — | — | *justify against consistency requirements and whether the engine supports it* | ☐ |
| RTO / RPO targets | *loose target implications* | *tight target implications* | — | — | — | *state the target numbers and what they imply for the rows above* | ☐ |
| Backup storage & redundancy | *Same-region (single-AZ vs multi-AZ) pros/cons* | *Cross-region pros/cons* | *Cross-provider/offline pros/cons* | *Immutable/WORM pros/cons* | — | *pick one or combine, justify against budget, compliance, and threat model (ransomware/credential compromise); flag key-custody separately from the primary store if encryption matters* | ☐ |
| Restore testing & failover | *Untested pros/cons* | *Periodic drills pros/cons* | *Warm/hot standby pros/cons* | — | — | *pick one, justify against RTO target and ops capacity; note the resulting DR tier (backup-and-restore / pilot light / warm standby / active-active); include backup-monitoring/alerting as a baseline regardless of which option is picked* | ☐ |

Note: for many projects — especially those on a managed backend-as-a-service with time-to-ship pressure and no compliance driver — most of these rows collapse to "use the provider's default," and the table can reasonably show a single consolidated row rather than five, with the managed-provider note carrying the justification. Don't force five rows when the project's answers make four of them a non-decision.

---

## Related files

- `02-data-storage.md` — this file's backup/PITR scope narrows when the storage decision there is a managed backend-as-a-service
- `04-file-storage.md` — covers backup/durability for files & media; this file covers the structured/transactional store only
- `14-resilience.md` — in-flight failure handling; this file covers already-occurred data loss instead
- `16-security.md` — general encryption-at-rest/in-transit policy; this file adds backup-specific key-custody
- `18-data-migration.md` — schema-versioning/rollback tooling; this file's pre-migration backup is a separate safety net
- `20-environment-strategy.md` — deploy rollback plan; this file's failover recovers from data/infra loss, not a bad release

