# Multi-Cloud & Hybrid Cloud Operations — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Multi-cloud | Two or more public clouds in active use, workloads split between them |
| Hybrid cloud | On-prem plus one public cloud, usually for data that can't leave the building |
| Cloud-agnostic | The portability goal — app runs on any cloud via portable pieces (K8s, Terraform, Postgres) |
| Real reason to go multi-cloud | Lock-in risk, a best-of-breed service, or a regulatory requirement — nothing else |
| VPN | Cheap, up in an afternoon, ~1.25 Gbps per tunnel, latency jitters |
| Private link | Direct Connect / ExpressRoute / Interconnect — stable latency, real monthly cost, weeks to provision |
| One IdP | Humans sign in once, federated via OIDC/SAML to every cloud |
| Workload identity | IRSA (EKS), Workload Identity Federation (GKE), managed identity (Azure) — no keys to leak |
| One pipeline | One image, one Helm chart, one Terraform module — differences in a values file per cloud |
| Data rule | Compute is portable, data is sticky — place the data first, the app follows |
| Egress | ~$0.09/GB out of AWS — 100 GB/day of replicated writes is ~$270/month just to copy |
| Networking rule | One IP allocation sheet, non-overlapping CIDRs, before the first VPC |
| DNS failover | Health check on the app endpoint (/healthz), not the load balancer |
| Active-passive | Primary serves all, secondary warm — cheaper, needs a real flip |
| Active-active | Both clouds serve traffic — survives an outage with no flip, costs double |
| One pane of glass | A labeling convention plus a single telemetry pipeline, not a dashboard purchase |
| Cost control | One tagging standard (owner, service, environment, cost-center) + weekly top-ten review |

## Hybrid vs multi-cloud vs cloud-agnostic

| Setup | Where workloads live | What connects them | Real example |
|---|---|---|---|
| Hybrid | On-prem + one public cloud | VPN or Direct Connect to the datacenter | Bank core on-prem, AWS for bursting |
| Multi-cloud | Two or more public clouds, active | VPN or interconnect between clouds | AWS for compute, GCP for BigQuery |
| Cloud-agnostic | Any cloud (design goal) | Portable pieces: K8s, Terraform, Postgres | One chart deploys to EKS or GKE unchanged |

Most "multi-cloud" teams actually run everything in one cloud with a second account — check where the workloads really live.

## VPN vs private link

| Option | Setup time | Throughput | Latency | Cost |
|---|---|---|---|---|
| Site-to-site VPN | An afternoon | ~1.25 Gbps per tunnel (use ECMP for more) | Jitters (public internet) | Low |
| Direct Connect / ExpressRoute / Interconnect | Weeks | 1–100 Gbps dedicated | Stable, low | Real monthly port + data charges |
| Cloud-to-cloud VPN | Hours | Same per-tunnel cap | Jitters | Low |
| Cloud interconnect partner | Weeks | Dedicated | Stable | Monthly cost |

Pick by traffic volume, not habit — check the tunnel's real throughput first.

## Identity across clouds

| Who | Mechanism | Never do this |
|---|---|---|
| Humans | One IdP (Okta / Entra ID), OIDC/SAML federation to each cloud | A second password or console login per cloud |
| EKS pods | IRSA — IAM role via service account | Long-lived access keys in a secret |
| GKE pods | Workload Identity Federation | Keys copied into another cloud's secret store |
| Azure workloads | Managed identity | A shared service account nobody owns |

Revoking access in the IdP kills it in every cloud at once — that's the point.

## One pipeline, two clouds

```hcl
# One module, two clouds: the app stays the same, only the wiring differs.
module "checkout_aws" {
  source        = "./modules/checkout-service"
  cloud         = "aws"
  image         = "checkout:1.42.0"
  load_balancer = "alb"
  database      = aws_db_instance.checkout.address
}

module "checkout_gcp" {
  source        = "./modules/checkout-service"
  cloud         = "gcp"
  image         = "checkout:1.42.0"
  load_balancer = "gclb"
  database      = google_sql_database_instance.checkout.private_ip_address
}
```

Never copy-paste the whole pipeline per cloud — one copy always drifts.

## Data across clouds

