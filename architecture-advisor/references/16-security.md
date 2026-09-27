# 16 — Security

## What this category covers

The security *policy* layer: threat modeling approach, encryption and key management, secrets management, least-privilege access scoping, security monitoring and audit logging, and data classification. Mechanical enforcement of these policies (TLS termination, firewalls, network segmentation) lives in `21-networking.md` instead — this file decides *what* the policy is, that file implements it at the network layer.

This is also the file several earlier files explicitly deferred to and need this file to actually deliver on: `04-file-storage.md`'s general access-control policy question, `06-auth.md`'s distinction between user-facing authorization and broader infrastructure access, `08-integrations.md` and `09-ai-integration.md`'s API-key-custody question, `13-caching.md`'s sensitive-data-in-cache concern, and `15-observability.md`'s security monitoring and audit logging.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Threat modeling approach

### Informal / ad hoc

- **Pros:** no process overhead — fast, and reasonable for a small team or a genuinely simple system
- **Cons:** relies entirely on the team's collective intuition and experience — real threats can be missed simply because no one happened to think to ask about them; doesn't scale as the system and team grow, and leaves no structured record of what was considered and dismissed

### Structured threat modeling (e.g. STRIDE, attack trees, or a similar framework)

- **Pros:** systematically walks through categories of threat (for STRIDE: spoofing, tampering, repudiation, information disclosure, denial of service, elevation of privilege) rather than relying on whoever happens to think of something; produces a documented record of what was considered, valuable for onboarding new team members and for revisiting as the system changes
- **Cons:** a real time investment, and a structured framework can still miss threats specific to the domain that don't map neatly onto its categories; can become a check-the-box exercise if not genuinely engaged with rather than treated as documentation theater

Structured threat modeling earns its cost specifically for systems handling sensitive data or with a real threat surface (public-facing APIs, financial transactions, PII, anything regulated) — proportionally lighter investment is reasonable for a low-stakes internal tool.

---

## Axis 2: Encryption & key management

Encryption in transit (TLS everywhere) and encryption at rest are both close to non-negotiable baselines for a production system — not really a genuine toss-up the way most axes in this skill are. The actual architectural decision worth a table row is **key custody**: who controls the encryption keys.

- **Encryption in transit:** protects data as it moves between client and server, and between internal services, from network-level interception. Universally expected; the mechanical implementation (TLS termination) is `21-networking.md`'s concern.
- **Encryption at rest:** protects data if the underlying storage medium is compromised or improperly accessed (a stolen disk, a misconfigured backup, unauthorized direct database access) — often bundled as a near-default feature by managed databases and storage providers (`02-data-storage.md`, `04-file-storage.md`). Doesn't protect against a compromised, *authorized* process reading data through normal application access — it protects the storage medium specifically, not the application layer.

### Provider-managed encryption keys

- **Pros:** minimal operational burden — the cloud provider or database service generates, rotates, and manages keys automatically; a sensible default for most projects
- **Cons:** the provider technically has access to the keys, which may not satisfy certain compliance regimes or threat models that specifically require the customer to hold exclusive key control

### Customer-managed encryption keys (e.g. via a dedicated key management service)

- **Pros:** stronger control and audit trail over exactly who and what can decrypt data; often required by specific compliance regimes (some data-residency or regulatory frameworks mandate customer-held keys)
- **Cons:** real operational burden — key rotation, secure key storage, and a genuine recovery plan if keys are lost, since losing a key can mean losing access to the data it protects permanently; this is the general version of the same key-custody question `03-backup-disaster-recovery.md` already flags specifically for backup encryption

---

## Axis 3: Secrets management

### Hardcoded / embedded in code or config committed to version control

- **Pros:** none genuinely worth calling a pro — named explicitly here as a real anti-pattern, not a legitimate option, since it happens by default if no better approach is deliberately set up
- **Cons:** secrets committed to version control are exposed to anyone with repository access, including in history even after later removal — a common and serious real-world breach vector

### Environment variables

