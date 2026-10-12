# Distributed Systems Study Plan

A step-by-step plan for every stage of the roadmap. It describes **what to study and what to do**, not code. Each stage ends with exit criteria; do not move on until they are met.

```
Single Server → Multiple Servers → Replication → Partitioning
      → Leader Election → Consensus → Failure Recovery → Distributed System
```

Suggested pace: about 2 to 3 weeks per stage at 8 to 10 hours/week, roughly 5 to 6 months total. The capstone (Stage 8) gets about 4 weeks.

---

## Stage 0: Setup (before anything else)

1. Choose one language and stick with it (default: Go).
2. Install the toolchain: language SDK, Docker + Docker Compose, protobuf compiler + gRPC plugins, k6 or ghz, Toxiproxy.
3. Create a shared "harness" plan used by every project:
   - A Docker Compose template that starts N nodes plus Prometheus and Grafana.
   - A Toxiproxy setup in front of each node for injecting latency and partitions.
   - A standard set of scripts (described in each README) to kill, pause, resume, and partition nodes.
4. Decide on conventions: structured JSON logging, a `/metrics` endpoint on every node, a `/health` endpoint, config via environment variables.
5. Set up a notes file per stage for reading notes and open questions.
6. Get the textbook: *Designing Data-Intensive Applications* (DDIA).

Exit criteria: you can start a 3-container cluster of a "hello" service, see its metrics in Grafana, and cut the network to one container using Toxiproxy.

---

## Stage 1: Single Server

**Objective:** Build a correct, durable, observable single-node service. Most distributed bugs are really single-node bugs (races, unsafe durability) amplified by the network.

### Study
1. DDIA chapter 1 (reliability, scalability, maintainability) and chapter 3 (storage and retrieval).
2. Concurrency primitives in your language: mutexes, read/write locks, atomics, channels; what a data race is.
3. Durability: how `write` vs `fsync` works, what the OS page cache does, what "acknowledged" should mean.
4. Write-ahead logging and snapshotting; how Redis AOF/RDB and SQLite WAL work at a high level.
5. Storage engines: append-only log + hash index (Bitcask), LSM trees, B-trees, compaction.
6. Latency percentiles and why averages lie; Little's Law.

### Build
1. **Ember (KV Store), Phase 1:** single-node `GET/SET/DELETE` over gRPC/HTTP with an in-memory map.
2. Add a WAL: every write is appended and fsynced before being acknowledged. Replay the WAL on startup.
3. Add snapshots + WAL truncation so restart time stays bounded.
4. Add TTLs and an eviction policy (LRU) to prepare for **Stash (Cache)**.
5. Build the single-node **Gatekeeper (Rate Limiter)** (token bucket and sliding window, in memory).
6. Build the single-node **Tally (Counter)** and observe races with a naive read-modify-write; fix with atomics.

### Break it
1. Kill the process with `SIGKILL` during a write-heavy load; verify that every acknowledged write survives.
2. Run the race detector / thread sanitizer under concurrent load.
3. Fill the disk; observe and handle the error instead of crashing silently.

### Measure
- Throughput and p50/p99 latency with and without fsync on every write. Try group commit (batching fsyncs) and record the difference.

### Exit criteria
- [ ] No acknowledged write is ever lost across 20 random `SIGKILL` restarts.
- [ ] Race detector is clean under concurrent load.
- [ ] Dashboard shows request rate, error rate, and latency percentiles.
- [ ] You can explain the durability vs latency trade-off of fsync with your own numbers.

---

## Stage 2: Multiple Servers

**Objective:** Experience partial failure, unreliable networks, and shared state for the first time.

### Study
1. DDIA chapter 8 (the trouble with distributed systems): unreliable networks, unreliable clocks, process pauses.
2. The eight fallacies of distributed computing.
3. Load balancing strategies; stateless vs stateful services; sticky sessions.
4. Timeouts, retries, exponential backoff with jitter, retry budgets, idempotency keys.
5. Clocks: wall vs monotonic, NTP, clock skew, Lamport clocks, vector clocks.
6. Redis as shared state: atomic commands, Lua scripts, `INCR`, `EXPIRE`, pipelining.

### Build
1. **Gatekeeper (Rate Limiter), distributed:** put 3+ API servers behind a load balancer (Nginx/Envoy). First show that per-server in-memory limits are wrong (3 servers × 100 = 300 allowed). Then move state to Redis with an atomic script. Then try a hybrid (local token buckets periodically synced) and compare accuracy vs latency.
2. **Tally (Counter), distributed:** many writers incrementing one counter. Compare: a single Redis key, sharded sub-counters summed on read, and local aggregation flushed in batches.
3. **Stash (Cache), Phase 1:** application → cache → database with cache-aside. Add TTLs and measure hit ratio.
4. Add request IDs and propagate them across services so a single request can be traced end to end.

