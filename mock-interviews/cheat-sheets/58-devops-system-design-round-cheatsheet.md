# DevOps System Design Round — Cheat Sheet

One page. Read it in the cab on the way to the interview.

## The 45-minute framework

| Phase | Time | Do this |
|-------|------|---------|
| Scope | 5 min | Scale numbers, priorities, existing constraints |
| Requirements | 5 min | Functional + non-functional, SLIs/SLOs |
| Happy-path sketch | 10 min | 3-5 boxes, no deep dives yet |
| Deep dives | 20 min | Follow the interviewer's probes |
| Wrap-up | 5 min | Tradeoffs + what changes at 10x |

## Scale numbers to quote

- 1M req/day ≈ 12 rps avg, ~100 rps peak | 100M req/day ≈ 1.2k rps avg
- p99 < p95 < p50 for latency SLIs; measure at the edge
- A single AZ failure is routine; design multi-AZ by default
- Multi-region ≈ 2x cost — always ask if the business needs it

## Component picks (with the tradeoff)

- Compute: containers/serverless for stateless+bursty; VMs/dedicated for stateful+steady
- Deploys: rolling (simple), blue-green (instant rollback), canary (risky releases)
- DB: managed RDS (ops-free, pricey) vs self-hosted (cheap, you own the pager)
- Cache: Redis in front of reads; TTLs you can defend
- Queues: decouple heavy work; bulkhead so one bad job can't eat the pool
- Secrets: vault/managed service, runtime injection, never in repo or image

## Resilience patterns (name 3, apply everywhere)

1. Timeouts + retries with exponential backoff
2. Circuit breakers — stop cascading slowness
3. Graceful degradation — define what the user sees when X is down

## Pipeline shape

`PR → preview env → merge → build once → test → canary (5%) → progressive rollout → auto-rollback on SLO burn` — pipeline itself is code, reviewed like code. Ship schema changes separately, backward-compatible.

## Cost levers (in order)

1. Data transfer out — usually the biggest line
2. Idle capacity → autoscaling on real utilization
3. Spot/preemptible for batch, on-demand for steady
4. Lifecycle policies: hot → warm → glacier

## Lines that land

- "Scope first, sketch the happy path, then let your questions pick the deep dives."
- "Every new requirement trades against something — here's what I'm giving up."
- "Page only on SLO burn; everything else is a ticket."
- "An untested fallback is just a hope — I'd game-day this."
- "At 10x I'd revisit: [sharding / multi-region / dedicated data tier]."

## Red flags to avoid

- Drawing before asking about scale
- "Kubernetes" as the answer to everything
- No alerting story on the diagram
- No rollback path drawn
- Treating cost as someone else's problem
