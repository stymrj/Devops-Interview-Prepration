# Database Reliability for DevOps — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Reliability | Availability + durability + p99 latency |
| RPO | How much data you can afford to lose |
| RTO | How fast you must be back |
| ACID | All-or-nothing transactions that survive a crash |
| BASE | Basically available, eventually consistent |
| Connection pool | Fixed set of reusable connections, no handshake per request |
| WAL / binlog | The write-ahead log that makes point-in-time recovery possible |
| Replication lag | How far behind the primary a replica is, in bytes or seconds |
| Sync replication | Confirm on a replica before answering — zero loss, slower writes |
| Async replication | Answer first, replica catches up — fast writes, possible loss |
| Failover | Promoting a replica to primary and repointing everything |
| Fencing | Making sure the old primary can never write again |
| Expand-and-contract | Add, dual-write, backfill, switch reads, then drop |
| pg_stat_statements | Rank queries by total time — the first place to look |
| Read replica | Scales stale-tolerant reads, does nothing for writes |
| Sharding | Splitting data across databases by a key, for write or size limits |

## SQL vs NoSQL, at a glance

| Signal | Pick |
|---|---|
| Relational data, joins, correctness matters | Postgres / MySQL |
| Fluid shape, single-key access, write scale | Document / key-value store |
| Money, inventory, half-applied = real bug | ACID (SQL) |
| Stale read harmless (like counts, feeds) | BASE is fine |

## Backup strategy checklist

- [ ] RPO and RTO written down before picking tools
- [ ] Nightly full backup + continuous WAL/binlog archiving
- [ ] Restore tested into staging on a schedule, timed end to end
- [ ] A copy in a different region / account
- [ ] Disk alert at 70%, not 95%

## Replication: sync vs async

| | Sync | Async |
|---|---|---|
| Write latency | Pays a network round trip | Fast, answers immediately |
| Loss on primary crash | Zero (confirmed on replica) | Writes not yet replicated can be lost |
| Use when | Losing a write is unacceptable, replicas close | Latency matters, seconds of loss survivable |

## Failover checklist

1. Promote the replica
2. Fence the old primary — it must never write again
3. Repoint the app (short DNS TTL on the database hostname)
4. Repoint monitoring and backup jobs
5. Replicas sized exactly like the primary — a replica is a primary waiting to happen

## Expand-and-contract migration

1. Add new column/table alongside the old one
2. Deploy code that writes to both, reads from the old
3. Backfill existing rows
4. Deploy code that reads from the new one
5. Drop the old column

Never: a NOT NULL column without a default, or a rename in one deploy. Big tables: pg_repack or gh-ost.

## Slow query fix order

1. `pg_stat_statements` sorted by total time — find the worst query
2. `EXPLAIN ANALYZE` — look for sequential scans, nested loops over millions of rows, sorts spilling to disk
3. Add the missing index (WHERE + JOIN columns) or rewrite the query
4. Verify with before-and-after numbers

## Replica vs shard

| Signal | Answer |
|---|---|
| Read-heavy, staleness tolerable | Read replica |
| Write load outgrows one database | Shard |
| One table too big to back up | Shard |
| Cost of sharding | No cross-shard transactions, painful cross-shard queries, rebalancing is a project |

## Disk-full incident, in order

1. Buy time: clear WAL archives, extend the volume live
2. Find the cause: vacuum, bloat, retention
3. Fix: tune autovacuum, add retention on append-only tables
4. Guardrail: disk alert at 70% + weekly bloat check
