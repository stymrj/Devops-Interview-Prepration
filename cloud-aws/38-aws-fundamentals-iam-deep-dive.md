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
