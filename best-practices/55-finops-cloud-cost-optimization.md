# FinOps & Cloud Cost Optimization Interview Preparation Guide

*How to Answer FinOps Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [FinOps Fundamentals](#finops-fundamentals)
2. [Tagging and Cost Visibility](#tagging-and-cost-visibility)
3. [Cutting Real Spend](#cutting-real-spend)
4. [Culture and Guardrails](#culture-and-guardrails)

---

## FinOps Fundamentals

### Q1: What is FinOps, and why should a DevOps engineer care?

**How to Answer:**

"FinOps is just the practice of keeping cloud spend under control without slowing the business down. It's not about saying no to engineers — it's about making sure every dollar maps to something that matters."

"I care because in DevOps, I'm the one provisioning the infra. If I spin up resources nobody watches, I'm spending someone else's budget. Cost is part of reliability — a service nobody can afford to keep running isn't production-ready."

"The trap answer is calling FinOps an accounting exercise. It's an engineering discipline: right architecture decisions at the right time."

**Key Point:** "FinOps makes cost an engineering concern, not a month-end surprise."

---

### Q2: How do you break down a cloud bill so each team owns its spend?

**How to Answer:**

"You make the bill attributable. On AWS that means Cost Allocation tags, on GCP labels, on Azure tags — and enforcing them from day one."

"I set up a tag policy: every resource gets `Team`, `Environment`, and `Service`. Then I build Cost Explorer reports grouped by those tags, so each team sees its own number."

"The part people miss is the shared stuff — NAT gateways, observability, data transfer. I split those by a fair rule, like share of total spend, and I publish the rule so nobody argues about it."

**Key Point:** "Untagged resources are unowned costs — tagging is the foundation of all FinOps."

---

### Q3: On-demand, Reserved, and Spot — when do you pick each?

**How to Answer:**

"On-demand is for the unknown — short-lived, unpredictable workloads. You're paying the premium for zero commitment."

"Reserved instances or Savings Plans are for the stuff that's always on. If I know a database runs 24/7 for the next year, committing saves 30-70%. I usually cover about 70-80% of steady-state with commitments and leave the rest on-demand."

"Spot is for work that can die and retry — batch jobs, CI runners, stateless workers. You get up to 90% off, but AWS can reclaim the capacity with two minutes' notice. I never put anything customer-facing on pure spot."

**Key Point:** "Commit to the baseline, spot the flexible, on-demand the unknown."

---

### Q4: What's the difference between Savings Plans and Reserved Instances?

**How to Answer:**

"Reserved Instances lock you to a specific instance family in a specific region. Savings Plans are simpler — you commit to a dollar-per-hour spend, and the discount applies across families and regions."

"In practice, Savings Plans are almost always easier. You don't have to guess whether you'll need m6i or c7g next year — you just commit to, say, $10/hour of compute."

"Both are one- or three-year commitments. I check actual usage before signing anything: commit to what you already spend, never to what you hope you'll spend."

**Key Point:** "Savings Plans trade a little flexibility cost for much less commitment risk."

---

### Q5: How do you use tags to keep costs visible?

**How to Answer:**

"I enforce a small, non-negotiable tag set: Team, Service, Environment, and CostCenter. Every IaC module I write has tags built in, so nobody can launch untagged resources."

"Then I turn on tag activation for Cost Explorer billing — tags don't show in the bill until you activate them. That one step gets skipped a lot."

"I also run a weekly report of untagged resources over a spend threshold. Anything unowned gets tagged or deleted. People learn fast when their mystery spend is public."

```bash
# Find EC2 instances missing a Team tag, using the resourcegroupstagging API
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters ec2:instance \
  --tag-filters Key=Team,Values=
```

**Key Point:** "Automate tags in IaC — asking engineers to tag manually never works."

---

### Q6: The AWS bill spikes 40% overnight. What's your first move?

**How to Answer:**

"I don't panic — I segment. I open Cost Explorer, group by service, and look at the last 30 days versus the spike day. Nine times out of ten, one service accounts for most of it."

"Common culprits: a NAT gateway pushing cross-AZ traffic, someone leaving a GPU instance running, or a new S3 bucket with request floods. The spike has a name within fifteen minutes if you filter right."

"Then I find who owns it and fix the root cause — resize, schedule, or delete — and I put an anomaly alert in place so we hear about it in hours, not at month-end."

**Key Point:** "A bill spike is a debugging exercise — segment first, then blame the resource, not the team."

## Cutting Real Spend

### Q7: How do you rightsize instances in practice?

**How to Answer:**

"I look at actual utilization, not the request. CloudWatch or the CSP's rightsizing recommendations show me instances running at 10% CPU for weeks."

"The pattern I see constantly: someone provisions for peak, then peak never comes back. I drop one size, watch for a week, and repeat. Do it in small steps — nobody gets fired for shrinking a VM slowly."

"For the confident cases I use tools like AWS Compute Optimizer. But I always sanity-check against the application — a CPU-bound batch job and an idle web server read differently at the same utilization number."

**Key Point:** "Rightsizing is boring and it works — most fleets are one to two sizes too big."

---

### Q8: How do you find and kill idle or orphaned resources?

**How to Answer:**

"I hunt for four things on a schedule: unattached EBS volumes, old snapshots, unattached Elastic IPs, and stopped-but-not-terminated instances. They bleed money quietly."

"I automate it — a weekly Lambda or a Terraform-managed script that lists candidates and tags them for review, then deletes after a grace period. Humans review, automation deletes."

```bash
# List EBS volumes that aren't attached to anything
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[].{Id:VolumeId,Size:Size,Created:CreateTime}'
```

**Key Point:** "Orphaned resources are the silent tax on every cloud bill — hunt them on a schedule."

---

### Q9: How do you cut storage costs — S3, snapshots, backups?

**How to Answer:**

"Storage is usually the second-largest line item and the easiest to cut. First: lifecycle policies — move anything older than 30-90 days to Infrequent Access or Glacier based on access patterns."

"Second: S3 Intelligent-Tiering for data with unpredictable access. It moves objects between tiers automatically and the monitoring fee is tiny compared to getting it wrong manually."

"Third: audit versioning and old snapshots. A bucket with versioning on and no lifecycle rule is a storage landfill. I cap noncurrent versions at a sane number and expire incomplete multipart uploads."

**Key Point:** "Most storage savings come from lifecycle rules, not cheaper tiers — automate the aging."

---

### Q10: What cost monitoring and alerts do you set up?

**How to Answer:**

"I start with AWS Budgets on every account — a monthly budget at expected spend, with alerts at 80%, 100%, and 120%. That catches the slow creep."

"Then I add anomaly detection. Cost Anomaly Detection learns the pattern per service and pings Slack when something deviates. That's what catches the 3 AM GPU instance."

"For real visibility I build a dashboard: spend by service, by team tag, and trend versus last month. I send the team-level numbers to each team weekly. People optimize what they can see."

**Key Point:** "Budgets catch the slow bleed, anomaly alerts catch the spikes — run both."

---

## Culture and Guardrails

### Q11: How do you do FinOps in Kubernetes?

**How to Answer:**

"Kubernetes hides cost behind shared clusters, so the first step is attribution. I use OpenCost or Kubecost to map namespace and label spend back to teams."

"The biggest win is usually requests and limits. Without limits, one team's memory leak eats the whole node pool. With requests set sanely, the scheduler packs better and you need fewer nodes."

"Then: cluster autoscaler plus scale-to-zero for dev namespaces, spot node groups for non-critical workloads, and image size discipline — smaller images pull faster and cost less to store and transfer."

**Key Point:** "In Kubernetes, requests and limits ARE your cost controls — everything else is reporting."

---

### Q12: Finance wants to cut 30% of the cloud budget. How do you handle it?

**How to Answer:**

"I treat it like an incident, not a negotiation. First I show them where the money actually goes — service by service, team by team. Most of the time the number is in three or four line items."

"I don't start by cutting reliability. I cut waste first: idle resources, oversized instances, storage without lifecycle rules, dev environments that run 24/7. That alone usually gets 15-25% without touching production."

"If the target needs real trade-offs — like downgrading a multi-AZ deployment — I make finance sign off on the risk in writing. Cost cuts have availability consequences, and the business should own that decision."

**Key Point:** "Cut waste before you cut reliability — and make the business own any trade-off in writing."
