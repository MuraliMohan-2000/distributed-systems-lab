# Silo: Distributed File Storage System

> Build a mini version of S3 / Google Drive. Split files into chunks. Store those chunks across multiple nodes. Now kill 2 nodes.

**Roadmap stages:** 3 Replication, 4 Partitioning, 7 Failure Recovery
**Real-world references:** Google File System (GFS), HDFS, Amazon S3, Ceph, MinIO, SeaweedFS
**Status:** Not started

## Learning goals

- Chunking large files
- Replication of chunks (and erasure coding as a stretch)
- Metadata management (namespace, chunk locations)
- Failure recovery and automatic re-replication
- Checksums and silent-corruption detection
- Consistency of metadata vs data

## Question to answer

If you kill 2 storage nodes, can every file still be read, and how long until every chunk is back to its target replication factor?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `PUT /buckets/{b}/objects/{key}` | Upload (multipart for large files) |
| `GET /buckets/{b}/objects/{key}` | Download (supports byte ranges) |
| `DELETE /buckets/{b}/objects/{key}` | Delete (garbage collected later) |
| `LIST /buckets/{b}?prefix=` | List objects |

## Architecture

```
 Client ──► API gateway ──► Metadata service (replicated via Raft/etcd)
                │              key → [chunk1, chunk2, ...]
                │              chunk → [nodeA, nodeD, nodeF], checksum
                ▼
   ┌─────────┬─────────┬─────────┬─────────┬─────────┐
   │ Store A │ Store B │ Store C │ Store D │ Store E │  chunk servers (heartbeat + chunk reports)
   └─────────┴─────────┴─────────┴─────────┴─────────┘
         background: scrubber, re-replicator, garbage collector
```

## Milestones

### Phase 1: Chunking and placement (Stage 4)
- [ ] Split uploads into fixed-size chunks (e.g., 4 MB) with SHA-256 checksums
- [ ] Single metadata service with object → chunk mapping
- [ ] Placement across chunk servers (rack/zone aware in config)
- [ ] Streaming upload/download; range reads

### Phase 2: Replication (Stage 3)
- [ ] Replication factor 3 with pipelined writes (client → A → B → C)
- [ ] Write is acknowledged only after all (or a quorum of) replicas confirm
- [ ] Chunk servers send heartbeats and chunk reports to metadata service

### Phase 3: Failure recovery (Stage 7)
- [ ] Detect dead chunk servers via missed heartbeats
- [ ] Re-replicate under-replicated chunks with prioritization (1 copy left first)
- [ ] Background scrubber verifies checksums and repairs corrupt chunks
- [ ] Garbage collection of orphaned chunks after deletes or failed uploads

### Phase 4: Highly available metadata
- [ ] Replicate the metadata service using Raft (Crown) or etcd
- [ ] Metadata failover with no lost object entries

### Phase 5: Stretch
- [ ] Erasure coding (Reed-Solomon, e.g., 6+3) and storage overhead comparison
- [ ] Content-addressed dedup of identical chunks
- [ ] S3-compatible API subset (test with the AWS CLI)

## Failure scenarios to test

- [ ] Kill 2 of 5 storage nodes: all files still readable
- [ ] Measure time until replication factor is restored
- [ ] Flip bytes in a chunk file on disk: detected on read and by scrubber, then repaired
- [ ] Kill a chunk server mid-upload: upload fails cleanly or completes; no partial object visible
- [ ] Kill metadata leader: follower takes over
- [ ] Disk full on one node: placement avoids it

## Success criteria

- Zero data loss with any 2 nodes lost (RF=3)
- Re-replication throughput and recovery time documented
- Upload/download throughput benchmark for 1 MB, 100 MB, 1 GB files

## Tech stack

Go, gRPC (internal), HTTP (public API), etcd or own Raft for metadata, local disk volumes, Docker Compose (5+ storage nodes), Prometheus (under-replicated chunks, scrub errors, throughput).

## Planned folder structure

```
silo/
├── README.md
├── docs/            # placement policy, recovery design, benchmark results
├── proto/
├── cmd/             # gateway, metadata service, chunk server, CLI
├── internal/        # chunker, placement, replication pipeline, scrubber, gc
├── deploy/
└── tests/
```

## References

- The Google File System paper (2003)
- HDFS architecture guide
- Ceph paper (CRUSH placement)
- Facebook f4 (warm BLOB storage, erasure coding)
- Amazon S3 consistency model docs
