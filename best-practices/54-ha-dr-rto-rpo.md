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
