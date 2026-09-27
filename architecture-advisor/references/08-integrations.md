# 08 — Third-Party Integrations

## What this category covers

Calling out to external services this project doesn't control — payment processors (Stripe, etc.), email/SMS providers (SendGrid, Twilio), and other third-party APIs a project depends on to do part of its job. The core concern is coupling and vendor dependency: how tightly the codebase binds to a specific provider, and how the system behaves when that provider is slow, down, or changes its API.

Distinct from `07-api-design.md`: that file covers designing the API *this* project exposes to its own clients; this file covers *consuming* APIs the project doesn't own or control.

Distinct from `09-ai-integration.md`: AI provider APIs are technically a third-party integration too, but are split into their own file because of AI-specific concerns (cost control per call, streaming, prompt/context handling, RAG) that don't apply to a typical payment or email integration. `09` should cross-reference here for the general integration-coupling reasoning rather than repeating it.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Integration coupling — direct SDK/API calls vs an abstraction layer

### Direct calls to the provider's SDK/API

- **Pros:** simplest and fastest to implement — call the provider's SDK directly where needed, no extra indirection to design or maintain; provider-specific features are immediately available without an abstraction layer lagging behind or needing to expose them
- **Cons:** the provider's types, error shapes, and API quirks leak throughout the codebase wherever the integration is used; switching providers later (forced migration, pricing change, provider shutdown) means touching every call site rather than one boundary; harder to mock/test in isolation without an abstraction to substitute at test time

### Abstraction layer / adapter (an internal interface the app calls, with the provider's SDK behind it)

- **Pros:** provider-specific details are isolated behind one boundary — swapping providers later means rewriting the adapter, not every call site; easier to test the rest of the application against a fake/mock implementation of the interface; can support multiple providers behind the same interface if genuinely needed (e.g. a fallback email provider)
- **Cons:** real upfront design and maintenance cost — the abstraction has to be designed well enough to accommodate the provider's actual capabilities, or it either leaks provider details anyway or becomes a bad fit and gets bypassed; can turn into unnecessary indirection for a provider that's very unlikely to ever be swapped (a classic over-engineering trap — see the design-principles lens applied throughout this skill); the abstraction is only as good as its ability to represent every provider it might need to support, which is hard to get right speculatively

