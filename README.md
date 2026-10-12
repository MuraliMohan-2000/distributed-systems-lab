# Distributed Systems Lab

Learning distributed systems by building them: 10 industry-style projects that follow one roadmap from a single server to a fault-tolerant distributed database.

> Don't just read about these concepts. Build them. Break them. Fix them.

## Roadmap

```
Single Server → Multiple Servers → Replication → Partitioning
      → Leader Election → Consensus → Failure Recovery → Distributed System
```

- [`PLAN.md`](PLAN.md): step-by-step study and build plan for every stage, with exit criteria
- [`SKILL.md`](SKILL.md): the roadmap in detail (concepts, key questions, stack, per-project workflow)

## Projects

| Project | What it is | Roadmap stages | Inspired by |
|---------|------------|----------------|-------------|
| [Ember](projects/ember) | Distributed key-value store | 1, 3, 4 | Redis, DynamoDB, Cassandra |
| [Crown](projects/crown) | Leader election (up to Raft) | 5, 6 | etcd, ZooKeeper, Consul |
| [Warden](projects/warden) | Distributed lock with fencing tokens | 5, 7 | Chubby, etcd locks, Redlock |
| [Courier](projects/courier) | Replicated message queue | 6, 7 | Kafka, RabbitMQ, SQS |
| [Gatekeeper](projects/gatekeeper) | Distributed rate limiter | 1, 2 | Envoy, Stripe rate limiters |
| [Tally](projects/tally) | High-throughput distributed counter | 2, 4 | YouTube likes, Redis INCR |
| [Stash](projects/stash) | Distributed cache | 1, 2, 3 | Memcached, Redis Cluster |
| [Foreman](projects/foreman) | Distributed job scheduler | 5, 7 | Temporal, Airflow, Sidekiq |
| [Silo](projects/silo) | Distributed file storage | 3, 4, 7 | GFS, HDFS, S3 |
| [Citadel](projects/citadel) | Mini distributed database (capstone) | all | CockroachDB, TiKV, Spanner |

Build order: Ember (single node) → Stash → Gatekeeper → Tally → Ember (replicated + partitioned) → Crown → Warden → Courier → Foreman → Silo → Citadel.

## Stack

Go, gRPC + Protobuf, Docker Compose, Toxiproxy for fault injection, Prometheus + Grafana, Porcupine for linearizability checks.

## Progress

- [ ] Stage 1: Single Server
- [ ] Stage 2: Multiple Servers
- [ ] Stage 3: Replication
- [ ] Stage 4: Partitioning
- [ ] Stage 5: Leader Election
- [ ] Stage 6: Consensus
- [ ] Stage 7: Failure Recovery
- [ ] Stage 8: Distributed System (Capstone)

## Credit

Project list inspired by [this thread by @MansiCodez](https://x.com/MansiCodez/status/2109290926548115494).
