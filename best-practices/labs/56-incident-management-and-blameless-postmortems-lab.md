# Incident Management & Blameless Postmortems — Hands-On Lab

Practice the mechanics of incident response: declaring incidents, writing postmortems, and building the guardrails that prevent repeats.

---

## Exercise 1: Write a Severity Matrix

**Goal:** Define Sev-1 through Sev-4 for a hypothetical service before any incident happens.

**Commands:**

```bash
cat > severity-matrix.md << 'EOF'
# Severity Matrix — example-web-app

| Sev  | Definition                        | Example                    | Response        |
|------|-----------------------------------|----------------------------|-----------------|
| Sev-1| Major user impact / revenue loss  | Checkout down              | Page now        |
| Sev-2| Partial degradation               | Slow dashboards, 10% users | Respond < 30 min|
| Sev-3| Minor, workarounds exist          | One endpoint flaky         | Next business day|
| Sev-4| Cosmetic / internal only          | Typo on admin page         | Backlog         |
EOF
cat severity-matrix.md
```

**Expected output:** A severity-matrix.md file with all four levels defined by impact and response time.

**Why it matters:** Severity debates at 3 AM waste time. Writing the matrix ahead of time turns a judgment call into a lookup.

---

## Exercise 2: Simulate an Incident Declaration

**Goal:** Practice declaring an incident the way you'd do it in a real channel.

**Commands:**

```bash
cat > incident-declaration.md << 'EOF'
## INCIDENT DECLARED — Sev-1
- **Time:** $(date -u +%Y-%m-%dT%H:%M:%SZ)
- **Symptom:** /checkout returning 500s for ~80% of requests
- **Incident Commander:** <your-name>
- **Comms channel:** #incident-2026-10-21-checkout
- **Status page:** Investigating — update in 30 min
EOF
cat incident-declaration.md
```

**Expected output:** A declaration template naming the commander, the channel, and the next update time.

**Why it matters:** A crisp declaration stops the "is this an incident?" debate and puts one person in charge immediately.

---

## Exercise 3: Build an Incident Timeline Log

**Goal:** Keep a running timeline during a simulated outage (timestamps + events only).

**Commands:**

```bash
cat > timeline.md << 'EOF'
## Timeline (UTC)
- 21:05 — PagerDuty fires: checkout 500 rate > 5%
- 21:08 — Incident declared, IC assigned
- 21:12 — Last deploy (v2.14.3) identified as suspect
- 21:16 — Rollback to v2.14.2 started
- 21:21 — Error rate back to baseline
- 21:25 — All-clear declared
EOF
cat timeline.md
```

**Expected output:** A timeline with at least 6 timestamped entries covering declare → mitigate → resolve.

**Why it matters:** The timeline is the raw material for the postmortem. Memory is unreliable after adrenaline; the log isn't.

---

## Exercise 4: Write a Status Page Update

**Goal:** Draft external updates that are honest, specific, and calm.

**Commands:**

```bash
cat > status-update.md << 'EOF'
## Update 1 (21:30 UTC) — Investigating
We're seeing elevated errors on checkout. Our team is investigating and rolling back
a recent change. Next update in 30 minutes.

## Update 2 (22:00 UTC) — Monitoring
The rollback is complete and error rates are back to normal. We're monitoring closely
and will publish a postmortem within 5 business days.
EOF
cat status-update.md
```

**Expected output:** Two status updates: one mid-incident, one after resolution, both with a promised next-update time.

**Why it matters:** Users forgive outages faster than they forgive silence. Regular updates — even with no news — preserve trust.

---

## Exercise 5: Write a Five Whys Root Cause

**Goal:** Practice going past "human error" to the systemic cause.

**Commands:**

```bash
cat > five-whys.md << 'EOF'
# Five Whys — checkout 500s
1. Why did checkout fail?          → Deploy v2.14.3 shipped a bad config value.
2. Why did the bad config ship?    → No validation ran on config changes in CI.
3. Why was there no validation?    → Config linting was never added to the pipeline.
4. Why was it never added?         → Postmortem action item from March was never done.
5. Why was it never done?          → Action items lived in a wiki with no owner or deadline.
EOF
cat five-whys.md
```

