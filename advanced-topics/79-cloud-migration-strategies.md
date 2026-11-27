# Cloud Migration Strategies for DevOps (6 Rs, Strangler Fig & Zero-Downtime Cutover) Interview Preparation Guide

*How to Answer Cloud Migration Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Migration Fundamentals](#migration-fundamentals)
2. [Planning & Discovery](#planning--discovery)
3. [Data & Cutover](#data--cutover)
4. [Networking & Security in Migration](#networking--security-in-migration)
5. [Migration in Production](#migration-in-production)

---

## Migration Fundamentals

### Q1: What is cloud migration, and why do companies actually migrate?

**How to Answer:**

"I'd say migration is moving workloads, data and operations from on-prem or another cloud into a target cloud — and the honest reasons are rarely just 'the cloud is cheaper'. Companies migrate to stop running hardware, to scale on demand instead of buying servers for peak, and to get managed services like RDS or EKS so their small team stops babysitting databases. Sometimes the trigger is a hard deadline — a data centre lease ending, or a vendor going end-of-life — and sometimes it's the business wanting to ship faster. My line: migration is a business move with a technical plan, not a technical move looking for a reason — so I always start with what the business gets, then pick the strategy."

**Key Point:** "Migration is a business move with a technical plan — start with what the business gets, then pick the strategy."

---

### Q2: The 6 Rs — rehost, replatform, repurchase, refactor, retire, retain — how do you actually choose between them?

**How to Answer:**

"I'd explain the 6 Rs as a decision ladder, from least change to most. Rehost is lift-and-shift — move the VM as it is, fastest, but you carry every bad habit with you. Replatform is lift-and-tinker — same app, but swap in a managed service like RDS for your self-managed Postgres, so you get real value without a rewrite. Repurchase is moving to SaaS — stop running it at all, like moving from self-hosted Jira to the cloud version. Refactor is re-architecting — breaking a monolith into services or going serverless, the most value but the most risk and time. Retire is deleting what nobody uses, and retain is deliberately leaving something where it is. My line: I pick per application, not per estate — rehost the boring stuff to move fast, replatform the databases, and refactor only the one or two apps where the payoff justifies the risk."

```bash
# A quick first pass: tag every app in the inventory with its R before any work starts.
# rehost = move as-is, replatform = managed service swap, refactor = rewrite,
# repurchase = SaaS, retire = delete, retain = leave it alone.
```

**Key Point:** "Pick per application, not per estate — rehost the boring stuff, replatform the databases, refactor only where the payoff justifies the risk."

---

### Q3: Lift-and-shift vs re-architect — when is each one right?

**How to Answer:**

"I'd say lift-and-shift is right when the deadline is the boss — a data centre lease ending in three months, or a compliance deadline — because it moves the risk, not the architecture. It's also right for stable apps that just work and nobody wants to touch. Re-architecting is right when the app itself is the problem — it can't scale, it falls over under load, or it costs a fortune because it's written for fixed servers. The trap is doing lift-and-shift and calling it done — you end up paying cloud prices for on-prem architecture, with none of the benefits. My line: lift-and-shift to escape a deadline, then replatform or refactor in waves once you're in the cloud — migration is a journey, not a single move."

**Key Point:** "Lift-and-shift to escape a deadline, then replatform or refactor in waves once you're in — migration is a journey, not a single move."

---

## Planning & Discovery

### Q4: How do you assess an estate before migrating — what does discovery actually involve?

**How to Answer:**

"I'd say discovery is the step everyone skips and then pays for during cutover. First I inventory everything — servers, apps, databases, and the stuff nobody documented, like the cron job on a forgotten VM that sends the nightly report. Then I map dependencies — which app talks to which database, which batch job feeds which report — because the app you think is standalone usually has three hidden callers. Then I measure: CPU and memory over a full business cycle, not one quiet weekend, so I size the cloud target on reality. And I check the non-technical stuff: licences that don't transfer, data that legally can't leave a country, and compliance rules. My line: inventory everything, map the hidden dependencies, measure a full business cycle, and check the licences and data rules before anyone writes a migration plan."

**Key Point:** "Inventory everything, map the hidden dependencies, measure a full business cycle, and check licences and data rules before writing a migration plan."

---

### Q5: How do you decide the migration order — waves, business risk, and what goes first?

**How to Answer:**

"I'd say I migrate in waves, and the first wave is deliberately boring. Wave one is a low-risk, low-dependency app — something real enough to prove the pipeline, the networking and the runbooks, but harmless if it wobbles. Wave two is the common services — the shared database patterns, the CI/CD, the monitoring — because everything else depends on them. The crown jewels go last, when the team has done it five times and the runbooks are proven. I also plan around business calendars — never cut over the payments system the week before the big sale. My line: boring first to prove the machinery, shared services second, crown jewels last — and never cut over next to a business-critical date."

**Key Point:** "Boring first to prove the machinery, shared services second, crown jewels last — and never cut over next to a business-critical date."

---

## Data & Cutover

### Q6: How do you migrate a live database with minimal downtime — replication, CDC, dual-write?

**How to Answer:**

"I'd say you never do a big-bang dump and restore on a live database — you replicate, then switch. The standard pattern is a snapshot plus change-data-capture: take a consistent snapshot, then stream every change after it with a tool like AWS DMS or Debezium until the replica is seconds behind the source. For a big table, I check the replica lag, the long-running transactions, and whether the schema has anything that doesn't replicate — like a stored procedure writing to a file. Dual-write — writing to both databases from the app — sounds clever but it's a trap: the two writes can disagree, and now you have two sources of truth and no way to know which is right. My line: snapshot plus CDC to keep the target seconds behind, verify the lag and the edge cases, and cut over only when the replica is provably current."

```bash
# Watch replication lag until it is stable in single-digit seconds before cutting over:
aws dms describe-replication-tasks --filters Name=replication-task-arn,Values=<task-arn> \
  --query 'ReplicationTasks[0].ReplicationTaskStats'
# Or for Debezium: check the connector lag metric and the snapshot phase in the connector status.
```

**Key Point:** "Snapshot plus CDC keeps the target seconds behind — cut over only when the replica is provably current, and never trust dual-write."

---

### Q7: How do you cut over with zero — or near-zero — downtime?

**How to Answer:**

"I'd say zero downtime is a plan, not a hope, and it has three parts. First, the data is already replicated and seconds behind, so the switch is a flip, not a copy. Second, traffic moves in a controlled way — DNS with a short TTL lowered days ahead, or a load balancer weighted shift, or a blue-green deploy where the new environment takes 10% of traffic first. Third, and most important, the rollback is rehearsed — I know exactly how to send traffic back and how to get the data back, and I've tested it, because a cutover without a tested rollback is a gamble. My line: data pre-replicated, traffic shifted gradually, rollback rehearsed — and if any of the three isn't ready, the cutover waits."

**Key Point:** "Data pre-replicated, traffic shifted gradually, rollback rehearsed — if any of the three isn't ready, the cutover waits."

---

### Q8: What is the strangler fig pattern, and how do you migrate a monolith service by service?

**How to Answer:**

"I'd say the strangler fig is how you migrate a monolith without a big-bang rewrite — you grow the new system around the old one until the old one withers. You put a proxy or API gateway in front, then carve out one slice at a time — say, the search endpoint — and route just that traffic to the new service while the rest still hits the monolith. Each slice proves itself in production before the next one moves, and the monolith shrinks until one day there's nothing left to strangle. The hard part is the shared database — if both systems write to the same tables, you need an anti-corruption layer so the new service doesn't inherit the old schema's mess. My line: proxy in front, one slice at a time, prove each in production, and keep the new service from inheriting the old database's mess."

**Key Point:** "Proxy in front, one slice at a time, prove each in production — and keep the new service from inheriting the old database's mess."

---

## Networking & Security in Migration

### Q9: How do you connect on-prem and the cloud during a migration — VPN vs Direct Connect, hybrid DNS?

**How to Answer:**

"I'd say during a migration you're running hybrid, so the connection between the two worlds is the whole game. A site-to-site VPN is quick to set up and fine for early waves and low traffic, but it rides the public internet, so latency and bandwidth vary. Direct Connect — or ExpressRoute, or Cloud Interconnect — is a private line: consistent latency, higher bandwidth, and it costs real money, so I bring it in when the data volume or the latency requirement justifies it. DNS is the part people forget — the app in the cloud has to resolve on-prem hostnames and vice versa, so I set up conditional forwarders or a private hosted zone that both sides trust. And the CIDR ranges must not overlap, because overlapping networks can't be routed to each other and fixing that mid-migration is painful. My line: VPN to start, private line when the traffic justifies it, DNS that both sides trust, and non-overlapping CIDRs decided on day one."

```bash
# Sanity check before the first wave: no overlapping CIDRs, DNS resolving both ways.
# On-prem: 10.0.0.0/16, Cloud VPC: 10.1.0.0/16 — distinct, routable, no overlap.
dig +short db.internal.example.com @10.0.0.2    # on-prem DNS answers for the cloud side
dig +short app.internal.example.com @10.1.0.2   # cloud DNS answers for the on-prem side
```

**Key Point:** "VPN to start, private line when traffic justifies it, DNS both sides trust, and non-overlapping CIDRs decided on day one."

---

### Q10: What breaks most often in a migration — identity, secrets, networking gotchas?

**How to Answer:**

"I'd say the app code is rarely what breaks — it's the stuff around it. Identity breaks first: the app used a service account or an on-prem AD group, and in the cloud it needs IAM roles or workload identity, and the mapping was never documented. Secrets break second: hardcoded connection strings and certificates baked into VMs don't travel, and the first deploy fails on a missing secret nobody knew existed. Networking breaks third: a hardcoded IP, a firewall rule that allowed a specific on-prem subnet, or a third-party allowlist that doesn't include the new cloud egress IPs. My line: identity, secrets, and network allowlists — I audit all three for every app before its wave, because the code itself is usually the easy part."

**Key Point:** "Identity, secrets, and network allowlists break first — audit all three per app before its wave; the code is usually the easy part."

---

## Migration in Production

### Q11: How do you validate after cutover — smoke tests, rollback triggers, hypercare?

**How to Answer:**

"I'd say cutover isn't done when traffic moves — it's done when the business confirms the system works. I run automated smoke tests immediately: login, checkout, the nightly report job — the flows that make money — and I watch the golden signals: latency, error rate, and traffic, against the pre-migration baseline, not against a guess. I define rollback triggers in advance — error rate above X for five minutes, or the payment success rate dropping — so nobody has to make a judgement call at 2 a.m. Then hypercare: the team watches the system closely for a week, with the old environment still warm and the rollback path still working. My line: smoke tests on the money flows, golden signals against the baseline, rollback triggers agreed in advance, and a week of hypercare with the old environment still warm."

**Key Point:** "Smoke tests on the money flows, golden signals against the baseline, rollback triggers agreed in advance, and a week of hypercare with the old environment warm."

---

### Q12: Design it end to end — migrate a payments monolith (app + Postgres + background jobs) to AWS with under 5 minutes of downtime. What moves, in what order?

**How to Answer:**

"I'd walk it in waves, with the database leading and the app following. First, discovery: map the dependencies — the app, the Postgres database, the cron jobs, the payment gateway allowlist, and the reporting queries that read the database directly. Second, replatform the database: snapshot plus DMS with CDC into RDS Postgres, and let it catch up until lag is seconds — the app still points at the old database meanwhile. Third, stand up the new app on ECS behind an ALB, wired to the new RDS through IAM auth and Secrets Manager, and run it in shadow mode against a copy of production traffic to prove it. Fourth, cut over: lower the DNS TTL days ahead, flip traffic to the ALB during the lowest-traffic window, and the database switch is just pointing the app at RDS — the replication lag means the data is already there, so downtime is minutes, not hours. And the guardrails: the old database stays as the rollback target for a week, the payment gateway's allowlist already includes the new egress IPs, and the rollback trigger is the payment success rate. My line: database replicates first, app proves itself in shadow, traffic flips at the quiet hour, old database stays warm for a week as the rollback."

**Key Point:** "Database replicates first, app proves itself in shadow, traffic flips at the quiet hour, old database stays warm for a week as the rollback."
