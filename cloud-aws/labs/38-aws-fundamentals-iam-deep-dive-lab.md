# AWS Fundamentals & IAM Hands-On Lab

*10 exercises to make IAM click — run these in a sandbox AWS account.*

---

## Lab 1: Create an IAM User and Enable MFA

**Goal:** Set up a human user the right way — no root usage.

**Commands:**

```bash
aws iam create-user --user-name dev-alice
aws iam create-login-profile --user-name dev-alice --password 'TempPass123!' --password-reset-required
```

**Expected output:** A JSON block showing the user's ARN: `arn:aws:iam::<ACCOUNT>:user/dev-alice`.

**Why it matters:** Interviewers expect you to know that humans get IAM users, never shared credentials, and never the root account. MFA is table stakes.

---

## Lab 2: Attach a Least-Privilege Policy to a Group

**Goal:** Give the user read-only EC2 access via a group, not direct attachment.

**Commands:**

```bash
aws iam create-group --group-name ec2-readers
aws iam attach-group-policy --group-name ec2-readers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
aws iam add-user-to-group --user-name dev-alice --group-name ec2-readers
```

**Expected output:** Empty JSON responses on success; `list-groups-for-user` shows `ec2-readers`.

**Why it matters:** Groups keep permissions manageable. Interviewers ask "would you attach policies directly to users?" — the answer is no, use groups or roles.

---

## Lab 3: Write and Test a Custom Least-Privilege Policy

**Goal:** Write a policy that allows reading objects only from one S3 prefix.

**Commands:**

```bash
cat > reports-read.json << 'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": ["arn:aws:s3:::company-reports/2026/*"]
  }]
}
JSON
aws iam create-policy --policy-name reports-read --policy-document file://reports-read.json
```

**Expected output:** The policy ARN, e.g. `arn:aws:iam::<ACCOUNT>:policy/reports-read`.

**Why it matters:** Writing JSON policies from scratch is a common interview exercise. Note the `/2026/*` prefix — without it, the Resource only covers the bucket itself.

---

## Lab 4: Create an IAM Role for EC2

**Goal:** Build a role with a trust policy that only EC2 can assume.

**Commands:**

```bash
cat > trust.json << 'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
JSON
aws iam create-role --role-name app-server-role --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name app-server-role \
  --policy-arn arn:aws:iam::<ACCOUNT>:policy/reports-read
```

**Expected output:** The role JSON with its ARN. Attach an instance profile and launch an EC2 instance with it — the instance can read the S3 prefix with zero stored keys.

**Why it matters:** This is THE pattern for workload identity. Saying "I'd use an instance profile" in an interview only lands if you can explain what it does.

---

## Lab 5: Assume a Role with the AWS CLI (STS)

**Goal:** See temporary credentials in action.

**Commands:**

```bash
aws sts assume-role --role-arn arn:aws:iam::<ACCOUNT>:role/app-server-role \
  --role-session-name lab-test
export AWS_ACCESS_KEY_ID=<from output>
export AWS_SECRET_ACCESS_KEY=<from output>
export AWS_SESSION_TOKEN=<from output>
```

**Expected output:** An `AccessKeyId` starting with `ASIA...` (temporary keys start with ASIA, long-lived ones with AKIA), plus an `Expiration` timestamp.

**Why it matters:** ASIA vs AKIA is a classic interview detail. Temporary credentials expire — that's the whole security point.

---

## Lab 6: Break It, Then Fix It — The Missing /* Trap

**Goal:** Feel the difference between bucket-level and object-level ARNs.

**Commands:**

```bash
aws s3 ls s3://company-reports/   # bucket-level ListBucket — fails
aws s3 cp s3://company-reports/2026/q3.pdf /tmp/   # object-level GetObject — works
```

**Expected output:** The `ls` fails with AccessDenied (policy only allows GetObject on objects), while the `cp` succeeds.

**Why it matters:** Interviewers love this trap. `arn:aws:s3:::bucket` ≠ `arn:aws:s3:::bucket/*`. Saying it is cheap; having debugged it is proof.

---

## Lab 7: Simulate Policies Before Deploying

**Goal:** Use the IAM policy simulator to validate access without guessing.

**Commands:** In the console: IAM → Policy Simulator → select `dev-alice` → simulate `s3:GetObject` on `arn:aws:s3:::company-reports/2026/q3.pdf`.

**Expected output:** An "allowed" or "denied" verdict with the exact statement that decided it.

**Why it matters:** "I tested it in the policy simulator" is the professional answer to "how would you verify this policy works?" Guessing in prod is not.

---

## Lab 8: Set a Password Policy and Audit Keys

**Goal:** Enforce MFA-friendly basics and find stale credentials.

**Commands:**

```bash
aws iam update-account-password-policy --minimum-password-length 14 \
  --require-symbols --require-numbers --require-upper-characters --require-lower-characters
aws iam generate-credential-report
aws iam get-credential-report --query Content --output text | base64 -d | cut -d, -f1,11
```

**Expected output:** A CSV listing users and their `password_mfa_active` status — spot anyone without MFA.

**Why it matters:** Credential audits come up in security-minded interviews. The credential report is the fastest way to find keys older than 90 days or users without MFA.

---

## Lab 9: Cross-Account Role with ExternalId

**Goal:** Build a role another account can assume, protected against confused deputy.

**Commands:**

```bash
cat > cross-trust.json << 'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::999988887777:root"},
    "Action": "sts:AssumeRole",
    "Condition": {"StringEquals": {"sts:ExternalId": "vendor-acme-2026"}}
  }]
}
JSON
aws iam create-role --role-name vendor-support --assume-role-policy-document file://cross-trust.json
```

**Expected output:** The role ARN. The other account can now assume it only by passing the matching ExternalId.

**Why it matters:** The confused deputy problem is an interview favorite. ExternalId is the one-line answer they want to hear.

---

## Lab 10: Deny Wins — Prove It to Yourself

**Goal:** Watch an explicit Deny override a wildcard Allow.

**Commands:**

```bash
# Attach AdministratorAccess AND this policy to a test user, then try the delete:
cat > deny-delete.json << 'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": ["s3:DeleteBucket"],
    "Resource": ["*"]
  }]
}
JSON
aws iam create-policy --policy-name deny-bucket-delete --policy-document file://deny-delete.json
```

**Expected output:** Even with admin rights, `aws s3 rb s3://company-reports` fails with AccessDenied.

**Why it matters:** This is the single most important IAM evaluation rule. Demonstrating it beats reciting it.

---

## Cleanup

```bash
aws iam delete-user --user-name dev-alice   # after removing group membership + login profile
aws iam delete-group --group-name ec2-readers
aws iam delete-role --role-name app-server-role
aws iam delete-role --role-name vendor-support
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT>:policy/reports-read
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT>:policy/deny-bucket-delete
```
