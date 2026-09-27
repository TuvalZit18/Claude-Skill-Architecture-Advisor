# 06 — Auth

## What this category covers

**Authentication** (verifying who a user is) and **authorization** (deciding what an authenticated user is allowed to do), plus where that logic physically lives in the system. This file covers three sub-decisions: session vs token-based (JWT) vs delegated (OAuth/OIDC) authentication; RBAC vs ABAC authorization models; and where auth checks are enforced (edge/gateway vs service/app layer).

Distinct from `16-security.md`: that file covers the broader security policy layer — threat modeling, encryption, secrets management, least-privilege principles in general. This file is specifically about the auth mechanism itself: how identity is established and how permission decisions are made and enforced.

Distinct from `21-networking.md`: that file covers the mechanical/topological enforcement layer (API gateway, TLS termination, firewalls); this file decides *what* the auth logic is and *where in principle* it should sit, which the networking layer then physically implements.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Authentication mechanism

### Server-side sessions

- **Pros:** simplest mental model — a session ID (usually in a cookie) maps to server-held state; trivial to revoke instantly (delete the session record) — the strongest story for "log this user out everywhere right now"; no client-side token parsing or expiry logic needed; well-trodden path with mature framework support
- **Cons:** requires a shared session store (Redis or equivalent) for anything horizontally scaled (ties directly into `05-scalability.md` Axis 2 — statelessness); doesn't fit cleanly across separate domains or native mobile clients the way a portable token does; adds a lookup on every request (though typically cheap with a fast session store)

### JWT (JSON Web Tokens) / stateless tokens

- **Pros:** self-contained — the token itself carries the claims, so no server-side lookup is needed to validate a request, which fits statelessness and horizontal scaling well; naturally portable across services/domains without a shared session store, a real advantage in a microservices topology (`01-system-level.md`); works well for mobile/native clients and API-to-API auth
- **Cons:** revocation is the core weakness — a JWT is valid until it expires, so instantly invalidating a compromised or logged-out token requires extra machinery (a denylist, short expiry plus refresh tokens, or similar), undoing some of the "stateless" benefit; token size grows with claims, adding overhead to every request; if signing keys leak, every issued token is compromised until keys are rotated and old tokens age out

### OAuth 2.0 / OpenID Connect (delegated / third-party auth)

