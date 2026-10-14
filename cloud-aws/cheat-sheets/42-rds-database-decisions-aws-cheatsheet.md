# RDS Cheat Sheet — Backups, HA, Aurora, Decisions

## Create an instance

```bash
aws rds create-db-instance --db-instance-identifier prod-db \
  --db-instance-class db.m6i.large --engine postgres --engine-version 16.3 \
  --allocated-storage 100 --storage-type gp3 --iops 3000 \
  --master-username admin --manage-master-user-password \   # Secrets Manager
  --vpc-security-group-ids sg-xxxx --db-subnet-group-name prod-private \
  --multi-az --backup-retention-period 30 --storage-encrypted \
  --enable-performance-insights --region ap-south-1
aws rds wait db-instance-available --db-instance-identifier prod-db
```

`--manage-master-user-password` stores the password in Secrets Manager. Encryption is set at creation — can't add later.

## Describe / modify / reboot

```bash
aws rds describe-db-instances --db-instance-identifier prod-db \
  --query 'DBInstances[0].[Endpoint.Address,DBInstanceStatus,MultiAZ]'
aws rds modify-db-instance --db-instance-identifier prod-db \
  --db-instance-class db.m6i.xlarge --apply-immediately
aws rds reboot-db-instance --db-instance-identifier prod-db
aws rds start-db-instance --db-instance-identifier prod-db   # stopped >7 days auto-starts
aws rds stop-db-instance --db-instance-identifier prod-db
```

## Snapshots and point-in-time recovery

```bash
aws rds create-db-snapshot --db-instance-identifier prod-db \
  --db-snapshot-identifier prod-db-$(date +%F)
aws rds describe-db-snapshots --db-instance-identifier prod-db \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,Status,SnapshotCreateTime]'
aws rds copy-db-snapshot --source-db-snapshot-identifier prod-db-2026-10-14 \
  --target-db-snapshot-identifier prod-db-2026-10-14-dr \
  --source-region ap-south-1 --region ap-south-2
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-db-restored \
  --db-snapshot-identifier prod-db-2026-10-14
LATEST=$(aws rds describe-db-instances --db-instance-identifier prod-db \
  --query 'DBInstances[0].LatestRestorableTime' --output text)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db --target-db-instance-identifier prod-db-pitr \
  --restore-time "$LATEST"
```

Restore always creates a NEW instance with a NEW endpoint. Automated backups: 0–35 days retention. Manual snapshots: kept until deleted.

## Multi-AZ and read replicas

```bash
aws rds modify-db-instance --db-instance-identifier prod-db --multi-az --apply-immediately
aws rds create-db-instance-read-replica --db-instance-identifier prod-replica-1 \
  --source-db-instance-identifier prod-db
aws rds promote-read-replica --db-instance-identifier prod-replica-1   # manual, one-way
```

Multi-AZ = sync standby, auto failover, same endpoint, NOT readable. Replicas = async, readable, manual promotion, endpoint changes.

## Aurora cluster

```bash
aws rds create-db-cluster --db-cluster-identifier aurora-prod \
  --engine aurora-postgresql --engine-version 16.3 \
  --master-username admin --manage-master-user-password \
  --vpc-security-group-ids sg-xxxx --db-subnet-group-name prod-private \
  --backup-retention-period 30 --storage-encrypted --region ap-south-1
aws rds create-db-instance --db-instance-identifier aurora-prod-writer \
  --db-cluster-identifier aurora-prod --db-instance-class db.r6g.large --engine aurora-postgresql
aws rds create-db-instance --db-instance-identifier aurora-prod-reader-1 \
  --db-cluster-identifier aurora-prod --db-instance-class db.r6g.large --engine aurora-postgresql
aws rds failover-db-cluster --db-cluster-identifier aurora-prod   # force failover drill
```

Shared storage across AZs, up to 15 readers, failover in seconds, storage auto-grows to 128 TB.

## Parameter groups and logs

```bash
aws rds create-db-parameter-group --db-parameter-group-name prod-pg16 \
  --db-parameter-group-family postgres16 --description "prod tuning"
aws rds modify-db-parameter-group --db-parameter-group-name prod-pg16 \
  --parameters "ParameterName=max_connections,ParameterValue=500,ApplyMethod=pending-reboot"
aws rds describe-db-log-files --db-instance-identifier prod-db
aws rds download-db-log-file-portion --db-instance-identifier prod-db \
  --log-file-name error/postgresql.log --starting-token 0 --output text | tail -50
```

Clone the default group before tuning. Some params need a reboot (`pending-reboot`).

## RDS Proxy

```bash
aws rds create-db-proxy --db-proxy-name app-proxy --engine-family POSTGRESQL \
  --vpc-subnet-ids subnet-a subnet-b --role-arn arn:aws:iam::123456789012:role/rds-proxy-role \
  --auth "AuthScheme=SECRETS,SecretArn=arn:aws:secretsmanager:ap-south-1:123456789012:secret:prod/rds-AbCdEf,IAMAuth=DISABLED"
```

Pools connections, cuts failover downtime. Reach for it with Lambda or spiky auto-scaling.

## Security checklist

- Private subnets only, `--no-publicly-accessible`
- SG allows 5432/3306 only from the app SG
- Secrets Manager with rotation (`--manage-master-user-password`) or IAM DB auth
- `--storage-encrypted` with a KMS key, TLS in transit
- Disable the default `admin`-style usernames where possible

## The decision tree

| Workload | Pick |
|---|---|
| Standard relational app | RDS (Postgres/MySQL), Multi-AZ |
| DB is the bottleneck / global | Aurora |
| Key-value, ms latency, huge scale | DynamoDB |
| Engine/extension AWS lacks | EC2 self-hosted (price the on-call) |
| Heavy reads, reports | Read replicas (+ RDS Proxy for connection spikes) |
| RPO minutes, RTO low | 30-day backups + cross-region snapshot copies |

## Numbers interviewers expect

- Multi-AZ failover: ~1–2 min (DB cluster: <35 s)
- Aurora failover: seconds
- Backup retention: up to 35 days; RPO ~5 min via transaction logs
- Aurora readers: up to 15; storage: 128 TB auto-growing, 6-way replicated across 3 AZs
- Free tier: 750 hrs/month db.t3.micro or db.t4g.micro, 20 GB storage (12 months)
