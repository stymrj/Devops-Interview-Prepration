# HA & DR Cheat Sheet

*RTO/RPO, availability math, and the commands that matter — one page.*

---

## The Acronyms

| Term | Meaning | Example |
|------|---------|---------|
| RTO | Max acceptable downtime | 2 hours |
| RPO | Max acceptable data loss (in time) | 15 minutes |
| MTTR | Mean time to repair — how fast you fix | 45 min |
| MTBF | Mean time between failures — how often it breaks | 90 days |
| Availability | MTBF / (MTBF + MTTR) | — |

---

## Nines → Downtime

| Nines | Yearly downtime | Monthly |
|-------|-----------------|---------|
| 99% | 3.65 days | 7.3 hrs |
| 99.9% | 8.76 hrs | 43.8 min |
| 99.99% | 52.6 min | 4.38 min |
| 99.999% | 5.26 min | 26.3 sec |

---

## DR Tiers (cheap → expensive)

| Tier | What you keep | RTO |
|------|---------------|-----|
| Backup & restore | Backups in 2nd region | Hours |
| Pilot light | Data replicated, infra idle | Tens of minutes |
| Warm standby | Scaled-down live copy | Minutes |
| Multi-site active | Full prod in 2 regions | Near zero |

---

## Backup & Restore (Postgres)

```bash
pg_dump -h primary -U postgres mydb > backup.sql        # backup
psql -h standby -U postgres restored < backup.sql       # restore
pg_basebackup -h primary -D /var/lib/pg/standby -R      # base backup for replica
```

## Replication Lag Check

```bash
# Postgres replica lag (your true RPO):
psql -h replica -c "SELECT now() - pg_last_xact_replay_timestamp() AS lag;"
# Promote replica:
pg_ctl promote -D /var/lib/postgresql/replica
```

## MySQL

```bash
SHOW SLAVE STATUS\G            # check Seconds_Behind_Master
STOP SLAVE; RESET MASTER;     # promote (then repoint app)
```

## Kubernetes HA Patterns

```bash
kubectl get nodes                        # spread across zones?
kubectl top nodes                        # headroom for failover
# PodDisruptionBudget: keep N pods alive during drains
# topologySpreadConstraints: force pods across zones
# affinity: nodeAffinity on topology.kubernetes.io/zone
```

## DNS Failover

```bash
dig your-svc.example.com | grep -i ttl   # TTL must be 60-300s for DR
# Route 53: health checks + failover routing policy
```

## AWS Building Blocks

| Need | Service |
|------|---------|
| Multi-AZ DB | RDS Multi-AZ / Aurora Global |
| Cross-region backup | S3 CRR, RDS cross-region snapshots |
| Traffic failover | Route 53 failover routing, ALB target groups |
| Infra rebuild | Terraform state in remote backend, tested `apply` |
| Chaos drills | Fault Injection Simulator (FIS) |

## DR Runbook Skeleton

1. Confirm (provider status + second signal)
2. Declare (who, bridge, severity)
3. Failover (ordered, copy-paste commands)
4. Verify real traffic (error rates, not dashboards)
5. Communicate (status page before customers ask)
6. Postmortem within 48h, blameless

---

**One-liner for interviews:** "RPO is data you can lose, RTO is downtime you can take — everything else is engineering to fit those two numbers at a price the business accepts."
