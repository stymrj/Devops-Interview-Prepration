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

