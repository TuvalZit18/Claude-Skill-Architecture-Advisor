# 04 — File / Media Storage

## What this category covers

Storing and serving **unstructured blob-type assets** — user uploads (images, video, documents, audio), generated files (PDFs, exports, thumbnails), and any binary content that isn't a row in the structured store.

Distinct from `02-data-storage.md`: that file covers the structured/transactional data model (relational, document, KV, etc.); this file covers the files that sit alongside it — typically referenced from the structured store by URL or key, not stored inside it.

Distinct from `13-caching.md`: caching is about avoiding repeated *computation or queries*; CDN delivery here is about avoiding repeated *origin fetches of static bytes*. They overlap at the edge-caching layer but solve different problems.

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

**Scope note (inbound from `03-backup-disaster-recovery.md`):** that file explicitly treats "the data" as the structured store and hands off file/media backup and durability concerns to this file — see Axis 5 below.

---

## Axis 1: Storage model

### Object storage (S3, GCS, Azure Blob, and equivalents)

- **Pros:** purpose-built for this — cheap at scale, effectively unlimited capacity, decoupled from compute so it survives instance/container churn; built-in versioning and lifecycle rules; the default for a reason
- **Cons:** not a filesystem — no directory locking, no partial-file edits, eventual consistency on some operations depending on provider; requires an SDK/API rather than plain file I/O

### Block storage / attached volumes (EBS, persistent disks, and equivalents)

- **Pros:** real filesystem semantics, lowest latency for local read/write, works with tools that expect a mounted path
- **Cons:** tied to a single instance or AZ — doesn't scale horizontally, awkward to share across multiple app servers, not built for distributing files to end users; wrong tool for anything client-facing

### Database BLOB storage

- **Pros:** transactional consistency with the owning row — file and metadata can be written/rolled back together; nothing extra to provision for small volumes
- **Cons:** bloats database size and backup time (see Axis 5); most engines explicitly discourage large blobs; no native CDN path — every read goes through the database and app tier instead of an edge cache

### Managed storage bundled with a backend-as-a-service (Supabase Storage, Firebase Storage, and equivalents)

- **Pros:** integrated with the platform's auth/row-level-security rules so access control (Axis 4) is close to free; minimal setup, consistent with a `02-data-storage.md` BaaS choice
- **Cons:** deeper vendor lock-in than raw object storage; less direct control over CDN configuration and backup/lifecycle posture, since it inherits whatever the platform provides

Most real projects land on plain object storage or the BaaS-bundled equivalent; block storage and database BLOBs are usually the wrong default and worth flagging as such rather than presenting as equally live options.

---

## Axis 2: CDN & delivery

Whether files are served directly from the storage origin or fronted by a content delivery network.

### Origin-only (no CDN)

- **Pros:** simplest — one less system to configure or invalidate
- **Cons:** every request hits the origin; higher latency for geographically distant users; origin bandwidth cost scales linearly with traffic

### CDN in front of storage

- **Pros:** edge caching drops latency for repeat reads and offloads bandwidth/cost from the origin; the standard setup for any publicly-served media at real traffic
- **Cons:** cache invalidation is a real operational concern (versioned/hashed filenames sidestep this more cleanly than cache-purge calls); private/signed content (Axis 4) needs deliberate cache-control handling so the CDN doesn't cache something it shouldn't