| Pattern | Consistency | Cost driver | When to use it |
|---|---|---|---|
| Async replication | Eventual — writes lag | Egress per GB on every write | Reads near the readers |
| Single-writer, one cloud | Strong — one source of truth | Cross-cloud read latency | Anything that must stay consistent |
| Active-active writes | Conflict-prone | Egress + conflict resolution | Almost never — only with CRDTs or sharding |
| No replication | N/A | None | Data that only one cloud needs |

Rule: replicate only what's needed, and answer "what happens to writes during a partition?" before you replicate anything.

## Networking checklist

| Rule | Detail |
|---|---|
| One IP sheet | Every cloud gets a non-overlapping CIDR — AWS 10.10.0.0/16, GCP 10.20.0.0/16, on-prem 10.30.0.0/16 |
| DNS story | One shared private zone, or a suffix per cloud (internal.aws.example.com, internal.gcp.example.com) |
| Service discovery | Works across the boundary — shared mesh control plane or a registry both clouds read |
| Classic mistake | Two teams each picking 10.0.0.0/16 on day one — a renumbering project three years later |

## Failover: active-passive vs active-active

| Design | How it works | Cost | RTO | Catch |
|---|---|---|---|---|
| Active-passive | Primary serves all; secondary warm, data replicating, apps scaled to zero | ~1.2x | Minutes (DNS flip) | The flip must be tested — untested failover is a hope |
| Active-active | Both clouds serve traffic all the time | ~2x | Near zero | Forces cross-cloud writes — the hardest problem |
| DNS failover | Health check on /healthz flips the record | Cheap | DNS TTL + check interval | Check the app endpoint, not the load balancer |

```bash
# Health check that actually catches an app failure:
aws route53 create-health-check --caller-reference checkout-primary \
  --health-check-config '{"Type":"HTTPS","ResourcePath":"/healthz",
  "FullyQualifiedDomainName":"checkout.example.com","Port":443,
  "RequestInterval":10,"FailureThreshold":3}'
# Failover routing policy: PRIMARY record with the health check attached,
# SECONDARY record in the other cloud.
```

## Observability & cost

| Area | Rule |
|---|---|
| Telemetry | One pipeline — OTLP / remote-write metrics, central logs, one tracing backend |
| Labels | Same labels everywhere: service, environment, cloud, region |
| Alerts | Live in the central system — a page means the same thing in any cloud |
| Tagging | owner, service, environment, cost-center on every resource, enforced |
| Cost review | Weekly top-ten spenders; anything untagged gets flagged |
| Egress watch | A chatty cross-cloud service is the bill nobody budgeted for |

## Commands to remember

```bash
# See a VPN tunnel's real throughput before trusting it:
aws cloudwatch get-metric-statistics --namespace AWS/VPN \
  --metric-name TunnelDataIn --dimensions Name=TunnelId,Value=vpn-xxxxxxxx \
  --start-time 2026-11-16T00:00:00Z --end-time 2026-11-16T06:00:00Z \
  --period 300 --statistics Average

# Find long-lived access keys hiding in a repo (each one is a leak):
grep -rE "AKIA[0-9A-Z]{16}" --include="*.tf" --include="*.yaml" --include="*.env" .

# Check which cluster context you're actually deploying to:
kubectl config get-contexts
kubectl config current-context

# List untagged resources before the weekly cost review (AWS example):
aws resourcegroupstaggingapi get-resources --tag-filters Key=owner --query 'ResourceTagMappingList' | head -20
```

## Interview traps

- Going multi-cloud for resilience alone — two clouds means two IAM models, two bills, double the pager load.
- Calling it multi-cloud when everything runs in one cloud with a second account.
- Routing production traffic between clouds over the open internet without encryption.
- Assuming a VPN tunnel will be enough — one IPsec tunnel caps around 1.25 Gbps.
- Long-lived access keys copied into a second cloud's secret store — a leak then works everywhere.
- Copy-pasting the whole pipeline per cloud — one copy always drifts.
- Replicating all data across clouds — the egress bill on every write, plus cross-cloud latency.
- Two clouds both on 10.0.0.0/16 — the renumbering project waiting to happen.
- A health check on the load balancer — the flip never fires while the app itself is down.
- A failover cloud that hasn't been deployed to in six months — the flip fails when you need it.
- Three consoles and three alerting systems — the on-call engineer clicking between tabs at 3 AM.
- Comparing clouds line by line on price — cost control is tagging and weekly review, not hunting the cheapest instance type.
