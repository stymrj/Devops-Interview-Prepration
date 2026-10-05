# ConfigMaps, Secrets & Environment Management Interview Preparation Guide

*How to Answer Kubernetes Configuration Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [ConfigMaps](#configmaps)
2. [Secrets](#secrets)
3. [Environment Variables and Configuration Strategy](#environment-variables-and-configuration-strategy)
4. [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)

---

## ConfigMaps

### Q1: What is a ConfigMap and why shouldn't I just hardcode config into my image?

**How to Answer:**

"Hardcoding config into the image means every environment change needs a rebuild and redeploy — different ports, different feature flags, different log levels for dev vs prod. A ConfigMap lets you keep configuration as key-value data in Kubernetes and inject it into Pods, so the same image runs everywhere."

"The image becomes environment-agnostic and config becomes someone else's problem — the platform team or the deployment manifest, not the Dockerfile. If a config value changes, you update the ConfigMap and roll the Pods instead of rebuilding anything."

**Key Point:** "ConfigMaps decouple config from the image — one build, many environments."

---

### Q2: How do you get ConfigMap data into a Pod?

**How to Answer:**

"Two ways: environment variables or files. For a handful of settings, you inject individual keys as env vars with `valueFrom.configMapKeyRef`, or dump the whole ConfigMap in with `envFrom`. For bigger config like an nginx.conf or app.properties, you mount the ConfigMap as a volume and the keys become files."

"I pick env vars when the app already reads `process.env` and the config is flat key-values. I pick volumes when the app expects a real config file on disk — then I just point the mount at the path the app already looks at."

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: log.level
```

**Key Point:** "Env vars for flat settings, volumes when the app expects real config files."

---

### Q3: Do ConfigMap changes reach running Pods automatically?

**How to Answer:**

"If the ConfigMap is mounted as a volume, yes — kubelet syncs the update into the Pod's mounted files within a minute or so. But if you injected the values as env vars, no — env vars are frozen at container start, so you have to restart the Pods to pick up the change."

"This is a classic interview trap, so I always state both paths. In production I don't rely on the volume sync anyway — I use a Reloader-style controller or a rolling restart on ConfigMap change, because silent config drift with no deployment record is how you get confused at 2 AM."

**Key Point:** "Volume mounts sync on their own; env vars need a Pod restart. Trigger a rollout either way so the change is traceable."

---

### Q4: What are ConfigMap limitations I should know about?

**How to Answer:**

"The hard one is the 1 MiB size limit on the whole ConfigMap — etcd is not a file store, so people stuffing big binaries or giant JSON blobs in there hit errors fast. ConfigMaps also aren't meant for anything sensitive: they're stored in plaintext in etcd and anyone who can `kubectl get configmap` can read every value."

"If config is large, it belongs in the image or fetched from object storage. If it's sensitive, it belongs in a Secret — or better, an external secret manager. ConfigMaps are for boring, non-secret configuration."

**Key Point:** "1 MiB limit, plaintext in etcd — big data and secrets don't belong in ConfigMaps."

---

## Secrets

### Q5: What is a Secret and how is it different from a ConfigMap?

**How to Answer:**

"A Secret is Kubernetes' object for sensitive data — passwords, tokens, TLS certs. The API surface is the same as a ConfigMap: same env-var and volume injection, same per-namespace scope. The difference is intent: Kubernetes treats Secrets with slightly more care."

"Secrets are only sent to nodes that actually run Pods using them, they're stored in tmpfs when mounted as volumes so they never hit the node's disk, and RBAC makes it easier to lock down who can read them. That's it — everything else about how you consume them is identical to ConfigMaps."

**Key Point:** "Same mechanics as ConfigMaps, but scoped for sensitive data — node-restricted distribution, tmpfs volumes, stricter RBAC."

---

### Q6: Are Kubernetes Secrets actually encrypted?

**How to Answer:**

"Not by default — and this is the trap everyone falls for. The values are base64-encoded, not encrypted. Anyone with `kubectl get secret -o yaml` can run base64 decode in two seconds. Base64 is encoding, not encryption, and interviewers love asking this."

"Real protection needs two things: encryption at rest in etcd, which you enable with an EncryptionConfiguration using KMS or a local key, and strict RBAC so almost nobody can read Secrets in the first place. Defense in depth — I treat etcd encryption as table stakes, not a bonus."

**Key Point:** "Base64 is encoding, not encryption — you need etcd encryption at rest plus tight RBAC for actual secrecy."

---

### Q7: What are the rules for handling Secrets in a real cluster?

**How to Answer:**

"First: Secrets never go in git. I know everyone says it, but someone always commits a .env, so we use tools like git-secrets or pre-commit hooks as the guardrail. Second: nobody reads Secrets through kubectl casually — if you need to debug, describe the Pod, don't dump the Secret."

"Third: rotate them. Mounted Secrets as volumes do update when the Secret changes, so a Pod restart picks up rotated credentials — plan for that. And fourth: encrypt etcd at rest, because plaintext-in-etcd is a breach waiting to happen if someone grabs a snapshot."

**Key Point:** "Never in git, never casually read, rotate regularly, encrypt etcd — the four rules."

---

### Q8: Why would I use External Secrets Operator or Vault instead of native Secrets?

**How to Answer:**

"Native Secrets are fine for static values, but they don't solve rotation or centralized management. When you have dozens of microservices sharing database passwords that rotate every 30 days, hand-editing Secret manifests is a losing game."

"External Secrets Operator syncs values from Vault, AWS Secrets Manager, or similar into native Secret objects on a schedule — so the app code doesn't change, but the source of truth lives in a real secret manager with audit logs and automatic rotation. Vault adds dynamic secrets too — credentials that only exist for the lifetime of the lease, which is the gold standard for blast radius."
