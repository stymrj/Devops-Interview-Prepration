# Secrets Management Interview Preparation Guide

*How to Answer Secrets Management Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Secrets Management Matters](#why-secrets-management-matters)
2. [HashiCorp Vault](#hashicorp-vault)
3. [AWS Secrets Manager](#aws-secrets-manager)
4. [Kubernetes Secrets](#kubernetes-secrets)
5. [Vault Agent and Sidecar Injection](#vault-agent-and-sidecar-injection)
6. [Rotation, Auditing and Interview Traps](#rotation-auditing-and-interview-traps)

---

## Why Secrets Management Matters

### Q1: Why can't I just store secrets in environment variables?

**How to Answer:**

"Env vars feel easy, but they're visible everywhere. Any process on the box can read `/proc/<pid>/environ`, they get dumped into crash reports and CI logs, and people paste their shell history into Slack."

"I've seen staging DB passwords leak because someone ran `env | grep DB` in a support session and screen-shared it. That was a fun week of rotating everything."

"Secrets belong in a store with access control and audit logs, fetched at runtime and kept in memory only as long as needed. Env vars are a transport, not a vault."

**Key Point:** "Env vars are visible to processes, logs, and crash dumps — use them as transport, not storage."

---

### Q2: What's the difference between encryption at rest and just base64-encoding a secret?

**How to Answer:**

"Base64 is encoding, not encryption — it's reversible by anyone, including an attacker who reads your etcd backup. Real encryption needs a key that's stored separately from the data."

"Encryption at rest means something like AES-GCM with a key kept in a KMS, with rotation and audit around it. Kubernetes Secrets in etcd are base64 by default, which is why encryption-at-rest for the cluster is a checkbox you never skip."

"The rule I give juniors: if you can decode it without a key, it's not protected."

**Key Point:** "Base64 is just encoding. Real protection = encryption with a separately stored key."

---

## HashiCorp Vault

### Q3: How does Vault actually work under the hood?

**How to Answer:**

"Vault is a central API that brokers access to secrets. Nothing reads a password from a file — your app authenticates to Vault, Vault checks a policy, and hands back the secret over TLS."

"Secrets are encrypted with an AES-GCM data key before they hit Vault's storage backend. The master key that protects everything is split into Shamir shards at init — you need a quorum of them to unseal."

"When the process restarts, Vault is sealed by design. Nobody auto-unseals in a serious setup; you use auto-unseal via a KMS so humans don't juggle shards in an incident at 3 AM."

**Key Point:** "Vault brokers secrets over an API: sealed-by-default, Shamir quorum to unseal, KMS auto-unseal in real setups."

---

### Q4: What are dynamic secrets, and why are they Vault's killer feature?

**How to Answer:**

"Static secrets rot. Dynamic secrets are generated on demand per client, with a lease and a TTL — Vault creates a real database user, hands you its credentials, and deletes the user when the lease expires."

"If the credential leaks, it stops working on its own in an hour. That's a fundamentally different security posture from a password that's been sitting in a config file for two years."

"My go-to pattern is a Postgres database secrets engine mounted with a 1-hour TTL and max 24 hours. Apps authenticate with their service identity, Vault generates the user, and rotation is free because credentials were never permanent."

```hcl
path "database/creds/myapp-readonly" {
  capabilities = ["read"]
}
```

**Key Point:** "Dynamic secrets = generated per client, expire automatically. Leaks stop working by themselves."

---

### Q5: How do Vault policies and auth methods work?

**How to Answer:**

"Auth methods answer who you are — AppRole, Kubernetes auth, AWS IAM auth. Policies answer what you're allowed to do, written as path-and-capability rules like `read` on `secret/data/payments/*`."

"Everything is least privilege. A service gets a token tied to exactly its policy, with a short TTL that it renews. No long-lived tokens floating around in CI."

"The trap interviewers love: the root token. It's for initial setup only, then you revoke it. If a root token exists in production, something is wrong."

```bash
vault policy write myapp - <<EOF
path "secret/data/myapp/*" { capabilities = ["read"] }
EOF
vault write auth/approle/role/myapp token_policies="myapp" token_ttl="1h"
```

**Key Point:** "Auth = identity, policies = permissions. Short-lived tokens, least privilege, root token revoked after setup."

---

## AWS Secrets Manager

### Q6: Secrets Manager vs SSM Parameter Store — which do I pick?

**How to Answer:**

"Parameter Store with SecureString is cheaper and fine for simple config values. Secrets Manager is built for credentials: native rotation, cross-account sharing, and random password generation out of the box."

"If the value needs automatic rotation — database passwords, API keys — it's Secrets Manager, no contest. The Lambda rotation templates for RDS are basically free rotation."

"I've used both in the same stack: parameters for feature flags and config, Secrets Manager for anything that could ruin your weekend if leaked. The cost difference is noise next to an incident."

**Key Point:** "Rotating credentials → Secrets Manager. Simple encrypted config → Parameter Store."

---

### Q7: How does automatic rotation actually work in Secrets Manager?

**How to Answer:**

"You attach a Lambda rotation function and set a schedule. On rotation, the function creates a new password, writes it to the database, then marks the new version as current — the old one stays valid during the cutover so apps never drop."

"For RDS it's nearly turnkey: single-user or multi-user rotation templates, and your app reads `AWSCURRENT` vs `AWSPENDING` version stages. With multi-user rotation you keep two sets of creds so rotation never causes downtime."

"The catch interviewers probe for: your app must handle the version switch gracefully. If it caches the password at startup and never re-reads, rotation breaks it. Design for re-reads or restart on rotation events."

```bash
aws secretsmanager rotate-secret \
  --secret-id prod/db/password \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123:function:rotator \
  --rotation-rules '{"AutomaticallyAfterDays": 30}'
```

**Key Point:** "Lambda rotates in stages (AWSCURRENT/AWSPENDING); apps must re-read or restart to survive rotation."

---

### Q8: How should an app running on AWS actually read a secret?

**How to Answer:**

"Server-side first: ECS task definitions have a `secrets` block that pulls from Secrets Manager into the container at launch, and IAM controls which task role can read which secret."

"For anything custom, the SDK `get-secret-value` call at startup or on demand. Cache in memory with a TTL — don't call the API on every request, it's slow and costs money."

"Never bake secrets into the image or pass them through CI as plaintext args. And on EKS, the Secrets Store CSI Driver mounts them as files with IAM Roles for Service Accounts, so no credential ever touches your manifest."

**Key Point:** "Pull at runtime via IAM-scoped access — ECS secrets block, SDK call, or CSI driver — never bake secrets into images."

---

## Kubernetes Secrets

### Q9: Why do people say Kubernetes Secrets are weak?

**How to Answer:**

"Because they're base64, not encrypted, by default. Anyone who can read etcd or a backup reads your secrets in plain text. That's why encryption at rest for etcd is non-negotiable."

"Also, anyone with `get secrets` RBAC in a namespace can read them all — there's no per-secret access control natively. And `kubectl describe pod` happily prints env-var secrets into terminal history and support tickets."

"I still use them for non-critical stuff, but for real credentials I reach for External Secrets Operator or the CSI driver backed by Vault or Secrets Manager. That keeps the actual secret material out of etcd entirely."

**Key Point:** "Native Secrets are base64 + broad RBAC. Enable etcd encryption, and use ESO/CSI driver for real credentials."

---

### Q10: How does the External Secrets Operator work?

**How to Answer:**

"ESO syncs secrets from an external store into Kubernetes. You define a `SecretStore` pointing at Vault or AWS, then `ExternalSecret` resources that declare which remote secret maps to which K8s Secret, with a refresh interval."

"The app sees a normal Kubernetes Secret — zero code changes. Rotation happens upstream, ESO picks it up on the refresh interval, and your pods get the new value on restart or volume re-mount."

"It's my default answer for 'how do secrets get into K8s safely' because the cluster never stores long-lived credentials and you keep one source of truth in Vault or Secrets Manager."

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-creds }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: vault-backend, kind: ClusterSecretStore }
  target: { name: db-creds }
  data:
    - secretKey: password
      remoteRef: { key: secret/data/myapp/db, property: password }
```

**Key Point:** "ESO mirrors external secrets into K8s on a refresh interval — apps see normal Secrets, cluster holds nothing long-lived."

---

## Vault Agent and Sidecar Injection

### Q11: What problem does the Vault Agent sidecar solve?

**How to Answer:**

"Without it, every app needs Vault client code and token handling. The agent sidecar runs next to your app, authenticates with the pod's identity, fetches secrets, and renders them to a file your app just reads."

"It handles token renewal and re-rendering on rotation, so the app stays dumb. For Kubernetes there's an injector that adds the sidecar automatically via annotations — no manifest surgery."

"The pattern I like: agent renders to a tmpfs volume, app reads the file. Secret material never touches etcd, and when the pod dies, the tmpfs dies with it."

```yaml
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp"
  vault.hashicorp.com/agent-inject-secret-config: "secret/data/myapp/config"
```

**Key Point:** "Sidecar authenticates and renders secrets to tmpfs — apps stay simple, secrets never touch etcd."

---

### Q12: How do you kill long-lived credentials entirely?

**How to Answer:**

"Long-lived credentials are the root of most secret incidents. The modern answer is workload identity: IRSA on EKS, Workload Identity on GKE, Managed Identity on Azure — the pod proves who it is with a signed OIDC token, no static key involved."

"Vault auth methods like Kubernetes or AWS IAM auth issue short-lived tokens from that identity. Combine with dynamic database credentials at a 1-hour TTL and there's simply nothing permanent to steal."

"It's a mindset shift: stop distributing secrets, start issuing identities. The only static credential left should be the break-glass one, and you monitor it like a fire alarm."

**Key Point:** "Replace static creds with workload identity + short-lived tokens. The only permanent secret is break-glass."

---

## Rotation, Auditing and Interview Traps

### Q13: How do you audit who accessed which secret?

**How to Answer:**

"Vault has audit devices that log every request and response — who, what path, when — to file, syslog, or a socket. You enable at least one before production, and treat those logs as sensitive because they show access patterns."

"On AWS, CloudTrail logs every `GetSecretValue` call with the caller identity. Correlate that with application logs and you know exactly which service read the prod DB password and when."

"In an interview I always add: audit logs prove access happened, but they don't prove non-access. Combine them with short TTLs and rotation so a missed log line isn't fatal."

```bash
vault audit enable file file_path=/var/log/vault-audit.log
```

**Key Point:** "Vault audit devices + CloudTrail log every secret read. Audit proves access; short TTLs cover what audit misses."

---

### Q14: Walk me through your response when a secret leaks.

**How to Answer:**

"First, assess blast radius: which secret, where did it land — a commit, a log, a screenshot? A key in a public repo is compromised the moment it's pushed, bots scan GitHub in seconds."

"Second, rotate immediately and revoke the old one — don't just delete the commit, Git history keeps it forever. Then check audit logs for suspicious use during the exposure window."

"Third, figure out the root cause and fix the system: add pre-commit secret scanning, tighten who can see the value, shorten the TTL. The postmortem asks why the secret was static and reachable, not who pasted it."

**Key Point:** "Assume instant compromise, rotate and revoke first, then fix the system that allowed it."

---

*Day 51 of 58 — Networking & Security series. Previous: [DevSecOps](50-devsecops.md). Next: Kubernetes NetworkPolicies.*