- **Pros:** simple and widely supported, and keeps secrets out of the codebase itself
- **Cons:** can still leak via process listings, crash dumps, or accidental logging if not handled carefully; no built-in rotation, versioning, or fine-grained access control — essentially a flat, unmanaged store

### Dedicated secrets manager (e.g. a cloud provider's secrets manager, Vault, or similar)

- **Pros:** centralized storage with access control over which service or person can retrieve which secret, audit logging of secret access (Axis 6), and often built-in rotation support; the standard approach for anything beyond a small or early-stage project
- **Cons:** another piece of infrastructure or vendor relationship to manage; adds a network dependency for secret retrieval, which needs the same failure-handling consideration `14-resilience.md` gives any external dependency if it becomes slow or unavailable

This axis is the concrete home for the API-key-custody question raised by `08-integrations.md` and `09-ai-integration.md` — a dedicated secrets manager is the standard answer for storing and retrieving third-party and AI-provider API keys specifically.

---

## Axis 4: Input validation & injection prevention

Covers where and how user-supplied input is validated and sanitized before it reaches a database, shell, or interpreter — the architectural side of preventing injection attacks (SQL, NoSQL, command injection, and similar). The actual coding practice itself — parameterized queries/prepared statements instead of string-concatenated queries, an ORM that does this by default, escaping shell arguments rather than concatenating them into a command string — is implementation-level, the same treatment this skill gives algorithms/data structures: a non-negotiable baseline practice, not a genuine architectural trade-off with two legitimate options. What *is* architectural is where validation happens in the request path.

### Validation at a single layer (e.g. only at the API/service boundary)

- **Pros:** one place to get validation right — simpler to reason about and audit than validation scattered across many layers
- **Cons:** any code path that bypasses that single layer (an internal service-to-service call, a batch job, a direct database script) has no validation at all — a single point of failure for the whole defense

### Validation at multiple layers (defense in depth: client, gateway, and service/data layer each validate)

- **Pros:** no single bypassed layer leaves the system fully exposed — client-side validation improves UX and catches obvious mistakes early, gateway-level validation blocks malformed requests before they reach application code, and service/data-layer validation (parameterized queries, ORM usage) is the actual enforcement point that matters most for injection prevention specifically, since it's the layer closest to the vulnerable operation
- **Cons:** real duplication of validation logic across layers, with the ongoing discipline required to keep them consistent as rules change; more places for validation logic to drift out of sync with actual requirements

Defense-in-depth — validating at multiple layers, with parameterized queries/prepared statements as the actual, non-negotiable enforcement point closest to the database or interpreter — is the standard expectation regardless of project size. Client-side-only or gateway-only validation without also enforcing safe query construction at the data-access layer leaves the core vulnerability class essentially unaddressed no matter how much validation happens upstream.

---

## Axis 5: Least-privilege & access scoping

Distinct from `06-auth.md`'s authorization model (RBAC/ABAC), which covers user-facing permissions: this axis is the broader principle applied to infrastructure and service-level access — service accounts, IAM roles, database credentials, deployment permissions — not just end-user access.

### Broad/shared credentials and permissions

- **Pros:** simplest to set up — one set of credentials or one broad role covers everything a service or team member might need
- **Cons:** a compromised credential or an over-permissioned account has a much larger blast radius than necessary — a service that only needs to read one table can, if compromised, do far more damage than it should be capable of; makes it genuinely hard to reason about what could go wrong from any given credential being misused

### Least-privilege, scoped credentials and roles

- **Pros:** each service, process, or person has exactly the access it needs and no more, containing the blast radius of any single compromised credential to only what that credential could legitimately do — the same isolation principle as `01-system-level.md`'s service boundaries and `14-resilience.md`'s bulkheads, applied to permissions instead of resources
- **Cons:** real ongoing overhead to define, maintain, and audit granular roles as the system evolves — permissions can drift out of sync with actual need over time without periodic review, most often accumulating unnecessary access ("permission creep") rather than becoming too restrictive

