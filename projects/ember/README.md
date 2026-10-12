# Ember: Distributed Key-Value Store

> Start with `GET key`, `SET key value`, `DELETE key`. Then distribute the data across multiple nodes.

**Roadmap stages:** 1 Single Server, 3 Replication, 4 Partitioning
**Real-world references:** Redis, Amazon DynamoDB, Apache Cassandra, Riak, etcd (as a KV store)
**Status:** Not started

## Learning goals

- Partitioning data across nodes
- Replication (leader/follower and leaderless quorums)
- Consistent hashing with virtual nodes
- Leader/follower architecture
- Handling node failures without losing acknowledged writes

## Question to answer

What happens to reads and writes when a node holding your data dies, and how do you guarantee no acknowledged write is lost?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `GET key` | Return value and version; optional consistency level (`ONE`, `QUORUM`, `ALL`) |
| `SET key value [ttl]` | Write a value; returns the new version |
| `DELETE key` | Remove a key (tombstone in replicated modes) |
| `SCAN prefix` (stretch) | Range/prefix scan |

Internal (node-to-node, gRPC): `Replicate`, `StreamRange` (for rebalancing), `Gossip/Membership`, `Health`.

## Architecture

```
            ┌──────────────┐
 Client ───►│ Router/Smart │──── ring lookup: key → [nodeA, nodeB, nodeC]
            │    Client    │
            └──────┬───────┘
         ┌─────────┼─────────┐
         ▼         ▼         ▼
     ┌───────┐ ┌───────┐ ┌───────┐
     │Node A │ │Node B │ │Node C │   each node: in-memory index + WAL + snapshots
     └───────┘ └───────┘ └───────┘
          ▲  replication (WAL shipping or quorum writes)  ▲
          └────────────────────────────────────────────────┘
```

## Milestones

### Phase 1: Single node (Stage 1)
- [ ] In-memory map with concurrent-safe access
- [ ] Write-ahead log with fsync before acknowledgement; replay on startup
- [ ] Snapshots + WAL truncation
- [ ] TTL support and background expiry
- [ ] Metrics: ops/sec, latency percentiles, WAL size

### Phase 2: Leader/follower replication (Stage 3)
- [ ] One leader, two followers; leader ships WAL entries
- [ ] Sync and async acknowledgement modes (configurable)
- [ ] Follower catch-up after disconnect (from log position or snapshot)
- [ ] Follower reads with read-your-writes via client log position
- [ ] Manual failover (automatic failover comes after Crown)

### Phase 3: Leaderless mode (Stage 3)
- [ ] Configurable N, W, R
- [ ] Version vectors for concurrent writes; return siblings on conflict
- [ ] Read repair
- [ ] Hinted handoff for temporarily unreachable replicas
- [ ] Merkle-tree anti-entropy (stretch)

### Phase 4: Partitioning (Stage 4)
- [ ] Consistent-hash ring with virtual nodes
- [ ] Replica set = next N distinct physical nodes on the ring
- [ ] Router/smart client with ring cache and change notifications
- [ ] Node join/leave with streaming migration of affected ranges only
- [ ] Gossip-based membership (SWIM-style) (stretch)

## Failure scenarios to test

- [ ] `SIGKILL` a single node during writes; all acknowledged writes survive restart
- [ ] Kill the leader in async mode right after an ack; measure lost writes
- [ ] Partition a follower, write, heal; follower catches up
- [ ] Concurrent writes to the same key from two partitions (leaderless); conflict detected
- [ ] Add a node under load; only ~1/N keys move, no write lost
- [ ] Crash a node permanently; replicas serve its keys

## Success criteria

- Linearizability checker passes for leader reads in sync mode
- Documented consistency guarantee per mode (sync, async, quorum)
- Benchmark table: throughput and p99 for 1, 3, 5 nodes

## Tech stack

Go, gRPC + Protobuf, Docker Compose (3 to 5 nodes), Toxiproxy, Prometheus + Grafana, Porcupine (linearizability checking).

## Planned folder structure

```
ember/
├── README.md
├── docs/            # design doc, consistency notes, benchmark results
├── proto/           # gRPC service definitions
├── cmd/             # node and client binaries
├── internal/        # storage, wal, replication, ring, router
├── deploy/          # docker-compose, toxiproxy config, grafana dashboards
└── tests/           # integration + fault-injection tests
```

## References

- DDIA chapters 3, 5, 6
- Amazon Dynamo paper (2007)
- Bitcask paper (log-structured hash table)
- Cassandra architecture docs (ring, hinted handoff, read repair)
