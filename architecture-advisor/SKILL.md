---
name: architecture-advisor
description: Use this skill whenever the person is making or discussing software architecture decisions - starting a new project and needing to decide its technical shape, evaluating or critiquing an existing system's structure, or asking a narrow comparison question about a specific technical approach (e.g. "monolith or microservices", "REST or GraphQL", "should this be its own service"). Trigger this even if the person doesn't say the word "architecture" - phrases like "how should I structure this", "what stack should I use", "is this over-engineered", "should I split this into services", or "help me design the backend for X" all qualify. Always consult this skill before giving architecture advice from general knowledge alone, since it encodes a specific decision process and format the person has asked for.
---

# Architecture Advisor

Helps the person make software architecture decisions - either walking through the full shape of a new project from scratch, or answering a single narrow architecture question mid-project. The process and output format matter as much as the content: this skill is as much about *how* the recommendation is delivered (tables, staged approval) as *what* is recommended.

## Step 0: Decide the mode

Before doing anything else, figure out which of these two modes applies:

- **Full walkthrough** - the person is starting a new project, or explicitly asks to think through "the architecture" broadly (e.g. "I'm building X, help me design the architecture", "let's figure out the whole stack for this"). Use the full clustered sequence below.
- **Narrow question** - the person asks about one specific decision (e.g. "should I use Redux here", "monolith or microservices for this", "what database should I use"). Skip straight to the single relevant reference file(s), answer only that, in table format (see "Table format" below). Do not run upfront questions or the cluster sequence. At the end, briefly offer: "Want to work through the rest of the architecture too, or just this piece?" - and only proceed further if they say yes.

Getting this distinction right matters: forcing the full sequence onto a narrow question is annoying and wastes the person's time; treating a full project like a narrow question means giving unfounded advice without context.

## For full walkthrough mode

### 1. Ask upfront questions, as one batch

Before recommending anything, gather context in a single round (not spread across many back-and-forth messages):

