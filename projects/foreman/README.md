# Foreman: Distributed Job Scheduler

> Imagine 10,000 jobs and 20 worker nodes. Build a scheduler that distributes jobs across workers.

**Roadmap stages:** 5 Leader Election, 7 Failure Recovery
**Real-world references:** Temporal, Apache Airflow, Sidekiq, Celery, Kubernetes CronJobs, Uber Cadence
**Status:** Not started

## Learning goals

- Distributing work across worker nodes
- Detecting and handling worker crashes
- Job retries and backoff
- Preventing duplicate execution
- Job timeouts
- Scheduling delays and fairness
- High availability of the scheduler itself

## Question to answer

How do you guarantee every job runs, and runs effectively once, when workers can crash halfway through and the scheduler itself can fail?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `Submit(job, runAt, cron, priority, maxRetries, timeout, idempotencyKey)` | One-off, delayed, or recurring job |
| `Cancel(jobId)` | Cancel pending/running |
| `GetStatus(jobId)` | `PENDING`, `LEASED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `DEAD` |
| Worker: `Poll(capabilities)`, `Heartbeat(leaseId)`, `Complete(leaseId, result)` | Worker protocol |

## Architecture

```
 Clients ──► Scheduler API ──► Job store (PostgreSQL)
                  │                 ▲
     Scheduler leader (elected; followers on standby)
                  │ assigns leases with fencing tokens
     ┌────────────┼────────────┬─────────────┐
     ▼            ▼            ▼             ▼
  Worker 1     Worker 2      ...         Worker 20
     └── heartbeats; lease expires → job reassigned
```

## Milestones

### Phase 1: Single scheduler, many workers
- [ ] Job store with state machine and transitions
- [ ] Workers pull jobs (`SELECT ... FOR UPDATE SKIP LOCKED` pattern)
- [ ] Leases with expiry; heartbeat extends lease
- [ ] Retries with exponential backoff; dead state after max retries

### Phase 2: Failure handling
- [ ] Reassign jobs whose lease expired (dead worker detection)
- [ ] Fencing tokens so a revived "dead" worker cannot complete a reassigned job
- [ ] Idempotency keys + guidance for idempotent job handlers
- [ ] Per-job timeout enforcement

### Phase 3: Scheduling
- [ ] Delayed jobs and cron schedules (timing wheel or indexed `run_at` column)
- [ ] Priorities and per-tenant fairness (avoid starvation)
- [ ] Measure scheduling delay (`actual start - runAt`) under load

### Phase 4: Highly available scheduler
- [ ] Multiple scheduler instances; leader elected via etcd or Crown
- [ ] Cron jobs fire exactly once per tick even across leader failover
- [ ] Partition job ranges across scheduler instances (stretch)

## Failure scenarios to test

- [ ] Kill a worker mid-job: job reassigned after lease timeout
- [ ] Pause a worker past its lease, resume: its completion is rejected (fencing)
- [ ] Kill the scheduler leader: standby takes over; no cron tick missed or doubled
- [ ] Job that always fails: lands in dead state after max retries
- [ ] Database slow/unavailable: workers back off; nothing is lost
- [ ] 10,000 jobs on 20 workers with random worker kills: every job completes, duplicates counted

## Success criteria

- 100% of jobs reach a terminal state in the chaos test
- Duplicate side effects = 0 with idempotent handlers
- Scheduling delay p99 under a target (e.g., 1 s) at 10k queued jobs

## Tech stack

Go, PostgreSQL, etcd (leader election), Docker Compose (scheduler × 3, workers × 20), Prometheus (queue depth, lease expirations, retries, delay).

## Planned folder structure

```
foreman/
├── README.md
├── docs/            # job state machine, delivery guarantees, chaos results
├── proto/
├── cmd/             # scheduler, worker, submit CLI
├── internal/        # store, leasing, cron, election, fencing
├── deploy/
└── tests/
```

## References

- Temporal architecture docs (history, matching, workers)
- "SKIP LOCKED" queue pattern in PostgreSQL
- Google SRE book: "Distributed Periodic Scheduling with Cron"
- Hashed and hierarchical timing wheels (Varghese & Lauck)