Least-privilege is close to a non-negotiable baseline principle rather than a genuine toss-up, similar to timeouts in `14-resilience.md` — the real decision is how much investment goes into granular role definition and periodic review, versus accepting broader default roles for lower-stakes internal systems.

---

## Axis 6: Security monitoring & audit logging

Distinct from `15-observability.md`'s operational monitoring: this axis is about detecting malicious or unauthorized activity and maintaining a compliance-grade record of who did what, not diagnosing general operational health — the concrete home for what `15-observability.md` explicitly deferred here.

### No dedicated security monitoring or audit logging

- **Pros:** nothing extra to build
- **Cons:** unauthorized access or malicious activity may go undetected until real damage is done; no record to investigate after the fact if an incident is suspected, and no way to satisfy compliance requirements that mandate an audit trail

### Security monitoring (intrusion/anomalous-access detection) plus audit logging

- **Pros:** surfaces malicious or unauthorized activity — repeated failed authentication attempts, access from unexpected locations, privilege-escalation attempts — as it happens, using the same anomaly-detection and alerting mechanisms `15-observability.md` covers generally, applied specifically to security-relevant signals; audit logging provides an immutable, queryable record of who did what and when, required by many compliance regimes and valuable for post-incident investigation regardless of compliance requirements
- **Cons:** real additional instrumentation and infrastructure, often a distinct pipeline and retention policy from general operational logs — audit logs typically need longer retention and stronger tamper-resistance than routine application logs; false positives in intrusion/anomaly detection carry the same tuning burden noted in `15-observability.md`'s anomaly-detection option

Configuration-change audit trails (flagged in `15-observability.md` as belonging here) are a specific case of this axis's audit logging — who changed what configuration, when.

---

## Axis 7: Data classification & handling policy

### No formal data classification

- **Pros:** nothing extra to define upfront
- **Cons:** without an explicit tiering of what's sensitive, encryption, access-control, and retention decisions elsewhere in the system tend to be applied uniformly — either over-protecting low-stakes data at needless cost, or under-protecting genuinely sensitive data by not recognizing it as such in the first place

### Formal data classification (e.g. public / internal / confidential / restricted tiers)

- **Pros:** gives every other security decision in this file — and in `02-data-storage.md`, `04-file-storage.md`, and `09-ai-integration.md`'s data-minimization note — a clear input to work from: which data needs the strongest encryption, the most restrictive access, the longest or shortest retention window; makes compliance conversations concrete rather than abstract
- **Cons:** real upfront effort to classify existing data, and ongoing discipline to classify new data as the system evolves; only pays off if the classification is actually consulted when making the encryption/access/retention decisions it's meant to inform, rather than existing as a document no one refers to again

Formal classification earns its cost specifically once a system handles genuinely sensitive data — PII, financial, health, or another regulated category. A system without any such data has little to classify and can reasonably skip this.

---

## Additional hardening considerations

Several other well-known vulnerability classes (drawing on the OWASP Top 10, for anyone mapping this file against it) don't have a genuine two-option architectural trade-off the way the axes above do — they're closer to baseline practices, or fall under a different file's territory. Worth naming explicitly here so none look like an oversight:

