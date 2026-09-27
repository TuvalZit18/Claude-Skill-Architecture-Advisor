# 13 — Caching

## What this category covers

Where cached data lives (client, CDN/edge, server, database), what granularity gets cached (a full rendered page vs individual data objects/fragments), how cached data gets invalidated or refreshed, and the pattern used to keep a cache in sync with its source of truth. As noted in `04-file-storage.md`: caching is about avoiding repeated *computation or queries*, distinct from that file's CDN axis, which is specifically about avoiding repeated *origin fetches of static bytes*. This file covers the general case — caching query results, computed values, API responses, and session data — of which static-asset CDN delivery is one specific instance.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Cache layer / location

### Client-side (browser cache, local/session storage, in-memory app state)

- **Pros:** zero network round-trip for cached data — the fastest possible access, and reduces server load entirely for repeat requests from the same client; some client-caching approaches keep working even with intermittent connectivity
- **Cons:** only benefits that one client — no sharing across users; invalidation can't be forced remotely without a deliberate mechanism (versioned keys, an explicit invalidation signal), so a client can hold stale data with no way for the server to correct it directly; storage limits vary by browser/device; caching sensitive data client-side raises real access-control considerations (`16-security.md`)

### CDN / edge caching

- **Pros:** serves cached content from a location geographically close to the user, cutting latency; offloads traffic from origin servers entirely for cache hits; scales to very high read traffic without touching application servers at all
- **Cons:** best suited to content that's the same for many users (or cacheable per some well-defined key) — highly personalized, per-user content doesn't cache well at the edge without careful cache-key design; invalidation/purging across a distributed edge network adds real operational complexity, echoing `04-file-storage.md`'s CDN axis, which covers this specifically for static assets — this axis is the more general case, including caching dynamic API responses at the edge

### Server / application-level cache

- **In-process (in-memory within the app itself):**
  - **Pros:** the fastest possible server-side cache access, with no network hop to a separate service, and zero additional infrastructure to run
  - **Cons:** not shared across multiple app instances — each horizontally-scaled instance (`05-scalability.md`) has its own separate cache, leading to duplicated cache population and inconsistency between instances; lost entirely on instance restart or redeploy
