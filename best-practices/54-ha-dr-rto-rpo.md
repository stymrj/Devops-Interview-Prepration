# High Availability & Disaster Recovery (RTO/RPO) Interview Preparation Guide

*How to Answer HA & DR Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [RTO and RPO Explained](#rto-and-rpo-explained)
2. [Designing for High Availability](#designing-for-high-availability)
3. [DR Strategies That Work](#dr-strategies-that-work)

---

## RTO and RPO Explained

### Q1: What's the difference between RTO and RPO?

**How to Answer:**

"RPO is how much data you can afford to lose, measured in time. RTO is how long you can afford to be down."

"A 1-hour RPO means your last backup or replica can't be more than an hour old. A 2-hour RTO means the business expects service back within two hours of the failure."

"I remember them this way: RPO is about the past — how much history you keep. RTO is about the future — how fast you get back on your feet."

**Key Point:** "RPO = data loss you're willing to accept. RTO = downtime you're willing to accept."

---

### Q2: How do you decide what RTO and RPO a service actually needs?

**How to Answer:**

"You don't guess — you ask what the failure costs. A checkout service losing an hour of orders during peak season is a very different number than an internal admin dashboard going dark."

"I sit down with the product folks and ask two questions: how many minutes of lost transactions hurt us, and how many minutes of downtime before customers leave. Those answers become the RPO and RTO."

"The trap is letting engineering pick them alone. We once built a 5-minute RTO system for a batch reporting job nobody looked at until Monday. That was expensive theatre."

**Key Point:** "RTO and RPO are business decisions wearing technical clothes — get the business to sign off."

---

### Q3: How does RPO relate to backup frequency?

**How to Answer:**

"Your backup frequency must be at least as aggressive as your RPO. A 24-hour RPO survives nightly backups. A 15-minute RPO means nightly backups are useless to you."

"For tight RPOs you stop thinking about backups and start thinking about replication — a standby that's seconds behind the primary. Replication is just backups taken continuously."

"The part people miss: the RPO also covers the time to *verify* the backup. An hourly snapshot that takes 45 minutes to restore into a usable state isn't an hourly RPO in practice."

**Key Point:** "RPO drives replication strategy, not just cron schedules — frequency alone isn't a plan."

---

### Q4: Why can't every service just be "five minutes RTO, zero RPO"?

**How to Answer:**

"Because zero RPO and five-minute RTO is the most expensive sentence in infrastructure. It means synchronous replication across regions, which means every write waits for the slowest link."

"Synchronous replication across 2000 km adds real latency to every database write. Your p99s get worse on the good days to buy protection for the bad days."

"So you tier it. The payments ledger gets the expensive treatment. The recommendation engine gets async replication and a 30-minute RPO, because nobody goes bankrupt over stale recommendations."

**Key Point:** "Gold-plated RTO/RPO everywhere means paying gold prices where bronze would do."

---

### Q5: How do you translate RTO and RPO into actual cost decisions?

**How to Answer:**

"You compare the cost of the protection against the cost of the failure. If an hour of downtime costs the business ₹10 lakh and a hot standby costs ₹2 lakh a month, the math is easy."

"If the standby costs ₹50 lakh a month for a service that earns ₹5 lakh a month, you're insuring a scooter with a Ferrari policy. Then you relax the targets."

"I like putting both numbers on one slide for stakeholders: cost of downtime per hour on the left, cost of each protection tier on the right. The conversation becomes rational very fast."

**Key Point:** "DR is insurance — buy a policy sized to the asset, not to your anxiety."

---

## Designing for High Availability

### Q6: What's the difference between high availability and disaster recovery?

**How to Answer:**

"High availability keeps you up through *expected* failures — a dead node, a bad deploy, an AZ outage. Disaster recovery brings you back from the *unexpected* — a whole region gone, data corruption, ransomware."

"HA is measured in nines and handles the Tuesday problems automatically. DR is measured in RTO/RPO and handles the once-a-decade problems with a runbook and humans."

"The confusion costs people money: they buy a second region thinking they've bought DR, then discover nobody ever tested the failover and the DNS TTL was 24 hours."

**Key Point:** "HA survives the ordinary. DR survives the extraordinary. You need both, and they're different muscles."

---

### Q7: How many 9s do I actually need, and what do they cost?

**How to Answer:**

"Three 9s — 99.9% — is about 8.7 hours of downtime a year. Four 9s is 52 minutes. Five 9s is five minutes. Each extra nine roughly multiplies your complexity and cost."

"Most internal tools are fine at 99.9%. Customer-facing checkout at a big retailer might justify 99.99%. Five 9s is for things like payment authorization networks, and they pay accordingly."

"The honest interview answer: you pick nines from revenue impact, not pride. I've seen teams chase 99.99% for a service that was down for days unnoticed because nobody checked the dashboard."

**Key Point:** "Each extra nine costs roughly an order of magnitude — buy them where downtime bleeds revenue."

---

### Q8: What's an active-active setup vs active-passive?

**How to Answer:**

"Active-active runs traffic in two or more places at once — both serve reads and writes. Active-passive keeps one side warm while the other idles, taking over only on failure."

"Active-active gives you better utilization and faster failover, but it's hard: you need conflict resolution for writes, data replication that's near-instant, and routing that stays consistent."

"Active-passive is simpler and cheaper, but failover is a bigger event — DNS flips, caches are cold, and the first five minutes are ugly. Most teams start here and only go active-active when the traffic math demands it."

**Key Point:** "Active-active is two engines running; active-passive is one engine and a spare in the trunk."

---

### Q9: How do multi-AZ deployments protect you, and where do they stop?

**How to Answer:**

"Multiple availability zones protect you from datacenter-level failures — power, cooling, a bad network switch. Your app runs in three AZs, one dies, traffic shifts. The user never notices."

"They don't protect you from a bad deploy, because the same broken image goes to all three AZs. They don't protect you from data corruption or a region-wide outage either."

"So multi-AZ is your first line of HA, not your DR plan. I always say: AZs protect against hardware. Regions protect against everything else."

**Key Point:** "Multi-AZ survives a dead building. It won't save you from your own code."

---

### Q10: What do I need to make a database highly available?

**How to Answer:**

"You need a primary that accepts writes, synchronous or near-synchronous replicas that can take over, and automated failover that actually promotes one of them without you typing commands at 3 AM."

"The failover piece is where teams fail. A standby that nobody ever promoted is just an expensive copy. You need health checks, a promotion mechanism, and application connections that follow the new primary."

"And test it. I've seen failover setups where the DNS record for the writer endpoint was hardcoded in three places. The replica promoted fine and the app still talked to the dead box."

**Key Point:** "A standby you can't automatically promote is decoration — the failover path is the feature, not the replica."

---

## DR Strategies That Work

### Q11: Walk me through the DR tiers — backup/restore, pilot light, warm standby, multi-site active.

**How to Answer:**

"Backup and restore is the cheapest: you keep backups in another region and rebuild when disaster strikes. RTO is hours. It's fine for non-critical workloads."

"Pilot light keeps the data replicated and a minimal version of the infra idling — like a gas pilot light. On failure you scale up. RTO drops to tens of minutes."

"Warm standby runs a scaled-down copy of production, always live. Failover is routing traffic over. Multi-site active is the top tier — full production in two regions, both serving. RTO is near zero, cost is near double."

"I usually draw it as a slider: cost goes up left to right, RTO goes down. The business picks where to sit for each workload."

**Key Point:** "DR tiers are a cost slider — match each workload to the tier its downtime economics justify."

---

### Q12: How do you test a DR plan without scaring everyone?

**How to Answer:**

"You test it the way you'd test anything dangerous: in layers. Tabletop exercises first — walk through the runbook on a call and find the steps that are lies."

"Then game days on non-production: actually fail over staging and see what breaks. Then chaos drills in production with blast radius limited — kill one pod, one AZ, then eventually one region during a quiet window."

"The rule: every drill gets easier because you fix what broke last time. A DR plan you've never tested is a hope, and hope is not a strategy."

**Key Point:** "Untested runbooks are fiction — tabletop it, game-day it, then chaos it."

---

### Q13: What's the most common reason DR plans fail in practice?

**How to Answer:**

"Stale runbooks. The plan says 'restore from the nightly snapshot' but the snapshot policy changed six months ago, the restore script references a deleted IAM role, and the person who knew the quirks left the company."

"Second place goes to DNS and certificates. The infra fails over beautifully and then the TLS cert for the new endpoint was never issued, or the DNS TTL was a day so nobody's traffic arrives."

"That's why DR testing isn't optional. Plans rot. The only DR plan that works is the one you ran last quarter."

**Key Point:** "DR plans fail from rot, not design — the last test date matters more than the document."

---

### Q14: How does chaos engineering fit into HA/DR?

**How to Answer:**

"Chaos engineering is how you prove your HA assumptions are true on a normal Tuesday instead of discovering they're false on a terrible Friday."

"You start tiny: kill a pod in staging and watch it reschedule. Then terminate a node, blackhole an AZ, simulate region latency. Each experiment has a hypothesis and a blast-radius limit."

"The cultural part matters more than the tooling. Teams that do chaos regularly stop fearing failures, and the incident reviews get shorter because the surprises already happened in a drill."

**Key Point:** "Chaos engineering trades small controlled failures now for no catastrophic surprises later."

---

### Q15: You get paged: primary region is down. Walk me through your first 10 minutes.

**How to Answer:**

"First minute: confirm it's really the region, not my laptop or the monitoring. Check the provider status page and a second signal."

"Minutes two to five: declare the incident, pull the DR runbook, get the right people on the bridge — the runbook names them, I don't figure this out live. Start the failover steps in order."

"Minutes five to ten: verify the secondary is actually serving — not just 'promoted' but handling real traffic. Check error rates, not just green dashboards. Communicate outward: status page updated before customers ask."

**Key Point:** "In a real DR event: confirm, declare, runbook, verify with real traffic, communicate — in that order."
