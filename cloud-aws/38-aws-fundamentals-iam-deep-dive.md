# AWS Fundamentals & IAM Deep Dive Interview Preparation Guide

*How to Answer AWS Fundamentals & IAM Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [AWS Core Concepts](#aws-core-concepts)
2. [Shared Responsibility Model](#shared-responsibility-model)
3. [IAM Core Identity Types](#iam-core-identity-types)
4. [IAM Policies](#iam-policies)
5. [IAM Roles and Trust Policies](#iam-roles-and-trust-policies)
6. [Cross Account Access and STS](#cross-account-access-and-sts)
7. [Interview Traps and Real Scenarios](#interview-traps-and-real-scenarios)

---

## AWS Core Concepts

### Q1: What is a Region, an Availability Zone, and an edge location?

**How to Answer:**

"A Region is a geographic area with multiple data centers — like `ap-south-1` for Mumbai. Each Region has Availability Zones, which are physically isolated data centers with independent power and networking. Edge locations are smaller points of presence that cache content for CloudFront. So Regions pick your latency and compliance zone, AZs give you fault isolation, and edge locations speed up delivery."

**Key Point:** "Regions are geographies, AZs are isolated data centers, edge locations are caching points."

---

### Q2: How do Regions and AZs affect your high availability design?

**How to Answer:**

"I never run a single-instance or single-AZ workload in production if I can avoid it. I spread instances across at least two AZs in the same Region, behind a load balancer that health-checks them. If one AZ has a power or network issue, the other keeps serving traffic automatically. Regions only come into the picture for disaster recovery — multi-Region is the next level up, and it's significantly more expensive and complex."

**Key Point:** "Multi-AZ is for HA, multi-Region is for disaster recovery — know which one the question needs."

---

### Q3: How do you pick an AWS Region for a workload?

**How to Answer:**

"I start with latency — deploy where the users are. Then compliance: some data legally has to stay in a specific country, so that can decide it for me. Cost is the third factor — prices differ slightly between Regions, and not every service is available everywhere. I usually go with the closest Region that has all the services I need at a price I'm fine with."

**Key Point:** "Latency, compliance, cost, service availability — in roughly that order."

---

## Shared Responsibility Model

### Q4: What is the AWS shared responsibility model?

**How to Answer:**

"AWS owns the security OF the cloud — the physical data centers, the hypervisors, the managed services themselves. I own security IN the cloud — my data, my IAM policies, my security groups, my OS patching. The line shifts with the service: on EC2 I patch the OS, but on RDS AWS patches the database engine for me. S3 is interesting — AWS encrypts and replicates, but I control who can read the bucket."

**Key Point:** "AWS secures the cloud itself; you secure everything you put in it."

---

### Q5: What's an example where people get the shared responsibility model wrong?

**How to Answer:**

"The classic one is S3 encryption. People say 'AWS encrypts my data, so I'm safe' — but encryption at rest doesn't protect against a public bucket. I've seen teams lose data because a bucket policy allowed anonymous access. AWS handled the encryption and replication perfectly. The breach was 100% the customer's IAM and policy config. In an interview, I say: AWS gives you the tools, but misconfigurations are yours."

**Key Point:** "Most cloud breaches are customer-side config mistakes, not AWS failures."

---

## IAM Core Identity Types

### Q6: What's the difference between an IAM user, group, and role?

**How to Answer:**

"An IAM user is a person or long-lived application identity with permanent credentials — a password or access key. A group is just a bucket of users that share permissions, so I'm not attaching policies one by one. A role is an identity with NO permanent credentials — you assume it and get temporary ones. I use users for humans, groups to organize permissions, and roles for everything else: EC2 instances, Lambda functions, cross-account access."

**Key Point:** "Users are people, groups are permission bundles, roles are temporary identities you assume."

---

### Q7: Why do you avoid long-lived access keys, and what do you use instead?

**How to Answer:**

"Access keys live forever until someone rotates them, which means they get leaked in Git repos and Slack channels all the time. AWS has literally scanned public GitHub for keys for years. Instead, I give workloads IAM roles — the SDK and CLI get temporary credentials from the metadata service automatically, with no secrets to store. For human CLI access I use SSO so there are no keys on laptops either."

**Key Point:** "Roles give short-lived credentials automatically; access keys are a leak waiting to happen."

---

## IAM Policies

### Q8: How is an IAM policy evaluated — what wins?

**How to Answer:**

"The golden rule: an explicit Deny always wins over any Allow. By default everything is implicitly denied, so a request needs at least one Allow and zero Denys across all policies that apply — identity policies, resource policies, and boundary policies. I remember it as 'deny first, then allow'. That's why troubleshooting access issues means checking every policy layer, not just the one attached to the user."

**Key Point:** "Explicit Deny beats everything; everything else starts denied."

---

### Q9: Walk me through an IAM policy JSON structure.

**How to Answer:**

"Every policy has a Version, then a Statement array. Each statement has Effect — Allow or Deny — an Action like `s3:GetObject`, a Resource ARN like `arn:aws:s3:::my-bucket/*`, and optionally a Condition for fine-tuning. The Resource is what trips people up: `my-bucket` alone covers the bucket, but you need `my-bucket/*` for the objects inside. I write policies least-privilege by default and test with the IAM policy simulator before attaching."

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": ["arn:aws:s3:::reports/*"]
  }]
}
```

**Key Point:** "Effect, Action, Resource, Condition — and the bucket ARN without /* only covers the bucket, not its objects."

---

### Q10: What's the difference between identity-based and resource-based policies?

**How to Answer:**

"Identity-based policies attach to users, groups, or roles and say what THAT identity can do. Resource-based policies attach to the resource — like an S3 bucket policy — and say who can touch it. The big practical difference: resource-based policies can grant access to identities in OTHER accounts, which identity policies can't do alone. I use identity-based policies for most things, and resource-based ones for S3, SQS, and KMS when cross-account is needed."

**Key Point:** "Identity-based policies follow the person; resource-based policies follow the resource and enable cross-account access."

---

## IAM Roles and Trust Policies

### Q11: What is a trust policy, and why does it matter?

**How to Answer:**

"A trust policy answers one question: who is allowed to assume this role? It's a resource-based policy attached to the role itself. For an EC2 role, the trusted principal is the `ec2.amazonaws.com` service. For cross-account, it's the other account's ARN. The classic interview trap: having the right permissions policy but a broken trust policy — then `sts:AssumeRole` gets denied and nothing works, even though the permissions look fine."

**Key Point:** "The trust policy decides WHO can assume the role; the permissions policy decides WHAT they can do."

---

### Q12: How do instance profiles work?

**How to Answer:**

"An instance profile is just a container that holds an IAM role for an EC2 instance. When I launch an instance with a profile, the metadata service at `169.254.169.254` serves temporary credentials to anything on the box — the AWS CLI and SDKs pick them up automatically. No keys on disk, no secrets in user-data scripts. The credentials rotate automatically, usually every few hours."

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-0abc123 \
  --iam-instance-profile Name=app-server-role
```

**Key Point:** "Instance profiles hand EC2 instances temporary role credentials — zero secrets to manage."

---

## Cross Account Access and STS

### Q13: How do you let another AWS account access your resources?

**How to Answer:**

"I create a role in MY account whose trust policy lists the other account's root or a specific role ARN as a principal. Then the other account grants its users permission to call `sts:AssumeRole` on my role's ARN. Their users assume it and get temporary credentials scoped to whatever my role's permissions policy allows. I never create IAM users in my account for someone else's team — that breaks the account boundary and creates credential sprawl."

**Key Point:** "Cross-account access flows through roles with trust policies — never share IAM users across accounts."

---

### Q14: What's the confused deputy problem, and how do you prevent it?

**How to Answer:**

"Imagine I let a service assume a role in my account on behalf of another customer. A malicious customer could trick that service into touching MY resources instead of theirs — the service is the confused deputy. The fix is the `ExternalId` condition in the trust policy: the caller has to know a secret-ish ID that only the real customer has. I always include ExternalId when a third party assumes roles in my account."

**Key Point:** "ExternalId in the trust policy stops a service from being tricked into acting on the wrong account."

---

## Interview Traps and Real Scenarios

### Q15: Walk me through debugging an AccessDenied error in production.

**How to Answer:**

"First I check CloudTrail for the exact API call and the error context — it often tells me which policy denied it. Then I verify the identity's attached policies and the resource policy for explicit Denys. I also check permission boundaries and SCPs if the account is in an Organization — those trip up a lot of people because they're invisible in the IAM console for the role. Once I find it, I reproduce with the IAM policy simulator before changing anything, then fix and re-test."

**Key Point:** "CloudTrail for the context, check every policy layer including SCPs and boundaries, simulate before you fix."

---

### Q16: Rapid-fire traps — what should I never say about IAM?

**How to Answer:**

"I never say the root user is fine for daily work — it should be locked down with MFA and almost never used. I never attach `AdministratorAccess` to a regular user or `*` resources in a production policy. I never commit access keys to Git. And I never confuse authentication — who you are — with authorization — what you're allowed to do. Interviewers ask these to see if I think like someone who's been paged at 2 AM."

**Key Point:** "No root for daily work, no wildcards in prod, no keys in Git, authn is not authz."
