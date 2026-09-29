# DevOps and SRE Fundamentals — Cheat Sheet

*One page. Revise in 5 minutes before the interview.*

## DevOps vs SRE

- **DevOps** = philosophy: break the dev/ops wall, shared ownership, automation, fast feedback.
- **SRE** = one prescriptive implementation of DevOps (Google, ~2003): error budgets, SLIs/SLOs, toil caps, blameless postmortems.
- Not competing choices — mix SRE practices into DevOps culture freely.
- Interview trap: naming tools (Jenkins, Docker, K8s) as the definition. Tools enable; culture defines.

## CALMS

- **C**ulture — no blame, shared ownership. Can't be bought; matters most.
- **A**utomation — kill manual repetitive work permanently.
- **L**ean — small batches, cut waste, flow over big-bang.
- **M**easurement — DORA four; dashboards people actually read.
- **S**haring — runbooks, demos, postmortems, knowledge flows.

## "You build it, you run it"

- Devs carry the pager → logging, metrics, health checks get built in, not bolted on.
- Ops skills spread into dev teams; no single silo bottleneck.
- Only works with real support: sane alerting, observability, on-call compensation.

## Toil — the 5-test checklist

Toil only if ALL true: manual, repetitive, automatable, tactical (not strategic), scales with service growth.

- SRE rule: **max 50% of time on ops work**, min 50% on engineering (automation/tooling).
- Past 50% = death spiral: no time to automate → toil grows → even less time.
- Google's enforcement: excess operational load goes back to the product team.

## Error budgets (risk as arithmetic)

- 100% reliability is the wrong target — infinitely expensive, freezes change.
- Budget = 100% − SLO. 99.9% monthly ≈ **43m** downtime; 99.95% ≈ **21m**; 99.99% ≈ **4m**.
- Budget remaining → ship fast. Budget burned → freeze deploys, invest in reliability.
- Ends the dev-vs-ops fight: a shared number instead of competing fears.

## DORA four (speed + stability move together)

| Metric | Elite benchmark |
|---|---|
| Deployment frequency | On demand / multiple per day |
| Lead time for changes | < 1 hour |
| Change failure rate | 0–15% |
| MTTR | < 1 hour |

- Fast teams are *more* stable, not less — small changes fail small.
- "We go slow to be safe" is wrong on both counts, per the research.

## Quick commands (measure without tooling)

```bash
git log --since="30 days ago" --oneline | wc -l   # deployment frequency, roughly
(df=deploys; echo "change failure: $(grep -c CRITICAL alerts.log)/$df")
```

- Small batches: `git diff --stat HEAD~1` should be boring. If it's scary, split it.
- Health check pattern: `kill -0 $(cat service.pid)` → OK else page.
- Watchdog pattern: failed check → restart → re-check (ancestor of every liveness probe).

## Interview one-liners

- "DevOps is shared ownership of software in production."
- "SRE treats operations as a software problem."
- "Toil eliminated beats toil done faster."
- "Small batches fail small; big bangs fail big."
- "Risk is a budget you manage, not a fear you obey."