### Break it
1. Add 500 ms latency between the API servers and Redis. What happens to p99, and do retries make it worse (retry storm)?
2. Kill Redis. Decide and implement a policy: fail open (allow traffic) or fail closed (reject). Document why.
3. Skew one server's clock by 30 seconds. Which algorithms (fixed window, sliding window) break?

### Exit criteria
- [ ] The rate limiter enforces a global limit within a measured error margin across 3+ servers.
- [ ] The counter handles 10k+ increments/second with no lost updates (verified by a final count).
- [ ] You have a written policy for every dependency outage (fail open vs fail closed).
- [ ] You can explain why a timeout does not tell you whether the request succeeded.

---

## Stage 3: Replication

**Objective:** Keep multiple copies of data and reason precisely about what clients observe.

### Study
1. DDIA chapter 5 (replication) in full.
2. Single-leader replication: synchronous vs asynchronous followers, replication logs (statement, WAL shipping, logical).
3. Replication-lag anomalies: read-your-writes, monotonic reads, consistent prefix reads.
4. Leaderless replication: quorums (N, W, R), sloppy quorums, hinted handoff, read repair, Merkle-tree anti-entropy.
5. Conflict resolution: last-write-wins and its data loss, version vectors, siblings, CRDTs (G-counter, PN-counter, OR-set).
6. Consistency models: linearizability vs serializability vs eventual consistency; the CAP theorem and PACELC.
7. Read the Dynamo paper.

### Build
1. **Ember (KV Store), Phase 2:** single-leader replication with 1 leader + 2 followers. The leader ships its WAL to followers. Support both async and sync acknowledgement modes.
2. Add follower reads and implement read-your-writes using a client-provided "last seen log position".
3. **Ember (KV Store), Phase 3 (alternative mode):** leaderless replication with configurable N/W/R, version vectors, and read repair.
4. **Stash (Cache), Phase 2:** replicate hot entries; implement write-through and explicit invalidation; measure stale-read windows.
5. **Tally (Counter):** replace sharded counters with a CRDT PN-counter replicated across nodes and compare.

### Break it
1. Kill the leader in async mode right after a write is acknowledged. Was the write lost? Measure how often.
2. Partition one follower; write on the leader; heal the partition; verify catch-up.
3. In leaderless mode, write concurrently to the same key from two clients during a partition; observe the conflict.
4. Run a linearizability checker (Porcupine) on recorded client histories for each mode.

### Exit criteria
- [ ] You can show, with test output, which modes are linearizable and which are not.
- [ ] Followers catch up automatically after a partition heals.
- [ ] You can explain W + R > N and give a concrete case where it still returns stale data (sloppy quorum, concurrent writes).

---

## Stage 4: Partitioning (Sharding)

**Objective:** Scale data and traffic beyond one machine and move data safely when the cluster changes.

### Study
1. DDIA chapter 6 (partitioning).
2. Range vs hash partitioning; the trade-off for range scans.
3. Consistent hashing with virtual nodes; rendezvous (highest random weight) hashing; jump consistent hash.
4. Rebalancing strategies: fixed number of partitions (Riak, Elasticsearch), dynamic splitting (HBase, CockroachDB ranges).
5. Request routing: client-side routing, routing tier/proxy, gossip membership (SWIM).
6. Hot keys and skew mitigation: key salting, local aggregation, read replicas for hot data.

### Build
1. **Ember (KV Store), Phase 4:** consistent-hash ring with virtual nodes; each key maps to a replica set of N nodes on the ring.
2. Add a router (or smart client) that caches the ring and is told when it changes.
3. Implement node join/leave with data migration that streams only the affected key ranges, while reads and writes continue.
4. **Tally (Counter):** shard counters by key and add hot-key detection that automatically splits a hot key into sub-counters.
5. **Silo (File Storage), Phase 1:** split files into fixed-size chunks and place chunks across nodes by hashing.

### Break it
1. Add a node during a write-heavy load; verify no key is lost and measure how much data moved (should be about 1/N).
2. Remove a node abruptly (crash, not graceful leave); verify replicas take over.
3. Send 50% of traffic to one key; observe the hot shard, then apply your mitigation and compare.

### Exit criteria
- [ ] Adding or removing a node moves roughly 1/N of the keys, measured.
- [ ] No acknowledged write is lost during rebalancing.
- [ ] A load imbalance graph (keys and requests per node) is in the project README.

---

## Stage 5: Leader Election

**Objective:** Ensure exactly one node acts as leader at a time, even when nodes crash, pause, or get partitioned.

