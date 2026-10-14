# RDS Hands-On Lab — Backups, Replicas, Failover

9 exercises to build real muscle memory on RDS. Run in `ap-south-1` (or your region) inside the **free-tier window** where noted, and **delete everything at the end** — RDS instances bill by the hour even idle. Use `db.t3.micro` with 20 GB gp3 storage.

## Setup

```bash
export REGION="ap-south-1"
export PASS="$(openssl rand -base64 18)"
aws rds create-db-instance \
  --db-instance-identifier rds-lab \
  --db-instance-class db.t3.micro \
  --engine postgres --engine-version 16.3 \
  --allocated-storage 20 --storage-type gp3 \
  --master-username labadmin --master-user-password "$PASS" \
  --backup-retention-period 7 \
  --no-multi-az --publicly-accessible \
  --region $REGION
aws rds wait db-instance-available --db-instance-identifier rds-lab --region $REGION
```

---

## Exercise 1 — Inspect the instance and find its endpoint

**Goal:** Locate the connection endpoint and confirm the config you asked for.

```bash
aws rds describe-db-instances --db-instance-identifier rds-lab --region $REGION \
  --query 'DBInstances[0].[Endpoint.Address,DBInstanceStatus,MultiAZ,BackupRetentionPeriod]'
```

**Expected:** Status `available`, endpoint like `rds-lab.xxxx.ap-south-1.rds.amazonaws.com`, `MultiAZ=false`, retention `7`.

**Why it matters:** The endpoint is what your app connects to — it never changes on reboot, only on restore/failover scenarios you need to plan for.

---

## Exercise 2 — Connect and create real data

**Goal:** Prove the database is reachable and writable from your machine.

```bash
ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier rds-lab --region $REGION \
  --query 'DBInstances[0].Endpoint.Address' --output text)
PGPASSWORD="$PASS" psql -h $ENDPOINT -U labadmin -d postgres \
  -c "CREATE TABLE lab_orders(id serial primary key, item text); INSERT INTO lab_orders(item) VALUES ('keyboard');"
PGPASSWORD="$PASS" psql -h $ENDPOINT -U labadmin -d postgres -c "SELECT * FROM lab_orders;"
```

**Expected:** `INSERT 0 1` then one row returned.

**Why it matters:** Publicly-accessible labs let you connect directly; in production this box sits in a private subnet and only the app SG reaches it.

---

## Exercise 3 — Take a manual snapshot (pre-deploy safety net)

**Goal:** Snapshot before a "risky migration", then simulate data loss.

```bash
aws rds create-db-snapshot --db-instance-identifier rds-lab \
  --db-snapshot-identifier rds-lab-pre-migration --region $REGION
aws rds wait db-snapshot-available --db-snapshot-identifier rds-lab-pre-migration --region $REGION
PGPASSWORD="$PASS" psql -h $ENDPOINT -U labadmin -d postgres -c "DROP TABLE lab_orders;"
```

**Expected:** Snapshot reaches `available`; the table is gone after the DROP.

**Why it matters:** Manual snapshots persist until you delete them — this is the habit that saves you before every migration.

---

## Exercise 4 — Restore from the snapshot (new instance, new endpoint)

**Goal:** Recover the dropped table and observe that restores never happen in place.

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier rds-lab-restored \
  --db-snapshot-identifier rds-lab-pre-migration --region $REGION
aws rds wait db-instance-available --db-instance-identifier rds-lab-restored --region $REGION
aws rds describe-db-instances --db-instance-identifier rds-lab-restored --region $REGION \
  --query 'DBInstances[0].Endpoint.Address'
