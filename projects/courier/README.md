# Courier: Your Own Message Queue

> Start simple: Producer → Queue → Consumer. Then add multiple consumers, ordering, acknowledgements, retries, dead-letter queues, and consumer failures.

**Roadmap stages:** 6 Consensus, 7 Failure Recovery
**Real-world references:** Apache Kafka, RabbitMQ, Amazon SQS, NATS JetStream, Apache Pulsar
**Status:** Not started

## Learning goals

- Multiple consumers and consumer groups
- Message ordering (global vs per-partition)
- Acknowledgements and visibility timeouts
- Retries with backoff and dead-letter queues
- Handling consumer and broker failures
- Durable, replicated append-only logs

## Question to answer

At-least-once vs at-most-once vs exactly-once delivery: which does your queue provide, how do you prove it, and what must the consumer do to get effectively-once processing?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `CreateTopic(name, partitions, replicationFactor)` | Create a topic |
| `Produce(topic, key, value, idempotencyKey)` | Append; key determines partition |
| `Consume(topic, group, maxMessages)` | Fetch messages for a consumer group |
| `Ack(topic, group, partition, offset)` | Commit progress |
| `Nack(...)` | Request redelivery (increments attempt count) |

## Architecture

```
 Producers ──► ┌──────────── Broker cluster ────────────┐ ──► Consumer group A (c1, c2, c3)
               │ Topic "orders"                         │
               │  P0: [0][1][2][3]...  leader B1, replicas B2,B3
               │  P1: [0][1][2]...     leader B2, replicas B3,B1
               │  P2: [0][1]...        leader B3, replicas B1,B2
               │ Offsets store per (group, partition)   │
               │ DLQ topic "orders.dlq"                 │
               └────────────────────────────────────────┘
```

## Milestones

### Phase 1: Single broker queue
- [ ] In-memory queue → durable append-only segment files with an offset index
- [ ] Produce / consume / ack
- [ ] Visibility timeout: unacked messages are redelivered
- [ ] Retry counter, exponential backoff, dead-letter queue after N attempts

### Phase 2: Partitions and consumer groups
- [ ] Topics with N partitions; key-hash partitioning gives per-key ordering
- [ ] Consumer groups with partition assignment and rebalancing on join/leave
- [ ] Committed offsets stored per group/partition
- [ ] Idempotent producer (dedup by producer ID + sequence number)
- [ ] Retention by time and size; segment deletion

### Phase 3: Replication (Stage 6)
- [ ] Each partition replicated to 3 brokers
- [ ] Option A: Raft per partition (reuse Crown)
- [ ] Option B: Kafka-style ISR (in-sync replicas) with a controller and high-water mark
- [ ] `acks=0/1/all` producer settings and their durability trade-offs
- [ ] Leader failover without losing committed messages

### Phase 4: Exactly-once processing (stretch)
- [ ] Transactional outbox pattern demo with a database
- [ ] Consumer-side dedup table for effectively-once processing

## Failure scenarios to test

- [ ] Consumer crashes after processing but before ack: message redelivered (at-least-once)
- [ ] Consumer acks before processing then crashes: message lost (at-most-once), documented
- [ ] Poison message: lands in DLQ after N attempts, the partition keeps flowing
- [ ] Broker leader killed with `acks=all`: no committed message lost
- [ ] Broker leader killed with `acks=1`: measure lost messages
- [ ] Consumer group rebalance during load: no message skipped
- [ ] Producer retries after timeout: no duplicates with idempotent producer

## Success criteria

- Each delivery mode has a test that proves its guarantee
- Benchmark: messages/sec and end-to-end p99 latency for 1 and 3 brokers
- Ordering verified per key across failovers

## Tech stack

Go, gRPC, file-based segment storage, Docker Compose (3 brokers), Toxiproxy, Prometheus (consumer lag, DLQ size, under-replicated partitions).

## Planned folder structure

```
courier/
├── README.md
├── docs/            # design, delivery-semantics proofs, benchmark results
├── proto/
├── cmd/             # broker, producer CLI, consumer CLI
├── internal/        # log segments, partitions, groups, replication, controller
├── deploy/
└── tests/
```

## References

- Kafka paper (LinkedIn, 2011) and Kafka design docs (ISR, idempotent producer, KRaft)
- DDIA chapter 11
- Amazon SQS docs (visibility timeout, DLQ)
- Jay Kreps, "The Log: What every software engineer should know about real-time data's unifying abstraction"