**Expected output:** A five-whys chain that ends at a process gap, not a person.

**Why it matters:** "Who ran the wrong command" teaches nothing. "Why was the wrong command possible" fixes a class of failure.

---

## Exercise 6: Draft a Postmortem Document

**Goal:** Write a complete postmortem skeleton with every required section.

**Commands:**

```bash
cat > postmortem.md << 'EOF'
# Postmortem: Checkout 500s (2026-10-21)
## Summary
One-paragraph overview of what happened and the impact.

## Impact
- 80% of checkout requests failed for 16 minutes
- ~1,200 failed orders

## Timeline
(Link to incident timeline log)

## Root Cause
Bad config value in v2.14.3; no config validation in CI.

## What Went Well
- Detection in 3 min via checkout error-rate alert
- Rollback completed in 9 min

## What Went Poorly
- Config change wasn't validated before deploy

## Action Items
- [ ] Add config schema validation to CI — Owner: ___, Due: ___
- [ ] Track postmortem action items as tickets, not wiki rows — Owner: ___, Due: ___
EOF
cat postmortem.md
```

**Expected output:** A postmortem with all seven sections filled in and every action item having an owner and due date.

**Why it matters:** Postmortems without owned, dated action items are documentation theater. The owners turn words into fixes.

---

## Exercise 7: Review a Near-Miss

**Goal:** Run the postmortem process on something that *almost* broke.

**Commands:**

```bash
cat > near-miss.md << 'EOF'
# Near-miss review
- **What almost happened:** Disk on DB primary hit 92% — auto-expand caught it.
- **Why it matters:** Without auto-expand, writes would have stalled (Sev-1).
- **Lesson:** Add a disk-usage alert at 75% so a human sees it before automation saves us.
- **Action:** Add disk alert — Owner: ___, Due: ___
EOF
cat near-miss.md
```

**Expected output:** A one-page near-miss review with at least one concrete action item.

**Why it matters:** Near-misses teach the same lessons as incidents, at a fraction of the cost. Teams that review them break less.

---

## Exercise 8: Practice the Blameless Reframe

**Goal:** Rewrite three blame-y incident statements into blameless, systems-focused ones.

**Commands:**

```bash
cat > blameless-reframe.md << 'EOF'
BEFORE: "Rahul deployed without testing."
AFTER:  "Our pipeline allows deploys without a test gate on config-only changes."

BEFORE: "The on-call ignored the alert."
AFTER:  "The alert fired 40 times this week for a non-issue, so it was being ignored."

BEFORE: "Nobody documented the rollback steps."
AFTER:  "Rollback docs live in a personal notes app, not the shared runbook."
EOF
cat blameless-reframe.md
```

**Expected output:** Three before/after pairs, each "after" pointing at a system or process gap.

**Why it matters:** This is the core blameless muscle. If you can reframe instinctively, your postmortems get honest data instead of defensive silence.

---

## Exercise 9: Define Your Error Budget Policy

**Goal:** Write a one-page policy connecting SLOs, error budgets, and incident response.

**Commands:**

```bash
cat > error-budget-policy.md << 'EOF'
# Error Budget Policy — example-web-app
- SLO: 99.9% availability on checkout (43 min/month budget)
- Budget remaining > 50%: ship normally
- Budget remaining < 50%: deploy freezes during peak hours
- Budget exhausted: feature work pauses, reliability work only
- Budget resets monthly; burn rate alerted at 2x and 5x
EOF
cat error-budget-policy.md
```

**Expected output:** A policy file tying SLO numbers to concrete team behavior at different budget levels.

**Why it matters:** Error budgets turn the dev-vs-ops fight into math. The policy makes "slow down and fix reliability" automatic, not political.
