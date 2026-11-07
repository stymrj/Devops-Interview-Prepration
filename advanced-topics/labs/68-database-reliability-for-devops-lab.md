# Database Reliability for DevOps — Hands-On Lab

Nine drills that turn database reliability answers into real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Measure your database's three reliability numbers

**Goal:** Availability, durability posture, and p99 latency written down for one real database.

**Commands:**

```bash
# Availability: can you connect and query?
pg_isready -h <host> -p 5432
# Latency: time a simple query from the app host
psql -h <host> -c "SELECT 1" -o /dev/null -t -c "\timing on"
# Durability posture: is WAL archiving on, and when was the last backup?
psql -h <host> -c "SHOW archive_mode; SHOW wal_level;"
```

**Expected output:** A yes/no on connectivity, a latency number in milliseconds, and a clear answer on archiving and last backup.

**Why it matters:** This is the Q1 starting point — you can't claim reliability until you can measure its three parts.

---

## Exercise 2: Prove the cost of an unpooled connection storm

**Goal:** See connection overhead with your own numbers, not just theory.

**Commands:**

```bash
# Open 100 short-lived connections in a loop and time it, then compare
# with 100 queries reused through one pooled connection (e.g. PgBouncer
# or your app's pool) — count the difference in total time and in
# pg_stat_activity rows created.
```

**Expected output:** Unpooled connections take far longer in total and spike the connection count; pooled queries stay flat.

**Why it matters:** This is Q4 made concrete — a handshake per request is a tax you pay on every query.

---

## Exercise 3: Run a backup and restore it somewhere else

**Goal:** A restore you've actually done, timed end to end.

**Commands:**

```bash
# Postgres: base backup + WAL, then restore into a fresh instance
pg_basebackup -h <primary> -D /tmp/restore-test -Ft -z -P
# Restore, start the instance, and time how long the app takes to connect
```

**Expected output:** A working restored database and a real RTO number — the minutes from "start restore" to "app can query".

**Why it matters:** This is Q5 — a backup you've never restored is a hope, not a plan.

---

## Exercise 4: Measure replication lag under load

**Goal:** A lag number you can defend, not a guess.

**Commands:**

```sql
-- On the primary, while generating writes:
SELECT client_addr, state, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS bytes_lag
FROM pg_stat_replication;
```

**Expected output:** Lag in bytes that stays small under normal load and grows visibly under a write burst — and a sense of how big "small" is for your system.

**Why it matters:** This is Q6 — sync vs async is a trade-off you can only make with a lag number in hand.

---

## Exercise 5: Practise a failover, including the fencing step

**Goal:** A failover you've done on purpose, with the old primary fenced.

**Commands:**

```bash
# Promote a replica, point the app at it, then verify the old primary
# cannot accept writes: revoke its write access or stop it entirely,
# and confirm the app only talks to the new primary.
```

**Expected output:** The app works against the new primary, and a write attempt against the old primary fails cleanly.

**Why it matters:** This is Q7 — the promotion is easy, the fencing and the DNS switch are where failovers actually break.

---

## Exercise 6: Do an expand-and-contract migration on a live table

**Goal:** A schema change with zero downtime and zero broken reads.

**Commands:**

```sql
-- Step 1: add the new column (nullable, with a default if needed)
ALTER TABLE users ADD COLUMN display_name text;
-- Step 2 (after dual-write + backfill): switch reads, then drop the old column
ALTER TABLE users DROP COLUMN nickname;
```

**Expected output:** Old code keeps working through step 1, new code reads the new column after step 2, and the drop is a non-event.

**Why it matters:** This is Q8 — two safe deploys beat one clever deploy every time.

---

## Exercise 7: Find the worst query and fix it with EXPLAIN ANALYZE

**Goal:** One query, measurably faster, with before-and-after numbers.

**Commands:**

```sql
-- Top queries by total time
SELECT query, calls, round(mean_exec_time::numeric, 2) AS mean_ms
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
-- Then EXPLAIN ANALYZE the worst one, add the missing index, and re-run
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42 AND created_at > now() - interval '7 days';
```

**Expected output:** A sequential scan replaced by an index scan, and a mean time that drops by an order of magnitude or more.

**Why it matters:** This is Q9 — sort by total time, explain the worst offender, verify with numbers.

---

## Exercise 8: Decide replica vs shard with real numbers

**Goal:** A scaling decision you can defend with a measurement, not a feeling.

**Commands:**

```bash
# Measure your read/write ratio and your largest table size:
# - If reads dominate and staleness is tolerable -> read replica
# - If writes dominate or one table is too big to back up -> shard
# Write the numbers down: reads/sec, writes/sec, largest table in GB
```

**Expected output:** A one-paragraph decision with the numbers that justify it, and the cost of the alternative named.

**Why it matters:** This is Q10 — replicas first, sharding only when the numbers prove one database can't hold it.

---

## Exercise 9: Set a disk alert at 70% and check table bloat

**Goal:** A guardrail that fires before the disk does.

**Commands:**

```sql
-- Find your most bloated tables (dead tuples vs live tuples)
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

**Expected output:** A disk-usage alert threshold at 70% and a bloat list with one table worth vacuuming first.

**Why it matters:** This is Q11 and the Q12 guardrail — the outage isn't the disk filling up, it's nobody watching it fill up.

---
