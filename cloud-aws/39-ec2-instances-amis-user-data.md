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

### Q7: What is Packer and how does it fit into image pipelines?

**How to Answer:**

"Packer builds machine images from code — I define the steps once in a template and it produces AMIs identically every time. It's the 'build' half of immutable infrastructure: Packer builds the image, Terraform deploys it, and nothing is ever hand-configured. I wire it into CI so every merged config change triggers a fresh AMI build and a pipeline that tests, then promotes it. If the tests fail, the old AMI stays in production — rollbacks are just pointing back at the previous image."
