# Crown: Leader Election System

> Have 5 servers. Only one should be the leader. If the leader crashes, another node must take over. Then deliberately kill the leader.

**Roadmap stages:** 5 Leader Election, 6 Consensus
**Real-world references:** etcd (Raft), Apache ZooKeeper (ZAB), HashiCorp Consul, Kafka KRaft controller
**Status:** Not started

## Learning goals

- Heartbeats and timeouts
- Failure detection and its false positives
- Leader election algorithms (bully, lease-based, Raft)
- Terms/epochs and majority voting
- Split-brain problems and how quorums prevent them

## Question to answer

When the leader is killed (or just paused, or partitioned away), how fast does the cluster elect a new one, and can two nodes ever believe they are leader in the same term?

## API (planned)

| Endpoint | Description |
|----------|-------------|
| `GET /leader` | Who this node believes the leader is, and the current term |
| `GET /status` | Role (follower/candidate/leader), term, last heartbeat, peers |
| `POST /work` | Accepted only by the leader; followers redirect |

Internal RPCs: `Heartbeat` / `AppendEntries`, `RequestVote`.

## Architecture

```
   ┌────────┐   heartbeats   ┌────────┐
   │ Node 1 │◄──────────────►│ Node 2 │
   │LEADER  │                │follower│
   └───┬────┘                └────────┘
       │  heartbeats to all followers
  ┌────┼──────────┬──────────┐
  ▼    ▼          ▼          ▼
Node 3  Node 4   Node 5    (majority = 3 of 5)
```

## Milestones

### Phase 1: Heartbeats + simple election
- [ ] 5 nodes with static peer config
- [ ] Leader sends periodic heartbeats; followers track last-seen time
- [ ] Bully algorithm (highest ID wins) or lease-based election
- [ ] Each node exposes its view of the leader

### Phase 2: Terms and quorum
- [ ] Monotonic term number persisted to disk
- [ ] A node votes at most once per term
- [ ] Leader requires majority votes; minority partitions cannot elect
- [ ] Nodes reject messages from stale terms; old leaders step down

### Phase 3: Raft election + log replication (Stage 6)
- [ ] Randomized election timeouts
- [ ] `RequestVote` with log up-to-date check
- [ ] `AppendEntries` log replication and commit index
- [ ] Apply committed entries to a simple state machine (reuse the store from Ember)
- [ ] Persistence of term, vote, log; snapshots and log compaction
- [ ] Linearizable reads via read index
- [ ] Single-server membership changes

## Failure scenarios to test

- [ ] Kill the leader; measure time to new leader (target: under 2× election timeout)
- [ ] Partition 2 | 3: only the 3-node side has a leader
- [ ] Partition the leader alone: it steps down; majority elects a new leader; heal and verify convergence
- [ ] `docker pause` the leader past the timeout, then resume: it must step down on seeing a higher term
- [ ] Network latency near heartbeat timeout: count unnecessary elections
- [ ] Restart all 5 nodes simultaneously: terms and votes survive

## Success criteria

- Never two leaders in the same term across 100+ randomized fault runs
- Raft phase: linearizable history under crashes, partitions, and message loss
- Graph of failover time vs election timeout settings

## Tech stack

Go, gRPC, Docker Compose (5 nodes), Toxiproxy, Prometheus (term, role, elections counter), a randomized fault-test runner.

## Planned folder structure

```
crown/
├── README.md
├── docs/            # election design, Raft notes, failure-test results
├── proto/
├── cmd/
├── internal/        # node, election, raft log, persistence, transport
├── deploy/
└── tests/           # randomized fault tests
```

## References

- Raft paper (extended version) and raft.github.io visualization
- MIT 6.5840 Raft lab
- DDIA chapters 8 and 9
- ZooKeeper paper (ZAB overview)
- Phi-accrual failure detector paper
