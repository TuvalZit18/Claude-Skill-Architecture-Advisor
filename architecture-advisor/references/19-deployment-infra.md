# 19 — Deployment & Infrastructure

## What this category covers

Compute/hosting model, cloud provider choice, CI/CD pipeline investment, and monorepo vs polyrepo. This is also the file several earlier files defer to: `16-security.md`'s CI/CD-integrated vulnerability scanning and software/data-integrity concerns, `17-testing-strategy.md`'s automated test execution, `15-observability.md`'s release tracking, and `18-data-migration.md`'s cloud-provider/data-center migration and hardware-upgrade scope.

Distinct from `01-system-level.md`: that file decides the *deployable-unit shape* (monolith, modular monolith, microservices, serverless, microkernel) — a decision about how the code is decomposed. This file decides how whatever was decided there actually *runs and ships* — the hosting mechanism, the pipeline, and the repository structure. If `01-system-level.md` already settled on serverless/FaaS, this file's compute-model axis is largely already answered by that choice.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Compute / hosting model

Relevant once `01-system-level.md`'s deployable-unit shape is settled and isn't already serverless/FaaS (which is itself a compute model, not just a code-shape decision) — this axis is about how a monolith, modular monolith, or set of microservices actually gets hosted and run.

### Containers (e.g. Docker, orchestrated via Kubernetes or similar)

- **Pros:** a consistent, portable runtime environment across dev, staging, and production — "it works the same everywhere it's run," reducing environment-drift bugs; efficient resource utilization compared to VMs, since containers share the host OS kernel rather than each needing a full OS; the dominant, most widely-supported approach for microservices specifically, with a large ecosystem of tooling
- **Cons:** real operational complexity, especially with a full orchestrator — a genuine skill and ops investment to run well; overkill for a simple, single-deployable monolith that doesn't need per-service scaling or orchestration

### Virtual machines

- **Pros:** the simplest, most familiar model — closest to "a regular server," well understood, and works with essentially anything without adaptation
- **Cons:** heavier resource overhead than containers, since each VM carries a full OS; slower to provision and scale than containers; environment consistency between dev and production is harder to guarantee without additional configuration-management tooling layered on top

### PaaS (Platform-as-a-Service)

- **Pros:** minimal ops burden — push code, and the platform handles provisioning, scaling, and much of the underlying infrastructure; the fastest path to a running production deployment, especially valuable for a small team without dedicated ops capacity
- **Cons:** less control over the underlying infrastructure and its configuration; can become expensive at scale compared to self-managed containers or VMs; deeper vendor lock-in than a portable container image

If `01-system-level.md` already settled on serverless/FaaS, this axis is largely moot — FaaS platforms are themselves the compute model in that case. This axis is most live for a monolith, modular monolith, or microservices shape that still needs a hosting decision made on top of it.

---

## Axis 2: Cloud provider / hosting choice

### Major cloud provider (broad, general-purpose platform)

- **Pros:** the broadest service ecosystem, the most mature tooling, and the largest talent pool familiar with the platform — generally the "everything is available" option
- **Cons:** real pricing complexity and potential for unexpectedly high bills without careful management; vendor lock-in risk grows with each provider-specific managed service adopted, echoing the vendor-dependency reasoning in `08-integrations.md`

### Smaller / specialized provider (a narrower, more opinionated platform)

- **Pros:** often simpler pricing and a more curated, opinionated set of choices, which can mean faster time-to-ship for a team that doesn't need a major provider's breadth; sometimes better developer experience for a specific niche (static sites, simple APIs)
- **Cons:** a narrower service ecosystem — may not have the specific managed service a project needs, forcing a workaround or a hybrid setup; typically a lower scale ceiling than a major provider before hitting real limits

### Self-hosted / on-premises

- **Pros:** full control over infrastructure and data location, which can matter directly for `16-security.md`'s data-residency and compliance considerations; potentially lower cost at sustained high scale compared to cloud, since there's no cloud margin being paid
- **Cons:** the team bears full responsibility for hardware, physical security, networking, and every layer cloud providers otherwise manage — a substantial ops burden; no elastic scaling the way cloud offers, so capacity has to be provisioned ahead of actual need rather than scaled on demand

