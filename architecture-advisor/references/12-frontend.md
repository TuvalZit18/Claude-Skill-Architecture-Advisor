# 12 — Frontend

## What this category covers

The internal architecture of the frontend application itself: framework/library paradigm choice, rendering strategy (where and when HTML gets generated), state management (client vs server state), and code organization as the codebase grows.

**Not covered here:** whether the frontend is split into independently-deployed pieces (micro-frontends) vs a single frontend app. That's the same underlying team-autonomy-vs-coordination trade-off as backend system shape, and is fully covered in `01-system-level.md`'s Sub-decision 2 — this file assumes that decision is already made and covers what happens *within* whichever shape was chosen, rather than re-litigating it.

**Scope note on naming specific frameworks:** unlike `08-integrations.md`, `09-ai-integration.md`, and `10-messaging.md`, which deliberately avoid naming specific vendor products, this file does name specific frontend frameworks (React, Vue, Svelte, Angular) as illustrative examples within Axis 1. This is a deliberate exception, not an inconsistency: frontend framework ecosystems are comparatively stable (React and Vue's core models have held for the better part of a decade, unlike AI models or specific message-broker products that shift month to month), and the differences between them are genuinely structural/paradigmatic — rendering model, bundled-vs-composable tooling — not just brand preference. Even so, this file stays focused on the paradigm each framework represents rather than a feature-by-feature product comparison.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Framework / component paradigm

### Virtual-DOM component libraries (e.g. React, Vue)

- **Pros:** the largest ecosystems and talent pools of any frontend approach, with mature tooling and extensive third-party component/library support; not a full opinionated framework, so routing, state management, and build tooling can be composed independently based on the project's actual needs rather than accepting a bundled default
- **Cons:** virtual-DOM diffing adds a runtime performance layer compared to compiled approaches; the flexibility of composing your own stack means more upfront decisions (routing, state, build tooling are all separate choices) rather than a single bundled default

### Compile-time reactive frameworks (e.g. Svelte)

- **Pros:** compiles away much of the runtime framework overhead, typically resulting in smaller bundle sizes and better raw performance since there's no virtual-DOM diffing at runtime; a more direct reactivity model that many find simpler to reason about for straightforward UI
- **Cons:** smaller ecosystem and talent pool than the virtual-DOM libraries; fewer third-party component libraries and integrations to draw on

### Comprehensive/opinionated frameworks (e.g. Angular)

- **Pros:** batteries-included — routing, state, forms, dependency injection, and testing conventions come bundled and consistent, reducing the number of separate tooling decisions a team has to make on its own; a strong fit for larger teams that benefit from convention-over-configuration and consistent structure across many contributors
- **Cons:** a steeper learning curve, with more concepts to learn upfront; less flexible than composing your own stack, so deviating from the framework's own conventions is harder than it would be with a lighter-weight library

### No framework / lightweight libraries

- **Pros:** minimal overhead and full control, with no framework lock-in — can be appropriate for a genuinely simple UI that doesn't need a framework's machinery at all
- **Cons:** reinventing solved problems (routing, state, reactivity) that a framework would otherwise provide out of the box; scales poorly as UI complexity grows without deliberate, self-built architecture

Team familiarity and existing organizational standards are often the deciding factor in practice once the paradigm-level trade-offs above are weighed — this file stays at the level of which paradigm fits the project's team-size and complexity needs, consistent with the scope note above.

---

## Axis 2: Rendering strategy

### Client-side rendering (CSR) — all rendering happens in the browser

- **Pros:** simplest deployment story — a static bundle served from anywhere, including a CDN (`04-file-storage.md`), with no server-side rendering infrastructure to run; snappy in-app navigation once the initial bundle has loaded, since subsequent views don't require a full page round-trip
- **Cons:** slower initial load and first-contentful-paint, since the browser must download and execute JavaScript before anything renders; weaker default SEO, since crawlers see an empty shell until JS executes — a real concern for any content that needs to be indexed reliably; a genuine performance/accessibility concern on slow networks or low-power devices

### Server-side rendering (SSR) — the server renders the initial HTML per request

- **Pros:** faster first-contentful-paint and stronger default SEO, since the browser receives fully-rendered HTML immediately; works reliably for crawlers regardless of their JavaScript-execution support
- **Cons:** requires a running server or serverless function to render on each request, adding infrastructure and per-request compute cost compared to serving a static bundle; a more complex deployment story than CSR's static-file simplicity

### Static site generation (SSG) — pages pre-rendered at build time

- **Pros:** combines SSR's fast-load and SEO benefits with CSR's simple, cheap static-file deployment — each page is built once and served from a CDN like any static asset, with no per-request rendering cost at all
- **Cons:** content is only as fresh as the last build — a poor fit for frequently-changing or per-user personalized content without additional client-side fetching layered on top of the initial static shell

### Incremental static regeneration / hybrid rendering

- **Pros:** combines SSG's performance with the ability to update stale pages without a full site rebuild, or mixes SSR/SSG/CSR per route within a single application depending on each route's actual freshness and personalization needs
- **Cons:** the most operationally complex of the options — requires a framework and hosting platform that supports the hybrid model, and reasoning about which rendering mode applies to which route adds a real design dimension the simpler options don't have

Which applies is largely a function of content freshness and SEO needs: mostly-static, SEO-sensitive content (marketing pages, blogs, product listings) fits SSG well; highly personalized or frequently-changing per-user content fits CSR or SSR; most production applications that care about both SEO and personalization end up on a hybrid approach mixing rendering modes by route rather than picking one for the whole app.

---

## Axis 3: State management

### Local/component state only

- **Pros:** simplest possible approach — no additional library or architecture, state lives exactly where it's used and nowhere else
- **Cons:** doesn't scale to state that needs to be shared across distant parts of the component tree without prop-drilling (passing state down through intermediate components that don't themselves need it, purely to reach a descendant that does)

