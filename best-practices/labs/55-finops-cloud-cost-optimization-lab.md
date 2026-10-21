# Hands-On Lab: FinOps & Cloud Cost Optimization

*9 exercises that turn your cloud bill into something you control. Run these against a personal account or sandbox — the first pass is read-only, and every destructive step is gated on your confirmation.*

---

## Exercise 1: Read Your Bill Like a Detective

**Goal:** Find where the money actually goes before optimizing anything.

**Commands:**

```bash
# Top 10 costliest services this month (AWS CLI)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%F),End=$(date +%F) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter file://<(echo '{"Dimensions":{"Key":"RECORD_TYPE","Values":["Usage"]}}') \
  | python3 -c "import json,sys; d=json.load(sys.stdin); rows=sorted(((r['Groups'][0]['Keys'][0], float(r['Metrics']['BlendedCost']['Amount'])) for r in d['ResultsByTime'][0]['Groups']), key=lambda x:-x[1]); [print(f'{s:40s} ${a:.2f}') for s,a in rows[:10]]"
```

**Expected output:** A ranked list — e.g. EC2 $84.20, RDS $41.10, S3 $12.55…

**Why it matters:** Most engineers have never seen their bill broken down. Optimization without attribution is guessing.

---

## Exercise 2: Find Untagged Resources Over $1

**Goal:** Prove that tagging discipline matters.

**Commands:**

```bash
# Instances missing the Team tag
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters ec2:instance \
  --tag-filters Key=Team,Values= \
  --query 'ResourceTagMappingList[].ResourceARN'
```

**Expected output:** A list of ARNs (hopefully short). If it's long, you just found your FinOps backlog.

**Why it matters:** Untagged resources can't be assigned to a team. No assignment means no owner, no owner means no optimization.

---

## Exercise 3: Hunt Orphaned EBS Volumes

**Goal:** Find money attached to nothing.

**Commands:**

```bash
# Unattached volumes, with size and monthly cost estimate
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[].{Id:VolumeId,GiB:Size,AZ:AvailabilityZone}' --output table
```

**Expected output:** A table of idle volumes. gp3 is roughly $0.08/GB-month — do the math.

**Why it matters:** Every terminated instance leaves its root volume behind if `DeleteOnTermination` wasn't set. This is the single most common cloud waste.

---

## Exercise 4: Audit Old Snapshots

**Goal:** See what snapshot sprawl costs.

**Commands:**

```bash
# Snapshots older than 90 days owned by you
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[?StartTime<='$(date -d '90 days ago' +%Y-%m-%d)'].{Id:SnapshotId,GiB:VolumeSize,Age:StartTime}" \
  --output table
```

**Expected output:** A list of ancient snapshots. At ~$0.05/GB-month, 100 GB of forgotten snapshots is $5/month of nothing.

**Why it matters:** Snapshot lifecycle is set-and-forget. One missed policy compounds every month.

---

## Exercise 5: Compare On-Demand vs Spot Pricing

**Goal:** Feel the size of the spot discount.

**Commands:**

```bash
# Current spot vs on-demand price for m6i.large in your region
aws ec2 describe-spot-price-history \
  --instance-types m6i.large --product-descriptions Linux/UNIX \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'SpotPriceHistory[0].{AZ:AvailabilityZone,SpotPrice:SpotPrice}' --output table
# Compare against on-demand: aws pricing get-products is the verbose route;
# quicker: check the EC2 console pricing page for m6i.large on-demand (~$0.096/hr in us-east-1)
```

**Expected output:** Spot price roughly 60-70% below on-demand for the same type.

**Why it matters:** Quoting the actual discount from your region makes "move batch jobs to spot" a one-line business case.

---

## Exercise 6: Spot a Cross-AZ Data Transfer Leak

**Goal:** Learn why architecture placement shows up on the bill.

**Commands:**

```bash
# NAT gateway data processed last month (a proxy for cross-AZ/subnet traffic)
aws ec2 describe-nat-gateways \
  --query 'NatGateways[].{Id:NatGatewayId,State:State}' --output table
# Then check Cost Explorer grouped by usage type for DataTransfer entries
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%F),End=$(date +%F) \
  --granularity MONTHLY --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE \
  | grep -i -A1 -i datatransfer | head -20
```

**Expected output:** DataTransfer-* usage types and their cost. A single NAT gateway processing 1 TB costs ~$45 in processing fees alone.

**Why it matters:** $0.09/GB cross-AZ transfer is invisible in architecture diagrams and very visible on the bill. VPC endpoints for S3/DynamoDB delete most of it.

---

## Exercise 7: Simulate Rightsizing With Compute Optimizer

**Goal:** Practice trusting (and verifying) automated recommendations.

**Commands:** None — console work.

1. Open EC2 → Compute Optimizer (opt in if needed, it takes ~12h for data).
2. Pick one "Over-provisioned" instance.
3. Note the recommended type and projected savings.
4. Ask yourself: is this workload spiky (keep headroom) or flat (take the recommendation)?

**Expected output:** A recommendation like "m6i.xlarge → m6i.large, est. savings $X/month".

**Why it matters:** Rightsizing is a judgment call the tool can't make for you — it doesn't know your traffic pattern.

---

## Exercise 8: Write an S3 Lifecycle Policy

**Goal:** Automate storage aging so you never think about it again.

**Commands:**

```bash
# Apply a lifecycle rule: IA after 30d, Glacier after 90d, expire noncurrent after 30d
cat > /tmp/lifecycle.json <<'JSON'
{"Rules":[{"ID":"age-out","Status":"Enabled","Filter":{"Prefix":""},
"Transitions":[{"Days":30,"StorageClass":"STANDARD_IA"},{"Days":90,"StorageClass":"GLACIER"}],
"NoncurrentVersionTransitions":[{"NoncurrentDays":30,"StorageClass":"STANDARD_IA"}],
"NoncurrentVersionExpiration":{"NoncurrentDays":60},
"AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}}]}
JSON
aws s3api put-bucket-lifecycle-configuration --bucket YOUR-BUCKET --lifecycle-configuration file:///tmp/lifecycle.json
```

**Expected output:** No error — the rule is live. Verify in the console under Management → Lifecycle rules.

**Why it matters:** This one rule, applied to every log/archive bucket, is often a five-figure annual saving for a mid-size org.

---

## Exercise 9: Set a Budget Alert Before You Need It

**Goal:** Make the bill someone's problem *before* month-end.

**Commands:**

```bash
# Monthly budget of $100 with alerts at 80% and 100% (via SNS topic)
aws budgets create-budget --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{"BudgetName":"monthly-guardrail","BudgetLimit":{"Amount":"100","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers \
  '[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"you@example.com"}]}]'
```

**Expected output:** Budget created. First alert fires when actual spend crosses 80%.

**Why it matters:** A budget alert is the cheapest FinOps control there is. It turns "we spent what?!" into a Tuesday-afternoon email.

---

*Done? You've touched attribution, waste hunting, pricing models, storage lifecycle, and guardrails — the full FinOps loop.*
