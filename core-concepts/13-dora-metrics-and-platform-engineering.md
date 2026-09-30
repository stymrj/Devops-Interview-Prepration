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

---

## Change Failure Rate and MTTR

### Q7: What counts as a 'failure' in change failure rate?

**How to Answer:**

"A failure is a deployment that causes degraded service and needs intervention — a rollback, a hotfix, or a fix-forward that you wouldn't have done otherwise. Not a failed pipeline run, not a test that caught a bug before prod. The change has to reach users and hurt.

In practice I count it from incident data: if a deploy's change window overlaps an incident and the rollback or hotfix resolved it, that's a failure. Some teams tag every prod rollback in their deploy system — that's the cleanest source of truth.

The judgment call: a deploy that gets rolled back because of a bad config is a failure; a deploy rolled back because someone pushed the wrong branch by accident is arguably a process failure, but I still count it. DORA cares about outcomes, not blame — count everything that touched users."

**Key Point:** "A change failure is a deployment that reached users and needed a rollback or hotfix — count it from incident and rollback data, never from failed pipeline runs."

---

### Q8: Why 'mean time to restore' instead of 'mean time to repair'?

**How to Answer:**

"Because users don't care what you fixed, they care when the pain stopped. Mean time to repair measures how long until the bug is truly fixed — that can take days of root-causing. Mean time to restore measures how long until the service is healthy again, which is usually a rollback in minutes.

DORA deliberately uses restore because it rewards the right behavior: detect fast, mitigate fast, investigate later. A team that rolls back in five minutes and root-causes over two days serves users better than a team that heroically debugs for six hours while the checkout page is down.

In an interview, this is a free insight to show: restore time is what users feel, repair time is what engineers feel. Optimize for the users."

**Key Point:** "Restore measures when users stopped hurting — usually a rollback — while repair measures when the bug was truly fixed; DORA picks restore because users feel service health, not root causes."

---

### Q9: How does gaming the metrics show up? What traps have you seen?

**How to Answer:**

"The classic one: a team slashes its change failure rate by making deploys rare and gated. Frequency collapses, lead time balloons, and the failure-rate number looks amazing while the product stagnates. That's why you always read the four together.

Another trap is redefining failures away — 'that rollback wasn't a real failure, it was a config issue.' If you let teams relabel outcomes, the metric becomes fiction. The rule has to be dumb and mechanical: rollback equals failure, no appeals.

I've also seen lead time gamed by merging code weeks before it's 'really done' — commits land in main but sit behind disabled flags. The lead time number drops while nothing reaches users. Count deployments to prod, not merges to main, and that trick dies."

**Key Point:** "Read all four metrics together — any single one is gameable, and the counterweight metrics are exactly what catch the gaming."

---

## Platform Engineering

### Q10: What is platform engineering, and how is it different from DevOps or SRE?

**How to Answer:**

"Platform engineering is building internal platforms that make the right way the easy way for developers. DevOps is the culture and practice of dev and ops working together; SRE is applying engineering to reliability. Platform engineering is the product discipline — the platform team treats developers as customers.

The shift happened because DevOps at scale left every team reinventing pipelines, Terraform, and observability. A platform team builds golden paths — paved roads — so a new service gets CI/CD, monitoring, and security defaults for free instead of each team hand-rolling it.

It doesn't replace DevOps culture. A platform team with a ticket-queue mindset is just the old ops team with a new name. The difference is the product mindset: self-service APIs, documentation, and measuring developer experience like you'd measure a real product."

**Key Point:** "Platform engineering is the product discipline of building internal platforms — paved roads so developers get secure, observable, deployable services by default — built on DevOps culture, not replacing it."

---

### Q11: What is an Internal Developer Platform? What are golden paths?

**How to Answer:**

"An Internal Developer Platform — IDP — is the self-service layer developers use to build, deploy, and run their services without filing tickets. Think Backstage, Port, or a well-built internal portal: service catalog, scaffolding, deploy buttons, runbooks, all behind one interface.

