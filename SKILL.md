---
name: distributed-systems-roadmap
description: Guides the study of distributed systems through an 8-stage roadmap (Single Server, Multiple Servers, Replication, Partitioning, Leader Election, Consensus, Failure Recovery, Distributed System) paired with 10 industry-style build projects (KV store, leader election, distributed lock, message queue, rate limiter, counter, cache, job scheduler, file storage, mini distributed database). Use when the user is studying distributed systems, working on any project in this folder, asks which concept or project comes next, or wants explanations, design reviews, or test plans for replication, partitioning, consensus, or failure handling.
disable-model-invocation: true
---

# Distributed Systems Roadmap

```
Single Server → Multiple Servers → Replication → Partitioning
      → Leader Election → Consensus → Failure Recovery → Distributed System
```

Principle: **don't just read about these concepts. Build them. Break them. Fix them.**

Every stage is learned through a project. Every project is finished only when it survives deliberately injected failures (killed processes, network partitions, slow disks, clock skew).

## Files in this workspace

- `PLAN.md`: the step-by-step study and build plan for every stage (no code).
- `projects/<project-name>/README.md`: the planned design, milestones, and test scenarios for each project.

Read `PLAN.md` before starting a stage, and read the project README before writing any code for that project.

---

## Stage 1: Single Server

**Goal:** Build a correct, observable, well-tested single-node service before distributing anything.

Concepts:
- Request/response lifecycle, TCP vs HTTP vs gRPC, serialization (JSON vs Protobuf)
- Concurrency inside one process: threads/goroutines, locks, race conditions, atomics
- Durability: fsync, write-ahead log (WAL), snapshots, crash recovery
- Storage engines: hash index, LSM tree vs B-tree, compaction
- Latency vs throughput, p50/p99, back-pressure, timeouts
- Observability basics: structured logs, metrics, traces

Key questions:
- What happens to an acknowledged write if the process crashes one millisecond later?
- Where is the bottleneck: CPU, memory, disk, or network?

Projects: single-node phase of **Ember (KV Store)**, **Gatekeeper (Rate Limiter)**, **Tally (Counter)**, **Stash (Cache)**.

## Stage 2: Multiple Servers

**Goal:** Understand what breaks the moment there is more than one machine.

Concepts:
- Fallacies of distributed computing (the network is not reliable, latency is not zero, ...)
- Stateless vs stateful services, load balancing (round robin, least connections, consistent hashing)
- Shared state: an external store (Redis) vs coordination between nodes
- Partial failure, timeouts, retries with exponential backoff and jitter, idempotency keys
- Time: wall clocks vs monotonic clocks, clock skew, Lamport clocks, vector clocks
- Service discovery and health checks

Key questions:
- If a request times out, did it succeed, fail, or is it still running?
- How do 10 servers agree on a shared limit without a bottleneck?

Projects: **Gatekeeper (Rate Limiter)**, **Tally (Counter)**, **Stash (Cache)**.

## Stage 3: Replication

**Goal:** Keep copies of data on multiple nodes and reason about what clients see.

Concepts:
- Leader/follower (single-leader), multi-leader, leaderless (Dynamo-style)
- Synchronous vs asynchronous replication, replication lag
- Consistency models: linearizable, sequential, causal, read-your-writes, monotonic reads, eventual
- Quorums: N, W, R, with W + R > N; sloppy quorums, hinted handoff
- Conflict resolution: last-write-wins, version vectors, CRDTs
- Anti-entropy: read repair, Merkle trees

Key questions:
- A follower is 5 seconds behind. Which reads are allowed to go to it?
- Two clients write the same key on two replicas at once. Which write wins, and is it a lost update?

Projects: **Ember (KV Store)**, **Silo (File Storage)**, **Stash (Cache)**.

## Stage 4: Partitioning (Sharding)

**Goal:** Split data so the system scales beyond one machine.

Concepts:
- Range vs hash partitioning, secondary indexes (local vs global)
- Consistent hashing, virtual nodes, rendezvous hashing
- Hot keys and skew, write aggregation, key salting
- Rebalancing: fixed partitions, dynamic splitting, minimal data movement
- Request routing: client-side, proxy/router, gossip-based membership

Key questions:
- When a node joins, how much data moves, and can reads continue during the move?
- One key receives 50% of traffic. What now?

Projects: **Ember (KV Store)**, **Tally (Counter)**, **Silo (File Storage)**, **Citadel (Mini DB)**.

## Stage 5: Leader Election

**Goal:** Make exactly one node responsible, and survive that node failing.

Concepts:
- Heartbeats, timeouts, failure detectors (phi-accrual), false positives
- Bully algorithm, ring algorithm, lease-based election
- Terms/epochs and fencing tokens
- Split brain and why a majority (quorum) prevents it
- Distributed locks: leases, TTLs, Redlock and its critique, ZooKeeper/etcd recipes

Key questions:
- The old leader was paused by GC for 30 seconds and wakes up still thinking it is the leader. What stops it from corrupting data?
- Why does a 5-node cluster tolerate 2 failures but a 4-node cluster also only tolerates 1?

Projects: **Crown (Leader Election)**, **Warden (Distributed Lock)**.

## Stage 6: Consensus

**Goal:** Get nodes to agree on an ordered log of decisions despite failures.

Concepts:
- The problem: agreement, validity, termination; FLP impossibility
- Raft: leader election, log replication, commit index, safety, membership changes, snapshots
- Paxos (single-decree and Multi-Paxos) at a conceptual level
- Replicated state machines, linearizable reads (read index, leases)
- Atomic broadcast and total order; how Kafka, etcd, ZooKeeper use these ideas