---

## Axis 3: CI/CD pipeline investment

### Minimal / manual pipeline (deploy scripts run by hand, or a very thin automation layer)

- **Pros:** simplest to set up initially — nothing to learn beyond running a script
- **Cons:** deploys are inconsistent and error-prone when done by hand under time pressure; no automatic gate stopping a broken build or failing tests from reaching production; doesn't scale as team size or deploy frequency grows

### Full CI/CD pipeline (automated build, test, security scan, and deploy on every commit/merge)

- **Pros:** consistent, repeatable deploys with automated gates — the concrete home for `17-testing-strategy.md`'s automated test suite and `16-security.md`'s vulnerability/dependency scanning to actually run on every change; enables the deploy frequency and confidence that makes the rollout strategies in `20-environment-strategy.md` (canary, blue-green) practical in the first place
- **Cons:** a real setup and maintenance investment — pipeline configuration is itself a piece of infrastructure that needs to be built, debugged, and kept working; a slow or flaky pipeline becomes its own source of friction if not maintained well

A full CI/CD pipeline is close to the standard expectation for anything beyond a solo or early-stage project — the automated gates it provides are what make every other file's automated-testing and security-scanning investments actually pay off in practice, rather than existing as unused capability.

**Release tracking & artifact integrity:** worth a one-line note rather than a separate axis — tagging each deployed build with a unique, traceable version or release identifier is what makes `15-observability.md`'s error tracking able to tie a given error back to the specific release that introduced it, and what `16-security.md`'s software/data-integrity concern (ensuring only signed, verified build artifacts get deployed) actually depends on. Close to a required companion to a full CI/CD pipeline rather than a standalone decision.

---

## Axis 4: Monorepo vs polyrepo

**If `01-system-level.md` settled on a monolith or modular monolith:** monorepo is essentially automatic — there's one deployable unit, so there's naturally one repository containing it. Not a genuine decision worth a table row; state it as a one-line consequence rather than presenting alternatives that don't meaningfully apply.

**If `01-system-level.md` settled on microservices:** this becomes a genuine, live decision.

### Polyrepo (each service in its own repository)

- **Pros:** the traditional default for microservices, and a clean match for independent deploy and ownership — each team or service has full autonomy over its own repo, versioning, and release cadence, with no risk of one service's changes accidentally touching another's code; access control can be scoped per-repo, cleanly matching team boundaries
- **Cons:** cross-service changes — a shared library update, a coordinated API contract change spanning several services — require coordinating changes across multiple repos and PRs, real friction; harder to enforce consistent tooling and standards across services without deliberate extra effort, since each repo can drift independently; understanding how services relate to each other is harder when their code isn't visible in one place

### Monorepo (all services in one repository)

- **Pros:** cross-service changes are a single PR or commit spanning multiple services, removing the coordination overhead polyrepo requires; consistent tooling, linting, and dependency versions can be enforced uniformly across all services from one place; easier to see and understand how the whole system fits together, since everything is in one place
- **Cons:** requires real investment in tooling built for monorepos at scale — a build system that only rebuilds and retests what actually changed, rather than the whole repo every time — or the repo becomes slow and unwieldy as it grows; weaker natural boundary enforcement between services' independence, since it's easier for accidental coupling to creep in when everything is physically adjacent, even if logically services are still meant to be independent; per-service access control is harder to scope tightly than polyrepo's natural repo-level boundaries

Polyrepo remains the traditional and more common default for microservices, and is the lower-friction choice absent a specific reason to centralize. Monorepo is a legitimate, increasingly common alternative specifically when strong cross-service tooling and versioning consistency matter more than deploy independence and repo-level access boundaries — a deliberate trade of some team autonomy for cross-cutting consistency and coordination ease, not a default to reach for without that specific need.

---