### Study
1. DDIA chapter 8 (process pauses, "the truth is defined by the majority") and chapter 9 (sections on leader election and fencing).
2. Failure detection: heartbeats, timeouts, phi-accrual failure detector; the trade-off between detection speed and false positives.
3. Classic algorithms: bully, ring; lease-based election.
4. Terms/epochs and fencing tokens; why a lease alone is not safe under GC pauses.
5. Split brain and quorum-based safety.
6. Distributed lock designs: single Redis `SET NX PX`, Redlock, ZooKeeper ephemeral sequential nodes, etcd leases + revisions. Read Kleppmann's "How to do distributed locking" and antirez's reply.
7. Read the Chubby paper.

### Build
1. **Crown (Leader Election), Phase 1:** 5 nodes with heartbeats and a bully or lease-based election. Every node exposes who it thinks the leader is.
2. **Crown (Leader Election), Phase 2:** add terms (epochs); a node only accepts messages from the highest term it has seen; require majority votes.
3. **Warden (Distributed Lock), Phase 1:** `acquireLock(name, ttl)` / `releaseLock(name, token)` on a single Redis instance with TTL and owner token.
4. **Warden (Distributed Lock), Phase 2:** add monotonically increasing fencing tokens and a protected storage service that rejects stale tokens.
5. **Warden (Distributed Lock), Phase 3:** implement the same API on etcd (lease + revision) and compare guarantees with the Redis version.

### Break it
1. Kill the leader; measure time to elect a new one.
2. Partition the cluster 2/3; verify the minority side cannot elect a leader (no split brain).
3. Pause (`docker pause`) the lock holder longer than the TTL, let another client acquire the lock, resume the first: does the fencing token prevent a corrupt write?
4. Introduce flapping network latency near the heartbeat timeout; count false elections.

### Exit criteria
- [ ] Never more than one leader per term, verified across 100 randomized fault runs.
- [ ] The paused-lock-holder scenario is safely rejected by fencing tokens.
- [ ] You can explain why a lease without fencing is unsafe.

---

## Stage 6: Consensus

**Objective:** Get a group of nodes to agree on an ordered log, which is the foundation of etcd, ZooKeeper, Kafka KRaft, and CockroachDB.

### Study
1. DDIA chapter 9 (consistency and consensus) in full.
2. The Raft paper (extended version) and the Raft visualization at raft.github.io.
3. MIT 6.5840 lectures on Raft; do the Raft lab if possible.
4. Paxos Made Simple (conceptual understanding only).
5. FLP impossibility (the intuition: no deterministic consensus in a fully asynchronous system with one faulty process).
6. Linearizable reads in Raft: read index and leader leases.
7. Total-order broadcast and its equivalence to consensus.

### Build
1. **Crown (Leader Election), Phase 3:** replace the custom election with Raft leader election (randomized timeouts, RequestVote, terms).
2. Add Raft log replication (AppendEntries, commit index, apply to a state machine). Use the KV store as the state machine.
3. Add persistence of term, vote, and log; add snapshots and log compaction.
4. Add linearizable reads via read index.
5. Add cluster membership changes (single-server changes).
6. **Courier (Message Queue), Phase 3:** replicate each partition's log using your Raft (or study how Kafka's ISR differs and implement a simplified ISR).

### Break it
1. Randomly kill, restart, and partition nodes while clients write; check the history with a linearizability checker.
2. Drop and reorder messages between nodes (unreliable-network mode in your test harness).
3. Restart all nodes at once; verify all committed entries survive.

### Exit criteria
- [ ] Raft passes randomized fault tests (crashes, partitions, message loss) with a linearizable history in 100+ runs.
- [ ] Committed entries survive full-cluster restarts.
- [ ] You can explain why a leader can only commit entries from its own term by counting replicas.

---

## Stage 7: Failure Recovery

**Objective:** Make the system detect failures, recover automatically, and make the right delivery guarantee explicit.

### Study
1. DDIA chapter 11 (stream processing): message brokers, acknowledgements, redelivery, exactly-once semantics.
2. Delivery semantics: at-most-once, at-least-once, effectively-once (idempotent consumers + dedup + transactional outbox).
3. Retry patterns: backoff + jitter, retry budgets, dead-letter queues, poison-message handling.
4. Resilience patterns: circuit breakers, bulkheads, load shedding, graceful degradation.
5. Recovery mechanics: WAL replay, checkpoints, re-replication, repair after data loss.
6. Read the Kafka paper and the GFS paper; skim Google SRE book chapters on handling overload and cascading failures.
7. Chaos engineering principles; read several Jepsen analyses.

