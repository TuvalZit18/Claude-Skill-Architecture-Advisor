# 07 — API Design

## What this category covers

The shape of the interface between clients and the backend (or between services): REST vs GraphQL vs RPC/tRPC as the core paradigm, how the API is versioned over time, and how public-facing surfaces are separated from internal-only ones.

Distinct from `06-auth.md`: that file decides how identity and permissions are established; this file decides the shape of the API surface those checks are applied to. Auth scheme does influence API design in places (e.g. token-in-header vs cookie auth affects whether an API is easily callable from non-browser clients) but the two are separate decisions.

Distinct from `08-integrations.md`: that file covers *calling out* to third-party APIs the project doesn't control; this file covers designing the API *this* project exposes to its own clients or other services.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: API paradigm

### REST

- **Pros:** the most widely understood paradigm — lowest onboarding cost for most teams and consumers; maps naturally onto HTTP semantics (verbs, status codes, caching via standard HTTP caching); mature tooling ecosystem (OpenAPI/Swagger for documentation and client generation, broad framework support); works well when resources map cleanly onto a CRUD-shaped domain
- **Cons:** over-fetching/under-fetching is a real cost — a client often gets more or less data than it needs per endpoint, sometimes requiring multiple round-trips to assemble a view; versioning (Axis 2) has to be handled explicitly since REST has no built-in schema evolution story; deeply nested or highly relational data can produce awkward endpoint shapes

### GraphQL

- **Pros:** clients request exactly the fields they need in a single query, solving REST's over/under-fetching problem directly — particularly valuable when different clients (web, mobile, third-party) need meaningfully different data shapes from the same underlying resources; a single endpoint with a strongly-typed schema that serves as living documentation and enables strong client tooling (codegen, introspection); schema can evolve additively without the versioning churn REST often needs
- **Cons:** significant upfront complexity — resolver design, the N+1 query problem (needing a batching layer like DataLoader), and query complexity/depth limiting to prevent abusive queries; HTTP-level caching doesn't work the way it does for REST (most GraphQL traffic is POST requests to one endpoint), requiring a different caching strategy (`13-caching.md`); steeper learning curve for teams unfamiliar with it; easy to accidentally expose more of the data graph than intended without deliberate access-control design at the resolver level

### RPC / tRPC

- **Pros:** the most direct mapping between client and server — calling a function feels like calling a local function, with minimal ceremony; tRPC specifically (TypeScript-first) gives end-to-end type safety between client and server without a separate schema definition language or codegen step, which is a significant DX win in a full-stack TypeScript project; lower overhead than REST or GraphQL for straightforward request/response calls that don't need REST's resource semantics or GraphQL's flexible querying
- **Cons:** tRPC's end-to-end type-safety benefit is specifically a same-language (TypeScript client + TypeScript server) win — it doesn't help a non-TypeScript client at all, making it a poor fit for a genuinely public API consumed by unknown third parties; RPC generally couples client and server more tightly than REST's resource-oriented contract, making independent evolution harder; less standardized tooling ecosystem than REST for cross-language interoperability

Note: gRPC (a separate RPC framework, protocol-buffer-based, common for service-to-service communication rather than client-facing APIs) is worth a brief mention here as the choice for internal, high-performance service-to-service calls specifically — strongly-typed contracts and binary serialization give it a real performance edge over REST/GraphQL for internal traffic, but it's a poor fit for public/browser-facing APIs due to limited browser support and higher integration complexity for external consumers. Treat it as a specialized internal-communication note alongside RPC/tRPC rather than a fourth full column, unless the project's own answers make internal service-to-service performance a first-order concern.

---

## Axis 2: Versioning strategy

Relevant once the API has external consumers (other teams, third parties, or even just older client app versions still in the field) that can't all upgrade in lockstep with the backend.

### No explicit versioning

- **Pros:** simplest — nothing to design or maintain; fine for an API with no external consumers, or where client and server always deploy together (a monolith serving its own frontend, `01-system-level.md`)
- **Cons:** any breaking change breaks every consumer simultaneously; not viable once there's a real external or slow-to-update client base

### URI/path versioning (e.g. `/v1/`, `/v2/`)

- **Pros:** explicit and highly visible — consumers can see exactly which version they're calling; simple to route at the infrastructure layer (different versions can even be different deployments); the most common and widely understood REST versioning approach
- **Cons:** encourages duplicating entire endpoint surfaces across versions even when only a small part changed; can accumulate stale, unmaintained old versions that still need to be kept running

### Header-based versioning