Golden paths are the blessed, supported ways to do things — the template for a new microservice that comes with CI/CD, logging, metrics, and security scanning wired in. Developers can go off-path, but then they own the consequences. The paved road is easy; the wilderness is allowed but unsupported.

The interview point: golden paths reduce cognitive load. A developer shouldn't need to understand Terraform, Kubernetes networking, and Vault to ship a CRUD API. The platform absorbs that complexity and exposes simple, safe choices."

**Key Point:** "An IDP is the self-service portal for building and running services; golden paths are the blessed templates with everything wired in — the paved road that's easy by default."

---

### Q12: What does 'cognitive load' have to do with platform engineering?

**How to Answer:**

"This comes from Team Topologies: every team has limited cognitive load, and you should spend it on your business problem, not on infrastructure plumbing. A product team whose load is eaten by Kubernetes YAML and pipeline debugging has nothing left for the actual product.

Platform engineering exists to move that plumbing off the product team's plate. When the platform handles provisioning, deploys, and observability defaults, the product team's cognitive load goes back to features and users.

Interviewers love this framing because it turns platform work into business math. Every hour a developer spends fighting the platform is an hour not building the product. The platform team's job is to make that number as close to zero as possible."

**Key Point:** "Cognitive load is the real budget — platform engineering moves infrastructure plumbing off product teams so their limited load goes to the product, not YAML."

---

### Q13: Is a platform team just the old ops team renamed? What's the trap?

**How to Answer:**

"That's the most common failure mode, and interviewers ask about it constantly. The difference is self-service versus ticket queues. If developers still file a ticket and wait three days for a database, you've just renamed ops — nothing changed.

A real platform team ships self-service products: an API or portal where a developer provisions what they need in minutes. They do user research with developers, they track adoption and satisfaction, and they deprecate things developers don't use.

The other trap is building a platform nobody asked for — a beautiful internal tool with zero users. The fix is starting from developer pain: find the most painful manual process, automate exactly that, and earn adoption one golden path at a time."

**Key Point:** "A platform team is self-service and product-minded; a renamed ops team still runs ticket queues — the test is whether developers help themselves in minutes without waiting on anyone."

---

## Traps and Judgment Calls

### Q14: Your company measures nothing today. How do you start with DORA?

**How to Answer:**

"I start with the data that already exists before building any dashboard. Deployment frequency comes from CI/CD deploy logs or release tags. Lead time comes from git merge timestamps to deploy timestamps. Change failures come from rollback records and incident tags. MTTR comes from incident start and resolve times.

Week one, I compute the last quarter by hand — a script over git log and the deploy system — and present the baseline without judgment. The numbers will be ugly and that's fine; the point is establishing the starting line.

Then I automate the collection and put one dashboard where leadership looks. But I never tie the metrics to performance reviews in the first year — the moment a metric decides a promotion, it becomes fiction. Measure to learn, not to punish."

```bash
# baseline deployment frequency from release tags (last 90 days)
git tag --sort=-creatordate --format='%(creatordate:short) %(refname:short)' \
  | awk '$1 > "2026-06-30"' | wc -l
```

**Key Point:** "Start with data that already exists — git log, deploy tags, incident records — baseline by hand, automate second, and never tie metrics to reviews or they become fiction."

---

### Q15: Your DORA dashboard is all red. What's your 90-day plan?

**How to Answer:**

"First two weeks: diagnose, don't prescribe. I pull the data apart — is lead time stuck in review queues, are deploys rare because of a manual change board, is MTTR high because nobody can roll back? The fix depends entirely on which bottleneck is real.

Days 15 to 45: attack the biggest bottleneck with the smallest change. Usually that's making rollbacks one-click — it directly cuts MTTR and makes everyone braver about deploying, which lifts frequency too. Or it's killing a manual approval gate that adds days to lead time.

Days 45 to 90: institutionalize. Golden path templates for new services, trunk-based development norms, and automated the metric collection so the dashboard updates itself. And I report progress monthly in the team's own words — the numbers improving because the work got easier, not because anyone was pressured."

**Key Point:** "Diagnose for two weeks, attack the biggest bottleneck with the smallest change — usually one-click rollbacks — then institutionalize golden paths and automated measurement."

