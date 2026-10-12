# AWS Fundamentals & IAM Cheat Sheet

*One-page command reference. Print it, pin it, memorize it before interviews.*

---

## Core Concepts (10-second answers)

| Term | One-liner |
|---|---|
| Region | Geographic area, e.g. `ap-south-1` — pick by latency, compliance, cost |
| AZ | Isolated data center within a Region — spread workloads across ≥2 for HA |
| Edge location | CloudFront cache point — speeds up content delivery |
| Shared responsibility | AWS secures the cloud; you secure what you put in it |
| Root user | Lock with MFA, never use daily |

## Identity Types

| Type | Use for | Credentials |
|---|---|---|
| IAM user | Humans | Long-lived (password / access key) |
| IAM group | Permission bundles for users | N/A |
| IAM role | Workloads, cross-account, humans via SSO | Temporary (STS, `ASIA...` keys) |

## Policy Evaluation (memorize this)

```
1. Default: IMPLICIT DENY everything
2. Look for an explicit DENY → if found, request DENIED (nothing overrides it)
3. Look for an explicit ALLOW → if found, request ALLOWED
4. Otherwise → DENIED
```

## Policy JSON Skeleton

```json
{"Version":"2012-10-17","Statement":[{
  "Effect":"Allow",
  "Action":["s3:GetObject"],
  "Resource":["arn:aws:s3:::bucket/prefix/*"],
  "Condition":{"StringEquals":{"aws:PrincipalTag/team":"platform"}}
}]}
```

**ARN traps:** `arn:aws:s3:::b` = bucket ops only · `arn:aws:s3:::b/*` = objects · most `*` ARNs need a justification.

## Policy Types

| Type | Attaches to | Cross-account? |
|---|---|---|
| Identity-based (managed/inline) | User, group, role | No |
| Resource-based (bucket, KMS key policy) | The resource | Yes |
| SCP (Organizations) | OU/account | Upper bound only — never grants |
| Permission boundary | User/role | Upper bound only |
| Trust policy | Role | Defines WHO can assume |

## IAM CLI Essentials

```bash
aws iam create-user --user-name alice
aws iam create-group --group-name admins
aws iam add-user-to-group --user-name alice --group-name admins
aws iam attach-group-policy --group-name admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam create-policy --policy-name p --policy-document file://p.json
aws iam attach-user-policy --user-name alice --policy-arn <arn>
aws iam create-role --role-name r --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name r --policy-arn <arn>

aws sts assume-role --role-arn <arn> --role-session-name s
aws sts get-caller-identity

aws iam generate-credential-report
aws iam get-credential-report --query Content --output text | base64 -d
aws iam list-access-keys --user-name alice
aws iam delete-access-key --user-name alice --access-key-id <AKIA...>
```

## Trust Policy Template

```json
{"Version":"2012-10-17","Statement":[{
  "Effect":"Allow",
  "Principal":{"Service":"ec2.amazonaws.com"},
  "Action":"sts:AssumeRole"
}]}
```

Cross-account with confused-deputy protection:

```json
{"Effect":"Allow",
 "Principal":{"AWS":"arn:aws:iam::999988887777:root"},
 "Action":"sts:AssumeRole",
 "Condition":{"StringEquals":{"sts:ExternalId":"vendor-acme-2026"}}}
```

## Instance Profile (EC2 gets role creds, no keys)

```bash
aws iam create-instance-profile --instance-profile-name app
aws iam add-role-to-instance-profile --instance-profile-name app --role-name r
aws ec2 associate-iam-instance-profile \
  --instance-id i-0abc123 --iam-instance-profile Name=app
# On the box: curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

## Debugging AccessDenied

```
1. CloudTrail → exact API call + error context
2. Identity policies → any explicit Deny?
3. Resource policy (bucket/KMS) → does it allow this principal?
4. SCPs + permission boundaries → invisible upper bounds
5. IAM Policy Simulator → reproduce before fixing
```

## Interview Rapid-Fire

- AuthN = who you are · AuthZ = what you may do
- Temporary keys start `ASIA`, long-lived start `AKIA`
- Roles > access keys, always · SSO for humans, instance profiles for EC2
- `Deny` beats `Allow`, always · default is deny
- Bucket policy = resource-based = can grant cross-account
- Root: MFA on, never daily · no keys in Git, ever