- **Pros:** keeps URIs stable and semantically clean (the resource identity doesn't change just because the API version does); can version more granularly than a whole-API version bump
- **Cons:** less discoverable/visible than URI versioning — easy to miss which version is being requested when debugging or reading logs; requires clients to correctly set headers rather than the version being self-evident in the URL

### Schema evolution (additive-only changes, GraphQL's typical approach)

- **Pros:** avoids discrete version numbers entirely — new fields/types are added, old ones deprecated in place with tooling support (e.g. GraphQL's `@deprecated` directive) rather than standing up a parallel version; consumers migrate at their own pace off deprecated fields without a hard cutover
- **Cons:** genuinely breaking changes (renaming or removing a field's meaning, not just its name) still eventually require a real break — this defers the problem rather than eliminating it; requires discipline to keep changes additive and to actually retire deprecated fields eventually, or the schema accumulates cruft indefinitely

Versioning strategy correlates strongly with the paradigm chosen in Axis 1 — REST typically pairs with URI or header versioning, GraphQL typically pairs with schema evolution — but isn't strictly forced by it; a REST API can be additive-only in spirit, and a GraphQL API can still stand up a `v2` schema for a genuine breaking overhaul.

---

## Axis 3: Public vs internal API boundaries

Whether the same API surface serves both this project's own frontend/services and any external/third-party consumers, or whether these are deliberately separated.

### Single unified API surface

- **Pros:** simplest to build and maintain — one API, one set of endpoints/schema, no duplication
- **Cons:** internal consumers (who can be trusted to move fast and adapt) and external consumers (who need stability and can't be broken without notice) end up constrained by the same change-management discipline, slowing internal iteration or risking external breakage; internal-only operational endpoints (admin actions, internal-only data) have to be carefully access-controlled within the same surface rather than being structurally separated

### Separate public and internal APIs

- **Pros:** internal API can evolve quickly without the version-stability guarantees a public API needs; public API surface can be deliberately minimal and well-documented, reducing both attack surface and support burden; internal-only functionality never risks accidental public exposure since it's not even on the same surface
- **Cons:** real duplication and maintenance cost — genuinely shared functionality may need to be implemented or exposed twice, or behind an internal service that both surfaces call; more infrastructure to stand up and secure (two API layers instead of one) and more decisions about which surface a given piece of functionality belongs on

For most early-stage or single-consumer (own frontend only) projects, a single unified surface is the right default — a separate public API is worth the added complexity specifically once there's a real, distinct external consumer (a partner integration, a public developer API as a product feature) rather than a hypothetical future one.

---

## What drives the recommendation (from upfront questions)

- **Platform**: multiple distinct client platforms (web + mobile + maybe third-party) with meaningfully different data needs per client strengthens the case for GraphQL's flexible querying over REST's fixed endpoint shapes; a single client (e.g. one web frontend) removes much of GraphQL's core advantage
- **System-level shape** (from `01-system-level.md`): a monolith serving its own frontend has the weakest case for heavy API-design investment — no explicit versioning and a single unified surface are both reasonable defaults; microservices increase the case for RPC/gRPC on internal service-to-service calls specifically, separate from whatever paradigm is chosen for client-facing APIs
- **Team language/stack**: a full-stack TypeScript project is the specific condition under which tRPC's type-safety benefit is real and substantial; it should not be recommended as strongly (or at all) outside that context, and this is worth stating explicitly rather than defaulting to it as a general RPC recommendation
- **Team size / ownership boundaries**: multiple teams or external partners consuming the API strengthens the case for explicit versioning (Axis 2) and a separated public/internal surface (Axis 3); a solo developer or single small team calling their own API has much less need for either
- **Time-to-ship pressure**: high pressure → REST with no explicit versioning and a single unified surface is a legitimate default, not a gap; GraphQL and a separated public API surface both carry real upfront cost that's hard to justify against a tight deadline unless a client-diversity or external-consumer need is already present
- **Data access pattern** (from `02-data-storage.md`'s CQRS axis): if CQRS was adopted, it's worth flagging here that command and query surfaces may reasonably be exposed as distinct API shapes (e.g. REST-style commands, GraphQL-style flexible queries) rather than forcing one paradigm to serve both symmetrically

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| API paradigm | *REST pros/cons* | *GraphQL pros/cons* | *RPC/tRPC pros/cons (note gRPC as internal-only alternative if service-to-service perf matters)* | *pick one, justify against client diversity / team stack / system shape* | ☐ |
| Versioning strategy | *No versioning pros/cons* | *URI or header versioning pros/cons* | *Schema evolution pros/cons* | *justify against whether there are external/slow-to-update consumers; note correlation with paradigm chosen above* | ☐ |
| Public/internal boundary | *Unified surface pros/cons* | *Separated surfaces pros/cons* | — | *justify against whether a real distinct external consumer exists today, not a hypothetical future one* | ☐ |

Note: for a monolith with a single frontend and no external consumers, this can reasonably collapse to a single consolidated row ("REST, no explicit versioning, single unified surface") rather than three rows — don't force the full table when the project's answers make versioning and the public/internal split non-decisions.

---

## Related files

- `01-system-level.md` — system shape informs internal service-to-service API needs (gRPC note) separately from client-facing paradigm choice
- `02-data-storage.md` — CQRS there may push toward distinct command/query API shapes here
- `06-auth.md` — auth scheme (token vs cookie) affects which clients can easily call the API; public/internal boundary here often aligns with auth boundary there
- `08-integrations.md` — that file covers calling out to others' APIs; this file covers designing the API this project exposes
- `13-caching.md` — GraphQL's single-endpoint POST pattern needs a different caching approach than REST's HTTP-cacheable GET endpoints
- `20-environment-strategy.md` — API versioning strategy here interacts with rollout/release strategy for breaking changes
