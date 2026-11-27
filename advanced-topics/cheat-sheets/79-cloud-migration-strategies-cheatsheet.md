# Cloud Migration Strategies — Cheat Sheet

## The 6 Rs

| Strategy | What it means | When to use it |
|---|---|---|
| Rehost | Lift-and-shift, move as-is | Deadline is the boss; stable app nobody wants to touch |
| Replatform | Same app, managed service swap (e.g. RDS) | Real value without a rewrite |
| Repurchase | Move to SaaS | Stop running it at all |
| Refactor | Re-architect | The app itself is the problem — scale, cost, fragility |
| Retire | Delete it | Nobody uses it — verify from logs first |
| Retain | Leave it where it is | Deliberate — compliance, latency, or cost says stay |

Pick per application, not per estate — rehost the boring stuff, replatform the databases, refactor only where the payoff justifies the risk.

## Lift-and-shift vs re-architect

| | Lift-and-shift | Re-architect |
|---|---|---|
| Speed | Weeks | Months |
| Risk | Low — same architecture | High — new architecture |
| Cloud benefit | Almost none at first | Full — scale, managed services |
| Cost trap | Cloud prices for on-prem architecture | Upfront investment, lower run cost |

Lift-and-shift to escape a deadline, then replatform or refactor in waves once you're in — migration is a journey, not a single move.

## Discovery checklist

| Step | What to capture |
|---|---|
| Inventory | Every server, app, database — including the forgotten ones |
| Dependencies | Who calls whom, batch jobs, reporting queries on the DB |
| Measurement | CPU/memory over a full business cycle, not one quiet weekend |
| Non-technical | Licences that don't transfer, data-residency rules, compliance |

Inventory everything, map the hidden dependencies, measure a full business cycle, and check licences and data rules before writing a migration plan.

## Migration waves

| Wave | Contents | Why |
|---|---|---|
| 1 — Boring first | Low-risk, low-dependency app | Proves the pipeline, networking and runbooks |
| 2 — Shared services | CI/CD, monitoring, shared DB patterns | Everything else depends on them |
| 3 — Crown jewels | Payments, core systems | Team has done it five times by now |
| Never | Cutover week before a big sale | Business calendar beats the migration calendar |

Boring first to prove the machinery, shared services second, crown jewels last.

## Live database migration

| Approach | Verdict |
|---|---|
| Snapshot + CDC (DMS / Debezium) | The standard — target stays seconds behind, cutover is a flip |
| Big-bang dump & restore | Only for small, quiet databases — downtime is the whole copy |
| Dual-write | A trap — two writers can disagree, and then you have two sources of truth |
| Check before cutover | Replica lag in single-digit seconds; schema features that don't replicate |

Snapshot plus CDC keeps the target seconds behind — cut over only when the replica is provably current, and never trust dual-write.

## Zero-downtime cutover

| Piece | How |
|---|---|
| Data | Pre-replicated, seconds behind — the switch is a flip, not a copy |
| Traffic | DNS with a short TTL lowered days ahead, or weighted load-balancer shift, or blue-green 10% → 50% → 100% |
| Rollback | Rehearsed, one command, with triggers agreed in advance (e.g. error rate > 1% for 5 min) |

Data pre-replicated, traffic shifted gradually, rollback rehearsed — if any of the three isn't ready, the cutover waits.

## Strangler fig

| Step | What happens |
|---|---|
| Proxy in front | Gateway routes traffic between monolith and new services |
| One slice at a time | Carve out one endpoint, route it to the new service, prove it in production |
| Shared database | New service needs an anti-corruption layer — don't inherit the old schema's mess |
| End state | Monolith shrinks until nothing is left to strangle |

Proxy in front, one slice at a time, prove each in production.

## Hybrid networking

| Piece | Choice |
|---|---|
| Early / low traffic | Site-to-site VPN — quick, rides the public internet |
| Heavy / latency-sensitive | Direct Connect / ExpressRoute / Interconnect — private line, consistent latency |
| DNS | Conditional forwarders or a private hosted zone both sides trust |
| CIDRs | Non-overlapping, decided on day one — overlapping networks can't route to each other |

VPN to start, private line when traffic justifies it, DNS both sides trust, and non-overlapping CIDRs decided on day one.

## What breaks most often

| Breaks | Why |
|---|---|
| Identity | Service account / AD group has no cloud equivalent mapped — IAM roles or workload identity needed |
| Secrets | Hardcoded connection strings and baked-in certificates don't travel |
| Networking | Hardcoded IPs, firewall rules for on-prem subnets, third-party allowlists missing the new cloud egress IPs |

Identity, secrets, and network allowlists break first — audit all three per app before its wave.

## Post-cutover validation

| Check | How |
|---|---|
| Smoke tests | The money flows — login, checkout, nightly report — immediately after the flip |
| Golden signals | Latency, error rate, traffic — against the pre-migration baseline, not a guess |
| Rollback triggers | Agreed in advance, so nobody judges at 2 a.m. |
| Hypercare | A week of close watching, old environment still warm, rollback path still working |

Smoke tests on the money flows, golden signals against the baseline, rollback triggers agreed in advance, and a week of hypercare with the old environment warm.

## Payments cutover order

| Order | Step |
|---|---|
| 1 | Discovery — app, Postgres, cron jobs, gateway allowlist, reporting queries |
| 2 | Database replicates first — snapshot + DMS/CDC into RDS, lag to seconds |
| 3 | App proves itself in shadow — ECS + ALB, IAM auth, Secrets Manager, test traffic |
| 4 | Cutover at the quiet hour — DNS TTL lowered days ahead, traffic flips, DB switch is a pointer change |
| 5 | Old database stays warm for a week as the rollback target |

Database replicates first, app proves itself in shadow, traffic flips at the quiet hour, old database stays warm for a week as the rollback.