- **Distributed (e.g. Redis, Memcached):**
  - **Pros:** shared across all app instances — a cache hit populated by one instance benefits every instance, which matters directly for a horizontally-scaled stateless app tier (`05-scalability.md`'s Axis 2); survives individual app instance restarts, persisting cached data across deploys
  - **Cons:** adds a network hop compared to in-process caching, and adds a piece of infrastructure to run and operate; becomes an external dependency in its own right, needing failure handling (`14-resilience.md`) if it becomes slow or unavailable

### Database-level caching

- **Pros:** often close to free — many databases offer built-in query result caching, or support materialized views (pre-computed, periodically refreshed query results) with minimal extra infrastructure; keeps caching logic close to the data rather than spread across the application
- **Cons:** query-level caching is often coarse, and can be invalidated by any write to the underlying tables, limiting its effectiveness for frequently-updated data; materialized views need an explicit refresh strategy (Axis 3) and add storage overhead for the precomputed result

**DNS caching:** worth a one-line note rather than a full option here — DNS resolution is itself cached at multiple levels (browser, OS resolver, recursive/ISP resolver), governed by the TTL set on DNS records. This is a real caching layer, but it's a networking-infrastructure concern rather than an application-caching-strategy one, and its actual treatment belongs with `21-networking.md`'s DNS coverage. Worth knowing that DNS TTL directly affects failover/rollout speed — a short TTL means faster propagation of an IP change (relevant to blue-green or failover strategies, `20-environment-strategy.md`) at the cost of more frequent DNS lookups, while a long TTL is the reverse trade-off. Flagged here only so it isn't mistaken for an oversight; the decision itself is made in `21-networking.md`.

---

## Axis 2: Cache granularity — full-page vs fragment/object caching

Distinct from Axis 1's question of *where* cached data lives: this axis is about *what unit* gets cached — an entire rendered response, or individual data objects and page fragments.

### Full-page caching

- **Pros:** the simplest and highest-leverage form of caching for content that's identical (or nearly identical) across requests — cache the entire rendered response once and serve it directly for every matching request with zero recomputation; pairs naturally with SSG (`12-frontend.md`) and CDN/edge caching (Axis 1), since a fully-cached page needs no origin computation at all on a cache hit
- **Cons:** an all-or-nothing granularity — a page with even one small personalized or frequently-changing element (a "logged in as X" header, a live comment count) either can't be fully page-cached at all, or requires deliberately carving out that dynamic piece (edge-side includes, client-side hydration of just the dynamic part) rather than caching the whole page as one unit; invalidating a full-page cache means recomputing the entire page even when only a small part of it actually changed

### Fragment / object caching

- **Pros:** caches at a finer granularity — individual data objects, API responses, or page fragments/components rather than an entire rendered page — letting a page with both static and dynamic parts cache the static pieces (a product description) while leaving the dynamic pieces (current stock count, personalized recommendations) uncached or cached separately under their own strategy; invalidation is more precise, since only the specific object that actually changed needs to be invalidated rather than an entire page
- **Cons:** more moving parts — the page has to be assembled from multiple cached (and uncached) pieces rather than served as a single cached unit, adding real composition complexity; doesn't capture full-page caching's maximum performance benefit for content that genuinely is uniform across requests

Most real applications need both, applied to different content: full-page caching (or SSG) for genuinely static or near-static pages, fragment/object caching for pages that mix static and dynamic content — this is more a "which granularity fits which piece of content" question than a single system-wide choice.

---

## Axis 3: Cache invalidation & refresh strategy

### Time-based expiration (TTL)

- **Pros:** the simplest strategy to implement — set an expiration time, and the entry naturally goes stale and gets refreshed on next access, with no explicit invalidation logic needed
- **Cons:** data can be stale for up to the full TTL window even after the underlying source has already changed — a real trade-off for anything sensitive to freshness; choosing the right TTL is a genuine tuning problem (too short gives little caching benefit, too long risks serving stale data)

### Event-driven / explicit invalidation (purge)

- **Pros:** the cache is invalidated exactly when the underlying data actually changes, rather than on a fixed schedule — gives fresher data than TTL without needing to guess a good expiration window; ties naturally into the event patterns already covered in `10-messaging.md` (an event like "product updated" can trigger invalidation of that product's cached data)
- **Cons:** requires deliberately wiring up invalidation logic wherever the underlying data can change — easy to miss a code path that mutates data without invalidating the corresponding cache entry, producing subtle stale-data bugs that are harder to track down than TTL's predictable staleness window

**Purge vs ban:** two common mechanisms for carrying out explicit invalidation. A **purge** immediately removes a specific cached object by its exact key. A **ban** (the term used by some reverse-proxy/CDN caches, e.g. Varnish) instead marks anything matching a pattern or rule as invalid — useful when the exact set of affected cache keys isn't easily known in advance (e.g. invalidate every cached response tagged with a given product ID, without having to enumerate every URL that happened to include that product's data). Ban trades precision for convenience: it can over-invalidate content that matches the pattern but wasn't actually affected, in exchange for not needing to track every specific key.

### Stale-while-revalidate (SWR)

- **Pros:** a refinement of TTL rather than a wholly separate strategy — when a cached entry's TTL has expired, the request that finds it stale is still served the stale content immediately, while a background refresh fetches updated data for subsequent requests, rather than making that one unlucky request wait on a synchronous refetch; meaningfully improves perceived latency and avoids the "thundering herd" problem of many simultaneous requests all hitting the source at once right after expiration; well-supported as a standard HTTP `Cache-Control: stale-while-revalidate` directive at the CDN/browser level, needing no custom invalidation code to get the basic behavior
- **Cons:** by design, at least the triggering request (and potentially others during the background refresh window) receives stale data — not appropriate for anything that can never tolerate serving outdated content even briefly; the caching layer (CDN, browser, or a caching library) has to actually support the pattern, not every cache implementation does

### Versioned / cache-busting keys (e.g. content-hashed filenames for static assets)

- **Pros:** sidesteps invalidation entirely for immutable content — since the cache key itself changes whenever the content changes, there's never a stale-vs-fresh ambiguity to resolve; works exceptionally well with CDN caching (`04-file-storage.md` already notes this as the cleaner alternative to cache-purge calls), since the CDN can cache a versioned asset indefinitely with no invalidation logic needed at all
- **Cons:** only works for content naturally chunked into discrete, replaceable units (files, bundles) — doesn't apply to something like "the current value of this database row," which has no natural notion of a new version becoming a new key

These aren't mutually exclusive — a real system commonly combines TTL (with stale-while-revalidate as a serving-time refinement of it) as a safety-net default, event-driven purge/ban for the specific data where freshness genuinely matters, and versioned keys for static assets, rather than picking one strategy to cover the whole system.

---

## Axis 4: Caching pattern — how the cache stays in sync with its source

### Cache-aside (lazy loading)

- **Pros:** the simplest and most common pattern — the application checks the cache first, falls back to the source on a miss, then populates the cache; the cache only ever holds data that's actually been requested, avoiding wasted space on unused data; works with almost any cache and data-store combination, since the application controls the logic directly
- **Cons:** the first request for any given piece of data is always a cache miss, paying the full source-lookup cost; a brief window exists where the cache can be out of sync with the source between a write and the next read repopulating it, unless combined with explicit invalidation (Axis 3)

### Write-through

- **Pros:** writes go to the cache and the source together, so the cache and source stay in sync at write time — no stale-data window the way cache-aside can have; reads are always fast, since the cache is always populated for anything that's ever been written
- **Cons:** adds latency to every write, since it must complete against both the cache and the source before being considered done; caches data that may never actually be read again, wasting cache space on entries that see no benefit

### Write-behind (write-back)

- **Pros:** the fastest possible write path from the caller's perspective, since the write only needs to hit the cache before returning; can batch multiple writes to the source together for efficiency
- **Cons:** a real risk of data loss if the cache fails before the asynchronous flush to the source completes — the source of truth is temporarily only the cache, a meaningfully different reliability posture than the other patterns; more complex to implement correctly

### Write-around

- **Pros:** writes go directly to the source, bypassing the cache entirely — avoids populating the cache with data that's written but may never be read again (bulk writes, logs, audit trails, data written once and rarely re-read), keeping the cache reserved for data that's actually been requested; a simpler write path than write-through, since the write never touches the cache at all
- **Cons:** a read immediately following a write is always a cache miss, since the write didn't populate the cache — a real latency cost for any read-after-write access pattern; needs to be paired with a read-side pattern (typically cache-aside) to actually get data into the cache at all, so it's rarely a complete strategy on its own

### Read-through

- **Pros:** simplifies application code, since the cache-population logic on a miss lives inside the caching layer itself rather than being repeated at every call site
- **Cons:** requires a caching layer or library that actually supports this pattern rather than the application handling population directly; less direct control over exactly when and how the source gets queried

Cache-aside is the most broadly applicable default and the easiest to reason about. Write-through and write-behind matter more for write-heavy workloads where read-after-write consistency or write latency specifically matters. Write-around is commonly paired with cache-aside as its read-side complement — write-around keeps writes out of the cache, cache-aside populates the cache lazily as data is actually read — a combination that suits write-heavy, rarely-re-read data (logging, audit trails, bulk imports) where populating the cache on every write would waste space on entries unlikely to be read again soon. Read-through is more of an implementation-convenience layered on top of whichever pattern above actually fits, rather than a wholly separate decision.

---

## What drives the recommendation (from upfront questions)

- **Expected scale / read volume**: high read volume relative to writes is the strongest general driver toward investing in caching at all (Axis 1) — low-traffic projects get comparatively little benefit and can reasonably defer this category
- **Read/write ratio**: read-heavy workloads benefit most from caching broadly; write-heavy workloads see less benefit from caching and more benefit from the database-scaling investment covered in `05-scalability.md`
- **Consistency requirements**: strict consistency needs push against aggressive caching or long TTLs (Axis 3) — anything that can't tolerate serving stale data should either skip caching for that specific data, avoid stale-while-revalidate, or lean on event-driven purge/ban instead of a TTL window
- **Content uniformity**: pages or responses that are genuinely identical across users/requests are a direct driver toward full-page caching (Axis 2) and toward Axis 1's CDN layer; pages mixing static and per-user/frequently-changing content push toward fragment/object caching instead
- **System-level shape / statelessness** (from `05-scalability.md`): a horizontally-scaled, stateless app tier needs a distributed cache, not an in-process one, for cache hits to actually benefit every instance rather than being duplicated per instance — a direct consequence of that file's Axis 2
- **Ops/deployment capacity**: low ops maturity weights toward in-process caching or a managed distributed-cache offering over self-hosting cache infrastructure, and toward simple TTL-based invalidation over building event-driven invalidation plumbing
- **Time-to-ship pressure**: high pressure → simple TTL-based caching at whichever single layer gives the most benefit for least effort (often CDN for public content, or a straightforward distributed cache for API responses) with cache-aside as the pattern, is a legitimate first version — deferring more sophisticated invalidation or write-pattern optimization until a real need is demonstrated

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Option D | Option E | Recommendation + why | Approve |
|---|---|---|---|---|---|---|---|
| Cache layer | *Client-side pros/cons* | *CDN/edge pros/cons* | *Server (in-process vs distributed) pros/cons* | *DB-level pros/cons* | — | *justify against read volume, personalization needs, and app-tier statelessness (`05-scalability.md`)* | ☐ |
| Cache granularity | *Full-page pros/cons* | *Fragment/object pros/cons* | — | — | — | *justify against content uniformity; most projects need both, applied to different content* | ☐ |
| Invalidation strategy | *TTL (+ SWR refinement) pros/cons* | *Event-driven purge/ban pros/cons* | *Versioned keys pros/cons* | — | — | *justify against consistency requirements; note these typically combine rather than being mutually exclusive* | ☐ |
| Caching pattern | *Cache-aside pros/cons* | *Write-through pros/cons* | *Write-behind pros/cons* | *Write-around pros/cons* | *Read-through pros/cons* | *cache-aside is the default; write-around commonly pairs with it; others justified by specific read-after-write or write-latency needs* | ☐ |

Note: for most projects under time pressure, this can reasonably collapse to a single consolidated row ("CDN or a distributed cache for the highest-read-volume data, full-page caching where content is uniform, TTL-based invalidation, cache-aside pattern") rather than four.

---

## Related files

- `04-file-storage.md` — that file's CDN axis is specifically about static asset delivery; this file covers the more general caching strategy, including dynamic content
- `05-scalability.md` — a horizontally-scaled, stateless app tier needs a distributed (not in-process) cache for benefits to be shared across instances
- `09-ai-integration.md` — caching repeated or near-duplicate AI queries is a direct cost-control application of this file's general strategy
- `10-messaging.md` — event-driven purge/ban (Axis 3) can be built on the same event infrastructure covered there
- `12-frontend.md` — client-side/CDN caching layer and full-page/SSG caching granularity for frontend assets are applications of this file's Axes 1-2
- `14-resilience.md` — a distributed cache is itself an external dependency that needs failure handling if it becomes slow or unavailable
- `16-security.md` — sensitive data cached client-side or in a shared cache raises access-control considerations covered there
- `20-environment-strategy.md` — DNS TTL (noted under Axis 1) affects failover/rollout propagation speed, decided there
- `21-networking.md` — DNS caching itself is a networking-infrastructure concern; its actual treatment lives there, not in this file
