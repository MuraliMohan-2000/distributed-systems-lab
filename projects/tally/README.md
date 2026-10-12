# Tally: Distributed Counter

> Imagine YouTube likes = 10,000,000. Millions of users can increment it simultaneously. A simple `counter = counter + 1` won't be enough.

**Roadmap stages:** 2 Multiple Servers, 4 Partitioning (plus CRDTs from 3 Replication)
**Real-world references:** YouTube view/like counts, Twitter/X engagement counters, Redis `INCR`, Cassandra counters, Facebook's sharded counters
**Status:** Not started

## Learning goals

- Atomic operations
- Race conditions and lost updates
- Sharding a single logical counter
- Hot keys
- Write aggregation (batching)
- Eventual consistency and CRDT counters

## Question to answer

How do you accept millions of increments per second on a single counter, and how stale is the number users see?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `Increment(counterId, delta, idempotencyKey)` | Add to a counter (likes can be undone with negative delta) |
| `Get(counterId, consistency)` | `exact` (slow) or `approximate` (fast, slightly stale) |
| `GetMany(counterIds)` | Batch read for feeds |

## Architecture

```
 Clients ──► Ingest servers (stateless, local aggregation buffers)
                │  flush every 100 ms / 1,000 increments
                ▼
        Counter store: counterId → K sub-counters (shards)
            likes:video42:#0  likes:video42:#1 ... likes:video42:#K-1
                │
                ▼
        Read path: sum of shards, cached with short TTL
```

## Milestones

### Phase 1: Single node and races (Stage 1)
- [ ] Naive read-modify-write with many concurrent goroutines: demonstrate lost updates
- [ ] Fix with mutex, then with atomics; benchmark both

### Phase 2: Shared store (Stage 2)
- [ ] Multiple servers using Redis `INCRBY`; measure single-key throughput ceiling
- [ ] Same thing on PostgreSQL with `UPDATE ... SET n = n + 1`: observe row-lock contention
- [ ] Idempotency keys so client retries do not double count

### Phase 3: Sharding and aggregation (Stage 4)
- [ ] Split one counter into K sub-counters; random shard on write, sum on read
- [ ] Local write aggregation on ingest servers with periodic flush
- [ ] Hot-key detection that increases K automatically for hot counters
- [ ] Read cache with short TTL; measure staleness

### Phase 4: CRDT counter (Stage 3)
- [ ] PN-counter replicated across 3 nodes, each node increments its own slot
- [ ] Merge on gossip; verify convergence after partitions
- [ ] Compare with the sharded approach (accuracy, latency, complexity)

## Failure scenarios to test

- [ ] Ingest server crashes with unflushed buffer: quantify lost increments; add a WAL or accept and document
- [ ] Client retries on timeout: no double counts with idempotency keys
- [ ] Partition a CRDT replica, increment on both sides, heal: counts converge
- [ ] Single viral counter receives 90% of traffic

## Success criteria

- Sustain a target load (e.g., 100k increments/sec on a laptop cluster) with zero lost updates in exact mode
- Documented staleness bound for approximate reads
- Final count matches total increments sent in a full load test

## Tech stack

Go, Redis, PostgreSQL (for the contention experiment), Docker Compose, k6, Prometheus (increments/sec, flush lag, shard distribution).

## Planned folder structure

```
tally/
├── README.md
├── docs/            # experiments: lock vs atomic vs sharded vs CRDT
├── cmd/             # ingest server, reader, load generator config
├── internal/        # aggregation buffer, shard manager, crdt, store adapters
├── deploy/
└── tests/
```

## References

- DDIA chapter 5 (concurrent writes) and chapter 7 (lost updates)
- Google App Engine "sharding counters" article
- CRDT paper: Shapiro et al., "Conflict-free Replicated Data Types"
- Redis `INCR` docs
