# Stash: Distributed Cache

> Build: Application → Cache → Database. Then shut down your cache. Does your application survive?

**Roadmap stages:** 1 Single Server, 2 Multiple Servers, 3 Replication
**Real-world references:** Memcached (Facebook's mcrouter), Redis Cluster, Hazelcast, CDN edge caches, Netflix EVCache
**Status:** Not started

## Learning goals

- Cache-aside, read-through, write-through, write-behind
- TTL and eviction (LRU, LFU)
- Cache stampede (thundering herd)
- Cache penetration (requests for keys that do not exist)
- Cache invalidation and stale reads
- Distributing cache nodes with consistent hashing

## Question to answer

When the cache disappears, does the database survive the full load, and how do you keep cached data from staying wrong after an update?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `Get(key)` | Returns value or miss |
| `Set(key, value, ttl)` | Store with expiry |
| `Delete(key)` | Invalidate |
| `GetOrLoad(key, loader)` | Client-library helper with request coalescing |

Demo application: a product-catalog service backed by PostgreSQL.

## Architecture

```
  App servers (with cache client library: ring + coalescing + local L1)
        │
        ├──► Cache node A ┐
        ├──► Cache node B ├─ consistent-hash ring
        ├──► Cache node C ┘
        │
        └──► PostgreSQL (source of truth) ──► change events (CDC) ──► invalidation
```

## Milestones

### Phase 1: Single cache node (Stage 1)
- [ ] In-memory cache server with LRU eviction and TTL
- [ ] Memory limit with eviction metrics
- [ ] Demo app with cache-aside; measure hit ratio and DB load reduction

### Phase 2: Distributed cache (Stage 2)
- [ ] 3+ cache nodes with consistent hashing in the client library
- [ ] Adding/removing a node remaps only ~1/N keys
- [ ] Two-level caching: small in-process L1 + shared L2

### Phase 3: Hard problems
- [ ] Stampede protection: request coalescing (single-flight), probabilistic early expiration, TTL jitter
- [ ] Penetration protection: negative caching and a Bloom filter for nonexistent keys
- [ ] Write-through and write-behind modes; compare consistency
- [ ] Invalidation via delete-on-write and via CDC events from PostgreSQL (Debezium or logical replication)
- [ ] Lease-based gets to prevent stale sets (Facebook memcache paper)

### Phase 4: Replication (Stage 3)
- [ ] Replicate hot keys to multiple nodes
- [ ] Measure stale-read window after invalidation

## Failure scenarios to test

- [ ] Kill all cache nodes under load: DB protected by load shedding / circuit breaker; app degrades instead of crashing
- [ ] Kill one cache node: only its keys miss; no cascading failure
- [ ] Popular key expires under 10k req/s: stampede prevented
- [ ] Flood of requests for nonexistent IDs: DB not hammered
- [ ] Update in DB then concurrent read: race that writes stale data back into the cache; fix with leases

## Success criteria

- Hit ratio and DB QPS graphs with and without cache
- App survives total cache loss with a documented degraded mode
- Stale-read window measured and bounded

## Tech stack

Go, PostgreSQL, optional Redis for comparison, Docker Compose, k6, Toxiproxy, Prometheus (hit ratio, evictions, DB QPS).

## Planned folder structure

```
stash/
├── README.md
├── docs/            # patterns comparison, stampede experiments
├── cmd/             # cache node, demo app
├── internal/        # lru, ttl, ring, client lib, coalescing, bloom filter
├── deploy/
└── tests/
```

## References

- "Scaling Memcache at Facebook" (NSDI 2013)
- Redis Cluster specification
- AWS caching best practices (cache-aside, TTL jitter)
- "Optimal Probabilistic Cache Stampede Prevention" (XFetch)
