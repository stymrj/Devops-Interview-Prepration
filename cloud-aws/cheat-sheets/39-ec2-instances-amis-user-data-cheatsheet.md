# EC2 Cheat Sheet

## Instance families

| Family | For | Examples |
|---|---|---|
| `t` burstable | Spiky, low-baseline work | t3.micro dev boxes, small APIs |
| `m` general | Balanced CPU/memory | m7i.large default choice |
| `c` compute | CPU-bound batch, HPC | c7i.large |
| `r` memory | Caches, in-memory DBs | r7i.large Redis |
| `i`/`d` storage | High local I/O | i4i.large |
| `p`/`g` accelerated | GPU/ML | g5.xlarge training |
| Naming: `c7i.large` = compute, 7th gen, Intel (`i`), a=AMD, g=Graviton | | |

## Pricing

| Model | Discount | Use for |
|---|---|---|
| On-Demand | — | Short-lived, unpredictable |
| Reserved / Savings Plans | up to ~72% | Steady 1–3 yr baseline |
| Spot | up to ~90% | Stateless, interruptible |

## Launch & manage

```bash
# Launch
aws ec2 run-instances --image-id ami-xxx --instance-type t3.micro \
  --key-name my-key --security-group-ids sg-xxx --user-data file://ud.sh

# Describe / stop / start / terminate
aws ec2 describe-instances --instance-ids i-xxx
aws ec2 stop-instances --instance-ids i-xxx
aws ec2 start-instances --instance-ids i-xxx
aws ec2 terminate-instances --instance-ids i-xxx

# Wait helpers
aws ec2 wait instance-running --instance-ids i-xxx
aws ec2 wait instance-terminated --instance-ids i-xxx
```

## AMIs

```bash
# Build golden image from an instance (no reboot = safer for prod)
aws ec2 create-image --instance-id i-xxx \
  --name "golden-2026-10-13" --no-reboot
aws ec2 wait image-available --image-ids ami-xxx

# Copy to another region / share with another account
aws ec2 copy-image --source-image-id ami-xxx \
  --source-region us-east-1 --region ap-south-1 --name golden
aws ec2 modify-image-attribute --image-id ami-xxx \
  --launch-permission "Add=[{UserId=123456789012}]"
```

## User data & cloud-init

```bash
# Pass at launch only; logs live here
/var/log/cloud-init-output.log   # user-data script output
/var/log/cloud-init.log          # cloud-init stage debug
```

- Runs **once** at first boot, as root, plaintext-visible in console — **no secrets**
- `#cloud-config` or `#!/bin/bash` headers; MIME multipart for combos

## EBS

```bash
# Snapshot / restore
aws ec2 create-snapshot --volume-id vol-xxx --description "pre-deploy"
aws ec2 create-volume --snapshot-id snap-xxx --availability-zone ap-south-1a
aws ec2 attach-volume --volume-id vol-new --instance-id i-xxx --device /dev/sdf

# Types: gp3 (general, tune IOPS/throughput), io2 (guaranteed IOPS),
# st1 (throughput HDD), sc1 (cold HDD)
```

- **EBS** = persistent network block storage. **Instance store** = ephemeral local SSD — scratch only.

## Security groups

```bash
aws ec2 authorize-security-group-ingress --group-id sg-xxx \
  --protocol tcp --port 443 --cidr 10.0.0.0/8
aws ec2 revoke-security-group-ingress --group-id sg-xxx \
  --protocol tcp --port 22 --cidr 0.0.0.0/0
```

- Stateful (return traffic auto-allowed), default-deny, changes apply instantly
- NACLs = stateless, subnet-level, explicit both directions

## Spot

```bash
aws ec2 describe-spot-price-history --instance-types c5.large \
  --product-descriptions "Linux/UNIX" \
  --start-time 2026-10-13T00:00:00Z
```

- 2-minute interruption notice — handle it or don't use Spot
- Diversify instance types + AZs in the fleet

## Placement groups

| Type | When |
|---|---|
| cluster | Low latency, HPC (same rack) |
| spread | Max physical isolation |
| partition | Large distributed systems (Kafka) |

## Interview one-liners

- **Golden image vs user data:** bake the stable base, inject the changing bits
- **Restore from snapshot:** create volume from snapshot → attach in same AZ → warm it
- **Instances not picking up new app:** check the launch template *default version* the ASG points at
- **Cleanup:** terminate instances, delete orphaned volumes, snapshots, and EIPs
