# S3 (Storage Classes, Lifecycle, Versioning) Interview Preparation Guide

*How to Answer S3 Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [S3 Buckets and Object Model](#s3-buckets-and-object-model)
2. [Storage Classes](#storage-classes)
3. [Lifecycle Policies](#lifecycle-policies)
4. [Versioning](#versioning)
5. [Encryption and Data Protection](#encryption-and-data-protection)
6. [Access Control](#access-control)
7. [Performance and Real-World Patterns](#performance-and-real-world-patterns)
8. [Interview Traps and Real Scenarios](#interview-traps-and-real-scenarios)

---

## S3 Buckets and Object Model

### Q1: What is S3, and why is it the default storage for almost everything in AWS?

**How to Answer:**

"S3 is object storage — you put files in as whole objects with a key, and you get them back with that same key. It's eleven nines of durability because every object is replicated across at least three AZs, so I don't design backup into the storage layer itself. Almost everything lands there: Terraform state, logs, backups, static sites, artifacts. I use it as the backbone because it's region-scoped, serverless, and everything else in AWS integrates with it."

**Key Point:** "S3 is durable, serverless object storage — it backs Terraform state, logs, artifacts, and backups."

### Q2: Buckets are global names but regional resources — what does that actually mean?

**How to Answer:**

"Bucket names are globally unique across all of AWS, so `satyam-logs` is taken if anyone anywhere has it. But the bucket itself lives in one region I choose, and data never leaves that region unless I replicate it. That's why I pick regions for compliance and latency — data residency is a real interview topic. I also prefix bucket names with the project or org, because collisions on common names are constant."

```bash
aws s3api create-bucket --bucket myorg-app-logs \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
```

**Key Point:** "Globally unique name, regional home — the bucket lives in one region and data stays there by default."

---

## Storage Classes

### Q3: Walk me through the S3 storage classes — when do you use which?

**How to Answer:**

"Standard is the default: millisecond reads, 99.99% availability, for anything hot. Intelligent-Tiering watches access patterns and moves objects between frequent and infrequent tiers automatically, and I use it for data with unpredictable access. Standard-IA and One Zone-IA are for backups and disaster recovery copies I'll rarely read — cheaper storage, but you pay more per read. Glacier Instant, Flexible, and Deep Archive are archive tiers with retrieval times from milliseconds to 12 hours — Deep Archive is the cheapest thing in S3 and I use it for compliance data I'll almost never touch."

**Key Point:** "Hot data gets Standard, unpredictable gets Intelligent-Tiering, backups get IA, archives get Glacier tiers."

### Q4: What's the real difference between Standard-IA and One Zone-IA?

**How to Answer:**

"One Zone-IA keeps a single copy in one AZ — it's cheaper than Standard-IA but one AZ failure loses the data. I only use it for data I can recreate, like processed logs or build artifacts. Standard-IA is multi-AZ like everything else in S3, so it's fine for backups and DR copies. Interviewers ask this to check whether you trade durability for pennies. My rule: if I can't recreate it, it never goes in One Zone-IA."

**Key Point:** "One Zone-IA is cheaper but single-AZ — only for recreatable data, never for the only copy."

### Q5: What is S3 Intelligent-Tiering, and does it actually save money?

**How to Answer:**

"It monitors every object's access pattern and automatically moves it between Frequent, Infrequent, Archive Instant, and Deep Archive tiers. The catch is a small per-object monitoring fee, so it's pointless for millions of tiny objects — the fee eats the savings. I use it for large, unpredictable datasets like data lakes and user uploads. There are no retrieval fees when it moves down tiers, which is what makes it genuinely hands-off."

**Key Point:** "Intelligent-Tiering auto-moves objects by access pattern — great for unpredictable data, wasteful for tiny objects."

---

## Lifecycle Policies

### Q6: How do lifecycle policies work? Give me a real example.

**How to Answer:**

"Lifecycle rules transition objects between storage classes or expire them, based on age or version count. My standard pattern: app logs stay in Standard for 30 days, move to Standard-IA for 60 more, then to Glacier Flexible for a year, then expire. I always include noncurrent-version rules too — delete noncurrent versions after 90 days, or versioning turns into a storage-cost leak. One trap: transitions to Standard-IA and Glacier have a minimum billable object size of 128KB and 40KB respectively, so lifecycle on tiny objects actually costs more."

```json
{
  "Rules": [{
    "ID": "logs-tiering",
    "Status": "Enabled",
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER"}
    ],
    "Expiration": {"Days": 365}
  }]
}
```

**Key Point:** "Lifecycle rules age objects into cheaper classes and expire them — and always cover noncurrent versions."

### Q7: How do lifecycle rules interact with versioning?

**How to Answer:**

"When versioning is on, a 'delete' just adds a delete marker — the data stays, and the lifecycle Expiration action applies to current versions while NoncurrentVersionExpiration applies to old ones. I set both, because without the noncurrent rule, old versions pile up silently and the bill creeps up every month. There's also a rule to remove expired delete markers, which cleans up markers whose versions are all gone. In interviews I always mention both current and noncurrent rules — forgetting noncurrent is the classic answer the interviewer is waiting for."

**Key Point:** "Lifecycle needs separate rules for current and noncurrent versions, or versioned data never really goes away."

---

## Versioning

### Q8: Why would you enable versioning on a bucket?

**How to Answer:**

"Versioning keeps every version of every object, so an accidental overwrite or delete is recoverable. For Terraform state buckets it's non-negotiable — a corrupted state file gets rolled back to the last good version instead of breaking every environment. The trade-off is storage cost, since every version is billed at full object size, so I pair it with noncurrent-version lifecycle rules. I also turn on MFA Delete for state buckets, which requires a hardware MFA code to permanently delete a version."

**Key Point:** "Versioning makes overwrites recoverable — essential for state files, always paired with lifecycle cleanup."

### Q9: Does enabling versioning encrypt old versions, and what happens when you delete?

**How to Answer:**

"Old versions keep whatever encryption and storage class they had — versioning doesn't change or re-encrypt anything retroactively. A normal delete adds a delete marker on top and hides the object, but the versions and the marker all stay and all get billed. To truly delete, I delete each version ID explicitly. Interviewers love this: 'I deleted the object, why is the bill the same?' — because delete markers aren't deletions with versioning on."

```bash
aws s3api list-object-versions --bucket my-bucket --prefix app.log
aws s3api delete-object --bucket my-bucket --key app.log --version-id abc123
```

**Key Point:** "With versioning on, delete just hides the object behind a marker — the data and the bill stay until versions are removed."

---

## Encryption and Data Protection

### Q10: How do you encrypt data in S3, and which option do you pick in practice?

**How to Answer:**

"I use SSE-S3 for most buckets — Amazon-managed keys, zero work, and it's the default since AWS enabled it by default. SSE-KMS is what I pick when I need audit trails and key control, like for customer data or anything compliance touches, and it logs every decrypt to CloudTrail. I avoid SSE-C because managing my own keys in headers is operational pain with no upside in most shops. Bucket Key cuts KMS costs dramatically for high-volume buckets by deriving a per-bucket data key, and I always enable it when using SSE-KMS."

**Key Point:** "SSE-S3 for the default, SSE-KMS with Bucket Key for anything audited — never manage your own keys without a reason."

### Q11: What's the difference between SSE-S3 and client-side encryption?

**How to Answer:**

"SSE-S3 encrypts server-side: I send plaintext over TLS and AWS encrypts at rest with managed keys. Client-side encryption means I encrypt before upload and S3 only ever sees ciphertext — I hold the keys, I own the risk. I only do client-side when a regulation or customer demands that AWS never sees plaintext, like some fintech or health workloads. For everything else server-side is enough, because data is encrypted on disk and in transit either way."

**Key Point:** "Server-side is the norm — client-side only when compliance requires that AWS never sees plaintext."

---

## Access Control

### Q12: How do you lock down an S3 bucket properly?

**How to Answer:**

"I start with Block Public Access on at the account level — four settings, all on, and I have never found a legitimate reason to turn them off. Private access goes through IAM policies and bucket policies; the bucket policy handles cross-account and service access. I use explicit deny statements for anything sensitive, like denying non-TLS requests with `aws:SecureTransport`. And I enable access logging and CloudTrail data events, because when someone asks 'who read that object?' I need an answer."

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
  "Condition": {"Bool": {"aws:SecureTransport": "false"}}
}
```

**Key Point:** "Block Public Access on, bucket policy for cross-account, explicit deny for non-TLS — and log everything."

### Q13: Pre-signed URLs — when do you use them and what's the security gotcha?

**How to Answer:**

"I use pre-signed URLs when my app needs to give a user temporary access to a private object — a download link that expires in 15 minutes, generated with my credentials but usable by anyone holding it. The gotcha is exactly that: whoever holds the URL has the access, so I keep expiry short and generate them per-request, never cached. They also work for uploads, which is how I let users PUT files directly to S3 without giving them IAM credentials."

```bash
aws s3 presign s3://my-bucket/report.pdf --expires-in 900
```

**Key Point:** "Pre-signed URLs grant temporary access to private objects — keep expiry short and generate per request."

---

## Performance and Real-World Patterns

### Q14: S3 has effectively unlimited throughput — so why do people still hit performance problems?

**How to Answer:**

"Throughput scales per prefix, not per bucket — AWS gives you 5,500 reads per second per prefix, so all objects under one 'folder' share that limit. The fix is what we all do naturally: random prefixes like UUIDs, which spread load across partitions. I also avoid thousands of tiny LIST operations because they're slow and billed. For big analytics workloads I use S3 Select or Athena to query in place instead of downloading everything."

**Key Point:** "S3 scales per prefix at 5,500 reads per second — randomize prefixes and don't hammer one 'folder'."

---

## Interview Traps and Real Scenarios

### Q15: "Our S3 bill doubled overnight. Walk me through your debugging."

**How to Answer:**

"First I'd check Cost Explorer broken down by S3 usage type — storage, requests, data transfer. If storage spiked, I'd look at incomplete multipart uploads with a lifecycle abort rule missing, because abandoned uploads are the classic silent bill inflator. If requests spiked, I'd look for versioning without noncurrent cleanup, or an app listing a bucket in a loop. I'd also check data transfer out, because cross-region replication and egress add up fast. Then I'd fix the root cause with lifecycle rules and CloudWatch billing alarms so it pages before it doubles."

**Key Point:** "Bill spikes come from abandoned multipart uploads, unmanaged versions, or runaway requests — lifecycle rules fix all three."
