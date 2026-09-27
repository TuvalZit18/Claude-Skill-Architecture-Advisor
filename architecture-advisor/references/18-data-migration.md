# 18 — Data Migration & Schema Evolution

## What this category covers

Safely changing the data model once a system is live: migration tooling and versioning, backward-compatibility strategy during a schema change, how existing data gets backfilled to match a new schema, and rollback strategy specific to a bad migration.

This is the concrete home for the "schema-versioning/rollback tooling" `03-backup-disaster-recovery.md` explicitly distinguishes from its own pre-migration-backup safety net — that file's backup is the fallback of last resort; this file covers the faster, more surgical recovery path for when the migration itself turns out to be the problem.

Distinct from `20-environment-strategy.md`'s deploy rollback: that reverts a bad *application release*; this file's rollback reverts a bad *data-model change*. The two often need to work together (a bad deploy that included a migration may need both), but they're separate mechanisms covering separate failure modes.

**Scope note on migration scope/subject:** this file covers evolving the data model *within* an already-chosen storage technology and platform (`02-data-storage.md`, `19-deployment-infra.md`) — adding a column, restructuring a table, changing a relationship — plus in-place version upgrades of that same technology (Axis 5 below). It does not cover wholesale platform migrations: swapping database engines entirely, on-prem-to-cloud or cloud-to-cloud adoption, moving into/out of a data center, or replacing/replatforming an entire application or legacy system onto a different stack. Those are fundamentally larger projects that effectively redo earlier decisions (`02-data-storage.md`'s storage model, `19-deployment-infra.md`'s compute/cloud choice, or `01-system-level.md`'s system shape for a full system replacement) rather than evolving within them. That said, the *cutover strategy* reasoning in Axis 2 below — big bang vs parallel run vs incremental/trickle — is the same underlying pattern at any of these scales, just with far more involved mechanics the larger the migration gets; it's worth reapplying this file's Axis 2 thinking to a platform-level migration even though this file doesn't own the storage/platform decision itself. Moving files or blobs between storage providers specifically is closer to `04-file-storage.md`'s territory; hardware refreshes and non-data-bearing infrastructure upgrades are `19-deployment-infra.md`'s territory.

**Out of scope entirely:** business-operation or business-process migration — moving an organization's operational workflows to a new system, change management, staff retraining — is an organizational/business-transformation concern, not a software architecture decision. Same treatment as process mining and MVP in `SKILL.md`'s excluded-categories list.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Migration tooling & versioning approach

### Manual / ad hoc scripts (no dedicated tooling)

- **Pros:** no additional tooling to learn or maintain; full, direct control over exactly what runs
- **Cons:** no built-in tracking of which migrations have already been applied to a given environment — easy to lose track, or apply the same script twice, or apply them out of order; no automatic rollback capability; doesn't scale to a team where multiple people are making schema changes concurrently without real coordination risk

### Migration framework with versioned migrations

- **Pros:** tracks exactly which migrations have been applied to which environment, preventing double-application or missed migrations; migrations are version-controlled alongside application code, giving a clear, ordered history of schema changes over time; most support at least a basic rollback/down-migration capability out of the box
- **Cons:** an additional tool and convention the team needs to learn and consistently follow; auto-generated migrations, where the tool infers the change from model diffs, can sometimes produce suboptimal or even incorrect SQL for complex changes, requiring manual review rather than blind trust in the generated output

A versioned migration framework is close to the standard default for any relational schema beyond a trivial project — the tracking and ordering guarantees it provides are hard to safely replicate with ad hoc scripts once more than one person is touching the schema.

---

## Axis 2: Migration cutover strategy

How the transition from old to new is actually executed — a decision that applies at any scale, from a single schema change up to the platform-level migrations flagged in the scope note above.

### Big bang (coordinated single-event cutover)

- **Pros:** simplest to reason about — the new schema and the code that expects it go live at the same instant, with no intermediate state to design for
- **Cons:** requires genuine downtime, or a carefully-coordinated simultaneous deploy, to avoid a window where running application code doesn't match the live schema — a real risk in any horizontally-scaled deployment (`05-scalability.md`) where a rolling deploy means old and new code run side by side for a period; a failed or partial cutover can leave the system in a broken, mismatched state with no gradual fallback

### Parallel run / dual-write (old and new run simultaneously, outputs compared, before full cutover)

- **Pros:** the lowest-risk option of the three — the old schema or system keeps serving as the source of truth while the new one is validated against real production data and traffic without yet being depended on; discrepancies between old and new get caught and fixed before the new path is ever fully trusted, rather than discovering a bug only after cutover
- **Cons:** real additional complexity and cost — running two systems, or writing to two schemas, simultaneously, plus a verification mechanism to actually confirm they agree; doubles write cost and operational complexity for the duration of the run; still needs real discipline to actually complete the cutover and decommission the old path, rather than leaving the parallel run going indefinitely

### Incremental / trickle (expand/contract: phased migration over multiple small steps)

- **Pros:** enables zero-downtime changes even with a horizontally-scaled, rolling deploy — trickles the change through in phases so old and new can coexist safely at every point along the way; each phase is independently small and reversible, reducing the blast radius of any single step going wrong
- **Cons:** significantly more steps and calendar time than a single cutover — one logical change becomes multiple deploys (expand, migrate/backfill, contract); requires real discipline to actually complete the final phase and remove the old structure, rather than leaving both indefinitely — a common real-world failure mode that leaves permanent, unnecessary cruft behind

Big bang is the simplest and lowest-effort choice, appropriate when brief downtime or a coordinated deploy is genuinely tolerable. Parallel run is the safest choice for the highest-stakes migrations, where validating correctness against real data before fully committing matters more than speed or simplicity. Incremental/trickle is the standard choice for anything that needs zero downtime and can't accept a parallel run's doubled operational cost. These aren't strictly mutually exclusive at larger scale either — a real platform-level migration might use parallel-run validation on the riskiest subset of data while trickling the bulk of the migration incrementally.

---

## Axis 3: Data backfill strategy

### Synchronous backfill (in the same migration/transaction as the schema change)

- **Pros:** simplest — one step, and the data is consistent with the new schema immediately
- **Cons:** for a large table, this can lock the table or run for an extended period, which is itself a form of downtime or degraded availability; risky as an all-or-nothing operation — a failure partway through can leave things in a partially-migrated state that's hard to reason about

### Incremental / batched background backfill

- **Pros:** avoids long-running locks or extended unavailability by processing the backfill in small batches over time, reusing the same background-job infrastructure already covered in `11-background-jobs.md`; can be paused, monitored, and resumed rather than being an all-or-nothing operation
- **Cons:** the table is in a mixed state — some rows migrated, some not — for the duration of the backfill, which application code needs to handle correctly during that window; more moving parts, and a longer overall timeline to fully complete the migration

For any table with real production data volume, incremental/batched backfill is the safer default. Synchronous, single-transaction backfills are reasonable only for genuinely small tables where the lock duration is a non-issue.

---

## Axis 4: Rollback strategy

This is the concrete home for the schema-versioning/rollback tooling `03-backup-disaster-recovery.md` explicitly distinguishes from its own pre-migration-backup safety net.

### No explicit rollback plan (roll forward only, or restore from backup)

- **Pros:** simplest — no reverse migration to design, write, or test for every forward change
- **Cons:** recovering from a bad migration means either fixing forward under pressure (writing and deploying a corrective migration quickly, with the incident still live) or falling back to `03-backup-disaster-recovery.md`'s backup restore, which loses any writes made since the backup was taken and is a much heavier, slower operation than a targeted rollback

### Explicit down-migrations / reverse migrations

- **Pros:** provides a tested, fast path back to the previous schema state without needing a full backup restore — meaningfully faster recovery than the backup-restore safety net, specifically for the case where the migration itself was the problem and the rest of the data is fine
- **Cons:** writing and testing a correct reverse migration for every forward migration is real additional work, and some migrations aren't cleanly reversible at all — a column drop that already discarded data, or a data transformation that isn't invertible, genuinely can't always be undone by a down-migration

Not every migration needs a tested down-migration — it's most valuable for the riskiest changes, while a backup restore (`03-backup-disaster-recovery.md`) remains the fallback of last resort regardless of whether a specific migration has one. This axis's rollback is a faster, more surgical complement to that file's backup safety net, not a replacement for it.

---

## Axis 5: In-place version upgrades (of an already-chosen, data-bearing technology)

Distinct from both this file's schema-evolution focus and the platform-replacement scope excluded above: upgrading the *version* of an already-chosen data-bearing technology — most commonly a database engine's major version — without changing the underlying technology itself. Far more common in practice than a full platform swap, and generally lower-risk, but still a real decision with its own trade-offs. Non-data-bearing version upgrades (an OS, a compute runtime, a framework with no persistent data of its own) are `19-deployment-infra.md`'s territory rather than this file's.

### In-place upgrade (upgrade the running instance directly)

- **Pros:** simplest — one instance, upgraded where it stands, with no parallel infrastructure to stand up
- **Cons:** typically requires downtime for the duration of the upgrade process, and a failed upgrade can leave the system in a broken, half-upgraded state, with recovery often harder than a clean restore from backup (`03-backup-disaster-recovery.md`)

### Blue-green / parallel-instance upgrade (a new, upgraded instance runs alongside the old one, cutover once verified)

- **Pros:** the old instance keeps running and serving traffic while the new, upgraded instance is provisioned and verified — genuinely low-risk, since cutover only happens once the new version is confirmed working, and reverting is as simple as pointing traffic back at the still-running old instance; the same underlying pattern as this file's Axis 2 parallel-run strategy, applied specifically to a version upgrade rather than a schema change
- **Cons:** requires double the infrastructure temporarily, plus a mechanism to keep the new instance's data in sync with the old one during the verification window (often via replication) — real additional complexity and cost compared to a straightforward in-place upgrade

Blue-green is the standard recommendation for any upgrade where downtime is genuinely unacceptable or the upgrade carries real risk (a major version bump with known breaking changes); in-place upgrade remains reasonable for lower-stakes upgrades where a maintenance window is tolerable.

---

## Multi-source data consolidation

Worth a dedicated note rather than folding into the axes above — merging data from multiple previously-separate systems into one raises two challenges the single-system migration axes above don't address:

- **Identity resolution:** the same real-world entity (a customer, a product) often exists independently in each source system, frequently with different identifiers and no natural, shared key connecting them. Resolving this — matching records across sources that represent the same thing — is a genuine data-modeling problem that has to be solved before consolidation, not an afterthought; getting it wrong produces duplicate or split records that are expensive to untangle later.
- **Conflict resolution:** when the same entity's data disagrees between sources (two different addresses on file for the same customer), a deliberate policy is needed for which source wins — the most recently updated record, a designated source-of-truth system, or manual review for high-value conflicts — rather than an implicit, undocumented default.

This file's Axis 2 (cutover strategy) still applies to how the consolidated system is rolled out once these problems are solved — parallel run is particularly well-suited to multi-source consolidation specifically, since validating the consolidated data against the still-running original sources before fully committing is exactly the scenario parallel run protects against.

Mergers and acquisitions are a common real-world trigger for this pattern, worth naming as context — though the M&A process itself (organizational, legal, business-side work) is out of this skill's scope entirely, the same treatment `SKILL.md` gives business-operation transfer.

---

## What drives the recommendation (from upfront questions)

- **Consistency requirements / uptime expectations**: systems that can't tolerate downtime are the direct driver for incremental/trickle (Axis 2) and incremental backfill (Axis 3) over big bang or synchronous approaches; genuinely high-stakes migrations where correctness matters more than speed push toward parallel run instead
- **System-level shape / scalability** (from `05-scalability.md`): a horizontally-scaled system doing rolling deploys essentially requires incremental/trickle — big bang assumes a deploy model that doesn't hold once multiple instances run different code versions simultaneously during rollout
- **Migration stakes / blast radius**: the direct driver for choosing parallel run over the other two — a migration where a subtle correctness bug would be very costly (financial data, anything hard to reverse) justifies parallel run's doubled cost specifically to validate against real data before committing, in a way a routine, low-stakes schema tweak doesn't
- **Expected scale / data volume**: large tables are the direct driver for incremental/batched backfill (Axis 3) over a synchronous in-transaction backfill; small tables or low data volume can reasonably take the simpler synchronous approach
- **Ops/deployment capacity**: low ops maturity weights toward a versioned migration framework (Axis 1 — lower long-term risk despite the initial learning curve) and toward accepting a big-bang/maintenance-window approach (Axis 2) if genuine downtime is tolerable, rather than building out incremental/trickle or parallel-run discipline
- **Time-to-ship pressure**: high pressure → a versioned migration framework with straightforward big-bang changes, accepting brief downtime or a maintenance window if the project can tolerate it, is a legitimate first version — incremental/trickle and parallel run's added process overhead are best adopted once zero-downtime or migration-correctness stakes actually demand them, not by default
- **Pending version upgrades / multi-source consolidation**: an approaching major-version upgrade of a data-bearing technology is the direct trigger for Axis 5; a project combining data from previously-separate systems (commonly following a merger or acquisition, or simply consolidating redundant internal systems) needs the identity-resolution and conflict-resolution reasoning in "Multi-source data consolidation" above addressed before Axis 2's cutover-strategy choice is even relevant

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| Migration tooling | *Manual/ad hoc scripts pros/cons* | *Versioned migration framework pros/cons* | — | *versioned framework is close to the standard default beyond a trivial project* | ☐ |
| Migration cutover strategy | *Big bang pros/cons* | *Parallel run/dual-write pros/cons* | *Incremental/trickle pros/cons* | *big bang for tolerable-downtime cases; parallel run for high-stakes correctness; incremental/trickle for zero-downtime, horizontally-scaled systems (`05-scalability.md`)* | ☐ |
| Backfill strategy | *Synchronous pros/cons* | *Incremental/batched background pros/cons* | — | *justify against table size/data volume; incremental is the safer default at real production scale* | ☐ |
| Rollback strategy | *No explicit rollback pros/cons* | *Explicit down-migrations pros/cons* | — | *justify against migration risk; not every migration needs one, backup restore remains the fallback of last resort regardless* | ☐ |
| Version upgrade approach | *In-place pros/cons* | *Blue-green/parallel-instance pros/cons* | — | *only relevant when upgrading an existing data-bearing technology's version; blue-green for genuinely risky/downtime-sensitive upgrades* | ☐ |

Note: for a small, early-stage project with tolerable downtime, this can reasonably collapse to two rows ("versioned migration framework with big-bang changes" and "synchronous backfill, no explicit rollback beyond backup restore") rather than five — incremental/trickle, parallel run, and the version-upgrade row are worth adopting once uptime requirements, migration stakes, or an actual pending upgrade demand them, not by default.

---

## Related files

- `02-data-storage.md` — migration tooling and approach depend heavily on the storage model chosen there; relational schemas need this file's full machinery far more than schema-flexible document stores do; a full database-engine swap is out of this file's scope and effectively redoes that file's decision
- `03-backup-disaster-recovery.md` — pre-migration backup is the safety net of last resort; this file's rollback (Axis 4) is the faster, more surgical complement, not a replacement
- `04-file-storage.md` — moving files/blobs between storage providers is closer to that file's territory than this one's schema-evolution scope
- `05-scalability.md` — horizontally-scaled, rolling-deploy systems are the direct driver toward incremental/trickle (Axis 2)
- `11-background-jobs.md` — incremental/batched backfill (Axis 3) reuses that file's job infrastructure
- `19-deployment-infra.md` — cloud-provider or data-center migration is out of this file's scope and effectively redoes that file's compute/cloud decision; this file's Axis 2 cutover-strategy reasoning still applies at that larger scale; non-data-bearing infrastructure/hardware upgrades belong there, not this file's Axis 5
- `01-system-level.md` — a full system replacement/legacy modernization effectively redoes that file's system-shape decision; out of this file's scope beyond its Axis 2 cutover-strategy reasoning still applying
- `20-environment-strategy.md` — deploy rollback (a bad release) is distinct from this file's migration rollback (a bad data-model change); rollout strategy there (canary/blue-green/rolling) interacts directly with whether incremental/trickle is actually needed