**Image/asset transformation at the edge:** worth a one-line note rather than a full option — some CDNs and dedicated services (Cloudinary, imgix, or a cloud provider's native image-resizing layer) can resize/transcode on the fly instead of pre-generating every variant at upload time. A real trade-off (on-the-fly compute cost and latency vs. storage cost for pre-generated variants) but narrow enough to flag rather than expand into its own axis.

---

## Axis 3: Upload handling

### Proxy through the app server

- **Pros:** full control at upload time — validation, virus/malware scanning, transformation, all before the file lands in storage
- **Cons:** server bears the full bandwidth and memory load of every upload; doesn't scale well for large files or high upload volume; the server becomes a bottleneck for something that doesn't need its compute

### Direct-to-storage via presigned URLs

- **Pros:** the client uploads straight to storage — the server only issues a short-lived signed URL, so bandwidth and memory load never touch the app tier; scales cleanly with upload volume
- **Cons:** less server-side control at the moment of upload — validation (file type, size, scanning) has to happen either via storage-side policy constraints or as a follow-up step after the file lands, not inline; more moving parts on the client side to implement correctly

### Chunked / resumable uploads

Not a third base option so much as a refinement layered onto either of the above — relevant once files are large enough or clients unreliable enough (mobile networks, large video) that a single-shot upload risks failing partway. Most object storage providers and upload libraries support this natively; worth a one-line note in the generated table rather than its own row unless the project's expected file sizes make it a real driver.

---

## Axis 4: Access control & privacy

### Public read

- **Pros:** simplest to serve, plays best with CDN caching since nothing needs per-request authorization
- **Cons:** no access control at all — anyone with the URL (or a guessed one) can read it; wrong default for anything not meant to be public

### Signed / time-limited URLs

- **Pros:** fine-grained, temporary access without making the object permanently public — the standard approach for private-but-shareable content (a user's uploaded document, a time-limited download link)
- **Cons:** added complexity to generate and refresh; needs deliberate cache-control tuning to get any CDN benefit without caching a URL that's meant to expire

### Private, served through the app

- **Pros:** every read goes through application-level authorization — the most control, easiest to audit
- **Cons:** no CDN caching benefit at all; the app server bears the bandwidth load of every file read, same trade-off as proxied uploads in Axis 3 but on the read path instead

The general policy question of *who should be allowed to access what* is `16-security.md`'s domain; this axis is about the storage-layer mechanism that enforces whatever that policy decides.

---

## Axis 5: Durability, versioning & backup

**Backup/durability posture:** this is the file/media counterpart to `03-backup-disaster-recovery.md`'s Axes 1 and 4 — but typically far less of a build-it-yourself decision, since object storage providers usually bake in high durability (e.g. S3's eleven-nines durability claim) and cross-region replication as a configurable feature rather than a pipeline to construct. For most projects this axis collapses to "which replication/versioning tier," not "design a backup system."

- **Versioning:** keep prior versions of an object on overwrite/delete vs. allow overwrite in place. Protects against accidental deletion or a bad re-upload; costs additional storage for every version retained.
- **Lifecycle policies:** automatic transition to cheaper cold-storage tiers after an age threshold, and/or automatic deletion after a retention window — relevant for compliance-driven retention limits or simply controlling storage cost on high-volume uploads (e.g. temporary export files).
- **Cross-region replication:** same trade-off as `03`'s Axis 4 — protects against a regional outage, adds cost and (usually negligible) replication lag.

---

## What drives the recommendation (from upfront questions)

- **Platform**: mobile clients with unreliable networks weight toward direct-to-storage uploads with chunked/resumable support (Axis 3) over a simple server-proxy upload
- **Expected scale**: high upload/read volume pushes toward object storage plus CDN (Axes 1–2) as a default rather than an optimization; low-volume internal tools can reasonably skip the CDN
- **Ops/deployment capacity**: low ops maturity weights toward the managed BaaS storage option (Axis 1) and provider-default durability tiers (Axis 5) over hand-building presigned-upload flows and lifecycle rules
- **Time-to-ship pressure**: high pressure → object storage with public or simple signed-URL access, no CDN, provider-default durability, is a legitimate answer, not a gap
- **Domain/compliance context**: data-residency or retention-window requirements would push Axis 5's lifecycle/replication choices — but per the skill's excluded-categories note, don't assume this; ask if it's relevant to the specific project rather than forcing it

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Option D | Recommendation + why | Approve |
|---|---|---|---|---|---|---|
| Storage model | *Object storage pros/cons* | *Block storage pros/cons* | *DB BLOB pros/cons* | *Managed BaaS storage pros/cons* | *pick one, justify against `02`'s storage decision (if BaaS, bundled storage is often the path of least resistance) and ops capacity* | ☐ |
| CDN & delivery | *No CDN pros/cons* | *CDN pros/cons* | — | — | *justify against expected scale and whether content is public-facing; note on-the-fly transformation if relevant* | ☐ |
| Upload handling | *Proxy-through-server pros/cons* | *Direct-to-storage (presigned) pros/cons* | — | — | *justify against expected upload volume/file size and platform; note chunked/resumable if large files or mobile* | ☐ |
| Access control | *Public read pros/cons* | *Signed URLs pros/cons* | *Private-via-app pros/cons* | — | *justify against privacy requirements; cross-reference `16-security.md` for the underlying policy* | ☐ |
| Durability & backup | *Versioning/lifecycle implications* | *Cross-region replication implications* | — | — | *state the durability tier and retention/lifecycle rules; note if this collapses to a provider default* | ☐ |

Note: for many projects — especially those already on a managed BaaS from `02-data-storage.md` with time-to-ship pressure — this can reasonably collapse to a single consolidated row ("use the platform's bundled storage with its default durability tier and a signed-URL access pattern") rather than five separate rows. Don't force five rows when the project's answers make most of them a non-decision.

---

## Related files

- `02-data-storage.md` — if that decision is a managed BaaS, its bundled storage is often the path of least resistance here too
- `03-backup-disaster-recovery.md` — covers backup/durability for the structured store; this file's Axis 5 is the file/media counterpart
- `13-caching.md` — general caching strategy; this file's CDN axis is specifically about static asset delivery
- `16-security.md` — general access-control policy; this file's Axis 4 is the storage-layer mechanism enforcing it
- `21-networking.md` — CDN/edge routing touches networking topology; this file covers the storage-and-delivery decision, not the network path itself