- **Pros:** offloads credential storage and management entirely to a trusted identity provider (Google, GitHub, Auth0, Okta, etc.) — no password storage, reset flows, or credential-breach liability for the app itself; lets users authenticate with an identity they already have, lowering signup friction; OIDC adds a standardized identity layer on top of OAuth's authorization framework, giving a consistent way to get verified user identity claims
- **Cons:** adds a hard external dependency — an outage or policy change at the identity provider affects your login flow directly; the OAuth/OIDC flow itself (redirects, token exchange, refresh) is more complex to implement correctly than a simple session or JWT issuance; still needs a mechanism for how the app itself tracks the resulting session (often combined with Axis 1's session or JWT approach *after* the OAuth handshake completes, rather than being a full replacement for it)

These three aren't fully mutually exclusive in practice: a common real-world combination is OAuth/OIDC for the initial login handshake, followed by the app issuing its own session or JWT for subsequent requests. Worth noting as a "combination" option in the generated table rather than forcing a single pick when the project's answers point that way.

---

## Axis 2: Authorization model

### RBAC (Role-Based Access Control)

- **Pros:** simple to reason about and implement — users are assigned roles (admin, editor, viewer), roles map to permissions; well-understood, widely supported by frameworks and managed auth providers; sufficient for the large majority of applications where permission needs are coarse-grained and stable
- **Cons:** becomes unwieldy when permission needs are fine-grained or contextual (e.g. "editor, but only for documents they created" doesn't fit a pure role model without role explosion — a proliferating set of narrow roles to cover every combination)

### ABAC (Attribute-Based Access Control)

- **Pros:** expresses fine-grained, contextual rules that RBAC can't (e.g. "allow if user.department == resource.department AND time is business hours") — attributes of the user, resource, action, and environment all factor into the decision; scales better than RBAC's role explosion when permission logic is genuinely complex or dynamic
- **Cons:** significantly more complex to design, implement, and audit — policies are harder to reason about at a glance than a role list; overkill for the common case where roles alone are sufficient; typically needs a dedicated policy engine or library (e.g. OPA, Casbin) rather than a simple lookup table, adding an operational dependency

Most projects should default to RBAC and only reach for ABAC when a genuine, demonstrated need for contextual/fine-grained rules exists — not as a default "more powerful, so better" choice. A hybrid is also common in practice: RBAC for the coarse admin/editor/viewer split, with a handful of ABAC-style contextual checks layered on top for specific resources (e.g. ownership checks) rather than building a full ABAC policy engine for the whole system.

---

## Axis 3: Where auth logic lives

### Enforced at the edge / API gateway

- **Pros:** centralizes authentication (and often coarse authorization) in one place before a request ever reaches application code — every service behind the gateway benefits without reimplementing the check; a natural fit alongside gateway-level rate limiting (`05-scalability.md` Axis 4) and the API-gateway pattern in `21-networking.md`
- **Cons:** the gateway can only reasonably do coarse-grained checks (is this a valid, authenticated request; does this token have the right scope) — fine-grained, resource-specific authorization (ABAC-style "can this user edit *this* document") usually still needs to happen at the service/app layer, which has the actual resource context; adds a hard dependency on the gateway being correctly configured for every route, and a gap here is a security gap for everything behind it

### Enforced at the service/application layer

- **Pros:** has full context — the actual resource being accessed, business rules, ownership — so fine-grained authorization decisions belong here regardless of what's also enforced upstream; necessary in a microservices topology where each service may have its own authorization nuances the gateway can't know about
- **Cons:** without a shared library or consistent pattern across services, authorization logic can drift or be inconsistently applied — a classic source of security gaps in a multi-service system; every service has to actually implement the check correctly rather than relying on a single upstream enforcement point

In practice these are layered, not either/or: coarse authentication/authorization at the edge (is this request even valid) plus fine-grained, resource-aware authorization at the service layer (can *this* user do *this* thing to *this* resource) is the standard pattern once there's more than a single deployable unit (`01-system-level.md`). For a monolith, "the edge" and "the service layer" collapse into the same codebase, and this axis becomes closer to a non-decision.

---

## What drives the recommendation (from upfront questions)

- **Platform**: mobile/native clients weight toward JWT or OAuth over server-side sessions, since portable tokens fit native auth flows and offline/reconnect scenarios more naturally than cookie-based sessions; a pure web app has no strong pull either way on this axis alone
- **System-level shape** (from `01-system-level.md`): a monolith has no strong need for JWT's cross-service portability — server-side sessions are a perfectly reasonable default; a microservices topology strengthens the case for JWT (or OAuth/OIDC with JWT as the resulting token) specifically because tokens can be validated by any service without a shared session store
- **Team size / ownership boundaries**: multiple teams each owning separate services benefits from centralizing authentication at the edge/gateway (Axis 3) so every team isn't reimplementing the same check; a single team on a monolith has less need for that separation
- **Ops/deployment capacity**: low ops maturity weights toward OAuth/OIDC via a managed identity provider (offloading credential management entirely) and away from building custom session infrastructure or an ABAC policy engine
- **Domain coupling / permission complexity**: genuinely fine-grained, contextual permission needs (multi-tenant systems, resource-level ownership rules, regulatory need-to-know restrictions) push toward ABAC or a hybrid; most CRUD-shaped apps don't need this and RBAC alone is the right default
- **Time-to-ship pressure**: high pressure → OAuth/OIDC via a managed provider (fastest to ship, no credential-management build-out) plus RBAC, is a legitimate and common answer, not a gap
- **Consistency requirements**: strict "log out everywhere instantly" requirements favor server-side sessions' easy revocation over JWT's revocation weakness, or at minimum push JWT implementations toward short expiry plus a refresh-token/denylist pattern

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| Authentication mechanism | *Server-side sessions pros/cons* | *JWT pros/cons* | *OAuth/OIDC pros/cons (note if combined with A or B for post-login session handling)* | *pick one or a stated combination, justify against platform / system-level shape / revocation needs* | ☐ |
| Authorization model | *RBAC pros/cons* | *ABAC pros/cons* | *Hybrid (RBAC + targeted ABAC-style checks)* | *justify against permission complexity; default to RBAC unless a genuine fine-grained need is demonstrated* | ☐ |
| Where auth logic lives | *Edge/gateway pros/cons* | *Service/app layer pros/cons* | *Layered (both)* | *justify against system-level shape — collapses to a non-decision for a monolith; layered is the default once multiple services exist* | ☐ |

Note: for a solo/small-team project on a monolith with time-to-ship pressure, this can reasonably collapse to a single consolidated row ("OAuth/OIDC via a managed provider for login, server-side sessions for the app's own state, RBAC for permissions, no separate edge/service split") rather than three full rows. Don't force three rows when the project's answers make the auth-logic-location axis a non-decision.

---

## Related files

- `01-system-level.md` — system shape (monolith vs microservices) is the main driver for Axis 1 and Axis 3 here
- `02-data-storage.md` — user/role/permission data lives in whichever storage model was chosen there
- `05-scalability.md` — statelessness (Axis 2 there) is why JWT is attractive for horizontally-scaled systems; session stores need the same shared-cache infra
- `07-api-design.md` — auth scheme affects API surface (e.g. token-in-header vs cookie-based auth, public vs authenticated endpoint boundaries)
- `16-security.md` — general security policy layer; this file is the specific auth mechanism that policy governs
- `21-networking.md` — API gateway is the mechanical layer that implements edge-level auth enforcement decided here
