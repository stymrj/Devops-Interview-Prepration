# Disaster Recovery & Business Continuity — Hands-On Lab

Nine drills that turn disaster-recovery answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Separate HA from DR for one real system

**Goal:** A written line between what HA covers and what DR covers for a system you actually run.

**Steps:**

```bash
# Pick one real system (yours, or a familiar one like a checkout service).
# List the failures HA handles: a node dies, a zone goes down, a bad pod
# gets restarted. Then list the failures only DR handles: the whole
# region is gone, the database is corrupted, a bad deploy takes all
# replicas out at once. Write the line in one sentence: "HA keeps us up
# through a bad day, DR brings us back from a bad week" — with your
# system's real examples under each half.
```

**Expected output:** One page with two failure lists and a one-sentence HA-vs-DR line grounded in the system's real failure modes.

**Why it matters:** This is Q1 made concrete — "three replicas in one region is HA, not DR" only counts when you can name the failure your setup doesn't survive.

---

## Exercise 2: Set real RTO and RPO numbers with the business

**Goal:** RTO/RPO numbers tied to money, not copied from a spec sheet.

**Steps:**

```bash
# Pick two systems: one that earns money per minute (checkout, payments)
# and one that doesn't (internal wiki, staging). For each, answer: what
# does one hour of downtime cost — in revenue, in fines, in trust?
# Write the RTO and RPO that follow from the answer, and which DR tier
# (backup-restore, pilot light, warm standby, active-active) each number
# forces. Get the numbers accepted in writing — even if it's just an
# email from the team lead — because the negotiation is the skill.
```

**Expected output:** Two systems with costed downtime, an RTO/RPO each, the DR tier each forces, and the acceptance in writing.

**Why it matters:** This is Q2 — the numbers are a negotiation, and this drill is the first time you run it.

---

## Exercise 3: Price the four DR patterns for one workload

**Goal:** A one-page table with RTO, RPO, and monthly cost for all four patterns.

**Steps:**

```bash
# Take one real workload and price all four patterns: backup and restore,
# pilot light, warm standby, active-active. For each, write the RTO, the
# RPO, and the monthly cost — including the second region's compute, the
# replication traffic, and the storage. The active-active row must include
# the cost of solving cross-region writes. End with the row you'd actually
# pick, and the sentence you'd use to sell it to the business.
```

**Expected output:** A four-row table (RTO, RPO, monthly cost) plus the picked row and the one-sentence pitch for it.

**Why it matters:** This is Q3 — "the business picks the row, you build it" only works when you can show all four rows with real numbers.

---

## Exercise 4: Back up the system, not just the data

**Goal:** A backup that includes the state and config, with an immutable copy in a second region.

**Steps:**

```bash
# List everything a restore needs beyond the database: Terraform state,
# etcd snapshots, Vault unseal keys, Helm values, DNS records, IAM
# policies, CI secrets. Back up each one on a schedule — etcd snapshot
# plus database dump plus object-storage versioning — and copy it to a
# second region with object lock (immutability) turned on, so ransomware
# can't encrypt the backups too. Now try the bad version: restore from
# the database dump alone, and write down everything that's missing.
```

**Expected output:** A scheduled backup covering data + state + config, an immutable second-region copy, and the list of what's missing when you restore from the database alone.

**Why it matters:** This is Q4 — "back up the system, not just the data" becomes real the moment a database-only restore leaves you with an app that can't start.

---

## Exercise 5: Restore into a clean environment and time it

**Goal:** A tested restore with a measured recovery time, not a green tick.

**Steps:**

```bash
# Spin up an isolated VPC (or a fresh kind cluster + fresh database).
# Restore last night's backup into it, point the app at the restored
# database, and run the smoke tests. Time the whole thing from "start
# restore" to "users could use it" — that's your real RTO, and it is
# almost never the number on the slide. Write down what broke and what
# was missing, and fix one of them. Repeat monthly.
```

**Expected output:** A working restore in a clean environment, a measured RTO in minutes, and a written list of what broke and what you fixed.

**Why it matters:** This is Q5 — the restore test is the only proof a backup works, and this drill is the proof behind the line.

---

## Exercise 6: Measure replication lag and answer "what did we lose?"

**Goal:** A real lag number and a written data-loss answer for a failover.

**Steps:**

```bash
# Set up async replication (Postgres streaming replica, or a managed
# replica in a second region) and watch the lag under load — check
# pg_stat_replication on the primary, or the managed replica lag metric.
# Write a burst of rows, kill the primary, and count how many of the
# burst are missing on the replica: that's your real RPO in rows and
# seconds. Now write the failover announcement you'd send: "we lost N
# seconds of writes, here's what they were" — because that sentence is
# what the business actually needs in a disaster.
```

**Expected output:** A measured replication lag, the rows lost in a real failover test, and the written data-loss announcement.

**Why it matters:** This is Q6 — "that's your real RPO" only lands when you've counted the missing rows yourself.

---

## Exercise 7: Run a traffic flip and measure the real RTO

**Goal:** A tested failover with a measured recovery time, including the DNS caching delay.

**Steps:**

```bash
# Set up a health check on /healthz of a real service, with a failover
# routing policy: PRIMARY in one region, SECONDARY in another, TTL 60s.
# Break the primary (stop the app, not the whole region) and time the
# flip: when did the health check fire, when did traffic move, and when
# was the service actually usable? Repeat with the health check pointed
# at the load balancer only — watch the flip fail to fire while the app
# is down. Write both numbers down.
```

**Expected output:** A working traffic flip, a measured RTO in minutes, and the comparison showing why the check must test the app endpoint.

**Why it matters:** This is Q7 — "the flip is a test, not a setting" is the line, and this drill is the test that turns it into a number.

---

## Exercise 8: Turn the runbook into one command

**Goal:** A recovery that runs from one command, with a runbook that stays true.

**Steps:**

```bash
# Write the runbook first: one page, numbered steps, exact commands,
# who runs each one. Then automate it: Terraform for the infrastructure,
# Helm or GitOps for the apps, a script for the database restore — until
# the recovery is one command that flips DNS, scales up the secondary,
# and pages the team. Run the command in a drill, and fix the runbook
# everywhere it lied. The test that matters: could someone who has never
# done this before run it at 3 AM?
```

**Expected output:** A one-page runbook, a one-command failover, and a drill run that fixed every place the runbook lied.

**Why it matters:** This is Q8 — "DR you can run from one command is a capability" is the line, and this drill builds the capability.

---

## Exercise 9: Run a tabletop, then a game day

**Goal:** Two drills — one that finds runbook gaps, one that finds real bugs.

**Steps:**

```bash
# Tabletop (one hour, in a room): "the region is down, what do we do?"
# Walk the runbook step by step and write down every gap — the missing
# phone number, the step nobody owns, the approval that needs someone
# asleep. Fix them. Then the game day: actually kill the primary, flip
# the traffic, and time it. Write down every technical bug the game day
# found that the tabletop couldn't — the health check that never fired,
# the backup that didn't restore, the secondary on an old schema.
```

**Expected output:** A tabletop gap list (fixed), a game-day run with a measured RTO, and the list of technical bugs only the game day could find.

**Why it matters:** This is Q9 — the tabletop finds the runbook gaps and the game day finds the real bugs, and this drill runs both.
