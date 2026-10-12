# Gatekeeper: Distributed Rate Limiter

> Start with 100 requests / minute / user. Then distribute it across 10 servers. Now the problem changes: your servers need to share state.

**Roadmap stages:** 1 Single Server, 2 Multiple Servers
**Real-world references:** Envoy global rate limit service, Stripe's rate limiters, Cloudflare rate limiting, GitHub API limits, Kong/NGINX limit_req
**Status:** Not started

## Learning goals

- Token bucket, leaky bucket, fixed window, sliding window log, sliding window counter
- Redis as shared state
- Atomic operations (Lua scripts / `INCR` + `EXPIRE`)
- Distributed state and its accuracy vs latency trade-off
- Fail-open vs fail-closed behavior when the limiter's store is down

## Question to answer

With 10 API servers behind a load balancer, how do you enforce exactly 100 requests/minute per user without adding a slow network call to every request, and what happens when Redis is unavailable?

## API (planned)

| Operation | Description |
|-----------|-------------|
| `Allow(key, cost)` | Returns allowed/denied, remaining, reset time |
| Middleware | Wraps HTTP handlers; returns `429` with `Retry-After` and `X-RateLimit-*` headers |
| `PUT /rules` | Configure limits per key pattern (user, IP, API key, endpoint) |

## Architecture

```
            ┌──────────── Load balancer (NGINX/Envoy) ────────────┐
            ▼              ▼              ▼                       ▼
        API server 1   API server 2   API server 3   ...   API server 10
        [local cache]  [local cache]  [local cache]        [local cache]
            └──────────────┴──────┬───────┴───────────────────────┘
                                  ▼
                         Redis (or Redis Cluster)
                     atomic token-bucket Lua script
```

## Milestones

### Phase 1: Single server (Stage 1)
- [ ] Implement all five algorithms in memory behind a common interface
- [ ] Concurrent-safe; benchmark each for memory and CPU per request
- [ ] HTTP middleware with standard headers

### Phase 2: Naive distribution (Stage 2)
- [ ] Run 3 to 10 servers behind a load balancer with per-server in-memory limits
- [ ] Demonstrate the bug: effective limit = servers × limit

### Phase 3: Centralized state
- [ ] Redis fixed window with `INCR` + `EXPIRE`
- [ ] Redis token bucket and sliding window counter as atomic Lua scripts
- [ ] Use Redis server time (not client clocks) to avoid skew
- [ ] Measure added latency per request

### Phase 4: Hybrid / scalable
- [ ] Local token buckets with periodic sync to Redis (approximate, low latency)
- [ ] Measure accuracy error vs sync interval
- [ ] Redis Cluster with key hashing per user
- [ ] Configurable fail-open / fail-closed with circuit breaker around Redis

## Failure scenarios to test

- [ ] Redis down: behavior matches configured policy; no request hangs
- [ ] 500 ms latency to Redis: p99 impact; timeouts bound the damage
- [ ] Clock skew of 30 s on one server: which algorithms still work?
- [ ] Burst at a window boundary: fixed window allows 2× limit; sliding window does not
- [ ] Hot key (one user sends 90% of traffic)

## Success criteria

- Global limit enforced within a documented error margin (e.g., ±2%) across 10 servers
- Comparison table: algorithm × accuracy × latency × memory
- Added p99 latency from limiting below a target (e.g., 2 ms in-cluster)

## Tech stack

Go, Redis / Redis Cluster, NGINX or Envoy, Docker Compose, k6 for load, Toxiproxy, Prometheus (allowed/denied rate, Redis latency).

## Planned folder structure

```
gatekeeper/
├── README.md
├── docs/            # algorithm comparison, accuracy experiments
├── cmd/             # api server, load generator config
├── internal/        # algorithms, redis scripts, middleware, rules
├── deploy/
└── tests/
```

## References

- Stripe engineering blog: "Scaling your API with rate limiters"
- Cloudflare blog: sliding window counter approach
- Envoy global rate limiting docs
- Redis docs: `INCR` rate limiter pattern, Lua scripting
