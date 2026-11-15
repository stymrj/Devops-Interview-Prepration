# Incident Response & On-Call Engineering — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Good rotation | Enough people, a secondary backup, a handover note — nobody dreads their week |
| Page-worthy | A human must act now or the customer loses something |
| Ticket-worthy | Matters, but can wait for working hours |
| Error budget | Allowed mistakes: 99.9% SLO ≈ 43 minutes of downtime a month |
| Burn rate | How fast you're spending the budget — 14.4x means it's gone in hours |
| SEV1 | Full outage or data loss — all hands, updates every 30 min |
| SEV2 | Major feature down for many users — owning team, hourly updates |
| SEV3 | Degraded for some users — working hours, ticket + workaround |
| SEV4 | Minor bug — backlog |
| Incident commander | Runs the incident so engineers can fix it — owns timeline, decisions, comms |
| Rollback first | Incident lines up with a recent reversible change? Restore service, debug later |
| Blameless postmortem | Root cause as a chain of events — ask why the system let a human err |
| Game day | Rehearsed failure with an abort button — finds broken runbooks, not broken systems |
| On-call health | Pages per rotation + sustainability survey — a reliability metric, not a toughness test |

## Page vs ticket vs delete

| Signal | Action | Why |
|---|---|---|
| Error rate burning budget fast | Page | Customer impact now, budget gone in hours |
| Checkout / payments down | Page | Revenue at risk |
| Database primary unreachable | Page | Data path broken |
| Disk at 70% | Ticket | Matters, can wait for working hours |
| Single pod restarting once | Ticket | Self-healed, watch the trend |
| CPU spike, no latency change | Delete | Symptom, not user impact |
| Alert fires and nobody acts | Delete or retune | Every false page is stolen sleep |

## Severity levels

| Level | User impact | Who gets woken | First update | Who closes it |
|---|---|---|---|---|
| SEV1 | Outage / data loss | All hands + IC | Within 30 min | IC, with owning team |
| SEV2 | Major feature down, many users | On-call + owning team | Within 1 hour | Owning team lead |
| SEV3 | Degraded for some users | Nobody (working hours) | Ticket + workaround | Owning team |
| SEV4 | Minor / cosmetic | Nobody | Backlog | Backlog triage |

Set severity by user impact, not by which system broke — a tiny service can be a SEV1 if it takes checkout down.

## First 15 minutes after a page

| Time | Action |
|---|---|
| 0–2 min | Acknowledge the page, open the alert link — don't guess from the title |
| 2–5 min | Check dashboards: error rate, latency, deploys in the last hour, scope |
| 5–10 min | Declare severity — early costs nothing, late costs trust |
| 10–15 min | Roll back the obvious suspect if it's recent and reversible |
| All along | Write timestamps in the incident channel — that's your postmortem |

## Rollback vs debug forward

| Situation | Action |
|---|---|
| Started minutes after a deploy/config change, reversible | Roll back first, debug later |
| Change was hours ago / blast radius doesn't match | Keep debugging, find the real cause |
| Rollback itself is risky (migration, schema) | Fail over or shed load instead |
| "I can fix this in five minutes" (third time) | Stop — rollback is the fastest safe path back to green |

## Multi-window burn alert (14.4x, page)

```yaml
- alert: ErrorBudgetBurningFast
  expr: |
    job:slo_errors_per_request:ratio_rate1h{job="checkout"} > (14.4 * 0.001)
    and
    job:slo_errors_per_request:ratio_rate6h{job="checkout"} > (14.4 * 0.001)
  for: 2m
  labels:
    severity: page
```

Two windows (1h + 6h) so a blip can't page you; 14.4x burn means the monthly budget is gone in about two hours.

## Incident comms

| Rule | Detail |
|---|---|
| First update | Within 30 min of declaring SEV1, even if it's only "investigating" |
| Cadence | Every 30 min for SEV1, hourly for SEV2 — even with no news |
| Never | Guess a cause in public, or give a recovery time you can't defend |
| Channels | Separate: responders / stakeholders / customers |
| All-clear | What broke, what fixed it, what's being watched |

## Postmortem: the five parts

| Part | Content |
|---|---|
| Timeline | Timestamps from the incident channel — no gaps |
| Impact | Numbers: users affected, minutes down, errors, revenue if known |
| Root cause | A chain of events, not one person's mistake |
| Action items | Each with ONE owner and ONE due date, reviewed in sprint planning |
| Learnings | What the system now prevents, not who will "be more careful" |

## Commands to remember

```bash
# See what changed recently — deploys, config, scaling events:
kubectl rollout history deployment/checkout --namespace prod
kubectl get events --namespace prod --sort-by=.lastTimestamp | tail -20

# Check error rate and latency for one service (PromQL):
rate(http_requests_total{job="checkout",status=~"5.."}[5m])
  / rate(http_requests_total{job="checkout"}[5m])
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="checkout"}[5m]))

# Roll back the last deploy (fastest safe path back to green):
kubectl rollout undo deployment/checkout --namespace prod

# Check error-budget remaining (example: 99.9% SLO over 30d):
# budget minutes ≈ 43.2 — count downtime minutes used this month and compare.
```

## Interview traps

- Paging on symptoms — a CPU spike is not a page; the latency it causes for real users is.
- Fixed thresholds — "error rate over 5%" can't tell a blip from an outage; burn rate can.
- Severity by system — a tiny service failing is a SEV1 if it takes checkout down.
- The commander debugging — the moment they type commands, nobody is coordinating.
- Debugging forward out of pride — "five more minutes" three times is fifteen minutes of outage.
- Guessing a cause in public — say what you know, what you don't, and when you'll say more.
- Silence during an incident — no news still gets an update, because silence reads as nobody working.
- A postmortem ending in "be more careful" — carefulness is not a system fix; add a guardrail.
- Game days without an abort button — a drill that causes a real incident is just an incident.
- Treating on-call health as toughness — a burned-out responder makes worse 3 AM decisions; that's an incident risk.
