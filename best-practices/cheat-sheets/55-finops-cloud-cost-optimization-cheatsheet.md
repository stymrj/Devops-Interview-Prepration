# FinOps & Cost Optimization Cheat Sheet

*Pricing models, waste patterns, and the commands that find them — one page.*

---

## Pricing Models

| Model | Discount | Commitment | Use for |
|-------|----------|------------|---------|
| On-demand | 0% | None | Unknown / spiky workloads |
| Reserved Instance | ~30-60% | 1-3 yrs, instance family | Steady 24/7 compute |
| Compute Savings Plan | ~30-60% | 1-3 yrs, $/hr commit | Steady compute, any family |
| Spot | up to 90% | None, 2-min reclaim | Batch, CI, stateless workers |

---

## The Usual Suspects (waste)

| Waste | Where to look | Fix |
|-------|---------------|-----|
| Unattached EBS volumes | `ec2 describe-volumes --filters Name=status,Values=available` | Delete or snapshot-then-delete |
| Old snapshots | `ec2 describe-snapshots --owner-ids self` | Lifecycle policies |
| Unattached Elastic IPs | `ec2 describe-addresses` | Release (idle EIPs are billed) |
| Stopped-but-alive instances | Console / `describe-instances` | Terminate after grace period |
| Oversized instances | Compute Optimizer | Downsize in steps |
| Cross-AZ data transfer | Cost Explorer by USAGE_TYPE `DataTransfer-*` | VPC endpoints, same-AZ placement |
| Unversioned log buckets | `s3api get-bucket-lifecycle-configuration` | Add lifecycle rules |

---

## Key Cost Commands

```bash
# Spend by service, last 30 days
aws ce get-cost-and-usage --time-period Start=<d1>,End=<d2> \
  --granularity MONTHLY --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Spend by team tag
aws ce get-cost-and-usage --time-period Start=<d1>,End=<d2> \
  --granularity MONTHLY --metrics BlendedCost \
  --group-by Type=TAG,Key=Team

# Find resources missing a tag
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters ec2:instance \
  --tag-filters Key=Team,Values=

# Apply S3 lifecycle (IA @30d, Glacier @90d)
aws s3api put-bucket-lifecycle-configuration \
  --bucket BUCKET --lifecycle-configuration file://lifecycle.json

# Current spot price for a type
aws ec2 describe-spot-price-history --instance-types m6i.large \
  --product-descriptions Linux/UNIX --start-time <ISO8601>
```

---

## Tagging Rules of Thumb

- **Enforce in IaC** — never rely on humans tagging manually.
- **Activate tags for billing** — they don't appear in Cost Explorer until activated.
- **Minimum set:** Team, Service, Environment, CostCenter.
- **Weekly sweep:** report untagged resources above a spend threshold.

---

## K8s Cost Levers

- Set **requests and limits** — the scheduler packs better, fewer nodes needed.
- **Cluster Autoscaler** + scale dev namespaces to zero after hours.
- **Spot node groups** for non-critical workloads.
- **OpenCost / Kubecost** for namespace-level attribution.

---

## Interview One-Liners

- "Commit to the baseline, spot the flexible, on-demand the unknown."
- "Untagged resources are unowned costs."
- "Cut waste before you cut reliability."
- "A budget alert is the cheapest FinOps control there is."
