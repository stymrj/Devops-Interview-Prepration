# RDS & Database Decisions on AWS Interview Preparation Guide

*How to Answer RDS Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [RDS Basics and Engine Choices](#rds-basics-and-engine-choices)
2. [High Availability with Multi-AZ](#high-availability-with-multi-az)
3. [Read Replicas and Scaling Reads](#read-replicas-and-scaling-reads)
4. [Backups Snapshots and Recovery](#backups-snapshots-and-recovery)
5. [Aurora When and Why](#aurora-when-and-why)
6. [Performance Tuning and RDS Proxy](#performance-tuning-and-rds-proxy)
7. [Security and Networking](#security-and-networking)
8. [Database Decisions on AWS](#database-decisions-on-aws)
9. [Interview Traps and Real Scenarios](#interview-traps-and-real-scenarios)

---

## RDS Basics and Engine Choices

### Q1: What is RDS, and why would I use it instead of running Postgres on an EC2 box?

**How to Answer:**

"RDS is managed relational database service — AWS handles patching, automated backups, failover, and OS-level maintenance. When I run Postgres on EC2, I'm the DBA: I own patching, backup scripts, replication setup, and failover logic. With RDS I get automated backups with point-in-time recovery and Multi-AZ failover out of the box, so my team focuses on schema and queries instead of database plumbing. For most production workloads, the managed service wins on reliability and engineer time."

**Key Point:** "RDS trades EC2 control for managed patching, backups, and automatic failover — fewer 3 AM database incidents."

### Q2: Which engines does RDS support, and how do I pick one in an interview?

**How to Answer:**

"RDS supports MySQL, PostgreSQL, MariaDB, Oracle, and SQL Server, plus Aurora variants of MySQL and Postgres. In startups and typical DevOps interviews, the answer is almost always Postgres or MySQL — Postgres when I want richer features like JSONB, extensions, and better concurrency, MySQL when the app ecosystem already speaks it. Oracle and SQL Server only come up when there's an existing enterprise license to honor. If the interviewer asks me to choose, I ask about the workload first: OLTP, analytics, or legacy app."

**Key Point:** "Default to Postgres or MySQL; reach for Oracle/SQL Server only for existing enterprise licensing."

---

## High Availability with Multi-AZ

### Q3: What does Multi-AZ actually do under the hood?

**How to Answer:**

"Multi-AZ keeps a synchronous standby replica of my database in a different availability zone. Every write to the primary is replicated synchronously, so on failure AWS automatically fails over to the standby — usually in about a minute or two, with the endpoint DNS flipping. My app never changes connection strings because the RDS endpoint stays the same. The trap interviewers love: the standby is not readable — it's purely for failover, not for scaling reads."

**Key Point:** "Multi-AZ is a synchronous, unreadable standby in another AZ — automatic failover with the same endpoint."

### Q4: Multi-AZ DB instance vs Multi-AZ DB cluster — what's the difference?

**How to Answer:**

"A Multi-AZ DB instance has one primary writer and one standby that's invisible to me. A Multi-AZ DB cluster has one writer plus two readable standbys, and it fails over much faster — typically under 35 seconds. I reach for the cluster when the app needs both high availability and read scaling from the same deployment. It costs more, so for most apps the plain Multi-AZ instance is enough. If the interviewer asks about sub-minute RTO, the cluster is the answer."

**Key Point:** "DB instance: one hidden standby. DB cluster: writer plus two readable standbys with faster failover."

---

## Read Replicas and Scaling Reads

### Q5: How do read replicas work, and when do I add one?

**How to Answer:**

"Read replicas are asynchronous copies of the primary that serve read traffic — I point reporting queries, dashboards, and heavy SELECT workloads at them instead of the primary. Replication lag is usually under a second on a healthy instance, but it's not zero, so I never read-after-write from a replica when freshness matters. I add one when the primary's CPU is dominated by reads or when a nightly report is slowing down the app. You can also promote a replica to standalone if the primary is lost."

**Key Point:** "Replicas offload reads asynchronously — great for reports and dashboards, bad for read-after-write freshness."

### Q6: Can I fail over to a read replica? What's the difference from Multi-AZ failover?

**How to Answer:**

"Yes, but it's manual — I promote the replica, which breaks replication and makes it a standalone primary. Multi-AZ failover is automatic and keeps the same endpoint; replica promotion changes the endpoint, so my app config has to update. That's exactly why I don't use read replicas as my HA strategy: they're for scaling reads, Multi-AZ is for surviving failures. Aurora blurs this line with its shared-storage replicas that can fail over automatically, which is one reason teams pick Aurora."

**Key Point:** "Replica promotion is manual and changes the endpoint — Multi-AZ is automatic HA, replicas are read scaling."

```bash
# Promote a read replica to a standalone primary (one-way, breaks replication)
aws rds promote-read-replica --db-instance-identifier prod-replica-1
```

---

## Backups Snapshots and Recovery

### Q7: Walk me through RDS backups — automated backups vs snapshots.

**How to Answer:**

"Automated backups run daily in the backup window and keep transaction logs, so I can restore to any point in time within the retention period — up to 35 days. Manual snapshots are point-in-time copies I take myself, and they stick around until I delete them, which makes them perfect for pre-deploy safety nets. Restoring always creates a brand-new DB instance — AWS never overwrites in place. That matters in interviews because the new instance gets a new endpoint, so I plan for config updates and DNS changes after a restore."

**Key Point:** "Automated backups give point-in-time recovery; manual snapshots are forever until deleted; restores always create a new instance."

### Q8: How do you design a backup strategy that actually survives a bad day?

**How to Answer:**

"I set automated backup retention to 30 days, take a manual snapshot before every migration or major deploy, and copy snapshots to a second region for disaster recovery. I test restores quarterly — an untested backup is just a hope. I also keep the backup window in off-peak hours and make sure the instance isn't in a state where snapshots stall, like storage-full. Interviewers love asking about RTO and RPO: my RPO here is five minutes thanks to transaction logs, and my RTO is however long a restore takes."

```bash
# Manual snapshot before a risky migration
aws rds create-db-snapshot \
  --db-instance-identifier prod-db \
  --db-snapshot-identifier prod-db-pre-migration-$(date +%F)

# Copy it cross-region for DR
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier prod-db-pre-migration-2026-10-14 \
  --target-db-snapshot-identifier prod-db-pre-migration-2026-10-14 \
  --source-region ap-south-1 --region ap-south-2
```

**Key Point:** "30-day automated retention, manual snapshots before migrations, cross-region copies, and tested restores."

---

## Aurora When and Why

### Q9: What is Aurora, and when does it beat plain RDS?

**How to Answer:**

"Aurora is AWS's cloud-native relational engine, compatible with MySQL and Postgres, with storage that automatically grows to 128 TB and replicates across three AZs six ways. It beats plain RDS on throughput — up to 5x MySQL and 3x Postgres in AWS's numbers — and failover takes seconds because replicas share the same underlying storage. I pick Aurora when the database is the bottleneck: high-throughput OLTP, global apps, or when I need fast cloning for dev environments. For a small internal app, plain RDS is cheaper and simpler."

**Key Point:** "Aurora is cloud-native MySQL/Postgres with shared storage, faster failover, and auto-scaling storage — pick it when the DB is the bottleneck."

### Q10: How does Aurora storage actually work? Why is it different?

**How to Answer:**

"Aurora separates compute from storage — my writer and up to 15 readers all talk to the same distributed storage layer, so adding a reader doesn't copy data. Storage is replicated six ways across three AZs and self-heals, and it grows in 10 GB chunks automatically with no provisioning. Backups are continuous and incremental against that shared storage, so they don't slow down the database. This is the 'why' behind Aurora's fast failover: a reader can be promoted in seconds because there's no data to copy."

**Key Point:** "Shared distributed storage across AZs: no data copying for replicas, seconds-fast failover, automatic growth."

---

## Performance Tuning and RDS Proxy

### Q11: A query is slow on RDS — what's your debugging order?

**How to Answer:**

"First I check Performance Insights to see whether the bottleneck is CPU, I/O, or waiting on locks — I don't guess. Then I look at the slow query log and `EXPLAIN ANALYZE` the top offenders for missing indexes or full table scans. After that I check the parameter group: things like `max_connections`, work memory, and buffer pool sizes for the instance class. Only then do I consider scaling up or adding a read replica, because throwing hardware at a missing index is the classic expensive mistake."

**Key Point:** "Performance Insights first, slow queries second, parameter group third — scale hardware last, not first."

```bash
# Pull the slow query log from RDS to a file for analysis
aws rds download-db-log-file-portion \
  --db-instance-identifier prod-db \
  --log-file-name slowquery/mysql-slowquery.log \
  --starting-token 0 --output text > slow.log
```

### Q12: What problem does RDS Proxy solve?

**How to Answer:**

"RDS Proxy is a connection pooler that sits between my app and the database. Serverless apps and auto-scaled services open thousands of short-lived connections, and each one costs the DB memory and CPU — Postgres especially suffers past a few hundred connections. The proxy pools and reuses connections, and during a failover it holds app connections while rerouting to the new primary, which cuts failover downtime for the app dramatically. I reach for it with Lambda-heavy architectures or any app where connection counts spike unpredictably."

**Key Point:** "RDS Proxy pools database connections and smooths failovers — essential for Lambda and auto-scaled apps."

```yaml
# Terraform: attach RDS Proxy to cut connection churn
resource "aws_db_proxy" "app" {
  name                   = "app-db-proxy"
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800
  require_tls            = true
  vpc_subnet_ids         = var.private_subnet_ids
}
```

---

## Security and Networking

### Q13: How do you lock down an RDS instance properly?

**How to Answer:**

"I put RDS in private subnets with no public access, and the security group only allows port 5432 or 3306 from the app's security group — never from 0.0.0.0. Credentials live in Secrets Manager with automatic rotation instead of hardcoded in env files, and I enable IAM database authentication where the app supports it. Storage encryption is on with a KMS key I control, plus encryption in transit with TLS. The interview trap: storage encryption is set at creation and can't be turned on later for an existing instance — I'd have to snapshot, copy with encryption, and restore."

**Key Point:** "Private subnets, SG-scoped access, Secrets Manager rotation, KMS encryption at rest, TLS in transit — encryption is decided at creation."

```bash
# Rotate the master password via Secrets Manager instead of touching the app
aws secretsmanager rotate-secret --secret-id prod/rds/master-password
```

---

## Database Decisions on AWS

### Q14: RDS vs Aurora vs DynamoDB vs self-hosted on EC2 — how do you decide?

**How to Answer:**

"I start from the data model and access pattern, not the service list. Relational with joins, transactions, and complex queries means RDS or Aurora — RDS for standard workloads, Aurora when I need scale, fast failover, or global distribution. Key-value or document access with single-digit millisecond latency at any scale means DynamoDB, especially serverless apps. Self-hosted on EC2 only wins when I need an engine AWS doesn't offer, a specific extension, or full control over the OS — and I price in the on-call cost honestly. When an interviewer asks me to 'just pick one,' I walk through this decision tree out loud."

**Key Point:** "Relational with joins: RDS or Aurora. Key-value at massive scale: DynamoDB. Self-hosted only for engines or extensions AWS doesn't offer."

---

## Interview Traps and Real Scenarios

### Q15: Your primary RDS instance just failed over at 2 AM — what do you check in the morning?

**How to Answer:**

"First I confirm the app recovered: check that the endpoint DNS flipped, connections re-established, and there are no lingering stale connections pointing at the old primary. Then I look at RDS events and CloudWatch to find the trigger — was it an AZ issue, storage full, or a crash? I verify the new primary's replication health and make sure a fresh standby got provisioned, because right after failover you're running without a safety net until AWS rebuilds it. Finally I write it up: timeline, root cause, and whether our RTO held. Interviewers want the boring operational discipline here, not heroics."

**Key Point:** "Verify app recovery, find the trigger in events and metrics, confirm a new standby exists, then write the timeline up."

---

*Day 42 of 58 — next: Load Balancing & Autoscaling (ALB, ASG).*
