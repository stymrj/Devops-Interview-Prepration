# Load Testing & Performance Engineering at Scale — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Load test | Does the system meet its targets at expected traffic |
| Stress test | Where does it break — keep adding load past the peak |
| Soak test | Does it survive at normal load for hours — catches slow leaks |
| Virtual users (VUs) | Simulated users generating load — a means to a request rate, not the target |
| Think time | Pause between requests so a VU behaves like a real user, not a machine gun |
| Throughput | Requests per second the system actually served — not the rate you asked for |
| p95 / p99 | The latency 95% / 99% of requests beat — the numbers that matter, never the average |
| Error rate | Percentage of failed requests — and which errors they are |
| Saturation | A resource so busy that new work queues — latency explodes, throughput flatlines |
| The knee | The load where latency shoots up and throughput stops growing — the saturation point |
| Circuit breaker | Fails fast to a slow dependency instead of hanging every request |
| Performance budget | An SLO the pipeline enforces — CI fails the PR when p95 breaks it |
| Headroom | How far past the target the system holds — prove 150% before a big event |

## Load vs stress vs soak

| Test | Question it answers | How it runs |
|---|---|---|
| Load | Does it meet targets at expected traffic? | Hold at the expected peak, measure p95 + errors |
| Stress | Where does it break? | Ramp past the peak until something fails |
| Soak | Does it survive for hours? | Steady normal load for 12+ hours, watch memory + pools |
| Spike | Does it survive a sudden jump? | Jump from low to peak instantly, watch recovery |

Load proves the target, stress finds the cliff, soak finds the leak — one test can't answer all three.

## Tool picks

| Tool | Tests are | Best for |
|---|---|---|
| k6 | JavaScript, runs in CI | DevOps teams — thresholds fail the build on p95 |
| JMeter | XML in a GUI | Unusual protocols, legacy teams |
| Locust | Python | Python-first teams, custom user behaviour |
| Gatling | Scala DSL | High-throughput HTTP |
| Artillery | YAML / JS | Quick HTTP + Socket.io tests |

k6 is the default — JavaScript tests that run in CI and fail the build on p95.

## The metrics that matter

| Metric | What it tells you | Watch out for |
|---|---|---|
| Throughput (served) | What the system actually handled | The gap vs requested rate is the first sign of trouble |
| p95 latency | What most users feel | Never quote the average — it hides the slow 1% |
| p99 latency | What the unlucky users feel | One slow request in a hundred barely moves the average |
| Error rate | How often requests fail | Split by error type — 500s and 429s are different stories |
| Saturation | How full each resource is | CPU + queue depth + pool usage, not CPU alone |

Throughput, p95/p99, error rate, saturation — and never trust an average.

## Designing a realistic test

| Piece | Rule |
|---|---|
| Traffic shape | Build it from production access logs — same endpoints, same ratios |
| Request rate | Match the real peak hour, not a round number |
| Test data | A pool of real IDs — one ID for everyone tests the cache, not the DB |
| Think time | Add pauses so VUs behave like users, not machine guns |
| Ramp | Stages — 25% → 50% → 100% — never jump straight to peak |
| Pass criteria | Written down before the run: p95, error rate, throughput |

The test is a recording of real traffic, not a number you invented.

## Finding the bottleneck

| Hop | Check | What it looks like |
|---|---|---|
| App | CPU, thread pool | CPU pegged, latency grows with load |
| Database | Active sessions, slow queries, lock waits | A 20ms query becomes 2s at 1,000 users |
| Network / pools | Connection counts, queue depth | Requests queuing before they reach the app |
| Downstream | Dependency latency + error rate | Your service is only as fast as its slowest call |

Measure each hop in order, fix the slowest one, re-run — one bottleneck at a time. Never scale the app to fix a database bottleneck.

## Saturation signals

| Resource | Saturated looks like |
|---|---|
| CPU | High utilisation + growing run queue |
| Connection pool | Zero idle connections, threads waiting for one |
| Database | Lock waits, I/O wait — while CPU sits at 40% |
| Message queue | Consumer lag growing, not shrinking |
| Threads | All threads busy, CPU idle — waiting on something else |

Saturation is the knee where latency explodes and throughput flatlines — watch utilisation plus queue depth, not CPU alone.

## Third parties under load

| Rule | Why |
|---|---|
| Know their rate limit | Hammering a real API at 10x its limit gives 429s and burns the vendor |
| Mock them when testing yourself | A stub answering in 20ms tests your service, not their API |
| Test the timeout path separately | A slow dependency should trip a breaker, not hang your threads |
| Timeout + circuit breaker | Fail fast — 500ms timeout, breaker opens after a few failures |

A slow third party should trip a circuit breaker, not exhaust your thread pool.

## Testing safely

| Rule | Why |
|---|---|
| Staging mirrors production | Same instance types, DB size class, autoscaling rules |
| Generators outside the system | A generator out of CPU looks like a slow app |
| Automatic stop condition | Error rate over 5% stops the test — no runaway runs |
| Never load test production | That's a game day — incident commander, gradual shift, rollback plan |
| Seed + snapshot the DB | An empty database answers in 2ms and proves nothing |

Staging that mirrors production, generators outside the system, an automatic stop — and production load tests are game days, not experiments.

## Performance budgets in CI

| Piece | How it works |
|---|---|
| SLO | Checkout API under 300ms at p95 — the contract |
| Budget | The same number as a k6 threshold — the pipeline enforces it |
| Smoke test | 2–5 minutes on every PR — fast feedback, not next week |
| Versioning | The budget lives in the same repo as the code |

A performance budget is an SLO the pipeline enforces — if the pipeline doesn't check it, it isn't a budget, it's a wish.

## Why a passing test still lies

| Lie | How it happens |
|---|---|
| Too-clean data | One product ID, one user — everything cached |
| Skipped the expensive path | Fast GETs tested, the five-write checkout never run |
| Staging ≠ production | Smaller DB, no autoscaling, a missing LB rule |
| Untouched failure | A midnight cron, a webhook, a cache expiring under real load |

Compare the test's traffic shape against production access logs and close the gap — a passing test that doesn't look like production is a rehearsal for a play nobody performs.

## Sale-day plan (checkout at 10x)

| Step | What you do |
|---|---|
| Target | Last year's sale + growth — e.g. 8,000 checkouts/min, p95 < 500ms, errors < 0.1% |
| Traffic | Mix from production logs; payment gateway mocked except one sandbox run |
| Ramp | 25% → 50% → 100%, hold at each stage |
| Watch | Throughput, p95, errors, saturation — app, DB, and queue together |
| Headroom | Prove 150% of target — the sale will exceed the forecast |
| Soak | 2 hours at target to catch the slow leak |
| Output | The sale-day runbook, written from the test's numbers |

Target from last year plus growth, test from real logs, ramp in stages, fix one bottleneck at a time, prove 150% headroom, then soak.