Key questions:
- Why can a log entry be on a majority of nodes and still not be committed?
- What does a client do when it is unsure whether its command was applied?

Projects: **Crown (Leader Election)** (upgrade to Raft), **Courier (Message Queue)** (replicated log), **Citadel (Mini DB)**.

## Stage 7: Failure Recovery

**Goal:** Detect, survive, and repair failures automatically.

Concepts:
- Failure models: crash-stop, crash-recovery, omission, Byzantine (awareness only)
- Delivery semantics: at-most-once, at-least-once, effectively-once (idempotency + dedup)
- Acknowledgements, retries, dead-letter queues, poison messages
- Checkpointing, WAL replay, snapshots, re-replication of lost data
- Circuit breakers, bulkheads, graceful degradation, load shedding
- Chaos engineering and Jepsen-style correctness testing

Key questions:
- A worker crashes halfway through a job. How does the system know, and how does it avoid running it twice?
- Two of three replicas holding a chunk die. How long until the data is safe again?

Projects: **Courier (Message Queue)**, **Foreman (Job Scheduler)**, **Silo (File Storage)**.

## Stage 8: Distributed System (Capstone)

**Goal:** Combine everything into one system that keeps serving correct data while nodes are killed at random.

Concepts combined: partitioning, replication, leader election, WAL, transactions (2PC, MVCC, snapshot isolation), consensus, failure recovery, plus operations (rolling upgrades, backups, observability).

Project: **Citadel (Mini Distributed Database)** (Client → Router → Nodes).

---

## Project map

| Folder | Project | Main stages | Real-world references |
|--------|---------|-------------|-----------------------|
| `ember` | Distributed Key-Value Store | 1, 3, 4 | Redis, DynamoDB, Cassandra, Riak |
| `crown` | Leader Election System | 5, 6 | etcd, ZooKeeper, Consul |
| `warden` | Distributed Lock | 5, 7 | Chubby, etcd locks, Redlock |
| `courier` | Message Queue | 6, 7 | Kafka, RabbitMQ, SQS |
| `gatekeeper` | Distributed Rate Limiter | 1, 2 | Envoy rate limit service, Stripe limiter |
| `tally` | Distributed Counter | 2, 4 | YouTube likes, Twitter counters, Redis INCR |
| `stash` | Distributed Cache | 1, 2, 3 | Memcached, Redis Cluster, CDN edge caches |
| `foreman` | Distributed Job Scheduler | 5, 7 | Temporal, Airflow, Sidekiq, Kubernetes CronJob |
| `silo` | Distributed File Storage | 3, 4, 7 | GFS, HDFS, S3, Ceph |
| `citadel` | Mini Distributed Database | all | CockroachDB, TiKV, Spanner, YugabyteDB |

Recommended build order (follows the roadmap): Ember (single node) → Stash → Gatekeeper → Tally → Ember (replicated + partitioned) → Crown → Warden → Courier → Foreman → Silo → Citadel.

---

## Default stack

Pick one stack and keep it for every project so effort goes into concepts, not tooling.

- **Language:** Go (the language of etcd, Consul, CockroachDB, Kubernetes). Java/Kotlin or Rust are acceptable alternatives.
- **RPC:** gRPC + Protocol Buffers for node-to-node; HTTP/JSON for the public API where convenient.
- **Local cluster:** Docker Compose (3 to 5 nodes per project).
- **Fault injection:** Toxiproxy (latency, partitions), `docker kill`/`docker pause` (crashes, GC-style pauses), `tc netem` (packet loss).
- **Observability:** Prometheus + Grafana for metrics, OpenTelemetry for traces, structured JSON logs.
- **Testing:** unit tests, property-based tests, integration tests against the Compose cluster, and a linearizability checker (Porcupine, or Jepsen's Knossos model) for consistency claims.
- **Load testing:** k6 or ghz (gRPC).

## Workflow for every project

Use this checklist whenever starting or reviewing a project:

```
- [ ] 1. Read the stage section in PLAN.md and the project README
- [ ] 2. Write a one-page design doc (API, data model, guarantees, non-goals)
- [ ] 3. Build the single-node version with tests and metrics
- [ ] 4. Distribute it (replication / partitioning / coordination as required)
- [ ] 5. Write the failure scenarios as automated tests before fixing them
- [ ] 6. Break it: kill, pause, partition, slow down, skew clocks
- [ ] 7. Fix it, then re-run every failure scenario
- [ ] 8. Benchmark (throughput, p99 latency) and record results in the README
- [ ] 9. Write a short post-mortem: what broke, why, and what guarantee it now provides
```

## Guidance for the agent

When helping in this workspace:
- Identify which stage and project the user is on before answering; tie explanations to that project.
- Prefer explaining trade-offs (consistency vs availability vs latency) over giving a single answer.
- When reviewing a design, always ask: what happens on node crash, on network partition, on a slow node, and on a retried request?
- Push the user to write the failure test first, then the fix.
- Do not generate the full solution unprompted; the goal is learning. Give hints, designs, and review feedback unless the user asks for code.

## Core reading list

- *Designing Data-Intensive Applications*, Martin Kleppmann (primary textbook; chapters 5 to 9 map to stages 3 to 7)
- MIT 6.5840 (formerly 6.824) Distributed Systems: lectures and labs
- Papers: Lamport "Time, Clocks, and the Ordering of Events"; Dynamo; GFS; Chubby; Raft ("In Search of an Understandable Consensus Algorithm"); Kafka; Spanner; ZooKeeper
- Kleppmann, "How to do distributed locking" (the Redlock critique)
- Jepsen analyses (jepsen.io) for how real systems fail