```

**Expected:** A *different* endpoint than `rds-lab`; connecting shows `lab_orders` back with the keyboard row.

**Why it matters:** Restores always create a new instance with a new endpoint — your runbook must include updating app config/DNS after a restore.

---

## Exercise 5 — Create a read replica and watch lag

**Goal:** Offload reads and measure replication lag.

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier rds-lab-replica \
  --source-db-instance-identifier rds-lab \
  --db-instance-class db.t3.micro --region $REGION
aws rds wait db-instance-available --db-instance-identifier rds-lab-replica --region $REGION
aws rds describe-db-instances --db-instance-identifier rds-lab-replica --region $REGION \
  --query 'DBInstances[0].ReadReplicaLag // DBInstances[0].Status'
```

**Expected:** Replica becomes `available`; lag metric appears in CloudWatch (`ReplicaLag`), usually under a second.

**Why it matters:** Replicas are async — lag is real, so read-after-write from a replica is a classic production bug.

---

## Exercise 6 — Promote the replica (manual failover drill)

**Goal:** See what a manual failover actually changes.

```bash
aws rds promote-read-replica --db-instance-identifier rds-lab-replica --region $REGION
aws rds wait db-instance-available --db-instance-identifier rds-lab-replica --region $REGION
aws rds describe-db-instances --db-instance-identifier rds-lab-replica --region $REGION \
  --query 'DBInstances[0].[DBInstanceStatus,ReadReplicaSourceDBInstanceIdentifier]'
```

**Expected:** The replica becomes a standalone primary; `ReadReplicaSourceDBInstanceIdentifier` disappears. One-way — no going back automatically.

**Why it matters:** Promotion is manual and changes the endpoint. This is why Multi-AZ (automatic) is the HA answer and replicas are the scaling answer.

---

## Exercise 7 — Modify the parameter group and reboot

**Goal:** Change a tunable the safe way and apply it.

```bash
aws rds create-db-parameter-group --db-parameter-group-name rds-lab-params \
  --db-parameter-group-family postgres16 --description "lab params" --region $REGION
aws rds modify-db-parameter-group --db-parameter-group-name rds-lab-params \
  --parameters "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
  --region $REGION
aws rds modify-db-instance --db-instance-identifier rds-lab \
  --db-parameter-group-name rds-lab-params --apply-immediately --region $REGION
aws rds reboot-db-instance --db-instance-identifier rds-lab --region $REGION
```

**Expected:** Parameter group shows `pending-reboot` until the reboot applies it; slow queries over 1s now get logged.

**Why it matters:** Never tune the default parameter group — clone it, version your changes, and know which parameters need a reboot.

---

## Exercise 8 — Point-in-time recovery from automated backups

**Goal:** Restore to a specific moment using transaction logs.

```bash
LATEST=$(aws rds describe-db-instances --db-instance-identifier rds-lab --region $REGION \
  --query 'DBInstances[0].LatestRestorableTime' --output text)
echo "Latest restorable: $LATEST"
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier rds-lab \
  --target-db-instance-identifier rds-lab-pitr \
  --restore-time "$LATEST" --region $REGION
```

**Expected:** New instance spins up restored to the latest restorable time (within ~5 min of now thanks to transaction logs).

**Why it matters:** This is your RPO story in interviews: automated backups + transaction logs = restore to any second in the retention window.

---

## Exercise 9 — Tear it all down

**Goal:** Leave no billable resources behind.

```bash
for id in rds-lab rds-lab-restored rds-lab-replica rds-lab-pitr; do
  aws rds delete-db-instance --db-instance-identifier $id \
    --skip-final-snapshot --delete-automated-backups --region $REGION 2>/dev/null
done
aws rds delete-db-snapshot --db-snapshot-identifier rds-lab-pre-migration --region $REGION
aws rds delete-db-parameter-group --db-parameter-group-name rds-lab-params --region $REGION
```

**Expected:** All deletes return without error; `describe-db-instances` shows nothing left.

**Why it matters:** RDS bills per hour per instance. The cheapest lab is the one you delete.

---

*Day 42 of 58 — next: Load Balancing & Autoscaling (ALB, ASG).*
