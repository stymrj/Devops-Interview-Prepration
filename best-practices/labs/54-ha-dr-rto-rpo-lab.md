# Hands-On Lab: High Availability & Disaster Recovery

*10 exercises that make RTO/RPO real. Run these on any VM or cloud sandbox — nothing here touches production.*

---

## Exercise 1: Measure Your Own RTO/RPO Baseline

**Goal:** Turn two abstract acronyms into numbers for a real service.

**Commands:** None — grab a notebook.

1. Pick one service you own.
2. Write down: "If this broke right now, how many minutes of lost data can we accept?" → that's your RPO.
3. Write down: "How many minutes of downtime before someone important calls me?" → that's your RTO.

**Expected output:** Two numbers, e.g. RPO = 15 min, RTO = 60 min.

**Why it matters:** You can't design DR until the business gives you targets. Every exercise below is about closing the gap between what you *have* and these two numbers.

---

## Exercise 2: Time a Real Backup Restore

**Goal:** Find out whether your backups actually meet your RPO.

**Commands:**

```bash
# Pick a test database, back it up, then time a full restore
pg_dump -h localhost -U postgres mydb > /tmp/mydb.sql
time psql -h localhost -U postgres -c "DROP DATABASE mydb_test;" -c "CREATE DATABASE mydb_test;"
time psql -h localhost -U postgres mydb_test < /tmp/mydb.sql
```

**Expected output:** The `time` output tells you your true restore time. Compare it against the RTO you wrote down in Exercise 1.

**Why it matters:** Most teams learn here that "nightly backups" plus a 2-hour restore means their real RTO is 26 hours, not 2. Restore time is part of the RTO.

---

## Exercise 3: Simulate an AZ Failure Locally

**Goal:** Feel what an AZ outage does to a running app.

**Commands:**

```bash
# Start a simple service on two "zones" (two containers)
docker run -d --name web-a -p 8081:80 nginx
docker run -d --name web-b -p 8082:80 nginx

# Verify both serve traffic
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8081
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8082

# Kill "AZ-a"
docker stop web-a
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8081  # fails
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8082  # still 200
```

**Expected output:** One endpoint fails, the other keeps serving — that's AZ isolation in miniature.

**Why it matters:** Multi-AZ only helps if traffic actually routes to the survivors. A load balancer in front of both containers is the missing half of this exercise — add one and re-run.

---

## Exercise 4: Test a Database Failover

**Goal:** Promote a replica and confirm the app follows it.

**Commands:**

```bash
# With a Postgres primary + replica running:
# 1. On the replica, promote it:
pg_ctl promote -D /var/lib/postgresql/replica

# 2. Point your app at the new primary (update connection string / DNS)
# 3. Write a row and read it back:
psql -h <new-primary> -U postgres mydb -c "INSERT INTO t VALUES (1); SELECT * FROM t;"
```

**Expected output:** The insert lands on the promoted replica. The app writes work with zero code changes after the connection switch.

**Why it matters:** The replica is the easy part. The failover path — health check, promotion, connection redirection — is the part that fails at 3 AM.

---

## Exercise 5: Find the TTL That Would Kill Your Failover

**Goal:** Discover whether DNS will betray your DR plan.

**Commands:**

```bash
dig +short your-service.example.com
dig your-service.example.com | grep -i ttl
```

**Expected output:** If the TTL is 86400 (24 hours), half your users won't reach the failover region until tomorrow. Healthy DR needs TTLs in the 60–300s range (or better, health-checked routing like Route 53).

**Why it matters:** DNS TTL is the most common silent DR killer. Fix it before the outage, not during.

---

## Exercise 6: Run a Tabletop DR Drill

**Goal:** Walk the runbook with your team and find the lies in it.

**Commands:** Book 45 minutes. No code.

1. Announce: "Primary region is down. Go."
2. Read the runbook step by step, out loud.
3. Mark every step that's outdated, ambiguous, or names someone who left.

**Expected output:** A list of 5–15 broken steps. That's normal — that's the point.

**Why it matters:** Every broken step you find in a calm room is one you won't find at 3 AM with customers down.

---

## Exercise 7: Measure Replication Lag

**Goal:** Know your *actual* RPO, not the one on the slide.

**Commands:**

```bash
# Postgres: check how far the replica lags the primary
psql -h <replica> -U postgres -c "SELECT now() - pg_last_xact_replay_timestamp() AS lag;"
```

**Expected output:** A lag number like `00:00:02`. If it says 20 minutes, your real RPO is 20 minutes regardless of what the architecture diagram claims.

**Why it matters:** Replication lag is the honest RPO. Monitor it as an alert, not a dashboard you glance at quarterly.

---

## Exercise 8: Practice a Restore Into a Fresh Environment

**Goal:** Prove your backups work *somewhere else*, not just on the same box.

**Commands:**

```bash
# Spin up a fresh VM/container, then restore there:
scp /tmp/mydb.sql fresh-box:/tmp/
ssh fresh-box "psql -U postgres -c 'CREATE DATABASE restored;' && psql -U postgres restored < /tmp/mydb.sql"
ssh fresh-box "psql -U postgres restored -c 'SELECT count(*) FROM important_table;'"
```

**Expected output:** Row counts match the source. The backup is portable.

**Why it matters:** A backup that only restores onto the machine that made it is a false sense of security. DR means rebuilding *elsewhere*.

---

## Exercise 9: Calculate Your "Nines" Honestly

**Goal:** See what your real availability has been.

**Commands:**

```bash
# Pull last 90 days of uptime from your monitoring (example: Prometheus)
# uptime_ratio = sum(up[90d]) / (90 * 24 * 60)
# Or simply: total downtime minutes in 90 days × 4.05 = projected yearly downtime
```

**Expected output:** Something like "99.4% — about 52 hours/year projected." Compare with what you promised.

**Why it matters:** Availability you haven't measured is marketing. This number tells you whether to invest in HA or stop apologizing.

---

## Exercise 10: Write the One-Page Runbook

**Goal:** Leave this lab with the document you'd actually grab in an outage.

**Commands:** Open an editor. One page max.

1. Who declares the incident and how (names + phone numbers).
2. The exact failover commands, in order, copy-paste ready.
3. How to verify real traffic is serving (not just "dashboard green").
4. Who updates the status page, and the template message.

**Expected output:** A single page you could hand to an on-call engineer who's never done a failover.

**Why it matters:** In a real region outage, nobody reads a 40-page DR plan. They read one page. Make it the right one.
