# Database Reliability for DevOps Interview Preparation Guide

*How to Answer Database Reliability for DevOps Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Reliability Basics](#reliability-basics)
2. [Backups, Replication & Failover](#backups-replication--failover)
3. [Performance, Scaling & Incidents](#performance-scaling--incidents)

---

## Reliability Basics

### Q1: What does "database reliability" actually mean to a DevOps engineer?

**How to Answer:**

"I'd say it means the database is up, correct, and fast enough when the app needs it — not just that the process is running. Up means it accepts connections and answers queries; correct means no lost or corrupted writes, even after a crash; fast enough means queries finish inside the latency your users tolerate. I think of it in three numbers: availability, durability, and p99 latency. If any of the three slips, the app feels it, no matter how healthy the rest of the stack looks."

**Key Point:** "Database reliability is availability plus durability plus p99 latency — a running process that loses writes or crawls is still a failure."

---

### Q2: How do you pick between SQL and NoSQL for a new service?

**How to Answer:**

"I start from the data, not the hype. If the data is relational, needs joins, and correctness matters more than scale — orders, payments, user accounts — I pick Postgres or MySQL. If the shape is fluid, the access pattern is a single key lookup, and I need to scale writes horizontally — sessions, feeds, event logs — I look at a document or key-value store. The interview point is that the wrong choice shows up later as either painful migrations or a consistency model that fights your product. I'd also name the escape hatch: you can run both, but every extra database is a backup, a failover plan, and an on-call burden you now own."

**Key Point:** "Relational data with correctness needs goes SQL; fluid shape with single-key access at write scale goes NoSQL — and every extra database is another backup and on-call burden."

---

### Q3: What are ACID and BASE, and when does each matter?

**How to Answer:**

"ACID is the promise a relational database makes: a transaction either fully happens or fully doesn't, and once it commits, it survives a crash. BASE is the relaxed version distributed systems settle for: basically available, soft state, eventually consistent — you get an answer now, and the data catches up. I'd use ACID wherever money or inventory moves, because a half-applied payment is a real-world bug, not a theoretical one. I'd accept BASE where a stale read is harmless, like a like count or a recommendation feed, because the availability and scale are worth the delay."

**Key Point:** "ACID for money and inventory where a half-applied transaction is a real bug; BASE where a stale read is harmless and availability is worth the wait."

---

### Q4: What is a connection pool, and why does it matter more than most people think?

**How to Answer:**

"A connection pool is a fixed set of reusable database connections that the app shares, instead of opening a new one per request. Opening a connection is expensive — a handshake, authentication, memory on the database side — so a thousand requests a second without pooling is a thousand handshakes a second. The trap I always mention is pool size: too small and requests queue up waiting for a connection, too big and the database spends all its time context-switching between idle connections. My rule of thumb is a small pool per app instance — a few dozen, not hundreds — and I size it by measuring, not by guessing."

**Key Point:** "A pool reuses connections instead of paying a handshake per request — too small queues requests, too big drowns the database in idle connections."

---

## Backups, Replication & Failover

### Q5: How do you design a backup strategy you can actually trust?

**How to Answer:**

"I design it backwards from the restore, not forwards from the backup. First I set the targets: RPO, how much data I can afford to lose, and RTO, how fast I must be back. Then I pick the mechanics — nightly full backups plus continuous WAL or binlog archiving for Postgres and MySQL, so I can restore to any point in time, not just last night. The part most teams skip is the test: I restore into a staging environment on a schedule and time it, because a backup you've never restored is a hope, not a plan. And I keep a copy in a different region, because a backup in the same account that gets deleted is no backup at all."

**Key Point:** "Design from the restore: RPO/RTO first, full backup plus WAL/binlog for point-in-time recovery, and test restores on a schedule — an untested backup is a hope, not a plan."

---

### Q6: How does replication work, and what's the difference between sync and async?

**How to Answer:**

"Replication copies writes from a primary to one or more replicas, so reads can spread out and a replica can take over if the primary dies. In synchronous replication, the primary waits until at least one replica confirms the write before telling the app it succeeded — you lose nothing on failover, but every write pays the network round trip. In asynchronous replication, the primary answers immediately and the replica catches up a moment later — writes are fast, but a crash can lose the writes that hadn't arrived yet. I choose sync when losing a write is unacceptable and the replicas are close, async when latency matters and a few seconds of loss is survivable."

```sql
-- Postgres: check replication lag on the primary in seconds
SELECT client_addr, state, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS bytes_lag
FROM pg_stat_replication;
```

**Key Point:** "Sync replication confirms before answering — zero loss, slower writes; async answers first — faster writes, possible loss on crash."

---

### Q7: What happens during a failover, and what usually goes wrong?

**How to Answer:**

"Failover is the moment a replica gets promoted to primary and everything has to point at it: the app's connection string, the DNS name, the monitoring, the backup job. What goes wrong is rarely the promotion itself — it's the things around it. The old primary comes back and starts accepting writes, so you get a split brain with two primaries. Or the app's pool keeps trying the dead primary because the DNS TTL was an hour. Or the new primary has half the memory because it was sized as a replica. My checklist is: fence the old primary so it can never write again, keep DNS TTLs short on the database hostname, and size replicas exactly like the primary, because a replica is a primary waiting to happen."

**Key Point:** "Failover breaks on the edges, not the promotion — fence the old primary, keep DNS TTLs short, and size replicas like primaries."

---

### Q8: How do you run schema migrations without downtime?

**How to Answer:**

"I use expand-and-contract: every change happens in two deploys, never one. First I add the new column or table alongside the old one, and deploy code that writes to both and reads from the old. Then, once the data is backfilled, I deploy code that reads from the new one, and only then do I drop the old column. The classic mistakes are adding a NOT NULL column without a default, which locks the table while it rewrites every row, and renaming a column in one deploy, which breaks the old code still running. For big tables I use online tools like pg_repack or gh-ost that copy the table in the background instead of locking it. If a migration can't be done in two safe steps, it's not ready to run."

**Key Point:** "Expand-and-contract in two deploys — add, dual-write, backfill, switch reads, then drop — never a NOT NULL column without a default or a rename in one deploy."

---

## Performance, Scaling & Incidents

### Q9: How do you find and fix a slow query in production?

**How to Answer:**

"I start with the slow query log or pg_stat_statements, sorted by total time, not by one bad run — the query eating the most cumulative time is the one to fix. Then I run EXPLAIN ANALYZE on it and look for the tell-tale signs: a sequential scan on a big table, a nested loop over millions of rows, or a sort spilling to disk. Most of the time the fix is an index on the columns in the WHERE and JOIN, or rewriting the query so it stops scanning what it doesn't need. If the query is fine and the database is just saturated, the fix is connection or resource tuning, not another index. And I always verify after the fix — the same query, the same data, the time before and after."

```sql
-- Postgres: top queries by total time, the first place to look
SELECT query, calls, round(mean_exec_time::numeric, 2) AS mean_ms
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

**Key Point:** "Sort pg_stat_statements by total time, EXPLAIN ANALYZE the worst offender, fix the scan or the missing index, then verify with before-and-after numbers."

---

### Q10: When do you add a read replica, and when do you shard instead?

**How to Answer:**

"A read replica is for read-heavy workloads where most queries can tolerate a second of staleness — dashboards, reports, feed reads. It scales reads, but it does nothing for write load, and it does nothing for a single table that's too big to fit or back up comfortably. Sharding is for when the write load or the data size itself is the problem: you split the data across many databases by a key like tenant or user ID. The cost is real — cross-shard queries become painful, transactions across shards don't exist, and rebalancing is a project. So I add replicas first, and I only shard when the numbers prove one database can't hold the writes or the data."

**Key Point:** "Replicas scale stale-tolerant reads; sharding is for when write load or data size itself outgrows one database — and it costs you cross-shard queries and transactions."

---

### Q11: How do you handle a database that's running out of disk?

**How to Answer:**

"I treat it as an incident with a clock, because a full disk stops writes and then crashes the database. First I buy time: drop old WAL archives, clear temp files, and extend the volume if the cloud lets me do it live. Then I find the cause: a table that never gets vacuumed, an index bloat, or a log table growing without retention. The long fix is the boring one — autovacuum tuned so dead rows actually get reclaimed, retention policies on anything append-only, and a disk-usage alert at seventy percent, not ninety-five. The mistake I never repeat is waiting until the alert fires at ninety percent, because by then you're one big import away from an outage."

**Key Point:** "Buy time by clearing WAL and extending the volume, fix the cause — vacuum, bloat, retention — and alert at 70% disk, not 95%."

---

### Q12: Walk me through a real database incident and how you'd handle it.

**How to Answer:**

"I'd tell it as a story with numbers and a timeline. Say the primary's disk hit a hundred percent at 2 a.m. because autovacuum hadn't run in a week and dead tuples had doubled the table size. Writes started failing, the app's error rate jumped, and the on-call got paged. The fix in order: extend the volume to restore writes, run a manual VACUUM to reclaim space, then tune autovacuum so it keeps up with the write rate. After the incident, I'd add the guardrails — a disk alert at seventy percent and a weekly check on table bloat — and write the postmortem around the real lesson: the outage wasn't the disk filling up, it was that nobody was watching it fill up. Interviewers remember an incident told with a cause, a fix in order, and a guardrail at the end."

**Key Point:** "Restore writes first, reclaim space second, tune vacuum third — then add the 70% disk alert and bloat check so nobody is surprised by a filling disk again."

---
