# SPEC: Distributed Web Crawler Architecture

## 1) Document Metadata
- Version: `0.1`
- Status: `Draft`
- Audience: Beginner-friendly implementation guide
- Last Updated: `2026-02-27`

## 2) Goal
Build a distributed web crawler where the frontier queue is the core system.  
The crawler must:
- avoid crawling the same URL more than once (deduplication),
- follow `robots.txt` rules,
- enforce per-domain crawl rate and concurrency limits (politeness),
- scale horizontally without a central scheduling bottleneck.

## 3) Non-Goals (for initial release)
- Full text indexing/search engine features.
- JavaScript rendering (headless browser).
- Advanced anti-bot evasion.
- Cross-region active-active crawling.

## 4) Recommended Architecture (High Level)
Use a **domain-sharded frontier** with **stateless fetchers**.

Key idea:
- Route each URL to a deterministic owner shard using the URL domain (`eTLD+1`).
- The owner shard handles dedupe + robots + politeness for that domain.
- Fetchers only fetch; they do not own long-lived scheduling state.

This avoids a central scheduler bottleneck and keeps per-domain coordination correct.

## 5) Beginner Glossary
- `Frontier queue`: The scheduler that decides what URL to crawl next.
- `Sharding`: Splitting work/state across multiple machines.
- `Domain sharding`: URLs from the same domain are handled by one shard.
- `Consistent hashing`: Stable way to map a domain to a shard.
- `Dedupe`: Prevent duplicate crawling of same canonical URL.
- `Canonicalization`: Converting equivalent URLs to one standard form.
- `Politeness`: Request limits to avoid overloading a site.
- `Stateless worker`: Worker with no critical local state.

## 6) Core Components

### 6.1 Ingest API
Purpose: Accept seed URLs and crawl jobs.

Responsibilities:
- Validate incoming seed URLs.
- Canonicalize URL early.
- Compute owner shard from domain.
- Publish `URL_DISCOVERED` event to the owner shard queue/topic.

### 6.2 Frontier Shard Service (Most Important)
Purpose: Own scheduling state for a subset of domains.

Responsibilities:
- Exact URL dedupe check (`add_if_absent`).
- `robots.txt` fetch/cache and path permission checks.
- Per-domain token bucket and in-flight concurrency checks.
- Per-domain queue storage.
- Ready-domain scheduling via min-heap (by `next_allowed_at`).
- Dispatch crawl tasks to fetchers.
- Persist state snapshots + changelog for recovery.

### 6.3 Fetcher Workers
Purpose: Download assigned URLs.

Responsibilities:
- HTTP(S) fetch with timeouts.
- Respect request headers and user-agent policy.
- Publish `FETCH_COMPLETED` with status/body metadata.
- No domain-level scheduling logic here.

### 6.4 Parser Workers
Purpose: Parse fetched content and extract links.

Responsibilities:
- Parse HTML and extract links.
- Resolve relative links to absolute URLs.
- Filter unsupported schemes (`mailto:`, `javascript:`).
- Publish discovered URLs back to frontier (`URL_DISCOVERED`).

### 6.5 Storage and Messaging
- Queue/Log: durable event transport between services.
- KV/DB: dedupe set, domain state, robots cache metadata.
- Snapshot storage: periodic checkpoints of frontier state.

## 7) URL Lifecycle (End-to-End)
1. Seed URL arrives at `Ingest API`.
2. URL is canonicalized and assigned to owner shard.
3. Owner shard checks dedupe:
   - already seen -> drop,
   - new -> add to dedupe set + enqueue in domain queue.
4. Scheduler selects domains that are eligible (`next_allowed_at <= now` and tokens available).
5. URL is dispatched to fetcher.
6. Fetcher downloads and emits `FETCH_COMPLETED`.
7. Parser extracts outlinks and emits new `URL_DISCOVERED` events.
8. Repeat until crawl budget/time constraints are reached.

## 8) Sharding Design

### 8.1 Ownership Function
- Domain key = normalized `eTLD+1` (example: `news.example.com` -> `example.com`).
- Owner shard = `consistent_hash(domain_key)`.

### 8.2 Hot Domain Handling
- Keep strict domain caps:
  - max in-flight requests,
  - max requests/second,
  - max domain queue size.
- For extreme hotspots, use an override map:
  - `example.com -> dedicated shard`.
- Preserve one shared rate limiter per domain even when moving ownership.

### 8.3 Rebalancing
- Use virtual nodes to reduce skew.
- Move domains in batches during scale-out.
- Transfer snapshot + changelog before cutover.

## 9) Dedupe Model
- Primary: exact dedupe store keyed by URL fingerprint.
- Optional optimization: Bloom filter pre-check before exact lookup.

Important:
- Bloom filter false positives can skip new pages.
- Exact store remains source of truth.

## 10) Politeness and Robots

### 10.1 Politeness Rules
- Per-domain request rate limit (token bucket).
- Per-domain max concurrent requests.
- Adaptive backoff on `429/503` and timeouts.

### 10.2 Robots Rules
- Cache `robots.txt` with TTL and validators (ETag/Last-Modified if available).
- Enforce user-agent specific `Allow/Disallow`.
- Respect `Crawl-delay` when available and policy allows.

## 11) Failure Handling and Recovery
- Keep shard as logical partition with at least one replica.
- Persist changelog events for every scheduling mutation.
- Periodic snapshots for fast recovery.
- On primary failure:
  1. promote replica,
  2. replay changelog after snapshot point,
  3. update shard routing.

Expected behavior after failover:
- dedupe history preserved,
- domain rate-limit timers preserved,
- minimal duplicate leakage.

## 12) MVP Scope (First Working Version)
- Single region.
- 2-4 frontier shards.
- Exact dedupe only (no Bloom initially).
- Basic robots handling.
- Fixed politeness policy.
- Metrics + logs + health endpoints.

## 13) Success Criteria
- No duplicate crawl for canonical URL in normal operation.
- No per-domain rate-limit violations in test harness.
- Shard failure recovery without full frontier reset.
- Horizontal scale by adding at least one shard with controlled rebalance.

## 14) Risks
- Hot domain concentration.
- Incorrect canonicalization causing over-merge or under-merge.
- Dedupe storage growth over long crawls.
- Operational complexity of stateful shard failover.

## 15) Future Enhancements
- Bloom/Cuckoo filter tier for lower exact-store load.
- Priority scheduling by page value/freshness.
- Multi-region federation with periodic dedupe summary exchange.
- Content-level dedupe (near-duplicate page detection).
