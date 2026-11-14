# Secrets Management at Scale — Hands-On Lab

Nine drills that turn secrets answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Run a secrets scanner across a repo's full history

**Goal:** A leaked secret found in git history, not just in current code.

**Steps:**

```bash
# Install gitleaks, then scan the whole history of a test repo:
gitleaks detect --source . --verbose
# Now commit a fake AWS key in a test repo, remove it in the next commit,
# and scan again — confirm the scanner still finds it in history.
# Then rewrite history (git filter-repo) and scan once more to confirm it's gone.
```

**Expected output:** Findings with file, commit, and secret type on the first two scans; a clean scan after the history rewrite.

**Why it matters:** This is Q9 made concrete — "scan the history, not just HEAD" is an answer you can only give confidently once you've watched a removed secret get caught.

---

## Exercise 2: Rotate a database password with an overlap window

**Goal:** A zero-downtime rotation you can describe from memory.

**Steps:**

```bash
# On a test database, create a second user with the same grants as the app user:
# (Postgres example)
CREATE USER app_v2 WITH PASSWORD 'strong-random-password';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_v2;
# Point one app instance at app_v2, confirm it works, then check the old
# user's connections in pg_stat_activity until they hit zero.
# Only then drop the old user.
```

**Expected output:** Both users working during the overlap, old-user connections draining to zero, and no failed requests in the app logs.

**Why it matters:** This is Q4 — the overlap window stops being theory once you've watched the old credential's usage drain to zero before deleting it.

---

## Exercise 3: Generate a dynamic database credential with Vault

**Goal:** A throwaway database user that Vault creates and revokes for you.

**Steps:**

```bash
# With a local Vault dev server, enable the database secrets engine:
vault secrets enable database
vault write database/config/my-postgres \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@localhost:5432/app" \
  username="vault-admin" password="vault-admin-password"
vault write database/roles/app-readonly db_name=my-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" max_ttl="24h"
# Now ask for a credential and watch it appear in the database:
vault read database/creds/app-readonly
```

**Expected output:** A fresh username and password with a 1-hour lease, a matching role visible in the database, and the role gone after the lease expires.

**Why it matters:** This is Q5 — "Vault creates the user and revokes it" becomes a real sentence once you've watched the role appear and disappear.

---

## Exercise 4: Give a pod cloud access with no stored key (IRSA)

**Goal:** A pod that reaches AWS with workload identity, not an access key.

**Steps:**

```bash
# On a test EKS cluster, create an IAM role trusting the cluster's OIDC provider,
# scoped to one service account:
eksctl create iamserviceaccount --cluster demo --namespace default \
  --name api-sa --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
# Deploy a pod using that service account and confirm it can list a bucket
# without any AWS keys in its env:
kubectl run test --image=amazon/aws-cli --serviceaccount=api-sa -- s3 ls
# Now try the same call from a pod with a different service account — it must fail.
```

**Expected output:** The right pod lists the bucket; the wrong pod gets AccessDenied — with zero keys anywhere in the pod spec.

**Why it matters:** This is Q6 and Q7 — workload identity stops being abstract once you've seen the deny on the wrong service account.

---

## Exercise 5: Inject a secret as a file, not an env var

**Goal:** A pod that reads its secret from a mounted file that never touches etcd.

**Steps:**

```bash
# Install the Secrets Store CSI driver and the Vault provider, then create
# a SecretProviderClass pointing at your Vault path and deploy a pod that
# mounts it:
kubectl apply -f secretproviderclass.yaml
kubectl apply -f pod-with-csi-mount.yaml
# Exec into the pod and read the secret file:
kubectl exec -it <pod> -- cat /mnt/secrets/db-password
# Confirm no Kubernetes Secret object exists for it:
kubectl get secrets | grep db-password   # should return nothing
```

**Expected output:** The secret readable inside the pod, and no matching Kubernetes Secret object in the cluster.

**Why it matters:** This is Q8 — "never lands in etcd" is the line that lands in interviews, and this is how you prove you know what it means.

---

## Exercise 6: Find and fix a secret in CI logs

**Goal:** A pipeline that masks secrets — and proof a leaked one gets caught.

**Steps:**

```bash
# In a test GitHub Actions workflow, deliberately echo a fake token once
# (in a private throwaway repo), then check the run log.
# Now fix the workflow: mark the value as a secret, use ::add-mask::, and
# re-run — confirm the log shows *** instead of the value.
# Add gitleaks as a CI step so the next leak fails the build instead of merging.
```

**Expected output:** The first run's log exposing the value, the fixed run masking it, and a failing gitleaks step on a branch that adds a secret.

**Why it matters:** This is Q11 — CI is where secrets leak most quietly, and a masked log plus a failing scan is the fix you can describe step by step.

---

## Exercise 7: Run a leak-response drill

**Goal:** A timed revoke-first drill you could run in a real incident.

**Steps:**

```bash
# Create a throwaway API key, then start a timer and run the drill:
# 1. Revoke the key (cloud console or CLI).
# 2. Check the audit log for calls made with it (CloudTrail / audit log).
# 3. Search the repo and chat for sibling secrets.
# 4. Write the three-line incident note: what leaked, what you rotated,
#    what the logs showed.
# Time yourself — the whole drill should take under 15 minutes.
```

**Expected output:** A revoked key, a clean audit-log check, and a three-line note — all inside 15 minutes.

**Why it matters:** This is Q10 — "revoke first, investigate second" only sounds confident when you've done it against a clock.

---

## Exercise 8: Set up OIDC federation for a CI pipeline

**Goal:** A pipeline that gets cloud credentials with no stored token.

**Steps:**

```bash
# Configure your cloud to trust your CI provider's OIDC issuer (GitHub Actions
# example: token.actions.githubusercontent.com), scoped to one repo and branch.
# Write a workflow that requests an id-token, exchanges it for short-lived
# credentials, and lists a bucket — with no secrets stored in the CI system:
#   permissions:
#     id-token: write
#     contents: read
# Confirm the job works, then confirm a job from a different repo is rejected.
```

**Expected output:** A green run with cloud access and zero stored credentials, and a denied run from an untrusted repo.

**Why it matters:** This is Q11's CI half — OIDC federation is the answer interviewers expect for pipelines, and the deny on the wrong repo is the proof it works.

---

## Exercise 9: Write the secrets-incident postmortem

**Goal:** A one-page write-up of a leak, ending with the category fix.

**Steps:**

```bash
# Take the leak from Exercise 1 (the fake key in git history) and write it up:
# What the scanner found, what you revoked, what the audit log showed,
# and which system change removes the whole category (pre-commit scanning?
# IRSA instead of long-lived keys? OIDC for CI?).
# Keep it to one page: timeline, root cause, fix, prevention.
# Read it aloud — if it takes more than 90 seconds, cut it down.
```

**Expected output:** A one-page postmortem you could paste into an interview answer, ending with the prevention (category fix), not the revoke.

**Why it matters:** This is Q12 — the story is the interview answer, and writing it once means you tell it cleanly under pressure.
