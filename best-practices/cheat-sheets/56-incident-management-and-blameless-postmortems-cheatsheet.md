# Incident Management & Blameless Postmortems — Cheat Sheet

## Severity levels

| Sev | Impact | Response |
|-----|--------|----------|
| Sev-1 | Major user impact / revenue loss | Page immediately |
| Sev-2 | Partial degradation | < 30 min |
| Sev-3 | Minor, workaround exists | Next business day |
| Sev-4 | Cosmetic / internal | Backlog |

Define these **before** anything breaks — never debate severity mid-incident.

## Incident lifecycle

1. **Detect** — alert fires (MTTD starts)
2. **Declare** — name it, page on-call, open channel
3. **Coordinate** — assign incident commander
4. **Mitigate** — stabilize first (rollback, scale, failover)
5. **Resolve** — verify metrics, declare all-clear (MTTR ends)
6. **Review** — postmortem within 5 business days

## Key roles

- **Incident Commander (IC)** — one decision-maker; coordinates, doesn't debug
- **Comms lead** — status page + stakeholder updates
- **Scribe** — keeps the timeline log
- **Subject-matter experts** — debug and mitigate

## Communication cadence

- Internal incident channel: continuous timeline updates
- Status page: every **30 min**, even with no news
- Update template: *what we know → what we're doing → when the next update lands*
- Never blame in writing, even internally

## Response order

- **Mitigate first, diagnose second** — rollback is usually the fastest fix
- Work the change timeline: *"what changed in the last hour?"*
- Timebox theories: **15 min** per hypothesis, then switch or escalate
- Check in layers: saturation → errors → latency → narrow the component

## Postmortem structure

1. **Summary** — one paragraph anyone can read
2. **Impact** — users affected, duration, cost
3. **Timeline** — minute-by-minute from the incident log
4. **Root cause** — the chain of events, not just the trigger
5. **What went well** — credit the response too
6. **What went poorly** — honestly
7. **Action items** — every item needs an **owner + deadline**, tracked as tickets

## Blameless reframing

- ❌ "Someone deployed without testing"
- ✅ "Pipeline allows deploys without a test gate on config changes"
- Always ask: *"What allowed this to happen?"* — never *"Who did it?"*

## Five whys

Keep asking "why" until you hit a **process gap**, not a person. Stop when the answer is "we should fix the system."

## Common traps

- Postmortem with no owned action items → documentation theater
- Only reviewing Sev-1s → near-misses teach the same lessons cheaper
- Stopping at "human error" → human error is a symptom of missing guardrails
- Fixing the instance, not the class → same failure repeats

## Error budget basics

- SLO 99.9% = **43 min/month** of allowed downtime
- Incidents **spend** the budget; exhausted budget = pause features, fix reliability
- Alert on burn rate: **2x** (fast burn) and **5x** (critical)
- Budget makes reliability a shared, measurable goal

## One-liners to remember

- Severity measures user impact, not cause.
- Declare, stabilize, then investigate.
- One commander, one voice.
- Silence is worse than bad news — update every 30 minutes.
- A postmortem without owned follow-ups is just documentation theater.
