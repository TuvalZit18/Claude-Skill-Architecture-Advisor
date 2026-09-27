# 01 — System-Level Shape

## What this category covers

The foundational structural decision: how the system is decomposed into deployable units, both on the backend and — using the same underlying trade-off — on the frontend. Every other category in every cluster assumes this decision has already been made, so it must be settled first.

This file covers two sub-decisions that share one reasoning lens:
1. **Backend shape** — monolith vs modular monolith vs microservices
2. **Frontend shape** — single frontend app vs micro-frontends

`12-frontend.md` (Cluster 3) does not repeat the frontend-shape reasoning — it cross-references back here and assumes the decision made in this file.

---

## Sub-decision 1: Backend System Shape

### The core trade-off

Team autonomy and independent deployability vs coordination overhead and operational complexity. This is fundamentally a Conway's Law question as much as a technical one — the org's team boundaries and the system's service boundaries tend to converge whether you plan for it or not.

### Options

**Monolith (single deployable unit, single codebase)**
- Pros: simplest to build, run, debug, and reason about; no network calls between internal components; easy transactional consistency; fastest to ship early; trivial local dev setup; refactoring across "boundaries" is just refactoring
- Cons: scales as one unit (can't scale hot paths independently); one team's bug/deploy can affect the whole app; large team = merge/ownership friction as codebase grows; harder to adopt different tech per component

**Modular monolith (single deployable unit, enforced internal module boundaries)**
- Pros: most of monolith's simplicity (single deploy, no network hops, easy consistency) while enforcing separation of concerns via internal boundaries (folder/package structure, dependency rules, sometimes internal APIs); de-risks a future microservices extraction if one becomes necessary; good middle ground for teams that expect to grow
- Cons: boundaries are conventions/tooling-enforced, not physically enforced — discipline can erode without linting/architecture tests; still scales and deploys as one unit; doesn't solve org-level team-autonomy needs if multiple teams must ship independently

**Microservices (multiple independently deployable services)**
- Pros: independent deployability and scaling per service; teams can own a service end-to-end including its own release cadence and (if needed) tech stack; failure isolation — one service crashing doesn't necessarily take down others
- Cons: significant operational overhead (service discovery, inter-service auth, distributed tracing, network reliability handling); distributed transactions/consistency become genuinely hard; local dev environment complexity multiplies; easy to get service boundaries wrong early and pay for it later; requires ops maturity (Cluster 4 categories) to run well — premature adoption is a common and expensive mistake

**Serverless / FaaS (function-level deployment unit)**
- Pros: maximal independent scaling and deployment — down to the individual function, not just the service; no server management (provider handles provisioning, patching, scaling to zero); pay-per-execution cost model can be significantly cheaper for spiky or low/irregular traffic; fastest path to production for event-triggered or lightweight API workloads; wide provider support (AWS Lambda, Vercel Functions, Cloudflare Workers, Azure Functions, Google Cloud Functions, and others)
- Cons: cold-start latency (varies by provider/runtime, but a real factor for latency-sensitive paths); execution time and resource limits per invocation; vendor lock-in tends to be higher than containers/VMs (provider-specific triggers, deployment tooling, local-dev parity is harder); debugging and local testing distributed function graphs is harder than a single running process; can get expensive at sustained high volume compared to always-on compute; orchestrating many functions with shared state/transactions across them re-introduces distributed-system complexity similar to microservices

**Microkernel (plug-in architecture)**
- Pros: high extensibility without modifying the core system — new functionality ships as plugins; enables third-party or team-contributed extensions without redeploying the core; clean separation between essential ("core") and optional ("plugin") functionality; core can stay small, stable, and well-tested while plugins iterate independently
- Cons: the core's plugin API becomes a long-term contract that's expensive to change once plugins depend on it; requires deliberate upfront design — hard to retrofit onto a system not built this way from the start; plugin isolation/sandboxing adds its own complexity if plugins need to be untrusted or fault-isolated; not a fit for typical CRUD/product apps — this pattern earns its cost specifically when extensibility *is* the product (IDEs, browsers with extensions, CMSs, CLI tools with plugin ecosystems)

### The distributed-monolith anti-pattern

Worth flagging explicitly in the recommendation whenever microservices is being considered: splitting into multiple services while keeping them tightly coupled (shared database, synchronous call chains where any one service being down breaks the others, no independent deploy in practice because of implicit ordering dependencies) yields all the operational cost of microservices with none of the isolation or independent-deployability benefit. If the services can't actually be deployed and scaled independently in practice, it isn't really microservices yet — it's a monolith with extra network hops and extra failure modes.

### Service boundary guidance (when microservices or modular monolith module boundaries are being defined)

- Boundaries should follow business/domain capabilities (bounded contexts, DDD-style), not technical layers (a "database service" or "validation service" is usually a smell)
- A boundary is a candidate service boundary if it has its own data it owns exclusively, changes for its own reasons, and could plausibly be owned by one team
- Prefer starting with fewer, coarser services and splitting further only when a real scaling or team-ownership need forces it — the cost of merging two services later is much lower than the cost of prematurely splitting a domain that wasn't actually separable

### What drives the recommendation (from upfront questions)

- **Team size / ownership boundaries**: single small team → monolith or modular monolith almost always wins; multiple teams needing independent release cadence → microservices becomes a real candidate
- **Ops/deployment capacity**: low ops maturity or no dedicated infra capacity → strongly weights against microservices regardless of team size, since the operational cost will dominate
- **Expected scale**: uniform, moderate scale → monolith/modular monolith fine; a small number of components with wildly different scaling needs (e.g. one heavy compute path, rest lightweight) → a case for splitting just that component out, not necessarily going full microservices
- **Time-to-ship pressure**: high pressure → monolith or modular monolith, virtually always; microservices' upfront cost rarely pays off before an early deadline
- **Domain coupling**: highly coupled, unclear domain boundaries (common pre-product-market-fit) → monolith/modular monolith, since premature service splits along wrong boundaries are expensive to undo
- **Traffic pattern** (spiky/irregular vs sustained/high-volume): spiky or low/irregular traffic → serverless/FaaS becomes attractive on cost grounds; sustained high volume → always-on compute (monolith/microservices) usually cheaper and avoids cold-start latency
- **Extensibility as a product requirement** (does the system need to support plugins/third-party extensions as a core feature, not just internal modularity): if yes → microkernel becomes a real candidate; if no → it should almost always be excluded from the shown table rather than forced in

### Recommendation table skeleton

| Category | Option A: Monolith | Option B: Modular Monolith | Option C: Microservices | Option D: Serverless/FaaS | Option E: Microkernel | Recommendation + why | Approve |
|---|---|---|---|---|---|---|---|
| System-level shape (backend) | *filtered pros/cons from above given upfront answers* | *filtered pros/cons* | *filtered pros/cons* | *filtered pros/cons* | *filtered pros/cons* | *pick one, justify against team size / ops capacity / scale / time pressure / extensibility needs* | ☐ |

Note: not every project's table needs to show all 5 columns — Microkernel in particular is a narrow, situational fit (extensibility-as-the-product) and may be reasonably omitted from the shown table entirely for projects where it's clearly irrelevant, rather than forcing a column that will obviously be rejected.

---

## Sub-decision 2: Frontend Shape (Single App vs Micro-Frontends)

### The core trade-off

Identical shape to the backend decision — team autonomy and independent deploys vs coordination and integration overhead — just applied to the presentation layer instead of the service layer.

### Options

**Single frontend application**
- Pros: simplest build/deploy pipeline; shared state, routing, and design system are trivial by default; no runtime integration complexity; smallest bundle-size overhead; easiest to keep UX consistent
- Cons: one team's release can block or risk another team's shipped work if ownership isn't cleanly modularized in-app; large single-team-or-multi-team codebases can develop the same ownership friction as a backend monolith

**Micro-frontends (independently deployed frontend pieces composed at runtime or build time)**
- Pros: independent deployability per team/feature area; teams can in principle use different frameworks/versions per piece (rarely advisable, but possible); scales team autonomy the same way microservices does for backend teams
- Cons: real integration complexity (shared design system/state/auth across independently-deployed pieces is hard to keep consistent); runtime composition approaches (module federation, iframes, web components) each carry their own performance and DX costs; multiplies bundle size and duplicated dependencies unless carefully managed; significantly higher up-front tooling investment; only pays off when there are genuinely multiple teams that need independent release cadence on the frontend specifically

### What drives the recommendation

Same upfront-question drivers as backend shape, but applied specifically to frontend team structure: does the frontend have (or will it soon have) multiple teams that need to ship independently of each other? If there's one frontend team (even if the backend is split into microservices), a single frontend app is almost always still correct — micro-frontends solve a frontend team-topology problem, not a backend architecture problem, and shouldn't be adopted just because the backend went microservices.

### Recommendation table skeleton

| Category | Option A: Single frontend app | Option B: Micro-frontends | Recommendation + why | Approve |
|---|---|---|---|---|
| System-level shape (frontend) | *filtered pros/cons* | *filtered pros/cons* | *pick one, justify against frontend team structure specifically* | ☐ |

---

## Related files

- `12-frontend.md` — assumes frontend shape decided here, no re-litigation
- `05-scalability.md` — scaling techniques depend on shape chosen here
- `19-deployment-infra.md` — monorepo vs polyrepo follows from this shape
- `21-networking.md` — load balancer/service mesh needs assume this topology