- **Security misconfiguration:** shipping secure defaults — not leaving default credentials, debug modes, or verbose stack-trace error pages enabled in production — and disabling unnecessary features, ports, or services rather than leaving them on "just in case." The environment-specific piece of this, hardened production configuration distinct from staging/dev, is `20-environment-strategy.md`'s dev/prod-parity territory; this file's role is naming the baseline expectation.
- **Identification and authentication failures beyond the mechanism choice:** `06-auth.md` covers the authentication mechanism itself (sessions/JWT/OAuth); this file adds two specific hardening considerations on top — multi-factor authentication for anything beyond low-stakes access, and brute-force/credential-stuffing protection on login endpoints specifically, a targeted application of `05-scalability.md`'s general rate-limiting axis to the authentication surface, since login endpoints are a disproportionately common attack target.
- **Server-side request forgery (SSRF):** any feature where the server fetches a user-supplied or user-influenced URL — webhook registration, URL preview/unfurling, an "import from URL" feature, or an AI agent with a fetch/browse tool per `09-ai-integration.md`'s Axis 6 — is a specific, well-known pattern that needs deliberate safeguards: validating and restricting fetch targets, blocking requests to internal/private IP ranges, and network-level egress filtering, which is `21-networking.md`'s mechanical territory. Worth flagging explicitly to anyone building a URL-fetching feature or an agentic system with browsing/tool access, rather than assuming it's covered by Axis 4's input validation alone — SSRF specifically abuses the server's own network position, not just malformed input.
- **Software and data integrity failures:** ensuring the CI/CD pipeline only pulls signed, verified dependencies and build artifacts, and that any auto-update mechanism verifies integrity before applying an update, is `19-deployment-infra.md`'s pipeline-security territory rather than a new mechanism this file builds.
- **Vulnerability & dependency management:** scanning dependencies for known vulnerabilities and keeping them patched. Whether to do this at all is close to non-negotiable, the same tier as least-privilege (Axis 5); the real decision is scan frequency and response policy — continuous, CI-integrated scanning that blocks a build on a critical finding, versus periodic manual review — a genuine trade-off between friction and thoroughness. The actual tooling and mechanism live in `19-deployment-infra.md`'s CI/CD pipeline and `17-testing-strategy.md`'s testing layers, not here.
- **Insecure design:** not a specific item to add on top of the above — it's the umbrella concept this entire file, and the design-principles lens applied throughout this skill, already addresses. A system arrived at through this file's axes, deliberately reasoned about rather than defaulted into, is what "secure by design" means in practice.

**On application type (web, mobile, cloud-native, containerized):** this file deliberately doesn't branch by platform or deployment model. Platform (web/mobile/desktop/CLI) is already one of the skill's upfront questions and filters which categories apply throughout the *entire* walkthrough, not just this file — so platform-specific security nuance is handled by that existing filter rather than duplicated here. Container- and cloud-specific security considerations (image scanning, cloud IAM specifics, container runtime hardening) are a direct function of the compute model chosen in `19-deployment-infra.md`, and belong there as a cross-reference rather than as separate axes in this file.

---

## What drives the recommendation (from upfront questions)

- **Domain/compliance context**: the primary driver for nearly every axis here — regulated domains (finance, health, anything handling PII) push hard toward structured threat modeling, customer-managed encryption keys, formal data classification, and comprehensive security monitoring/audit logging; per the skill's excluded-categories note, don't assume this applies — ask if it's relevant to the specific project
- **Ops/deployment capacity**: low ops maturity weights toward provider-managed keys, a managed secrets manager rather than self-hosting an equivalent, and broader default roles over painstakingly granular least-privilege definitions — the cheaper end of each axis
- **Team size / ownership boundaries**: larger teams and more services benefit more from least-privilege scoping (Axis 5), since there's more surface area for a broad-permission mistake to matter; a solo developer's single service has comparatively less to gain from granular role definition
- **Time-to-ship pressure**: high pressure → TLS everywhere (non-negotiable), a managed secrets manager for API keys, provider-managed encryption keys, and basic least-privilege defaults, deferring structured threat modeling, formal data classification, and dedicated security monitoring until the system handles genuinely sensitive data or reaches real scale, is a legitimate first version
- **Consistency requirements**: not usually a primary driver for this file specifically, but connects directly to Axis 6 — anything requiring an immutable record for correctness or compliance reasons needs this file's audit logging specifically, not just `15-observability.md`'s general operational logging
- **Attack surface / feature set**: any URL-fetching feature (webhooks, link previews, an agentic tool with browsing access per `09-ai-integration.md`) is the direct driver for taking the SSRF note in "Additional hardening considerations" seriously rather than treating it as boilerplate; a system with no such feature has nothing to mitigate there

---

## Recommendation table skeleton