### Build
1. **Courier (Message Queue), Phases 1 to 2:** producer → queue → consumer with acknowledgements, visibility timeouts, retries, dead-letter queue, consumer groups, and per-partition ordering.
2. **Foreman (Job Scheduler):** a scheduler that assigns jobs to 20 workers with leases and heartbeats; reassigns jobs from dead workers; prevents duplicate execution with fencing tokens and idempotency keys.
3. **Silo (File Storage), Phases 2 to 3:** checksums on every chunk, background scrubbing, automatic re-replication when a chunk falls below its replication factor.
4. Add circuit breakers and load shedding to at least one project.

### Break it
1. Kill a consumer after processing but before acknowledging; observe redelivery; make the consumer idempotent.
2. Kill a worker mid-job; measure time to reassignment; verify the job runs effectively once.
3. Kill 2 of 3 storage nodes holding a chunk; verify the file is still readable and re-replicated to the target factor.
4. Corrupt a chunk on disk; verify the checksum catches it and it is repaired.
5. Overload the system 3× capacity; verify it sheds load instead of collapsing.

### Exit criteria
- [ ] Each project documents its delivery guarantee and has a test proving it.
- [ ] Mean time to recovery (detection + repair) is measured for each failure scenario.
- [ ] No silent data corruption: corrupted chunks are always detected.

---

## Stage 8: Distributed System (Capstone)

**Objective:** Combine every concept into one system that keeps serving correct data while nodes are killed at random.

### Study
1. DDIA chapter 7 (transactions): ACID, isolation levels, MVCC, snapshot isolation, write skew, serializability.
2. Two-phase commit and its blocking problem; how Spanner and CockroachDB combine 2PC with Raft.
3. Papers: Spanner, Percolator (transaction model), and the CockroachDB design docs.
4. Hybrid logical clocks (HLC) for timestamps.
5. Operations: rolling upgrades, backups and restore, capacity planning, runbooks.

### Build (see `projects/citadel/README.md` for full detail)
1. Reuse your Raft from Stage 6 as the replication layer; each range (partition) is a Raft group.
2. Range-based partitioning with automatic range splits and a metadata range describing where every range lives.
3. A router layer that routes requests to range leaders and retries on leadership changes.
4. MVCC storage with timestamps from a hybrid logical clock.
5. Multi-key transactions with snapshot isolation using 2PC across ranges with a transaction record.
6. Failure recovery: replica replacement, range rebalancing, recovering in-flight transactions.
7. A simple query interface (KV + range scan; optional tiny SQL subset).

### Break it
1. A chaos runner that randomly kills, pauses, partitions, and skews clocks of nodes for hours.
2. A bank-transfer workload (total balance must stay constant) and a linearizability check on single-key ops.
3. Rolling restart of every node under load with zero failed client requests (after retries).

### Exit criteria
- [ ] The bank invariant holds across a multi-hour chaos run.
- [ ] Losing any minority of nodes causes no data loss and only a brief unavailability for affected ranges.
- [ ] A final write-up: architecture, guarantees, benchmark results, and a list of known limitations.

---

## Project-to-stage schedule

| Order | Project | Phases to do now | Stage |
|-------|---------|------------------|-------|
| 1 | Ember (KV Store) | Phase 1 (single node + WAL) | 1 |
| 2 | Stash (Cache) | Phase 1 | 1 to 2 |
| 3 | Gatekeeper (Rate Limiter) | all | 1 to 2 |
| 4 | Tally (Counter) | Phases 1 to 2 | 2 |
| 5 | Ember (KV Store) | Phases 2 to 3 (replication) | 3 |
| 6 | Stash (Cache) | Phase 2 (replication, invalidation) | 3 |
| 7 | Ember (KV Store) | Phase 4 (consistent hashing) | 4 |
| 8 | Tally (Counter) | Phase 3 (hot keys, CRDT) | 4 |
| 9 | Crown (Leader Election) | Phases 1 to 2 | 5 |
| 10 | Warden (Distributed Lock) | all | 5 |
| 11 | Crown (Leader Election) | Phase 3 (Raft) | 6 |
| 12 | Courier (Message Queue) | all | 6 to 7 |
| 13 | Foreman (Job Scheduler) | all | 7 |
| 14 | Silo (File Storage) | all | 3, 4, 7 |
| 15 | Citadel (Mini Distributed DB) | all | 8 |

## Progress tracker

```
- [ ] Stage 0: Setup
- [ ] Stage 1: Single Server
- [ ] Stage 2: Multiple Servers
- [ ] Stage 3: Replication
- [ ] Stage 4: Partitioning
- [ ] Stage 5: Leader Election
- [ ] Stage 6: Consensus
- [ ] Stage 7: Failure Recovery
- [ ] Stage 8: Distributed System (Capstone)
```
