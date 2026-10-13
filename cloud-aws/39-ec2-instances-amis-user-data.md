# EC2 (Instances, AMIs, User Data) Interview Preparation Guide

*How to Answer EC2 Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Instance Types and Families](#instance-types-and-families)
2. [Pricing Models](#pricing-models)
3. [AMIs and Golden Images](#amis-and-golden-images)
4. [User Data and Cloud-Init](#user-data-and-cloud-init)
5. [Storage: EBS and Instance Store](#storage-ebs-and-instance-store)
6. [Networking: Security Groups and Placement](#networking-security-groups-and-placement)
7. [Interview Traps and Real Scenarios](#interview-traps-and-real-scenarios)

---

## Instance Types and Families

### Q1: What are EC2 instance types and families? How do you pick one?

**How to Answer:**

"EC2 instance types are grouped into families by what they're good at — general purpose like `m` and `t`, compute like `c`, memory like `r`, and storage/GPU types like `i`, `p`, and `g`. The naming is `family + generation + size`, so `c7i.large` is compute-optimized, 7th gen, Intel. I pick by profiling the workload first: CPU-bound batch jobs go to `c`, in-memory caches go to `r`, bursty web servers go to `t`. When in doubt I start with general purpose `m` and let CloudWatch metrics tell me if I need to move."

**Key Point:** "Pick the family by the bottleneck, not by guessing — profile, then choose."

---

### Q2: What are burstable instances (T family)? How do CPU credits work?

**How to Answer:**

"T instances earn CPU credits when idle and spend them when bursting — they run at a low baseline and spike up when needed. If you run out of credits on a `t2`/`t3`, the CPU gets throttled down to the baseline, which is the classic gotcha. `t3` is unlimited mode by default, so it bursts past zero credits and you just pay a small surcharge instead of throttling. They're perfect for dev boxes, small APIs, and anything spiky — but I never put sustained workloads on them."

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUCreditBalance --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time 2026-10-12T00:00:00Z --end-time 2026-10-13T00:00:00Z \
  --period 3600 --statistics Average
```

**Key Point:** "T instances burst on credits — zero credits means throttling unless unlimited mode is on."

---

## Pricing Models

### Q3: Compare On-Demand, Reserved, Savings Plans, and Spot pricing.

**How to Answer:**

"On-Demand is pay-as-you-go, no commitment — good for short-lived or unpredictable workloads. Reserved Instances and Savings Plans are 1- or 3-year commitments in exchange for big discounts — Savings Plans are more flexible since they apply across instance families and Regions. Spot is spare AWS capacity at up to 90% off, but AWS can reclaim it with a 2-minute warning. My rule: Spot for stateless, fault-tolerant work like batch and CI agents; Reserved or Savings Plans for the steady-state baseline I know will run all year; On-Demand for everything else."

**Key Point:** "Spot for interruptible work, commitments for the steady baseline, On-Demand for the rest."

---

### Q4: When should you use Spot Instances, and what are the gotchas?

**How to Answer:**

"I use Spot for stateless, restartable workloads — build agents, batch jobs, dev environments — where losing an instance is just an inconvenience. The gotcha is the interruption: AWS gives you a 2-minute reclaim notice, so the app must checkpoint or be behind a queue. I also diversify across multiple instance types and AZs in the Spot fleet so one pool draining doesn't take everything down. Mixing Spot with a small On-Demand base keeps the critical path alive while cutting the bill."

```bash
aws ec2 describe-spot-price-history --instance-types c5.large m5.large \
  --product-description "Linux/UNIX" --start-time 2026-10-13T00:00:00Z
```

**Key Point:** "Spot is for interruptible work only — handle the 2-minute warning or don't use it."

---

## AMIs and Golden Images

### Q5: What is an AMI? How do you build a golden image?

**How to Answer:**

"An AMI is a snapshot of a root volume plus launch permissions and block-device mappings — it's the template every EC2 instance boots from. For a golden image I launch a clean instance from the official Amazon Linux AMI, install and harden everything, then create my own AMI and bake it into the pipeline. That gives me fast, repeatable launches with no drift between environments. I rebuild it regularly with Packer so the image never goes stale, and I share it to other accounts only with explicit launch permissions."

```bash
aws ec2 create-image --instance-id i-0abc123 \
  --name "web-server-golden-2026-10-13" --no-reboot
```

**Key Point:** "Golden images trade slow provisioning-time installs for fast, identical launches."

---

### Q6: Golden image vs. user-data bootstrapping — which do you choose?

**How to Answer:**

"Golden images win when boot speed matters — auto-scaling events don't wait for a 10-minute install. User-data bootstrapping wins when things change fast, because you always pull the latest packages and config at launch instead of rebuilding images. In practice I use both: a golden image with the stable base baked in, plus a short user-data script for environment-specific config. That way launches are fast and each environment still gets its own settings."

**Key Point:** "Bake the stable base into the AMI, inject the changing bits with user data."

---

## User Data and Cloud-Init

### Q8: What is EC2 user data? How is it different from an AMI?

**How to Answer:**

"User data is a script or cloud-config that runs once at instance launch to customize the box — install packages, write config files, start services. An AMI is the full starting template; user data is the last-mile personalization on top of it. User data runs as root through cloud-init, and the output lands in `/var/log/cloud-init-output.log`, which is the first place I check when a launch doesn't come up right. It's plaintext in the console by default, so I never put secrets in it — those come from SSM Parameter Store or Secrets Manager."

```bash
#!/bin/bash -xe
yum update -y
yum install -y nginx
systemctl enable --now nginx
echo "healthy" > /usr/share/nginx/html/health
```

**Key Point:** "User data customizes at launch; never put secrets in it — they're plaintext-visible."

---

### Q9: What is cloud-init and what are its stages?

**How to Answer:**

"Cloud-init is the service that runs user data and instance metadata on first boot — it's how a generic AMI becomes a specific server. It runs in stages: init finds the data source and sets the hostname and networking, config applies packages and files, and final runs the user script. It only runs the per-instance modules once, which is why rebooting doesn't re-run user data by default. When a launch fails, `/var/log/cloud-init.log` shows which stage broke — usually a bad package URL or a syntax error in my script."

**Key Point:** "Cloud-init turns a generic AMI into a specific server, once per instance."

---

## Storage: EBS and Instance Store

### Q10: EBS vs. instance store — what's the difference?

**How to Answer:**

"EBS is network-attached block storage that survives instance stops and terminations — it's where my real data lives. Instance store is ephemeral SSD physically attached to the host — blazing fast, but the data dies with the instance. The interview trap is calling instance store 'free fast storage' — it's only safe for scratch data, caches, and temp files. I size EBS volumes by IOPS needs with `gp3` for general work and `io2` when I need guaranteed performance, and I snapshot them before risky changes."

```bash
aws ec2 create-snapshot --volume-id vol-0abc123 \
  --description "pre-deploy-backup-2026-10-13"
```

**Key Point:** "EBS persists, instance store doesn't — treat them as permanent vs. scratch."

---

### Q11: What are EBS snapshots, and how do you restore from one?

**How to Answer:**

"Snapshots are incremental, point-in-time backups of an EBS volume stored in S3 — only changed blocks are stored after the first one. I automate them with Data Lifecycle Manager policies instead of cron hacks, with retention rules so old ones age out. To restore, I create a new volume from the snapshot in the same AZ, then attach it to the instance. The gotcha is first-launch latency: blocks are fetched lazily from S3, so I warm a restored volume with `dd` or use fast snapshot restore for critical restores."

**Key Point:** "Snapshots are incremental and cheap — restore by creating a volume from one, then warm it."

---

## Networking: Security Groups and Placement

### Q12: What are security groups? How do they differ from NACLs?

**How to Answer:**

"Security groups are stateful virtual firewalls on the instance — if I allow inbound 443, the return traffic is automatically allowed. NACLs are stateless subnet-level rules, so both directions need explicit rules. Security groups default-deny everything, which is the right starting posture, and I keep them tight — port, protocol, source — never `0.0.0.0/0` on SSH. In interviews I also call out that changing a security group takes effect immediately, no reboot needed."

**Key Point:** "Security groups are stateful per-instance; NACLs are stateless per-subnet — SG first."

---

### Q13: What are placement groups? When do you use them?

**How to Answer:**

"Placement groups control where instances physically land. Cluster puts them on the same rack for low latency — that's my pick for HPC and tightly-coupled workloads that use ENA. Spread puts them on separate racks for the rare cases I need maximum physical isolation. Partition spreads across groups of racks for big distributed systems like Kafka where I want failures isolated to a partition. I rarely need them — most workloads are fine with default placement and multi-AZ distribution."

**Key Point:** "Cluster for low latency, spread for physical isolation, partition for large distributed systems."

---

## Interview Traps and Real Scenarios

### Q14: Your auto-scaling launch template change isn't picking up — the new instances still run the old app version. How do you debug it?

**How to Answer:**

"This is the classic immutable-template trap: editing the launch template's default version isn't automatic — you have to set the new version as default or reference it explicitly in the ASG. First I check which launch template version the ASG is actually pointing at, then confirm the AMI in that version is the new golden image. If it's user-data based, I pull `/var/log/cloud-init-output.log` to see if the script failed or the artifact download 404'd. My fix discipline: never SSH in and hot-patch — build the new template version, roll it, and let the old instances drain out."

**Key Point:** "New instances come from the launch template's default version — check what the ASG actually points at."

---

*That's the guide. Read it out loud, tweak the phrasing to your own voice, and you'll walk in sounding like someone who's actually run EC2 in production.*


---

### Q7: What is Packer and how does it fit into image pipelines?

**How to Answer:**

"Packer builds machine images from code — I define the steps once in a template and it produces AMIs identically every time. It's the 'build' half of immutable infrastructure: Packer builds the image, Terraform deploys it, and nothing is ever hand-configured. I wire it into CI so every merged config change triggers a fresh AMI build and a pipeline that tests, then promotes it. If the tests fail, the old AMI stays in production — rollbacks are just pointing back at the previous image."