1. **Platform** - web / mobile (native or React Native) / desktop / CLI / other. Ask this first and explicitly, because it's a filter: it determines which later categories even apply (a CLI tool has no meaningful "frontend state management" decision; a mobile app has different deployment concerns than a web service).
2. **Team size and ownership** - solo, small team, or larger, and who owns which parts (relevant to Conway's Law - service boundaries work best when they mirror team boundaries, and splitting things up when one person owns everything usually just adds coordination overhead).
3. **Expected scale** - rough user/traffic expectations, now and anticipated growth.
4. **Ops/deployment capacity** - how much infrastructure complexity the person or team can realistically operate and maintain.
5. **Domain coupling** - how tightly related the core pieces of the system are to each other.
6. **Consistency requirements** - where strong consistency actually matters vs where eventual consistency is fine.
7. **Time-to-ship pressure** - how much runway there is before this needs to work.
8. **Budget/cost constraints** - this shapes decisions silently if not surfaced explicitly (e.g. why someone avoids a managed service, why self-hosting is or isn't viable).

These answers get applied as filters throughout every cluster below - don't just collect them and forget them.

### 2. Apply design principles as a lens, not a checkpoint

Throughout every recommendation, keep separation of concerns, SOLID, DRY, and appropriate use of layered/hexagonal/DDD-style thinking in mind as reasoning tools - e.g. flag it in the trade-off column if an option would violate separation of concerns, or if a "simple" option is simple specifically because it defers a concern that will need to be split out later. Don't present these principles as their own checkpoint or table row; they're background judgment applied while filling in every other row, not a category the person approves separately.

### 3. Work through clusters in order, one at a time

The 21 architecture categories are grouped into four clusters. Categories within a cluster can be reasonably recommended in parallel; a cluster as a whole should not be attempted until the previous cluster is approved, because later categories build on assumptions the earlier ones settle.

| Cluster | Categories | Reference files |
|---|---|---|
| **1 - Foundation** | System-level shape, Data storage, Backup & disaster recovery, File/media storage, Scalability | `references/01-system-level.md`, `references/02-data-storage.md`, `references/03-backup-disaster-recovery.md`, `references/04-file-storage.md`, `references/05-scalability.md` |
| **2 - Access & interface** | Auth, API design, Third-party integrations, AI integration | `references/06-auth.md`, `references/07-api-design.md`, `references/08-integrations.md`, `references/09-ai-integration.md` |
| **3 - Runtime behavior** | Messaging (conditional), Background jobs/scheduling, Frontend, Caching, Resilience (conditional) | `references/10-messaging.md`, `references/11-background-jobs.md`, `references/12-frontend.md`, `references/13-caching.md`, `references/14-resilience.md` |
| **4 - Operations** | Observability, Security, Testing strategy, Data migration & schema evolution, Deployment & infra, Environment & release strategy, Networking | `references/15-observability.md`, `references/16-security.md`, `references/17-testing-strategy.md`, `references/18-data-migration.md`, `references/19-deployment-infra.md`, `references/20-environment-strategy.md`, `references/21-networking.md` |

**Conditional categories:** Messaging is only relevant if Cluster 1's system-level decision produced multiple services or an async/event-driven need - if the system is a single deployable app, skip it with a one-line note why, don't write a full row. Resilience (retries, timeouts, circuit breakers) is thin or skippable for a simple monolith with no external calls - same treatment.

For each cluster:

1. Read the relevant reference file(s) for that cluster's categories.
2. Produce **one master decision table** covering every category in the cluster (format below).
3. Present it and stop - wait for the person's approval before generating the next cluster.
4. If anything is rejected, rework only that row (new options if warranted), and explicitly flag anything in *this* cluster or *later, not-yet-generated* clusters whose reasoning depended on the rejected decision. Never silently patch around a rejection or quietly carry forward a decision that no longer holds.
5. Once every row in the cluster is approved, move to the next cluster.

### 4. Table format (used in every mode, both full walkthrough and narrow question)

Every recommendation - whether one row in a cluster table or the single table for a narrow question - uses this format:

| Category | Option A (pros/cons) | Option B (pros/cons) | Option C (pros/cons) | Recommendation + why | Approve |
|---|---|---|---|---|---|
| e.g. System-level shape | Monolith: fast to ship, simple ops \| harder to scale teams independently | Modular monolith: clear internal boundaries, single deploy \| discipline required to keep modules decoupled | Microservices: independent scaling/deploy \| ops overhead, network complexity, needs multiple owners | Modular monolith - solo project, need speed now, can extract services later if team grows | ☐ |

Include every real alternative considered, not just the winner - the person wants to see what was ruled out and why before checking a box. Keep pros/cons to the ones that actually matter given the upfront-question answers, not a generic textbook list.

### 5. End state

Once all clusters (or as many as the person wants) are approved, offer a single consolidated summary document of every approved decision, organized by cluster, as the final artifact.

## For narrow question mode

Identify the one relevant reference file (or occasionally two, if the question spans a boundary - e.g. "should auth live in the API gateway or the service" touches both auth and networking). Produce a single table row in the format above. No upfront-question batch, no clustering, no multi-step approval chain - just the answer, in table form, with real alternatives shown.

## Reference files

This skill's content lives in `references/`, organized by category, numbered in dependency order (see cluster table above for what's in each). Each reference file should be read only when its category is actually relevant to the conversation - don't load files outside the current cluster or narrow question.

Categories intentionally **not** included as their own files, with reasoning (mention if the person asks why something's missing, don't volunteer unprompted):
- **Algorithms/data structures** - implementation-level, not system-shape; only becomes architectural when it needs dedicated infrastructure (e.g. "we need full-text search" is a system-level or background-jobs question, not an algorithm question).
- **Prompt engineering / system-prompt engineering** - crafting prompt wording, few-shot examples, chain-of-thought instructions, or system-prompt content is implementation-level craft applied within whichever AI-integration architecture `09-ai-integration.md` settles on - same treatment as algorithms/data structures above. The one adjacent structural question - whether the prompt is hardcoded or externalized as a versioned, independently-deployable artifact - is covered as a brief note in `09-ai-integration.md`, cross-referenced to `20-environment-strategy.md`'s config-management reasoning.
- **Process mining** - a business-analytics/BPM discipline (discovering and monitoring processes from event logs), not an AI-integration or system-shape architecture decision. Doesn't fit `09-ai-integration.md` or any other of the 21 reference files; out of scope for this skill entirely rather than force-fit somewhere.
- **Automation** - not a standalone concern; distributed across deployment-infra, background-jobs, and testing-strategy instead.
- **Multi-tenancy, compliance/data-residency, offline support/sync** - genuinely architectural when relevant, but not assumed relevant by default. Ask the person if their project needs these before treating them as in-scope; add ad hoc reasoning if so rather than forcing them into the numbered sequence.
- **MVP** - product-scoping concept, not a system-shape/architecture pattern. Out of scope for this skill.
- **SOA (Service-Oriented Architecture)** - microservices' predecessor, mostly seen in legacy enterprise contexts now. Not a full table column in `01-system-level.md`; worth a one-line mention there so it doesn't look like an oversight, but not a live default option.
- **Event-driven architecture** - a runtime-behavior/communication-style question, not a deployment-unit shape question; already covered by `10-messaging.md`.
- **Business-operation / business-process migration** - moving an organization's operational workflows to a new system, change management, staff retraining, is an organizational/business-transformation concern, not a software architecture decision. Flagged explicitly in `18-data-migration.md`'s scope note so it doesn't look like an oversight there.
- **Specific tool/vendor selection within a decided category** - e.g. DB client GUIs like DBeaver/DataGrip, specific AI provider/model choice (`09-ai-integration.md` covers hosted-vs-self-hosted as an architecture axis but not which provider), specific message broker products like Kafka/RabbitMQ/BullMQ (`10-messaging.md` covers broker properties but not which product), other developer-workflow tooling. Not an architecture decision, it's tooling/vendor choice, and moves faster than a project-agnostic reference file can track. Doesn't fit any of the 21 reference files. This is the seed for a possible *separate*, future skill that would pick specific tools/vendors based on decisions made by this skill - not part of `architecture-advisor` itself.
