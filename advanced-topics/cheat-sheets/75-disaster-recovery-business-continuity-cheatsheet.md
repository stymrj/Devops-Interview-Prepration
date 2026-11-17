# Disaster Recovery & Business Continuity — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| High availability | Survives a component or zone failing — keeps you up through a bad day |
| Disaster recovery | Survives the whole region or dataset failing — brings you back from a bad week |
| RTO | How long you can be down — from "it's broken" to "users are back" |
| RPO | How much data you can lose — the gap between the last good backup and the failure |
| Backup & restore | Cheapest pattern — data copied elsewhere, nothing running, RTO in hours |
| Pilot light | Data replicated, minimal core running — scale up on failure, RTO ~30 min |
| Warm standby | Scaled-down copy of everything running — RTO in minutes, ~1.5x cost |
| Active-active | Both regions serve traffic — RTO near zero, ~2x cost, cross-region writes to solve |
| Immutable backup | A copy ransomware can't encrypt — object lock in a second region or account |
| Restore test | The only proof a backup works — restore monthly into a clean environment |
| Split-brain | Both sides think they're primary and accept writes — two versions of the truth |
| Fencing | Blocks the old primary before the new one accepts writes — one writer at a time |
| Health check | Must test the app endpoint (/healthz), not the load balancer |
| Failover button | One command that flips DNS, scales up the secondary, and pages the team |
| Tabletop | A room walkthrough of the runbook — finds gaps, quarterly |
| Game day | Actually killing the primary — finds the real bugs, twice a year |

## HA vs DR

| | High availability | Disaster recovery |
|---|---|---|
| Survives | A node dies, a zone goes down | A region goes down, data corrupted, site gone |
| Mechanism | Replicas standing by | Rebuild service somewhere else |
| Time scale | Seconds to minutes | Minutes to hours |
| Classic mistake | Three replicas in one region, called DR | A backup nobody has restored |

Three replicas in one region is HA, not DR — a bad deploy can take all three out at once.

## The four DR patterns

| Pattern | RTO | RPO | Cost | Catch |
|---|---|---|---|---|
| Backup & restore | ~4 hours | ~1 hour | ~1.05x | Rebuild everything under pressure |
| Pilot light | ~30 min | ~5 min | ~1.2x | Scale-up must actually work |
| Warm standby | ~5 min | ~1 min | ~1.5x | You're paying for a second system |
| Active-active | ~0 | ~0 | ~2x | Cross-region writes — the hardest problem |

The business picks the row, you build it — get the tier accepted in writing.

## RTO / RPO: the negotiation

| Question | Answer |
|---|---|
| What does an hour down cost? | Revenue + fines + trust — that's the number that sets the RTO |
| What can we lose? | The gap between the last good backup and the failure — that's the RPO |
| Who decides? | The business, not engineering — the numbers are a signed decision |
| The trap | Quoting "RTO 15 minutes" with no cost behind it — a spec, not a plan |

## What to back up (beyond the database)

| Item | Why it matters |
|---|---|
| Database | The data itself — the part everyone remembers |
| Terraform state | Without it, you can't rebuild the infrastructure |
| etcd snapshot | The cluster's brain — lose it, lose the cluster |
| Vault unseal keys / secrets | Backups you can't decrypt are files, not recovery |
| Helm values, DNS records, IAM policies | The wiring — a database dump alone is a weekend |
| CI secrets | The pipeline can't deploy without them |

Back up the system, not just the data — and keep an immutable copy in a second region.

## Replication: async vs sync

| Mode | Data loss on failover | Cost | When to use it |
|---|---|---|---|
| Async | The lag window — seconds of writes lost | Low | Reads, non-critical data |
| Sync | Near zero — every write waits for the other region | Latency on every write | The ledger — anything that must not lose a write |
| Split-brain | Both sides accept writes | Two versions of the truth | Never — fence the old primary first |

Rule: one writer at a time, fencing before the new primary accepts writes, and a written "what did we lose?" before declaring recovery done.

## The traffic flip

| Piece | Rule |
|---|---|
| Health check | Test the app endpoint (/healthz), not the load balancer |
| DNS failover | Failover routing policy, PRIMARY with health check, SECONDARY without |
| TTL | 60s — a 300s TTL keeps users on the dead region for five more minutes |
| Global load balancer | Flips faster than DNS, costs more |
| Sessions | Long-lived connections drop — the app must handle reconnects |
| The test | Run the flip in a game day and write down the real RTO |

```bash
# etcd snapshot — the cluster's brain, backed up on a schedule:
ETCDCTL_API=3 etcdctl snapshot save /backups/etcd-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
aws s3 cp /backups/etcd-$(date +%F).db \
  s3://dr-backups-eu-west-1/etcd/ --storage-class STANDARD_IA

# Health check on the app endpoint + failover routing policy:
aws route53 create-health-check --caller-reference payments-dr \
  --health-check-config '{"Type":"HTTPS","ResourcePath":"/healthz",
  "FullyQualifiedDomainName":"payments.example.com","Port":443,
  "RequestInterval":10,"FailureThreshold":3}'

# Check replication lag before you trust the secondary (Postgres):
SELECT client_addr, state, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)
  AS lag_bytes FROM pg_stat_replication;
```

## Drills: tabletop vs game day

| Drill | What it finds | How often |
|---|---|---|
| Tabletop | Runbook gaps, missing phone numbers, steps nobody owns | Quarterly |
| Game day | The health check that never fires, the backup that doesn't restore, the rotted secondary | Twice a year, for anything that earns money |
| Restore test | Whether the backup actually restores — and the real RTO | Monthly, into a clean environment |

A drill that never touches production only tests the script, not the system.

## Selling DR to the business

| Argument | How it sounds |
|---|---|
| The pitch | "An hour down costs X, warm standby costs Y a month — it pays for itself in one saved hour" |
| The honesty | Backup and restore costs almost nothing, but the RTO is hours — valid if accepted in writing |
| The worst outcome | A tier nobody chose — everyone assumed someone else bought the expensive one |
| The line | DR is a signed decision about how much downtime costs, not a technical default |

## Interview traps

- Three replicas in one region, called disaster recovery.
- Quoting RTO/RPO like specs, with no cost behind the numbers.
- A backup you've never restored — the green tick only proves the file was written.
- Backing up the database and forgetting Terraform state, etcd, Vault keys, and configs.
- Backups in the same account and region — ransomware encrypts them too.
- Async replication with no answer to "what did we lose?" — the lag window is the real RPO.
- Sync replication without counting the latency it adds to every write.
- No fencing — split-brain gives you two versions of the truth to reconcile by hand.
- A health check on the load balancer — the flip never fires while the app itself is down.
- A 300-second DNS TTL — users keep hitting the dead region for five more minutes.
- A secondary that hasn't been deployed to in six months — the flip lands on an old schema.
- A runbook in one person's head — nobody can trigger the failover at 3 AM.
- A drill that never touches production — you've tested the script, not the system.
- Buying active-active for everything — double the cost, plus the cross-region write problem.
- Everyone assuming someone else bought the expensive tier — a tier nobody chose.