### Context / built-in framework state sharing

- **Pros:** avoids prop-drilling for state that needs broader sharing, without adding an external library — built into the framework itself
- **Cons:** can trigger unnecessary re-renders across the whole subtree consuming the context whenever the shared state changes, unless deliberately optimized; not designed for high-frequency updates or complex derived state

### External global state library (e.g. Redux, Zustand, Pinia)

- **Pros:** purpose-built for larger-scale client state — predictable update patterns, devtools for inspecting state changes over time, and better performance characteristics for frequent updates than context; scales to complex, deeply-shared application state more gracefully than context alone
- **Cons:** an additional library and learning curve; can be genuine over-engineering for an app whose actual shared-state needs are modest — reaching for a global store when local state or context would have sufficed is a common and avoidable complexity trap

### Server-state / cache library (e.g. React Query/TanStack Query, SWR)

- **Pros:** addresses a genuinely distinct problem from client UI state — data fetched from a server (and its caching, refetching, staleness, and synchronization with the server's own source of truth) has different needs than purely client-side state like "is this dropdown open"; handles loading/error states, caching, and background refetching in ways general-purpose client-state libraries don't natively provide
- **Cons:** another library and concept to learn; the underlying distinction between client state and server state still has to be recognized and applied consistently by the team, or the tool's benefit is lost by mixing the two anyway

Most real applications benefit from distinguishing client state (UI-only, e.g. is a modal open) from server state (data fetched from the backend) and using different tools for each — a global state library for client state plus a server-state/cache library for data fetching is a common, coherent combination rather than a single tool covering everything.

---

## Axis 4: Code organization / folder structure

### Type-based organization (folders like `components/`, `hooks/`, `pages/`, `utils/`)

- **Pros:** simplest to start with and intuitive for a small codebase — easy to find "all the hooks" or "all the components" at a glance
- **Cons:** scales poorly as the app grows — files related to a single feature end up scattered across many top-level folders, making it harder to see everything relevant to one feature, or to cleanly remove a feature when it's no longer needed

### Feature/domain-based organization (folders like `features/checkout/`, `features/profile/`, each containing its own components/hooks/state)

- **Pros:** everything related to a feature lives together, making it easier to understand, modify, or remove that feature as a unit; scales considerably better than type-based organization as the app and team grow, and mirrors the domain-boundary thinking (DDD-style) applied as a lens throughout this skill
- **Cons:** genuinely shared or cross-cutting components and utilities don't fit neatly into any single feature folder, requiring a deliberate `shared/` or `common/` area and real discipline about what belongs there versus what's feature-specific

### Atomic design (folders organized by component complexity tier: atoms, molecules, organisms, templates, pages)

- **Pros:** gives a principled vocabulary for component composition and reuse, particularly valuable for design-system-heavy projects where consistent, reusable UI primitives matter a great deal
- **Cons:** the tier boundaries (is this a molecule or an organism?) are often more ambiguous in practice than the framework suggests, leading to unproductive debate; a weaker natural fit for organizing business logic or state than a feature-based structure, since atomic design is primarily a UI-composition vocabulary rather than a full application-organization scheme

Feature-based organization is the most broadly applicable default for a growing application, since it mirrors how work actually gets done (one feature at a time) and scales well with team size; type-based organization is reasonable for a genuinely small project; atomic design is best treated as a complementary vocabulary for the UI-component layer specifically — which can live inside a feature-based structure — rather than a replacement for it.

---

## What drives the recommendation (from upfront questions)

- **Platform**: this file assumes a web frontend; a native mobile app has a different (though analogous) set of framework/state-management questions largely outside this file's scope, and a CLI or non-UI platform has no frontend decision here at all
- **Team size / ownership boundaries**: larger teams benefit more from an opinionated framework's convention-over-configuration (Axis 1) and from feature-based organization's clean ownership boundaries (Axis 4) than a solo developer would; also connects to `01-system-level.md`'s micro-frontends decision, which this file doesn't re-litigate but does assume
- **Domain/SEO requirements**: content that needs strong SEO (public marketing pages, content sites, product listings meant to be discoverable) is the direct driver toward SSG or SSR over pure CSR (Axis 2); an internal tool or authenticated-only app behind a login has no SEO need and can default to CSR's simpler deployment
- **Time-to-ship pressure**: high pressure → the team's already-familiar framework (Axis 1) plus CSR (Axis 2, simplest deployment) plus local/context state until a real need for a global or server-state library emerges, is a legitimate first version — not a gap
- **Expected scale / complexity**: an app expected to stay small can reasonably stay on local/context state and type-based organization; an app expected to grow substantially benefits from planning for a global/server-state split and feature-based organization earlier rather than migrating under pressure later
- **Ops/deployment capacity**: low ops maturity weights toward CSR or SSG (both deployable as static files with minimal infrastructure) over SSR or hybrid rendering, which require running server/serverless rendering infrastructure

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Option D | Recommendation + why | Approve |
|---|---|---|---|---|---|---|
| Framework/component paradigm | *Virtual-DOM library pros/cons* | *Compile-time reactive pros/cons* | *Comprehensive framework pros/cons* | *No framework pros/cons* | *justify against team size, familiarity, and complexity needs* | ☐ |
| Rendering strategy | *CSR pros/cons* | *SSR pros/cons* | *SSG pros/cons* | *Hybrid/ISR pros/cons* | *justify against SEO/content-freshness needs and ops capacity* | ☐ |
| State management | *Local/context pros/cons* | *Global state library pros/cons* | *Server-state/cache library pros/cons* | — | *most apps need a combination — client state tool plus server-state tool; justify against actual shared-state complexity* | ☐ |
| Code organization | *Type-based pros/cons* | *Feature-based pros/cons* | *Atomic design pros/cons (as UI-layer complement, not full replacement)* | — | *justify against expected app size and team growth; feature-based is the broadly applicable default* | ☐ |

Note: for a small project under time pressure, this can reasonably collapse to two rows ("team's familiar framework, CSR for simplicity" plus "local/context state, type-based folders until the app grows") rather than four.

---

## Related files

- `01-system-level.md` — micro-frontends vs single frontend app is decided there, not here; this file assumes that decision
- `04-file-storage.md` — CSR/SSG's static-bundle deployment model relies on the CDN/object-storage decisions covered there
- `07-api-design.md` — the frontend's data-fetching pattern (Axis 3's server-state tools) is the client side of whatever API paradigm was chosen there
- `13-caching.md` — client-side and CDN caching strategy for frontend assets is covered in full there
- `19-deployment-infra.md` — SSR/hybrid rendering's need for running server infrastructure ties into the compute-model decision covered there
