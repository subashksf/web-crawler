# Distributed Web Crawler Backlog

## Scope
This backlog is aligned to:
- `specs/SPEC_DISTRIBUTED_WEB_CRAWLER_ARCHITECTURE.md`
- `specs/SPEC_DATA_AND_EVENT_CONTRACTS.md`
- `RECOMMENDED_ARCHITECTURE.md`

## Status Legend
- `TODO`: not started
- `IN PROGRESS`: currently being built
- `DONE`: completed

## Work Items

| ID | Status | Work Item | Depends On | Acceptance Criteria |
|---|---|---|---|---|
| W-001 | DONE | Create architecture spec document | None | `SPEC_DISTRIBUTED_WEB_CRAWLER_ARCHITECTURE.md` exists with goals, components, lifecycle, failover |
| W-002 | DONE | Create data and event contract spec | W-001 | `SPEC_DATA_AND_EVENT_CONTRACTS.md` exists with API/events/entities/invariants |
| W-003 | DONE | Create standalone recommendation doc | W-001 | `RECOMMENDED_ARCHITECTURE.md` exists and explains recommended design |
| W-004 | DONE | Initialize repo structure (`common`, `services`, `tests`, `docs`) | W-001 | Directory structure committed and consistent with specs |
| W-005 | DONE | Add project tooling (`pyproject`, lint, format, test config) | W-004 | `ruff`/`pytest` run locally without errors |
| W-006 | IN PROGRESS | Add local runtime stack (`docker-compose`) for queue + DB + cache | W-004 | Services start with one command and pass health checks |
| W-007 | TODO | Implement shared config loader and environment validation | W-005 | App fails fast on missing critical env vars |
| W-008 | TODO | Implement shared event/type definitions | W-005 | Event payloads match spec and are importable by all services |
| W-009 | TODO | Implement URL canonicalizer module | W-008 | Canonicalization rules in spec are covered by unit tests |
| W-010 | TODO | Implement URL fingerprint helper | W-009 | Same canonical URL always yields same fingerprint |
| W-011 | TODO | Implement domain extraction (`eTLD+1`) helper | W-009 | Subdomains map to expected domain key |
| W-012 | TODO | Implement consistent hash ring for shard routing | W-011 | Same domain always maps to same shard in stable cluster |
| W-013 | TODO | Add shard override map for hot domains | W-012 | Configurable domain override takes precedence over hash ring |
| W-014 | TODO | Build ingest API `POST /v1/seeds` endpoint | W-008 | Accepts seeds, validates URLs, emits `URL_DISCOVERED` |
| W-015 | TODO | Build ingest API status endpoint | W-014 | Returns crawl-level queue/fetch metrics |
| W-016 | TODO | Implement frontier dedupe store (`add_if_absent`) | W-010 | Duplicate URLs are rejected deterministically |
| W-017 | TODO | Implement frontier per-domain queue storage | W-016 | URLs are grouped and retrievable by domain |
| W-018 | TODO | Implement robots fetch + cache module | W-011 | Robots rules cached with TTL and used for allow/deny checks |
| W-019 | TODO | Implement token-bucket politeness limiter | W-017 | Domain dispatch rate never exceeds configured limit |
| W-020 | TODO | Implement per-domain in-flight concurrency guard | W-017 | Concurrent fetches per domain capped at configured value |
| W-021 | TODO | Build scheduler ready-heap by `next_allowed_at` | W-019 | Scheduler only emits eligible domains/URLs |
| W-022 | TODO | Build dispatcher emitting `FETCH_REQUESTED` | W-021 | Valid tasks published to fetch queue/topic |
| W-023 | TODO | Build fetcher worker consumer loop | W-022 | Fetcher consumes tasks and handles cancellation/shutdown cleanly |
| W-024 | TODO | Implement resilient HTTP client (timeouts/retries/redirect policy) | W-023 | Retries only retryable failures and records status correctly |
| W-025 | TODO | Publish `FETCH_COMPLETED` from fetcher | W-023 | Completion event emitted for success and failure paths |
| W-026 | TODO | Build parser worker for HTML parsing | W-025 | Extracts links from HTML bodies only |
| W-027 | TODO | Implement link resolution + filtering | W-026 | Relative links resolved; unsupported schemes filtered |
| W-028 | TODO | Publish `OUTLINKS_EXTRACTED` and rediscovery events | W-027 | Outlinks re-enter discovery pipeline end-to-end |
| W-029 | TODO | Implement frontier handlers for fetch completion updates | W-025 | In-flight counters/backoff/timers update correctly |
| W-030 | TODO | Implement backoff policy for `429/503/timeouts` | W-029 | Domain `next_allowed_at` increases after throttling errors |
| W-031 | TODO | Add state snapshot writer for frontier shard | W-017 | Snapshot captures dedupe, queues, and domain state |
| W-032 | TODO | Add append-only changelog events for state mutation | W-031 | All scheduler mutations are replayable from log |
| W-033 | TODO | Implement shard recovery (snapshot + log replay) | W-032 | Restart restores prior state without frontier reset |
| W-034 | TODO | Add replica promotion and routing switch workflow | W-033 | Simulated shard failure recovers via promoted replica |
| W-035 | TODO | Add metrics + health endpoints to all services | W-014 | Metrics listed in spec are exported and scrapeable |
| W-036 | TODO | Add structured logging with correlation IDs | W-014 | Logs can trace one URL across services |
| W-037 | TODO | Add unit tests for canonicalization edge cases | W-009 | Test suite covers query, fragment, port, and path normalization |
| W-038 | TODO | Add scheduler correctness tests | W-021 | Tests prove politeness and concurrency invariants |
| W-039 | TODO | Add integration test for full crawl loop | W-028 | Seed -> fetch -> parse -> rediscover loop passes |
| W-040 | TODO | Add failover integration test | W-034 | System restores from snapshot/log after shard crash |
| W-041 | TODO | Add load test for hot domain scenarios | W-034 | Hot domain does not starve unrelated domains |
| W-042 | TODO | Document runbook and operational playbooks | W-035 | Runbook includes startup, scaling, failover, and recovery steps |
| W-043 | TODO | Security and abuse hardening (UA policy, blocklists, limits) | W-042 | Configurable safeguards prevent accidental aggressive crawling |
| W-044 | TODO | MVP release checklist and go-live review | W-039, W-040, W-042 | All MVP success criteria in spec are satisfied |

## Suggested Execution Order (Beginner-Friendly)
1. Foundation: `W-004` to `W-008`
2. URL and routing core: `W-009` to `W-013`
3. Frontier scheduler core: `W-016` to `W-022`
4. Fetch and parse pipeline: `W-023` to `W-030`
5. Recovery and failover: `W-031` to `W-034`
6. Observability and tests: `W-035` to `W-041`
7. Ops and release: `W-042` to `W-044`
