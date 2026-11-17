# Disaster Recovery & Business Continuity for DevOps Interview Preparation Guide

*How to Answer Disaster Recovery & Business Continuity Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [DR Foundations: HA vs DR, RTO/RPO & the Four Patterns](#dr-foundations-ha-vs-dr-rtorpo--the-four-patterns)
2. [Data, Backups & Failover Mechanics](#data-backups--failover-mechanics)
3. [Running DR in Real Life](#running-dr-in-real-life)

---

## DR Foundations: HA vs DR, RTO/RPO & the Four Patterns

### Q1: What's the difference between high availability and disaster recovery — and why do interviewers separate them?

**How to Answer:**

"I'd draw the line by what fails, not by what the tools look like. High availability is about surviving a component failing — a node dies, a zone goes down, and the system keeps serving because there's a replica standing by. Disaster recovery is about surviving the whole thing failing — a region goes down, the database is corrupted, the primary site is gone — and you rebuild service somewhere else. The confusion I call out is people buying HA and calling it DR: three replicas in one region are still one region, and a bad deploy can take all three out at once. My interview line: HA keeps you up through a bad day, DR brings you back from a bad week — and you need both, budgeted separately."

**Key Point:** "HA survives a component or zone failing; DR survives the whole region or dataset failing — three replicas in one region is HA, not DR, and a bad deploy can kill all three at once."

---

### Q2: RTO and RPO — how do you actually set them, not just define them?

**How to Answer:**

"I'd define them in one line each, then say the numbers come from the business, not from me. RTO is how long you can be down — the clock from 'it's broken' to 'users are back'. RPO is how much data you can lose — the gap between the last good backup and the failure. The mistake I call out is quoting them like specs: 'our RTO is 15 minutes' means nothing until the business says what an hour of downtime costs. So in an interview I flip it — I ask which systems earn money per minute, and those get the tight numbers and the expensive warm standby; the internal wiki gets an RTO of a day and a backup. The numbers are a negotiation, and the negotiation is the real skill."

**Key Point:** "RTO is how long you can be down, RPO is how much data you can lose — and the numbers come from what an hour of downtime costs, not from a spec sheet."

---

### Q3: What are the four DR patterns — backup & restore, pilot light, warm standby, and active-active?

**How to Answer:**

"I'd lay them out as a ladder of cost versus speed. Backup and restore is the cheapest: data copied to another region, nothing running there — you rebuild when disaster strikes, so the RTO is hours. Pilot light keeps the data replicated and a minimal copy of the core running, so you scale up on failure — RTO in tens of minutes. Warm standby runs a scaled-down copy of everything, ready to take traffic — RTO in minutes, but you're paying for a second system. Active-active serves traffic from two regions all the time — RTO near zero, cost double, and you have to solve cross-region writes. The interview point is the trade-off: nobody picks active-active for everything, and nobody admits their backup-and-restore RTO is four hours until the day they need it."

```bash
# The honest way to compare patterns: write the RTO, RPO, and monthly
# cost on one line each — the business picks the row, you build it.
# backup-restore:  RTO ~4h,  RPO ~1h,  cost ~1.05x
# pilot-light:     RTO ~30m, RPO ~5m,  cost ~1.2x
# warm-standby:    RTO ~5m,  RPO ~1m,  cost ~1.5x
# active-active:   RTO ~0,   RPO ~0,   cost ~2x (+ write conflicts)
```

**Key Point:** "Backup and restore is cheap but slow, pilot light is the middle, warm standby is minutes at 1.5x cost, active-active is near zero at double cost plus write conflicts — the business picks the row, you build it."

---

## Data, Backups & Failover Mechanics

### Q4: How do you back up a real system — what do you back up, and what do people forget?

**How to Answer:**

"I'd start with the list everyone gets right — the database — and then name the three things teams forget. One: the state around the data — Terraform state, Kubernetes etcd snapshots, the Vault unseal keys, the CI secrets — lose those and your backups are files you can't use. Two: the configuration — the Helm values, the DNS records, the IAM policies — because rebuilding an app from a database dump alone is a weekend. Three: the backups themselves need a copy somewhere the disaster can't reach — a second region, a different account, with immutability turned on so ransomware can't encrypt the backups too. My line: back up the system, not just the data — the data is useless without the wiring that runs it."

```bash
# The backup that saves you is the boring one, run on a schedule:
# etcd snapshot (the cluster's brain), database dump, and object
# storage versioning — all copied to a second region, immutable.
ETCDCTL_API=3 etcdctl snapshot save /backups/etcd-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
aws s3 cp /backups/etcd-$(date +%F).db \
  s3://dr-backups-eu-west-1/etcd/ --storage-class STANDARD_IA
```

**Key Point:** "Back up the system, not just the data — Terraform state, etcd, Vault keys, and configs, plus an immutable copy in a second region, or ransomware encrypts your backups too."

---

### Q5: How do you test a restore — and why is an untested backup worthless?

**How to Answer:**

"I'd be blunt: a backup you've never restored is a file, not a recovery plan. The test is simple — restore into a clean environment on a schedule, not when you're already in a disaster. I do it monthly at minimum: spin up an isolated VPC, restore the database from last night's backup, point the app at it, and run the smoke tests. Then I write down the real number — how long it actually took, what broke, what was missing. The trap I call out is the 'backup succeeded' green tick in the console — that only means the file was written, not that it restores. The restore test is the only proof, and the interviewers who've lived through a real outage always ask for it first."

**Key Point:** "A backup you've never restored is a file, not a recovery plan — restore monthly into a clean environment and write down the real time, because the 'backup succeeded' tick only proves the file was written."

---

### Q6: How do you keep data consistent during failover — replication lag and split-brain?

**How to Answer:**

"I'd say this is the hardest part of DR, and it's where most designs quietly fail. Async replication means the secondary is always a few seconds behind — so when you fail over, you lose those seconds of writes, and that's your real RPO, whatever the slide says. Sync replication loses nothing but makes every write wait for the other region, so latency goes up and a slow secondary slows the primary. And the nightmare is split-brain: both sides think they're primary and accept writes, and now you have two versions of the truth to reconcile by hand. My rule: one writer at a time, fencing that blocks the old primary before the new one accepts writes, and a written answer to 'what did we lose?' before anyone declares recovery done."

**Key Point:** "Async replication means you lose the lag window on failover — that's your real RPO — and the fix is one writer at a time with fencing, or split-brain gives you two versions of the truth."

---

### Q7: How does traffic actually fail over — DNS, load balancers, and what breaks in the flip?

**How to Answer:**

"I'd say the flip is three steps, and step two is where it breaks. First the health check has to notice — and it has to test the app endpoint, not the load balancer, or it never fires while the app is down. Then DNS or the load balancer moves traffic — DNS is slow because of TTL caching, so a 300-second TTL means some users hit the dead region for five more minutes; a global load balancer flips faster but costs more. Third, the clients that were already connected — long-lived sessions, WebSockets, sticky sessions — drop and have to reconnect, so the app needs to handle reconnects gracefully. The interview line: the flip is a test, not a setting — run it in a game day, time it, and write down the real RTO, because the number on the slide is always wrong."

```bash
# Health check on the app endpoint, not the load balancer — plus a
# low TTL so the flip actually reaches users in minutes, not hours.
aws route53 create-health-check --caller-reference payments-dr \
  --health-check-config '{"Type":"HTTPS","ResourcePath":"/healthz",
  "FullyQualifiedDomainName":"payments.example.com","Port":443,
  "RequestInterval":10,"FailureThreshold":3}'
# Failover routing policy: PRIMARY record with the health check,
# SECONDARY record in the other region — and a 60s TTL on both.
```

**Key Point:** "Health check on the app endpoint, failover record with a low TTL — DNS caching means a 300s TTL keeps users on the dead region for five more minutes, so time the real flip in a game day."

---

## Running DR in Real Life

### Q8: How do you automate DR — IaC, runbooks, and the failover button?

**How to Answer:**

"I'd say the rule is simple: if a human has to remember the steps, it won't work at 3 AM. Everything that rebuilds the system lives in code — Terraform for the infrastructure, Helm or GitOps for the apps, the database restore as a script — so the recovery is a command, not a memory test. The runbook is the same thing in words: one page, numbered steps, with the exact commands and who runs them, and it gets tested in the drill so it stays true. The best teams go one step further and build a failover button — one command that flips DNS, scales up the secondary, and pages the team — because a button gets pressed in a real outage and a 40-step runbook doesn't. My interview line: DR you can run from a runbook is a plan; DR you can run from one command is a capability."

**Key Point:** "If a human has to remember the steps, it won't work at 3 AM — recovery lives in code and one failover command, with a runbook that gets tested in the drill so it stays true."

---

### Q9: How often should you run DR drills — game days vs tabletop exercises?

**How to Answer:**

"I'd give two rhythms and say which one actually finds the bugs. A tabletop exercise is cheap — the team sits in a room, walks through 'the region is down, what do we do?', and finds the gaps in the runbook, the missing phone numbers, the step nobody owns. Do it quarterly. A game day is the real thing — you actually kill the primary, flip the traffic, and time it — and that's the only exercise that finds the technical bugs, like the health check that never fires or the backup that doesn't restore. Do it at least twice a year for anything that earns money. The trap I call out is the drill that never touches production — if you've only ever failed over in staging, you've tested the script, not the system."

**Key Point:** "Tabletops quarterly find the runbook gaps; game days twice a year find the real bugs — and a drill that never touches production only tests the script, not the system."

---

### Q10: What does DR cost — and how do you sell the tier to the business?

**How to Answer:**

"I'd frame it as insurance, and sell it in the business's own numbers. The pitch is never 'we need active-active' — it's 'an hour of downtime costs us X, and warm standby costs Y a month, so it pays for itself if it saves us one hour a year.' That's the only argument that works, because the business is buying risk reduction, not infrastructure. The second part is honesty about the cheaper tiers: backup and restore costs almost nothing but the RTO is hours, and if the business accepts that in writing, that's a valid decision — the worst outcome is the one nobody chose, where everyone assumed someone else had bought the expensive tier. My line: DR is a signed decision about how much downtime costs, not a technical default."

**Key Point:** "Sell DR as insurance in the business's numbers — 'an hour costs X, warm standby costs Y, it pays for itself in one saved hour' — and get the tier accepted in writing, because the worst outcome is a tier nobody chose."

---

### Q11: What breaks most often in DR — the classic mistakes?

**How to Answer:**

"I'd give the four I see in every postmortem. One: the untested restore — the backup was green for a year and the first restore attempt fails on a corrupted file or a missing key. Two: the secondary that rotted — it hasn't been deployed to in six months, so the flip lands on an old version with a different schema. Three: the failover that nobody can trigger — the runbook lives in one person's head, or the DNS change needs a ticket and an approver who's asleep. Four: the data nobody backed up — the secrets, the Terraform state, the config — so the database comes back and the app can't start. The pattern is the same as every other ops failure: DR breaks on the boring things you didn't automate in the place you look at least."

**Key Point:** "DR breaks on the boring things — an untested restore, a secondary that hasn't been deployed to in months, a failover nobody can trigger at 3 AM, and the secrets nobody backed up."

---

### Q12: Walk me through designing DR for a real product.

**How to Answer:**

"I'd take a payments service, because the stakes make every choice obvious. First the numbers: the business says an hour down costs real money and losing a payment is unacceptable, so the RTO is 15 minutes and the RPO is near zero for the ledger. That rules out backup and restore — I'd pick warm standby in a second region, with the ledger on sync replication and the read-heavy data on async. Traffic flips through a global load balancer with a health check on /healthz, DNS is the backup path with a 60-second TTL. Backups are continuous to a second region in an immutable bucket, and the restore is tested monthly in a clean VPC with the real time written down. The runbook is one page and the flip is one command, and the whole thing gets a game day twice a year. The interview close: the design is the easy half — the monthly restore test and the twice-yearly game day are what decide whether it works when the region actually goes down."

**Key Point:** "Warm standby, sync replication for the ledger, async for reads, health-checked flip with a 60s TTL — and the monthly restore test plus the twice-yearly game day, because the design is the easy half."
