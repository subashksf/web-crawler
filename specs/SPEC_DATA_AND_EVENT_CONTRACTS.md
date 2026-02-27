# SPEC: Data Model and Event Contracts

## 1) Document Metadata
- Version: `0.1`
- Status: `Draft`
- Last Updated: `2026-02-27`
- Scope: Contracts needed for MVP crawler pipeline

## 2) Canonical URL Contract

Input: raw URL string  
Output: canonical URL string + URL fingerprint

Mandatory normalization rules:
1. Lowercase scheme and host.
2. Remove default ports (`80` for `http`, `443` for `https`).
3. Remove fragment (`#...`).
4. Resolve dot segments in path (`.` and `..`).
5. Sort query parameters by key then value.
6. Drop known tracking params (`utm_*`, `gclid`, `fbclid`).
7. Remove trailing slash only for non-root paths (policy-driven).

Non-goals for MVP:
- Site-specific custom canonicalization rules.
- JS-driven canonical tags.

## 3) Primary Entities

### 3.1 URLRecord
- `url_fingerprint` (PK)
- `canonical_url`
- `domain_key`
- `first_seen_at`
- `last_seen_at`
- `seen_count`

### 3.2 DomainState
- `domain_key` (PK)
- `owner_shard_id`
- `robots_etag`
- `robots_last_fetched_at`
- `robots_expires_at`
- `next_allowed_at`
- `tokens`
- `refill_rate_per_sec`
- `in_flight`
- `max_in_flight`
- `backoff_until`

### 3.3 FrontierItem
- `item_id` (PK)
- `domain_key`
- `canonical_url`
- `priority`
- `discovered_at`
- `status` (`queued`, `dispatched`, `done`, `failed`, `dropped`)

## 4) API Contract (Ingest Service)

### 4.1 `POST /v1/seeds`
Purpose: submit one or more seed URLs.

Request:
```json
{
  "crawl_id": "crawl-001",
  "urls": ["https://example.com", "https://example.com/docs"],
  "priority": 50
}
```

Response:
```json
{
  "crawl_id": "crawl-001",
  "accepted": 2,
  "rejected": 0
}
```

### 4.2 `GET /v1/crawls/{crawl_id}/status`
Purpose: get aggregate crawl progress.

Response:
```json
{
  "crawl_id": "crawl-001",
  "queued": 1234,
  "fetched": 875,
  "failed": 31,
  "last_updated_at": "2026-02-27T12:00:00Z"
}
```

## 5) Event Contracts

### 5.1 `URL_DISCOVERED`
Published by: ingest service and parser  
Consumed by: frontier shard owner

```json
{
  "event_id": "uuid",
  "event_type": "URL_DISCOVERED",
  "crawl_id": "crawl-001",
  "raw_url": "https://example.com/a/../b?utm_source=x",
  "source_url": "https://example.com",
  "discovered_at": "2026-02-27T12:00:00Z",
  "priority": 50
}
```

### 5.2 `FETCH_REQUESTED`
Published by: frontier shard  
Consumed by: fetcher

```json
{
  "event_id": "uuid",
  "event_type": "FETCH_REQUESTED",
  "crawl_id": "crawl-001",
  "canonical_url": "https://example.com/b",
  "domain_key": "example.com",
  "request_timeout_ms": 10000
}
```

### 5.3 `FETCH_COMPLETED`
Published by: fetcher  
Consumed by: parser and frontier shard

```json
{
  "event_id": "uuid",
  "event_type": "FETCH_COMPLETED",
  "crawl_id": "crawl-001",
  "canonical_url": "https://example.com/b",
  "status_code": 200,
  "content_type": "text/html",
  "fetched_at": "2026-02-27T12:00:02Z",
  "retryable": false
}
```

### 5.4 `OUTLINKS_EXTRACTED`
Published by: parser  
Consumed by: ingest/frontier

```json
{
  "event_id": "uuid",
  "event_type": "OUTLINKS_EXTRACTED",
  "crawl_id": "crawl-001",
  "source_canonical_url": "https://example.com/b",
  "outlinks": [
    "https://example.com/c",
    "https://docs.example.com/guide"
  ],
  "extracted_at": "2026-02-27T12:00:03Z"
}
```

## 6) Scheduler Invariants
1. Same canonical URL should be accepted only once by exact dedupe store.
2. A domain cannot exceed configured `max_in_flight`.
3. A domain cannot be scheduled before `next_allowed_at`.
4. Robots disallowed paths cannot be dispatched.
5. Ownership rule must be deterministic from `domain_key`.

## 7) Storage Contract (Logical Tables)

### 7.1 `seen_urls`
- key: `url_fingerprint`
- value: `canonical_url`, `domain_key`, timestamps

### 7.2 `domain_state`
- key: `domain_key`
- value: politeness + robots metadata + queue counters

### 7.3 `frontier_items`
- key: `item_id`
- indexes: `(domain_key, status)`, `(status, priority)`

### 7.4 `event_log`
- append-only changelog for recovery and replay
- index by `shard_id` + timestamp

## 8) Observability Contract (MVP Metrics)
- `crawler_urls_discovered_total`
- `crawler_urls_deduped_total`
- `crawler_fetch_success_total`
- `crawler_fetch_fail_total`
- `crawler_domain_throttled_total`
- `crawler_robots_disallowed_total`
- `crawler_frontier_queue_depth`
- `crawler_shard_lag_seconds`

## 9) Test Contract
- Unit tests for canonicalization edge cases.
- Property tests for deterministic shard routing.
- Scheduler tests for rate-limit and in-flight invariants.
- Integration test: ingest -> frontier -> fetch -> parser -> rediscover loop.
- Failover test: restore from snapshot + changelog replay.
