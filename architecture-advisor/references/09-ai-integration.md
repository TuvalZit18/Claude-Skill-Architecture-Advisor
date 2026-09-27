# 09 — AI Integration

## What this category covers

Integrating AI model calls (LLMs, embedding models, or other AI services) into the system: whether the model is consumed as a hosted API or run self-hosted, where the call is made from, whether responses are synchronous, streamed, or async, whether retrieval-augmented generation (RAG) and a vector store are needed, how cost is controlled, whether/how the system uses agentic behavior — tool/function calling, MCP, multi-agent orchestration, and reusable "skill" packages — versus a simple single-shot call, whether a human reviews AI output/actions before they take effect, and how context is managed as agent sessions and available capabilities grow.

Distinct from `08-integrations.md`: a hosted AI provider is technically a third-party integration, and that file's coupling/resilience reasoning (abstraction layers, sync vs async, timeouts/retries) still applies here and isn't repeated — this file covers what's specific to AI calls: hosted-vs-self-hosted trade-offs, API key exposure risk, streaming UX, RAG/context-retrieval infrastructure, and per-call cost, none of which have a real analog in a typical payment or email integration.

**Scope note:** this file covers the *architectural* shape of AI integration — not which specific provider or model to use (OpenAI vs Anthropic vs Google, or which model within a provider's lineup). Provider/model selection is a fast-moving, benchmark-driven choice much closer to the DB-tooling/vendor-selection exclusion already noted in the skill's excluded-categories list than to a structural architecture decision, and doesn't belong in a project-agnostic reference file that would go stale quickly. Axis 1 below covers the one part of "which model" that *is* architectural — hosted API vs self-hosted/open-weight — without naming specific vendors or models.

Also out of scope: **prompt engineering and system-prompt engineering** — the actual crafting of prompt wording, few-shot examples, chain-of-thought instructions, or system-prompt content. This is implementation-level craft applied *within* whichever architecture this file's axes settle on, the same category as the skill's existing algorithms/data-structures exclusion — it has no real "Option A vs Option B" architectural table to build, since the choice is about wording and technique, not structure. Where a genuine structural decision does hide nearby — *how the prompt/system-prompt is managed as a deployable artifact* — that's covered as a one-line note under Axis 8 below, since it is architectural (versioned/externalized config vs hardcoded) even though the prompt's actual content isn't.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Hosted API vs self-hosted / open-weight model

The one part of "which model" that's a genuine architecture decision rather than a vendor pick — does the model run behind a third-party's hosted API, or does the project run the model itself (open-weight model on owned or rented infrastructure).

### Hosted API (provider-run, billed per call/token)

- **Pros:** zero infrastructure to provision or maintain for the model itself — the provider handles serving, scaling, and hardware; access to frontier-capability models without needing the infrastructure or expertise to run them; pay-per-use cost model with no idle-capacity waste; fastest path to shipping an AI feature
- **Cons:** ongoing per-call cost that scales directly with usage, versus a largely fixed cost for self-hosted infrastructure at high sustained volume; data sent to the provider leaves the project's own infrastructure, a real consideration for sensitive or regulated data (`16-security.md`) — the standard mitigating practice is data minimization, sending only the fields or context a given call actually needs rather than a full record or document, which reduces both privacy exposure and token cost; dependent on the provider's uptime, rate limits, and pricing/model-availability changes over time — the same vendor-dependency reasoning as `08-integrations.md` applies, just with a harder-to-replace dependency given model-specific behavior differences; hosted models are also periodically deprecated or sunset by the provider on a timeline outside the project's control, forcing a migration to a newer model version whether or not the project is ready — a stronger and more certain version of the general vendor-lock-in risk, and a concrete reason to isolate model calls behind an abstraction layer (`08-integrations.md` Axis 1) even when swapping providers entirely feels unlikely; latency includes a network round-trip to the provider

### Self-hosted / open-weight model

- **Pros:** data never leaves the project's own infrastructure — the clear answer when data residency, regulatory, or confidentiality requirements rule out sending data to a third party; cost becomes largely fixed (infrastructure) rather than scaling per call, which can be meaningfully cheaper at high sustained volume; full control over model version, fine-tuning, and availability — no risk of a provider deprecating or changing a model out from under the project
- **Cons:** significant infrastructure and ops burden — provisioning and maintaining (often GPU) compute, managing model deployment/updates, and building the serving layer the hosted providers otherwise handle; open-weight models generally trail the best hosted frontier models on raw capability, though the gap varies by task and narrows over time; upfront and ongoing infrastructure cost has to be justified by volume or by a hard requirement (data residency, compliance) rather than by cost alone at low-to-moderate usage; requires in-house ML/infra expertise that most teams building an AI *feature* (as opposed to an AI *product*) don't have

For the large majority of projects adding an AI feature to a broader product, a hosted API is the right default — self-hosting is justified specifically by a hard data-residency/compliance requirement, or by sustained volume high enough that the infrastructure cost genuinely undercuts per-call billing, not by a general preference for control or cost-consciousness alone.

---

## Axis 2: Where the AI call is made from

### Client-side (browser or mobile app calls the AI provider directly)

- **Pros:** lowest latency path — no extra hop through the app's own backend; simplest to implement for a quick prototype; server has less request volume to handle
- **Cons:** exposes the API key to the client, a serious security problem — anyone can extract it from the client bundle/traffic and use it, running up cost or abusing the account on the project's dime; effectively never appropriate for a real product with a provider API key that bills per use, which is the overwhelming majority of hosted AI providers; no opportunity to apply server-side cost controls, rate limiting, or content filtering before the call is made

### Server-side (app backend proxies the call to the AI provider)

- **Pros:** the API key stays server-side, never exposed to the client — the standard and effectively required pattern for any provider with a billed API key; enables cost control (Axis 5), rate limiting, request validation/moderation, and business logic (e.g. checking user entitlement) before the call is made; the app controls exactly what context/prompt is sent, rather than trusting the client to construct it correctly — the natural enforcement point for data-minimization practice (Axis 1), since the server can strip or exclude fields the call doesn't actually need before it ever reaches the provider
- **Cons:** adds a hop (client → app server → AI provider) versus calling the provider directly, though this is rarely the dominant source of latency compared to the model call itself; the app server has to handle streaming responses back to the client if streaming is used (Axis 3), which is more involved than a simple request/response proxy

Client-side calling is essentially never the right default once real API keys and billing are involved — worth stating plainly rather than presenting as a genuinely close call. The one legitimate client-side exception is a fully local/on-device model with no billed API key and no server round-trip needed at all, which is a different architecture entirely (no proxy needed because there's no key to protect).

---

## Axis 3: Response handling — synchronous, streaming, or async

### Synchronous (wait for the full response, return it in one piece)

- **Pros:** simplest to implement — a normal request/response cycle, no special client-side handling needed
- **Cons:** for anything beyond a short response, the user waits with no feedback for however long generation takes, which for LLMs can be several seconds to tens of seconds — a poor UX for anything conversational or long-form

### Streaming (send tokens/chunks to the client as they're generated)

- **Pros:** the standard UX pattern for chat-style or long-form AI output — the user sees progress immediately rather than staring at a blank loading state; perceived latency is dramatically lower even though total generation time is the same
- **Cons:** more implementation complexity on both server (proxying a streamed response, typically via Server-Sent Events or a similar mechanism) and client (incrementally rendering partial output); harder to apply certain server-side checks that need the full response first (e.g. a full-response content moderation pass has to either run post-hoc on the complete streamed output or be skipped/approximated)

### Async (submit a job, poll or notify when the result is ready)

- **Pros:** appropriate when generation is long-running (batch processing, large document analysis) or when the user doesn't need to watch it happen in real time; decouples the request from the AI call entirely, fitting naturally with `11-background-jobs.md`'s queuing patterns; avoids holding a request/connection open for a long-running call
- **Cons:** not appropriate for interactive/conversational use cases where the user is actively waiting on a response in the moment; adds the same complexity as any async flow (`08-integrations.md` Axis 2) — the client needs a way to learn the result is ready

Streaming is the default for any interactive/chat-style feature; synchronous is fine for short, non-conversational outputs (e.g. a single classification or short summary call); async fits batch or long-running generation that isn't part of an interactive loop.

---

## Axis 4: RAG (Retrieval-Augmented Generation) and vector store needs

Whether the AI calls need to be grounded in the project's own data rather than relying solely on the model's built-in knowledge.

### No retrieval (direct prompting only)

- **Pros:** simplest — no additional infrastructure; appropriate when the task doesn't depend on private/proprietary/current data the model wouldn't already know
- **Cons:** the model can't answer questions about the project's own data, can hallucinate specifics it doesn't actually know, and has no way to stay current on information after its training cutoff without this being fed in some other way

### RAG with a vector store

- **Pros:** grounds responses in the project's actual data (documents, support tickets, product catalog, etc.) by retrieving relevant context and including it in the prompt — meaningfully reduces hallucination on domain-specific questions and lets the system answer about private/proprietary/current data the base model was never trained on; the vector store itself (Pinecone, pgvector, Weaviate, or similar) can often piggyback on an existing storage choice (e.g. pgvector on an already-chosen Postgres instance, per `02-data-storage.md`) rather than requiring a wholly new system
- **Cons:** real additional infrastructure and pipeline to build and maintain — chunking strategy, embedding generation and refresh as source data changes, retrieval quality tuning (is the right context actually being retrieved); adds latency (the retrieval step happens before the generation call) and cost (embedding calls, vector store hosting); retrieval quality is its own ongoing tuning problem, not a one-time setup

Whether RAG is needed is a direct consequence of what the AI feature actually needs to know — a general-purpose writing assistant likely doesn't need it; a support chatbot that should answer from the project's own documentation almost certainly does.

---

## Axis 5: Cost control

AI API calls are usually billed per token or per call in a way that scales directly and sometimes unpredictably with usage, making this a more deliberate architectural concern than most other integrations.

### No explicit cost controls

- **Pros:** simplest — nothing extra to build
- **Cons:** usage-driven cost can grow unpredictably, especially with a public-facing feature or one vulnerable to abuse (e.g. a user scripting many rapid requests); no ceiling on worst-case spend

### Rate limiting / per-user quotas

- **Pros:** bounds worst-case cost per user or in aggregate — directly reuses the rate-limiting patterns from `05-scalability.md`'s rate-limiting axis, applied specifically to AI-call volume rather than general API traffic; can be tiered (different quotas for different user/subscription tiers)
- **Cons:** needs a tracking mechanism (usage counters, typically in the same shared store used for general rate limiting) and a policy decision about what happens when a user hits their quota (hard block vs degraded service)

### Model/prompt cost optimization

- **Pros:** reduces cost per call directly — using a smaller/cheaper model where task complexity allows, trimming unnecessary context or prompt length, caching repeated or near-duplicate queries (an AI-specific application of `13-caching.md`'s general caching strategy) rather than only bounding call *volume*
- **Cons:** requires ongoing attention as models, prices, and the project's actual usage patterns change; model-selection trade-offs (cheaper model vs output quality) are a real product decision, not a free optimization

These aren't mutually exclusive — a mature system typically combines per-user rate limiting (bounding worst-case volume) with prompt/model cost optimization (reducing cost per call within that volume) rather than relying on just one.

---

## Axis 6: Agentic behavior — tool use & orchestration

Whether the AI integration is a single prompt-in/response-out call, or involves the model taking multi-step actions, calling tools, or coordinating across multiple specialized agents. This is the more fundamental question that MCP (below) is one implementation detail of, not a synonym for it.

### Single-shot (no tool use)

- **Pros:** simplest, most predictable, cheapest and fastest — one call, one response, nothing else to design; the right default for anything that's fundamentally a text-in/text-out task
- **Cons:** can't take actions or reach live external systems/data beyond what's already in the prompt — not viable once the feature genuinely needs the model to *do* something, not just respond

### Single agent with tool/function calling

- **Pros:** the model can decide when to call defined tools/functions and incorporate results into further reasoning — opens up real actions (querying a database, calling an internal API, running a calculation) beyond static prompt context; most hosted providers support this natively via a function-calling API, so no extra framework is required to get basic tool use working
- **Cons:** non-deterministic — the model decides when and whether to call a tool, so behavior needs testing and guardrails beyond typical deterministic software testing; each tool call is a further round trip to the model, adding latency and cost on top of the base call; tool definitions and descriptions need careful design, since ambiguous or overlapping tools lead to the model misusing or confusing them

  **MCP (Model Context Protocol):** once tool use is warranted, this is the specific choice of *how* tools are connected — a standardized protocol for exposing tools and data sources to the model in a consistent way, versus defining functions directly against a provider's own function-calling API. MCP's case strengthens once there's more than one or two tools/data sources to connect (avoids bespoke, model-specific glue code per integration, and a growing ecosystem of pre-built MCP servers reduces custom work) or when tool access needs to be shareable/reusable across different AI features or even different model providers. Its cost is a real added protocol layer and, depending on architecture, additional running infrastructure (MCP servers) — unnecessary overhead for one or two simple tools, where calling a provider's function-calling API directly is the lower-overhead path.

### Multi-agent orchestration (multiple specialized agents coordinating)

- **Pros:** decomposes a complex task into specialized sub-agents (e.g. a research agent, a writing agent, a review agent), each with a narrower and more reliable scope than one agent trying to do everything — the same specialization-and-separation-of-concerns reasoning as `01-system-level.md`'s service-boundary guidance, applied to agents instead of services; independent sub-tasks can run in parallel
- **Cons:** significant added complexity — orchestration logic (which agent runs when, how results are combined or handed off), failure modes compound across agents rather than staying contained to one, and cost/latency multiply with every additional agent hop; debugging a multi-agent chain is substantially harder than reasoning about one agent's behavior; genuinely justified only when a single agent with tools demonstrably can't handle the task's complexity or breadth reliably — not a default architecture to reach for because it sounds more sophisticated

Single-shot should be the default; tool use is justified once the feature genuinely needs the model to act or fetch live data; multi-agent orchestration is the narrowest, most advanced tier and should be flagged as advanced/rare and generally omitted from a project's shown table, the same way space-based architecture (`05-scalability.md`) and microkernel (`01-system-level.md`) are — include it only when a real, demonstrated need for multiple coordinating specialized agents exists, not as a default for "agentic" work.

**Skills / reusable capability packages:** worth a one-line note rather than a full option — some platforms support bundling reusable instructions, context, and sometimes tool bindings into a named, invokable unit (distinct from a raw tool/function call — closer to a packaged capability or sub-prompt the agent can pull in on demand). This is an emerging, platform-specific pattern rather than a settled cross-provider architectural decision; worth flagging as a consideration for extensibility and reuse in agent-heavy systems with many recurring, well-defined sub-tasks, rather than treating it as its own table row.

This whole axis should be explicitly excluded from a project's shown table when the AI integration is simple prompt/response (summarization, classification, content generation) with no tool-calling or external-action component — forcing it in for every AI-touching project would misrepresent agentic behavior as more broadly necessary than it is.

---

## Axis 7: Augmentation vs automation (human-in-the-loop)

Whether the AI's output or action requires human review/approval before it takes effect (augmentation — the AI assists a human who remains the decision-maker) or takes effect autonomously with no human checkpoint (automation). Distinct from Axis 6: agentic behavior is about *whether* the AI takes actions or calls tools at all; this axis is about whether a human checks those outputs/actions before they're real, and applies just as much to a simple content-generation call (does a human approve the drafted email before it sends?) as to a tool-using agent.

### Augmentation (human-in-the-loop)

- **Pros:** a human catches errors or hallucinations before they cause real-world harm or cost; builds trust incrementally — the human validates the system's actual reliability over time before any consideration of removing the checkpoint; often a legal, regulatory, or professional-standards requirement in some domains (e.g. medical, financial, hiring decisions) rather than a purely technical preference; the reasonable default for a new or unproven AI feature where reliability hasn't yet been established in practice
- **Cons:** adds friction and latency to the workflow — the feature's value is capped by how much of the human's time it actually saves once review is factored in; doesn't scale as well, since a human review step is a bottleneck that grows roughly linearly with volume; can degrade into "review theater" if the checkpoint becomes a rubber stamp that doesn't add real oversight while still costing the UX friction of an extra step

### Automation (autonomous, no human checkpoint)

- **Pros:** full time/labor savings with no human bottleneck in the loop; scales without a linear human-review cost as volume grows; appropriate once a task's reliability at scale is well-understood and errors are low-stakes, cheap, or easy to reverse
- **Cons:** errors and hallucinations reach the real world or downstream systems directly, so the blast radius of a mistake is larger than in an augmented flow; requires materially higher confidence in the model's reliability for the *specific* task before removing the human checkpoint is appropriate, not just general confidence in AI capability; raises sharper liability, compliance, and audit questions (`16-security.md`) than an augmented flow where a human remains accountable for the final action; can damage trust with stakeholders or end-users if introduced before reliability is actually demonstrated, harder to walk back than it was to avoid in the first place

Which applies is mostly a function of task stakes and error reversibility, not team preference: low-stakes, cheaply-reversible tasks (drafting, suggesting, first-pass classification a human can still override downstream) are reasonable automation candidates fairly early; high-stakes or hard-to-reverse tasks (financial transactions, medical or legal decisions, anything costly or impossible to undo) should default to augmentation until reliability is proven, and in regulated domains this may be a compliance requirement rather than a discretionary choice at all. A common and realistic path is to ship augmentation first and graduate specific, well-understood sub-tasks to automation as confidence is established through actual production experience, rather than committing to one posture permanently at design time.

This axis interacts directly with Axis 6: an agent taking real-world actions through tools is exactly where the automation side carries the most risk, since a tool call (sending an email, making a purchase, modifying a record) is a real-world effect a human can't silently disregard the way they might ignore a wrong line of generated text.

---

## Axis 8: Context management strategy

A finite-resource problem that becomes architecturally significant once a system has real agentic behavior (Axis 6) — many available tools/skills, long multi-step sessions, or large intermediate results — rather than a single short prompt/response call. The context window is a limited, costed resource; how it's managed is a real design decision, not just an implementation detail.

### Static / full context (everything loaded upfront)

- **Pros:** simplest to implement — all available tool definitions, instructions, and reference material go into the system prompt or context every call; no discovery or loading logic needed
- **Cons:** doesn't scale — as the number of available tools, skills, or reference documents grows, they consume context budget whether or not they're relevant to the current task, competing directly with the space available for actual conversation and reasoning; can degrade output quality even before hitting a hard token limit, since irrelevant loaded content can distract from what's actually relevant to the current step

### Progressive disclosure / on-demand loading

- **Pros:** keeps the base context small by only loading a tool's, skill's, or document's full definition when it's actually about to be used — a lightweight index or search step (e.g. searching available tools/skills by keyword, then loading only the matched one's full definition) scales to far more available capabilities than would ever fit if all were loaded upfront; keeps each individual call focused on what's relevant to the current step, rather than diluted by every possible capability the system has
- **Cons:** adds a real design and latency cost — the discovery/search step itself is an extra round trip or lookup before the actual work happens; requires each tool/skill/document to have a good enough short description for the discovery step to reliably surface the right one, which is its own design and maintenance burden; more moving parts than simply loading everything upfront

### External state / file-based scratchpad

- **Pros:** lets the agent read and write large intermediate results, working data, or accumulated findings to an external store (a filesystem, a database, a dedicated "memory" or scratchpad store) rather than keeping them in the conversation context itself — the agent can reference a file by path/key instead of re-stating its full contents in-context on every turn; particularly valuable for long-running or multi-step agent sessions that accumulate more working data than would reasonably fit or stay affordable in-context
- **Cons:** needs actual storage infrastructure to back it (ties into `04-file-storage.md` or `02-data-storage.md` depending on what's being stored), plus a policy for what belongs in-context versus externalized; the agent has to be deliberately prompted/designed to use the external store rather than defaulting to stuffing everything into context, which doesn't happen automatically just because the storage exists

### Context compaction / summarization

- **Pros:** for long-running conversational or agentic sessions, periodically summarizing or pruning older turns keeps the active context within budget without simply truncating and losing everything past a cutoff — preserves the gist of earlier context while freeing room for continued work
- **Cons:** summarization is inherently lossy — details that turn out to matter later can be dropped in the process; adds its own model call (or a rules-based pruning step) and design effort to decide what's safe to compact versus what must be preserved verbatim (e.g. exact figures, prior tool outputs a later step depends on)

These aren't mutually exclusive — a mature agentic system commonly combines progressive disclosure for tool/skill/reference-document access with a file-based scratchpad for large working data and, for genuinely long sessions, periodic compaction of conversational history. A simple, short-lived AI feature with few tools has little need for any of this; static/full context is the right default until the number of tools/skills or the length of a working session actually strains context budget in practice.

**Prompt/system-prompt management as an artifact:** worth a one-line note rather than a full option — separately from what the prompt *says* (out of scope for this file, see the scope note above), *where the prompt lives and how it's deployed* is a small but real architectural fork: hardcoded inline in application code (simplest, but changing prompt wording requires a full code deploy) versus externalized into a versioned, independently-updatable store (a config service, a database table, or a dedicated prompt-management platform), which lets prompts be iterated on and rolled back without a code deploy — at the cost of an extra system to run and a version-skew risk between prompt and code if they're not deployed together deliberately. This is closer in kind to the feature-flag/config-management reasoning in `20-environment-strategy.md` than to anything else in this file, and is worth flagging only when prompt iteration speed independent of code deploys is a real, stated need — not a default for every project.

---

## What drives the recommendation (from upfront questions)

- **Domain/compliance context**: a hard data-residency, regulatory, or confidentiality requirement (Axis 1) is close to the only strong driver toward self-hosting — absent that, a hosted API is the default; per the skill's excluded-categories note, don't assume a compliance driver applies, ask if it's relevant to the specific project
- **Platform**: no strong direct driver on its own, but combined with the API-key exposure reasoning in Axis 2, any platform with a real billed provider key should default to server-side calls regardless of platform — this is closer to a non-negotiable than a project-specific trade-off
- **Time-to-ship pressure**: high pressure → hosted API (Axis 1), server-side proxying (non-negotiable per Axis 2), synchronous or basic streaming, no RAG unless the feature fundamentally requires grounding in project data, and simple rate-limiting for cost control, is a legitimate first version
- **Budget/cost constraints**: tight budget strengthens the case for Axis 5's rate limiting and model/prompt cost optimization from day one rather than as a later addition, and may push toward a cheaper model tier even at some quality cost; note that self-hosting (Axis 1) is not itself a budget-saving default at low-to-moderate volume, since the fixed infrastructure cost usually exceeds hosted-API billing until volume is genuinely high
- **Domain coupling / task complexity**: features that need to answer from or act on the project's own proprietary data are the direct driver for RAG (Axis 4); whether the feature needs to *act* (take steps, call tools, reach live systems) rather than just respond is the direct driver for Axis 6 — most AI features are single-shot and general-purpose (writing help, summarization) and need neither; tool use and especially multi-agent orchestration should only enter the table when a specific, demonstrated task need justifies the added complexity
- **Ops/deployment capacity**: low ops maturity weights strongly against self-hosting (Axis 1) regardless of other pressures, and against building custom RAG infrastructure or standing up MCP servers or multi-agent orchestration (Axis 6) unless the feature genuinely requires them — a managed vector-store option, direct function-calling instead of MCP, or a single agent instead of multiple, are all the lower-cost path
- **Consistency requirements**: not usually a primary driver here, but worth flagging if the AI feature's output needs to be deterministic/auditable (e.g. anything AI-assisted feeding into a compliance-sensitive workflow) — this pushes toward synchronous handling with logging of the full prompt/response pair, rather than a fire-and-forget async call, and pushes against multi-agent orchestration's compounding non-determinism specifically; it also weighs against lossy context compaction (Axis 8) for anything where losing earlier detail is unacceptable
- **Domain/compliance context and task stakes**: the direct driver for Axis 7 — regulated domains (medical, financial, legal, hiring) often push toward augmentation as a compliance requirement rather than a discretionary choice; even outside regulated domains, high-stakes or hard-to-reverse actions should default to augmentation, while low-stakes, cheaply-reversible outputs are reasonable automation candidates; a new or unproven AI feature of any kind should generally start augmented and graduate toward automation as production reliability is demonstrated, not the reverse
- **Scale of agentic capability / session length**: the direct driver for Axis 8 — a system with a handful of tools and short, single-turn interactions has little need for progressive disclosure, external scratchpads, or compaction; a system with many available tools/skills, or agent sessions that run long and accumulate substantial working data, pushes toward some combination of them well before context limits are actually hit in practice, not as an emergency fix after they are

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| Hosted vs self-hosted model | *Hosted API pros/cons* | *Self-hosted/open-weight pros/cons* | — | *hosted API is the default; self-hosting needs a hard data-residency/compliance or high-sustained-volume justification, not general preference* | ☐ |
| Call origin | *Client-side pros/cons* | *Server-side pros/cons* | — | *server-side is close to non-negotiable with a billed API key; state this plainly rather than as a genuine toss-up (n/a if fully local/on-device, per Axis 2)* | ☐ |
| Response handling | *Synchronous pros/cons* | *Streaming pros/cons* | *Async pros/cons* | *justify against interactivity — streaming for chat/conversational, sync for short outputs, async for long-running/batch* | ☐ |
| RAG / retrieval | *No retrieval pros/cons* | *RAG + vector store pros/cons* | — | *justify against whether the feature needs grounding in project-specific data; note if vector store can piggyback on existing storage choice* | ☐ |
| Cost control | *No controls pros/cons* | *Rate limiting/quotas pros/cons* | *Model/prompt optimization pros/cons* | *pick a combination; justify against budget constraints and abuse/scale risk* | ☐ |
| Agentic behavior | *Single-shot pros/cons* | *Single agent + tool use (incl. MCP sub-choice) pros/cons* | *Multi-agent orchestration pros/cons, flagged advanced/rare* | *pick one; single-shot is the default, tool use requires a demonstrated action/live-data need, multi-agent orchestration only for demonstrated task complexity a single agent can't handle* | ☐ |
| Augmentation vs automation | *Augmentation (human-in-the-loop) pros/cons* | *Automation (autonomous) pros/cons* | — | *justify against task stakes and error reversibility; default to augmentation for new/unproven features or regulated domains, automation for low-stakes/cheaply-reversible outputs* | ☐ |
| Context management | *Static/full context pros/cons* | *Progressive disclosure pros/cons* | *External scratchpad / compaction pros/cons* | *justify against number of available tools/skills and expected session length; static/full is the default until either strains context budget in practice* | ☐ |

Note: for a simple, low-stakes AI feature (e.g. a single summarization or content-generation call) under time pressure, this can reasonably collapse to three rows ("hosted API, server-side proxying, synchronous or basic streaming" plus "basic rate limiting, no RAG, single-shot with no tool use, augmented by default") rather than eight — drop the RAG, agentic-behavior, and context-management rows by default per their respective axis guidance above, and the hosted-vs-self-hosted and augmentation-vs-automation rows can each collapse into a single stated assumption ("hosted API"; "augmented") when no compliance/volume/stakes driver pushes otherwise, rather than full pros/cons rows.

---

## Related files

- `08-integrations.md` — general third-party integration coupling/resilience reasoning applies to AI providers too; this file covers what's AI-specific on top of that
- `01-system-level.md` — multi-agent orchestration (Axis 6) mirrors that file's service-boundary/specialization reasoning applied to agents instead of services
- `02-data-storage.md` — a vector store for RAG (Axis 4) can often piggyback on the storage model already chosen there (e.g. pgvector); an external scratchpad (Axis 8) may also live in whichever store was chosen there
- `04-file-storage.md` — a file-based scratchpad (Axis 8) for large agent working data ties into general file/blob storage decisions there
- `20-environment-strategy.md` — externalized prompt/system-prompt management (Axis 8 note) mirrors that file's feature-flag/config-management reasoning
- `05-scalability.md` — per-user rate limiting for cost control (Axis 5) reuses the general rate-limiting patterns from that file
- `11-background-jobs.md` — async AI call handling (Axis 3) fits the same queuing patterns covered there
- `13-caching.md` — caching repeated/near-duplicate AI queries is a direct cost-control lever, covered generally there
- `16-security.md` — API key custody for hosted providers, data-residency reasoning behind self-hosting (Axis 1), and liability/audit implications of automation (Axis 7) all fall under general security policy there
