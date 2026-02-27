# Recommended Architecture for This Project

## Recommendation
Use a **domain-sharded frontier** with **stateless fetcher workers**.

In simple terms:
1. Normalize URL.
2. Map URL domain to one owner shard.
3. Owner shard does dedupe + robots checks + rate limiting.
4. Owner shard dispatches fetch tasks to stateless fetchers.
5. Parser extracts links and sends them back into the same pipeline.

## Why This Is Recommended
1. Correctness is easier because one shard coordinates each domain.
2. Deduplication is simpler because ownership is deterministic.
3. Politeness is safer because one shard enforces per-domain limits.
4. It scales by adding shards instead of scaling one central scheduler.

## Important Terms (Beginner-Friendly)
- `Sharding`: splitting work across multiple machines.
- `Domain sharding`: grouping by domain so each domain has one owner.
- `Politeness`: not sending requests too fast to the same site.
- `Dedupe`: making sure the same URL is not crawled repeatedly.
- `robots.txt`: site rules that tell crawlers what paths are allowed.

## Main Tradeoffs
1. Hot domains can overload a shard.
2. Failover is harder because shard state is important.
3. Rebalancing requires moving state when shard count changes.

## Mitigations
1. Per-domain rate and queue caps.
2. Replica + changelog + snapshot for failover.
3. Dedicated shard override for very hot domains.
