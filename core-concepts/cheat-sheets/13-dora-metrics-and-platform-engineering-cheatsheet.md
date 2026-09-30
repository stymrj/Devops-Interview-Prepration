# DORA Metrics and Platform Engineering — Cheat Sheet

## The four metrics

| Metric | Definition | Measures |
|--------|-----------|----------|
| Deployment frequency | Successful prod releases per unit time | Throughput |
| Lead time for changes | Commit → running in prod (median) | Throughput |
| Change failure rate | % of deploys needing rollback/hotfix | Stability |
| MTTR | Mean time to restore service after incident | Stability |

Restore, not repair — users feel service health, not root causes.

## Performance bands (know the direction cold)

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| Deploy frequency | On demand, multiple/day | Weekly–monthly | Monthly–6 months | < 6 months |
| Lead time | < 1 day | 1 week–1 month | 1–6 months | > 6 months |
| Change failure rate | < 15% | < 15% | 16–45% | > 45% |
| MTTR | < 1 hour | < 1 day | < 1 week | > 1 week |

## What counts

- **Deploy** = successful release to prod reaching users. Not merges, not staging.
- **Failure** = deploy that needed rollback/hotfix after reaching users. Not failed CI runs.
- **Lead time** = median of (deploy timestamp − merge timestamp). Exclude design time.

## Measuring from git

```bash
# deployment frequency from release tags (last 90 days)
git tag --sort=-creatordate --format='%(creatordate:short) %(refname:short)' \
  | awk '$1 > "2026-06-30"' | wc -l

# lead time per deploy: merge commit date -> tag date
git log --merges --format='%H %ci' -10 main

# count rollbacks (change failures) from deploy history
kubectl rollout history deployment/api | grep -c -i rollback
```

## Measuring in Python (pandas-free)

```python
import csv, statistics
rows = list(csv.DictReader(open("deploys.csv")))
freq = len(rows) / 12.86                       # deploys per week
lead = statistics.median(float(r["lead_hours"]) for r in rows)
cfr  = 100 * sum(r["result"]=="fail" for r in rows) / len(rows)
mttr = statistics.mean(float(r["restore_minutes"]) for r in rows if r["result"]=="fail")
```

## Cutting lead time

- Small PRs + trunk-based dev (short-lived branches)
- Kill manual approval gates; parallelize the test suite
- Deploy on merge; feature flags for risky changes
- Lead time drops when waiting disappears, not when people hurry

## Cutting MTTR

- One-click rollback before perfect deploys
- Detect fast (burn-rate alerts) → mitigate fast → investigate later
- Auto-rollback on failed health checks

## Platform engineering, condensed

| Concept | One line |
|---------|----------|
| Platform engineering | Product discipline: build internal platforms devs use self-service |
| IDP | Self-service portal: catalog, scaffolding, deploys, runbooks (Backstage, Port) |
| Golden path | Blessed template with CI/CD, observability, security pre-wired; off-road allowed but unsupported |
| Cognitive load | Team Topologies: teams have limited load — spend it on the product, not plumbing |
| Platform team ≠ ops renamed | Self-service in minutes vs ticket queues in days |

## Traps

- Optimizing one metric in isolation: gated deploys → great failure rate, dead frequency. Always read all four.
- Redefining failures away ("that wasn't a real failure"). Rollback = failure, no appeals.
- Counting merges instead of prod deploys for frequency/lead time.
- Tying DORA metrics to performance reviews — the metric becomes fiction.
- Building a platform nobody asked for: start from the most painful manual process, earn adoption one golden path at a time.

## Interview one-liners

- "Top teams are fast AND stable — DORA killed the speed-vs-safety tradeoff."
- "Restore time is what users feel; repair time is what engineers feel."
- "A platform team ships self-service products; a renamed ops team still runs ticket queues."
- "Lead time doesn't drop because people work faster — it drops because waiting disappears."
