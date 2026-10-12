# Warden: Distributed Lock

> Imagine 10 servers processing the same job. Only one should execute it. Build `acquireLock()` and `releaseLock()`.

**Roadmap stages:** 5 Leader Election, 7 Failure Recovery
**Real-world references:** Google Chubby, etcd locks (`concurrency` package), ZooKeeper lock recipe, Redis `SET NX PX` and Redlock
**Status:** Not started

## Learning goals

- Lock expiration (TTL/leases)
- Handling lock-holder crashes
- Network delays and process pauses (GC, VM migration)
- Concurrent acquire requests and fairness
- Deadlocks and how leases avoid them
- Fencing tokens

## Question to answer

What happens if the lock holder pauses for longer than the lock TTL, another server acquires the lock, and then the first server wakes up and keeps writing?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `acquireLock(name, ttl, waitTimeout)` | Returns `(token, fencingToken)` or times out |
| `renewLock(name, token)` | Extends the lease (keep-alive) |
| `releaseLock(name, token)` | Releases only if `token` matches the owner |
| `getLockInfo(name)` | Current owner, expiry, fencing token |

Plus a **protected resource service** (a simple storage server) that accepts writes only with a fencing token greater than the last one it has seen.

## Architecture

```
 Worker 1 ─┐
 Worker 2 ─┼──► Lock Service ──► backend: Redis (v1) / etcd (v2) / own Raft (v3, from Crown)
   ...     │        │
 Worker 10─┘        └── issues fencing token N
                              │
 Worker (holder) ──write(token N)──► Protected Storage (rejects tokens < max seen)
```

## Milestones

### Phase 1: Single Redis lock
- [ ] `SET name owner NX PX ttl` acquire
- [ ] Atomic compare-and-delete release (Lua script) so a client cannot release someone else's lock
- [ ] Lease renewal (watchdog) while work is in progress
- [ ] Blocking acquire with retry + jitter, and a wait timeout

### Phase 2: Fencing tokens
- [ ] Monotonically increasing fencing token per lock
- [ ] Protected storage service that rejects stale tokens
- [ ] Demonstrate the GC-pause scenario failing without fencing and passing with it

### Phase 3: Consensus-backed lock
- [ ] Implement the same API on etcd (lease + revision as the fencing token)
- [ ] Optional: implement on top of your Raft from Crown
- [ ] Fair queueing (waiters ordered by revision, like the ZooKeeper recipe)

### Phase 4: Redlock analysis (stretch)
- [ ] Implement Redlock across 5 Redis instances
- [ ] Reproduce its weaknesses under clock jumps and pauses; document findings

## Failure scenarios to test

- [ ] Holder crashes: lock becomes available after TTL, no deadlock
- [ ] Holder paused beyond TTL (`docker pause`): stale write rejected by fencing
- [ ] 10 workers race to acquire 1,000 times: exactly one holder at a time (verified via protected storage log)
- [ ] Redis primary fails over to a replica that missed the lock write: two holders (document why)
- [ ] Clock jump on a Redis node (Redlock phase)
- [ ] Network delay between worker and lock service longer than TTL

## Success criteria

- Mutual exclusion verified by the protected storage log under all scenarios when using fencing + a consensus backend
- Written comparison: Redis single vs Redlock vs etcd (safety, latency, operational cost)

## Tech stack

Go, Redis, etcd, Docker Compose, Toxiproxy, Prometheus (acquire latency, contention, expired leases).

## Planned folder structure

```
warden/
├── README.md
├── docs/            # design, safety analysis, Redlock findings
├── cmd/             # lock service, worker simulator, protected storage
├── internal/        # redis backend, etcd backend, fencing, client lib
├── deploy/
└── tests/
```

## References

- Martin Kleppmann, "How to do distributed locking"
- Salvatore Sanfilippo (antirez), "Is Redlock safe?"
- Chubby paper (Google)
- etcd concurrency API docs; ZooKeeper lock recipe
- DDIA chapter 8 (fencing tokens)