The classic trade-off here is YAGNI vs future-proofing: an abstraction layer pays off when a provider swap is a real, foreseeable possibility (multiple viable competitors, provider lock-in risk, planned multi-provider fallback) and is wasted effort when the provider is a near-permanent, deeply-integrated choice (e.g. a payment processor's compliance/PCI surface is rarely worth abstracting away, since a swap would be a major project regardless of the abstraction).

---

## Axis 2: Handling provider failure — synchronous dependency vs decoupled/async

### Synchronous, in-request dependency

- **Pros:** simplest to reason about — the request either succeeds with the integration's result or it doesn't; no eventual-consistency complexity; appropriate when the integration's result is genuinely required to complete the request (e.g. a payment must actually succeed before an order is confirmed)
- **Cons:** the provider's latency and availability become part of *this* request's latency and availability — a slow or down provider directly degrades or breaks the user-facing request; without resilience patterns (`14-resilience.md` — timeouts, retries, circuit breakers), a single flaky integration can cascade into broader outages

### Decoupled / asynchronous (queued, background-processed, or webhook-driven)

- **Pros:** the user-facing request isn't blocked on the third-party call — appropriate when the integration's result isn't needed immediately (e.g. sending a confirmation email, syncing to an analytics provider); isolates the provider's failure modes from the critical request path; pairs naturally with `11-background-jobs.md` for the queuing/retry mechanics
- **Cons:** adds real complexity — a queue or job system to build/operate, and the user-facing flow needs to account for the integration's result arriving later (webhook handling, polling, or simply not surfacing failure to the user in real time); not viable when the integration's result genuinely must be known before the request can complete

Which one applies is almost entirely determined by whether the integration's result is on the critical path of the request or not — this isn't really a stylistic choice so much as a direct consequence of what the integration is for. Payment confirmation before order completion is inherently synchronous; a post-signup welcome email is inherently a background-safe candidate.

---

## Axis 3: Resilience posture for integration calls

A third-party call is an external dependency by definition, so how the system handles it being slow or unavailable is a real decision, not an afterthought.

### No explicit resilience handling

- **Pros:** simplest — nothing extra to implement
- **Cons:** a slow or down provider directly becomes a slow or down feature (or, if synchronous and unhandled, a slow or down request) with no graceful degradation; acceptable only for genuinely non-critical, low-stakes integrations or very early-stage projects

### Timeouts, retries, and circuit breakers

- **Pros:** bounds how long a call to a misbehaving provider can hold up the caller (timeout); recovers automatically from transient failures without manual intervention (retry, ideally with backoff); stops hammering a provider that's confirmed down, giving it room to recover and protecting the caller's own resources (circuit breaker)
- **Cons:** genuine implementation complexity — retries need idempotency handling on the provider side to avoid duplicate side effects (e.g. double-charging a payment), and circuit breaker thresholds need tuning to avoid false positives; this is the full domain of `14-resilience.md`, so this file should point there rather than re-deriving the pattern

This axis is largely a pointer rather than a full decision made here — the actual retry/circuit-breaker/timeout patterns live in `14-resilience.md`. What belongs specifically in this file is recognizing that *every* third-party integration is a candidate for those patterns, and flagging which integrations are critical-path enough to warrant them versus which are low-stakes enough to skip.

---

## What drives the recommendation (from upfront questions)

- **Domain coupling / how central the integration is**: a payment processor at the core of the business model is worth different treatment than an optional analytics integration — the more central and swap-unlikely the provider, the less an abstraction layer (Axis 1) pays for itself, and the more resilience investment (Axis 3) is justified regardless of ops capacity
- **Ops/deployment capacity**: low ops maturity weights toward direct SDK calls (Axis 1) over building an abstraction layer, and toward simpler resilience handling over a full circuit-breaker implementation — a managed retry library or the provider's own SDK-level retry support is often a lower-cost starting point than hand-building the pattern
- **Time-to-ship pressure**: high pressure → direct SDK calls, synchronous where the integration is genuinely required, no explicit resilience layer beyond what the SDK provides by default, is a legitimate answer for a first version — not a gap, but worth flagging as technical debt if the integration is payment-critical
- **Consistency requirements**: integrations with financial or otherwise irreversible side effects (payments, anything charging a user) push hard toward careful idempotency handling in any retry logic (Axis 3) — a naive retry without idempotency keys is a real correctness bug, not just a resilience nicety
- **Team size / ownership boundaries**: a system with multiple services each needing the same integration (e.g. several services all needing to send email) strengthens the case for a shared abstraction layer (Axis 1) so the integration logic isn't duplicated per service

---

## Recommendation table skeleton

| Category | Option A | Option B | Recommendation + why | Approve |
|---|---|---|---|---|
| Integration coupling | *Direct SDK calls pros/cons* | *Abstraction layer pros/cons* | *pick one per integration if they differ (e.g. abstract email, don't abstract payments); justify against swap likelihood and ops capacity* | ☐ |
| Sync vs async handling | *Synchronous pros/cons* | *Decoupled/async pros/cons* | *state which integrations are critical-path (sync) vs background-safe (async); note background-jobs dependency if async* | ☐ |
| Resilience posture | *No explicit handling pros/cons* | *Timeouts/retries/circuit breakers pros/cons* | *flag which integrations warrant full resilience treatment (financial/critical-path) vs which can skip it; point to `14-resilience.md` for mechanics* | ☐ |

Note: real projects usually have several distinct integrations (payments, email, maybe a couple more) that don't all land the same way on these axes — it's reasonable and often clearer to show per-integration rows (e.g. "Payments: sync, direct SDK, full resilience" / "Email: async, abstraction layer, basic retry only") rather than forcing one answer to cover every integration the project has.

---

## Related files

- `07-api-design.md` — that file covers the API this project exposes; this file covers APIs this project consumes
- `09-ai-integration.md` — AI provider calls are integrations too, but split out for AI-specific cost/streaming/RAG concerns
- `11-background-jobs.md` — the queuing/retry mechanics behind any async-handled integration (Axis 2) live there
- `14-resilience.md` — full timeout/retry/circuit-breaker pattern detail for Axis 3; this file only flags which integrations need it
- `16-security.md` — API key/credential storage for third-party providers falls under general secrets-management policy there