## What drives the recommendation (from upfront questions)

- **System-level shape** (from `01-system-level.md`): the direct gate for whether Axis 4 is even a real decision — a monolith collapses it to a non-decision; microservices makes it live. Also determines whether Axis 1 is live at all, per that axis's serverless carve-out
- **Team size / ownership boundaries**: multiple teams needing independent release cadence strengthens polyrepo's case (Axis 4) even within a microservices monorepo setup, unless the team is prepared to invest in the tooling that makes monorepo coordination costs manageable at scale
- **Ops/deployment capacity**: low ops maturity weights toward PaaS (Axis 1) over containers or VMs, toward a major cloud provider with managed services (Axis 2) over self-hosted, and toward relying on a CI platform's built-in pipeline templates over building extensive custom CI/CD tooling
- **Domain/compliance context**: data-residency or full-infrastructure-control requirements push toward self-hosted (Axis 2) — but per the skill's excluded-categories note, don't assume this applies; ask if it's relevant to the specific project
- **Time-to-ship pressure**: high pressure → PaaS or a major cloud provider's simplest managed offering, a full but simple CI/CD pipeline using common templates rather than bespoke tooling, and monorepo-by-default for a monolith, is a legitimate first version
- **Expected scale**: sustained high scale is where self-hosted's cost advantage (Axis 2) and containers/VMs' greater control (Axis 1) start to genuinely pay off over PaaS's simplicity — though PaaS remains a reasonable choice for many projects even at real scale

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| Compute/hosting model | *Containers pros/cons* | *VMs pros/cons* | *PaaS pros/cons* | *justify against ops capacity and `01-system-level.md`'s shape; note this collapses if serverless was already chosen there* | ☐ |
| Cloud provider choice | *Major provider pros/cons* | *Smaller/specialized provider pros/cons* | *Self-hosted/on-prem pros/cons* | *justify against compliance/data-residency needs, ops capacity, and cost at scale* | ☐ |
| CI/CD pipeline investment | *Minimal/manual pros/cons* | *Full automated pipeline pros/cons* | — | *full pipeline is close to the standard expectation beyond a solo/early project; note release tracking and artifact integrity as required companions* | ☐ |
| Monorepo vs polyrepo | *Polyrepo pros/cons* | *Monorepo pros/cons* | — | *collapses to a one-line non-decision for a monolith (monorepo automatic); for microservices, polyrepo is the traditional default, monorepo a deliberate trade for cross-service tooling consistency* | ☐ |

Note: for a small, early-stage monolith under time pressure, this can reasonably collapse to two rows ("PaaS or a major cloud provider's managed offering, monorepo automatic" and "a simple CI/CD pipeline using common templates") rather than four.

---

## Related files

- `01-system-level.md` — the gating decision for compute model (Axis 1, collapses if serverless was already chosen) and for monorepo vs polyrepo (Axis 4, collapses to a non-decision for a monolith)
- `08-integrations.md` — vendor lock-in reasoning for cloud provider choice (Axis 2) echoes that file's general integration-coupling reasoning
- `09-ai-integration.md` — self-hosting an AI model, or standing up MCP servers, is a specific instance of this file's compute/hosting decisions
- `15-observability.md` — error tracking tying errors to a specific release depends on this file's release-tracking practice
- `16-security.md` — the CI/CD pipeline (Axis 3) is where vulnerability/dependency scanning and software/data-integrity (signed artifacts) actually run
- `17-testing-strategy.md` — the CI pipeline (Axis 3) is where all of that file's testing layers get wired in and run automatically; installation testing there ties to self-hosted deployment (Axis 2)
- `18-data-migration.md` — cloud-provider/data-center migration and hardware upgrades are this file's territory, out of that file's schema-evolution scope
- `20-environment-strategy.md` — rollout strategy (canary/blue-green/rolling) depends on having a full CI/CD pipeline (Axis 3) in place to execute it
- `21-networking.md` — the compute model chosen here (Axis 1) shapes what the networking layer needs to route to and secure
