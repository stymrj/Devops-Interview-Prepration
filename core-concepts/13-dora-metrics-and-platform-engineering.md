# DORA Metrics and Platform Engineering Interview Preparation Guide

*How to Answer DORA Metrics and Platform Engineering Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [The Four DORA Metrics](#the-four-dora-metrics)
2. [Frequency and Lead Time](#frequency-and-lead-time)
3. [Change Failure Rate and MTTR](#change-failure-rate-and-mttr)
4. [Platform Engineering](#platform-engineering)
5. [Traps and Judgment Calls](#traps-and-judgment-calls)

---

## The Four DORA Metrics

### Q1: What are the four DORA metrics? Why does anyone care about them?

**How to Answer:**

"The four DORA metrics come from Google's DevOps Research and Assessment program — years of research on what actually predicts high-performing engineering teams. The four are: deployment frequency, lead time for changes, change failure rate, and mean time to restore service.

The first two measure throughput — how fast you ship. The last two measure stability — how safely you ship. That pairing is the whole point: speed without safety is chaos, and safety without speed is stagnation.

Interviewers care because these metrics are the closest thing our industry has to an evidence-based scoreboard. When I say 'we're an elite team,' DORA gives me numbers to back it up instead of vibes."

**Key Point:** "DORA's four metrics — deployment frequency, lead time, change failure rate, MTTR — score throughput and stability together, and they're backed by years of research on what makes teams actually perform."

---

### Q2: What are the performance bands — elite, high, medium, low?

**How to Answer:**

"For deployment frequency, elite teams deploy on demand, multiple times a day. High is weekly to monthly, medium is monthly to every six months, low is less than every six months. Lead time for changes goes elite at under a day, high at a week to a month, medium at one to six months, low beyond six.

Change failure rate is under 15% for elite and high, 16 to 45% for medium, above that is low. Mean time to restore is under an hour for elite, under a day for high, under a week for medium, and longer for low.

The bands move a bit year to year as the industry improves, so I don't memorize them like scripture. What I know cold is the direction: elite is multiple-deploys-a-day, lead time under a day, failures under 15%, restore under an hour."

**Key Point:** "Elite means: deploy on demand, lead time under a day, change failure under 15%, restore under an hour — know the direction cold, not every band like scripture."

---

### Q3: Why measure all four instead of just velocity or just stability?

**How to Answer:**

"Because any single metric is gameable and optimizing one thing in isolation breaks something else. Measure only deployment frequency and teams ship garbage faster. Measure only change failure rate and teams gate everything until shipping takes months.

The four metrics are designed as counterweights. Throughput metrics pull you toward speed, stability metrics pull you toward care, and together they keep you honest. A team that improves frequency while failure rate stays flat is genuinely getting better.

The research found something surprising too: teams don't trade speed for stability. The top performers are fast AND stable. That's the insight that kills the 'move fast and break things versus slow down and be safe' debate — the best teams refuse the tradeoff."

**Key Point:** "Single metrics get gamed — speed-only ships garbage, safety-only freezes shipping; the four are designed as counterweights, and the research shows top teams are fast AND stable, no tradeoff."

---

## Frequency and Lead Time

### Q4: What counts as a 'deployment' when measuring deployment frequency?

**How to Answer:**

"A deployment is a change that reaches production and affects real users — a successful release to prod. Not a merge to main, not a staging deploy, not a PR that passed CI. The metric is 'how often do users get value,' so it has to mean prod.

In practice I count successful production releases from the CI/CD system. If you deploy a monolith once a month but a microservice daily, you measure per service — one number for the whole org hides everything.

The trap answer is 'we deploy every commit.' What the interviewer wants is the judgment: a deploy that nobody can see isn't a deploy. Count what reaches users."

**Key Point:** "A deployment is a successful release to production that reaches users — measure per service from your CI/CD system, not merges or staging pushes."

---

### Q5: How do you actually measure lead time for changes?

**How to Answer:**

"Lead time for changes is the time from code committed to code running in production. In practice I compute it as: merge-to-main timestamp minus the commit timestamp, plus deploy pipeline duration — or simpler, the time from PR merge to prod deploy, since that's where the number lives.

The easiest honest measurement: take the git merge timestamp of a PR and the deployment timestamp from your CI system, subtract, and take the median over the month. The median matters — averages get dragged around by one month-long PR.

Where teams get tripped up is including 'idea to commit' time. DORA's definition starts at commit, so I keep design and backlog time out of it. That keeps the metric about the delivery system, which is the thing I can actually improve."

```bash
# rough lead time: PR merge time -> deploy tag time (per deploy)
git log --merges --format='%H %ci' -5 main | while read sha ts; do echo "$sha merged $ts"; done
# median of (deploy_ts - merge_ts) across deploys = lead time
```

**Key Point:** "Lead time is commit-to-prod, measured as the median of merge-timestamp to deploy-timestamp — exclude design time, keep it about the delivery system."

---

### Q6: Your lead time is two weeks. How do you cut it?

**How to Answer:**

"I look at where the time actually goes first. Almost always it's one of three things: huge PRs that take days to review, a manual approval gate nobody owns, or a test suite that takes an hour and gets retried.

The fixes are unglamorous: shrink PRs so reviews take hours not days, move to trunk-based development with short-lived branches, and parallelize the test suite. I once watched a team's lead time drop from nine days to two just by killing a required two-approver policy on low-risk services.

The structural fix is making deploys boring — automated pipelines, feature flags for risky changes, and deploy-on-merge. Lead time doesn't drop because people work faster. It drops because waiting disappears."

**Key Point:** "Cut lead time by removing waits — small PRs, trunk-based flow, parallel tests, automated deploys — people don't need to work faster, the system needs less waiting."
