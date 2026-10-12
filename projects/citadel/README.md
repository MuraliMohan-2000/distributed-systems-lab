# Citadel: Mini Distributed Database (Final Boss)

> Combine partitioning, replication, leader election, WAL, transactions, consensus, and failure recovery. Build Client → Router → Nodes. Then start killing nodes randomly. Your goal: the system keeps working.

**Roadmap stages:** all, especially 8 Distributed System
**Real-world references:** CockroachDB, TiKV/TiDB, Google Spanner, YugabyteDB, FoundationDB
**Status:** Not started (begin after Ember, Crown, and Courier are complete)

## Learning goals

- Range partitioning with automatic splits and rebalancing
- Per-range replication using Raft (multi-Raft)
- Leader election and leaseholders
- Write-ahead logging and storage engine integration
- Transactions: MVCC, snapshot isolation, two-phase commit across ranges
- Consensus-based metadata
- Failure recovery under random node failures

## Question to answer

While nodes are killed at random, does every committed transaction stay committed, and does the total balance in a bank-transfer workload stay exactly constant?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `Get(key)` / `Put(key, value)` / `Delete(key)` | Single-key, linearizable |
| `Scan(start, end, limit)` | Range scan at a snapshot timestamp |
| `Begin()` / `Commit()` / `Rollback()` | Multi-key transactions (snapshot isolation) |
| Optional SQL subset | `CREATE TABLE`, `INSERT`, `SELECT ... WHERE pk BETWEEN` |

## Architecture

```
 Client ──► Router (stateless, caches range descriptors)
               │  key → range → leaseholder node
               ▼
 ┌───────── Node 1 ─────────┐ ┌───────── Node 2 ─────────┐ ┌───────── Node 3 ─────────┐
 │ Range [a-f) replica (L)  │ │ Range [a-f) replica      │ │ Range [a-f) replica      │
 │ Range [f-m) replica      │ │ Range [f-m) replica (L)  │ │ Range [f-m) replica      │
 │ Range [m-z) replica      │ │ Range [m-z) replica      │ │ Range [m-z) replica (L)  │
 │ MVCC store + WAL         │ │ MVCC store + WAL         │ │ MVCC store + WAL         │
 └──────────────────────────┘ └──────────────────────────┘ └──────────────────────────┘
   each range = one Raft group; meta range stores all range descriptors
```

## Milestones

### Phase 1: Single-range replicated KV
- [ ] Reuse Raft from Crown; one Raft group storing all keys
- [ ] Storage engine (own LSM from Ember, or embed Pebble/BadgerDB) with WAL
- [ ] Linearizable reads via leaseholder or read index

### Phase 2: Multi-range (multi-Raft)
- [ ] Range-based partitioning with a meta range holding range descriptors
- [ ] Router that caches descriptors and retries on `NotLeader`/`RangeMoved` errors
- [ ] Automatic range splits by size and by load
- [ ] Rebalancer that moves replicas to keep nodes balanced

### Phase 3: MVCC
- [ ] Versioned keys with timestamps from a hybrid logical clock (HLC)
- [ ] Snapshot reads at a timestamp
- [ ] Garbage collection of old versions

### Phase 4: Distributed transactions
- [ ] Transaction record + write intents (Percolator/CockroachDB style)
- [ ] Two-phase commit across ranges
- [ ] Conflict detection: write-write conflicts, read timestamp pushes
- [ ] Recovery of abandoned transactions (coordinator crashed)
- [ ] Deadlock detection or wound-wait

### Phase 5: Operations and failure recovery
- [ ] Replace dead replicas automatically (up-replicate on another node)
- [ ] Rolling restarts with zero failed client requests (after retries)
- [ ] Backup and restore of a consistent snapshot
- [ ] Dashboards: ranges per node, leaseholders, Raft proposals, txn retries

## Failure scenarios to test

- [ ] Chaos runner: random kill, pause, partition, clock skew for multiple hours
- [ ] Bank workload: random transfers between 100 accounts; total must never change
- [ ] Single-key linearizability check with Porcupine
- [ ] Kill the coordinator mid-commit: transaction resolves to committed or aborted, never half-applied
- [ ] Kill a minority of nodes for every range: no data loss, brief unavailability only
- [ ] Partition the leaseholder: new leaseholder; old one cannot serve stale reads

## Success criteria

- Bank invariant and linearizability hold across a multi-hour chaos run
- Documented isolation level with an example anomaly it prevents and one it allows (write skew under SI)
- Benchmark: YCSB-style workloads (A: 50/50 read/write, B: 95/5, E: scans) on 3 and 5 nodes
- Final write-up: architecture, guarantees, limitations, what you would do differently

## Tech stack

Go, gRPC, own Raft (Crown) or etcd/raft library, Pebble or own LSM, Docker Compose (5 nodes), Toxiproxy, Porcupine, a custom chaos runner, Prometheus + Grafana, go-ycsb.

## Planned folder structure

```
citadel/
├── README.md
├── docs/            # architecture, txn protocol, chaos results, final write-up
├── proto/
├── cmd/             # node, router, client CLI, chaos runner
├── internal/        # raft, storage, mvcc, hlc, ranges, txn, router, rebalancer
├── deploy/
└── tests/           # bank workload, linearizability, chaos
```

## References

- DDIA chapters 7 and 9
- Spanner paper (Google, 2012)
- Percolator paper (Google, 2010)
- CockroachDB design docs and "CockroachDB: The Resilient Geo-Distributed SQL Database" (SIGMOD 2020)
- TiKV deep-dive docs (multi-Raft, Percolator transactions)
- Jepsen analyses of CockroachDB, TiDB, YugabyteDB
