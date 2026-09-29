# SLI, SLO, SLA and Error Budgets — Cheat Sheet

## The stack

| Term | What it is | Stakes |
|------|-----------|--------|
| SLI | Raw measurement (e.g. % 2xx < 300ms) | None — just the number |
| SLO | Internal target on the SLI over a window | Feature freeze when missed |
| SLA | External contract with the customer | Money (service credits) |

## Budget math (30-day window)

| SLO | Error budget | = minutes/month |
|-----|--------------|-----------------|
| 99% | 1% | 432 min (~7.2 h) |
| 99.9% | 0.1% | 43.2 min |
| 99.95% | 0.05% | 21.6 min |
| 99.99% | 0.01% | 4.32 min |

Budget minutes = (1 − SLO) × 43,200.

## Good SLI rules

- Measure the user, not the machine: success rate + latency at the service boundary.
- Never: CPU, memory, pod restarts, raw VM uptime.
- Green SLO + angry users = wrong SLI. Fix the SLI, it's a hypothesis.

## Burn-rate alerting (Google SRE workbook defaults)

| Alert | Long window | Short window | Rate | Action |
|-------|-------------|--------------|------|--------|
| Fast burn | 1 h | 5 min | 14× | Page |
| Slow burn | 6 h | 30 min | 6× | Ticket |

Multiwindow = both windows must fire. Kills false pages.

## PromQL patterns

```promql
# availability SLI
sum(rate(http_requests_total{status=~"2.."}[5m]))
/ sum(rate(http_requests_total[5m]))

# latency SLI: share under 0.3s (histogram)
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
/ sum(rate(http_request_duration_seconds_count[5m]))

# p99 latency (watch it, don't average it)
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

## Policy defaults

- SLA looser than SLO (e.g. SLO 99.9%, SLA 99.5%) — tripwire before the cliff.
- Budget < 10% → feature freeze, deploys gated.
- Review SLOs quarterly with product; change SLI when users complain while green.
- Different nines per service: payments 99.99%, internal tools 99%.

## Interview one-liners

- "Reliability is a feature with a cost — each nine costs ~10× for downtime nobody feels."
- "The error budget turns dev-vs-ops into a number both sides can see."
- "100% is the wrong target: infinitely expensive and it freezes all change."