| Category | Option A | Option B | Recommendation + why | Approve |
|---|---|---|---|---|
| Threat modeling | *Ad hoc pros/cons* | *Structured (e.g. STRIDE) pros/cons* | *justify against domain/compliance context and the system's actual threat surface* | ☐ |
| Encryption & key management | *Provider-managed keys pros/cons* | *Customer-managed keys pros/cons* | *TLS in transit and encryption at rest are non-negotiable regardless; key-custody choice justified against compliance requirements* | ☐ |
| Secrets management | *Environment variables pros/cons* | *Dedicated secrets manager pros/cons* | *dedicated manager is the default beyond a small/early project; hardcoded secrets are never acceptable* | ☐ |
| Input validation & injection prevention | *Single-layer validation pros/cons* | *Defense-in-depth (multi-layer) pros/cons* | *defense-in-depth is the standard expectation regardless of project size; parameterized queries are non-negotiable baseline practice* | ☐ |
| Least-privilege access scoping | *Broad/shared credentials pros/cons* | *Scoped, least-privilege roles pros/cons* | *close to non-negotiable as a principle; investment level justified against team/service count* | ☐ |
| Security monitoring & audit logging | *None pros/cons* | *Monitoring + audit logging pros/cons* | *justify against compliance requirements and threat surface* | ☐ |
| Data classification | *No formal classification pros/cons* | *Formal tiers pros/cons* | *only for systems handling genuinely sensitive/regulated data; otherwise omit this row* | ☐ |

Note: for an early-stage or low-stakes project under time pressure, this can reasonably collapse to four rows ("TLS everywhere plus a managed secrets manager," "parameterized queries/ORM as baseline practice, gateway-level validation as the second layer," "least-privilege defaults without heavy role granularity," and "provider-managed keys, no formal classification or dedicated security monitoring yet") rather than seven — defer structured threat modeling, formal data classification, and dedicated security monitoring until real sensitive data or scale is present. Input validation's baseline practice (parameterized queries) is never deferred, regardless of project stage.

---

## Related files

- `01-system-level.md` — least-privilege scoping (Axis 5) applies the same blast-radius-containment principle as service boundaries there, to permissions instead of deployables
- `02-data-storage.md` — encryption at rest is often a near-default feature of the managed database chosen there; this file's key-custody question (Axis 2) applies on top
- `03-backup-disaster-recovery.md` — that file's backup-specific key-custody note is a specific case of this file's Axis 2, applied to backups
- `04-file-storage.md` — that file's access-control axis is the storage-layer mechanism enforcing the policy this file decides
- `05-scalability.md` — brute-force/credential-stuffing protection on login endpoints is that file's rate-limiting axis, applied to the authentication surface specifically
- `06-auth.md` — user-facing authorization model (RBAC/ABAC) is covered there; this file's least-privilege axis covers infrastructure/service-level access, the broader principle; MFA and brute-force protection are this file's additions on top of that file's mechanism choice
- `08-integrations.md` / `09-ai-integration.md` — API key custody for third-party and AI providers is the concrete case of this file's Axis 3; an agentic tool with URL-fetching access is a classic SSRF vector flagged in this file's hardening notes
- `13-caching.md` — sensitive data cached client-side or in a shared cache is this file's access-control policy applied specifically to caching
- `14-resilience.md` — a secrets manager is itself an external dependency subject to that file's timeout/circuit-breaker reasoning if it becomes slow or unavailable
- `15-observability.md` — security monitoring and audit logging (Axis 6) is the concrete home for what that file explicitly deferred here
- `17-testing-strategy.md` / `19-deployment-infra.md` — vulnerability/dependency scanning mechanism and CI/CD pipeline/software-supply-chain integrity live there
- `20-environment-strategy.md` — the environment-specific hardening piece of security misconfiguration (production vs staging/dev config) lives there
- `21-networking.md` — TLS termination, firewalls, network segmentation, and SSRF egress filtering are the mechanical enforcement of this file's encryption and access policy
