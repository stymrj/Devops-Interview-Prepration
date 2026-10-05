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

**Key Point:** "External managers own the lifecycle — rotation, audit, short-lived credentials — while apps keep consuming plain native Secrets."

---

## Environment Variables and Configuration Strategy

### Q9: Why do container apps prefer environment variables for config?

**How to Answer:**

"Env vars are the lowest common denominator — every language, every framework, every container runtime supports them with zero dependencies. They're set at deploy time, so the same image works in every environment without modification."

"This is also the 12-factor app principle in action: config that varies between deploys lives in the environment, not the code. I'd say for simple key-values env vars are ideal; the moment config becomes structured or large, that's when I reach for a mounted config file instead."

**Key Point:** "Env vars are universal and deploy-time — the simplest way to keep one image for all environments."

---

### Q10: immutable ConfigMaps — when and why would I use them?

**How to Answer:**

"Marking a ConfigMap immutable means once it's created, it can never be updated — to change config you create a new version and roll the Pods to reference it. That sounds annoying but it removes a whole class of bugs: no silent config drift, no 'who changed this and when', every config change is a new versioned object."

"There's a performance win too — the API server can serve immutable ConfigMaps straight from its watch cache instead of hitting etcd, which matters at scale. I recommend immutable for stable config; keep mutable only for things that genuinely need hot updates like feature-flag style toggles."

**Key Point:** "Immutable ConfigMaps trade convenience for safety — every config change becomes a versioned, traceable rollout."

---

### Q11: How do you manage configuration across dev, staging, and production?

**How to Answer:**

"One image, environment-specific config layered on top. The base Deployment manifest references ConfigMaps and Secrets by name, and then each environment gets its own overlay — Kustomize overlays or Helm values files — that swaps in the right values."

"Common config lives in the base, environment overrides live in the overlay. And for anything shared across environments, like a central artifact registry URL, I put it once in the base so nobody redefines it per env and they inevitably drift apart."

```bash
kubectl apply -k overlays/production/     # same image, prod config
```

**Key Point:** "Build once, configure per environment with overlays — never fork the image or the manifest per env."

---

## Troubleshooting and Best Practices

### Q12: My Pod can't see an environment variable — how do you debug it?

**How to Answer:**

"First I check the obvious: `kubectl describe pod` shows exactly what env vars were resolved at startup. If the variable is missing, the usual suspects are a typo in the ConfigMap or Secret name, or a wrong key name — Kubernetes fails silently on missing optional references unless you set `optional: false`."

"If it references a ConfigMapKeyRef, I `kubectl get configmap <name>` to confirm the key actually exists. And I always remember: env vars freeze at container start. If someone changed the ConfigMap an hour ago, the running Pods won't see it — restart the Deployment and the problem usually vanishes."

**Key Point:** "Describe the Pod, verify the referenced key exists, and remember env vars freeze at startup — a restart is often the fix."

---

### Q13: Someone committed a Secret to git — what's your response?

**How to Answer:**

"The moment a secret touches git, it's compromised — full stop. You rotate it immediately, everywhere: database password, API key, whatever it was. Deleting the commit from history is cosmetic; anyone who pulled or forked already has it."

"Then I fix the process: add the pattern to gitignore, set up pre-commit scanning like gitleaks or git-secrets, and enable push protection on the repo so it can't happen again. The interview lesson: the fix is rotation first, tooling second, blame never."

**Key Point:** "A committed secret is a compromised secret — rotate first, then add guardrails, never just rewrite history."

---

### Q14: What's the one config mistake you see teams repeat in Kubernetes?

**How to Answer:**

"Shipping secrets as environment variables in CI/CD logs. Someone adds `env:` with a hardcoded value in a pipeline or a debug `echo $DB_PASSWORD` in a build step, and now the password is in the pipeline logs forever — and pipeline logs get exported, archived, and searched by more people than you'd think."

"My rule: Secrets are referenced by name in manifests, never pasted as values, and pipelines run with masked variables. If I can't `kubectl get deploy -o yaml` and see only a reference — not a value — the setup is wrong."

**Key Point:** "Secrets by reference, never by value — anywhere a secret appears as plaintext, that's the bug."
