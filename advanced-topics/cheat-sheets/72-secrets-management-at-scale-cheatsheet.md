# Secrets Management at Scale — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Secrets at scale | Storage is solved; lifecycle and blast radius are the job |
| Static secret | Created once, lives until someone rotates it |
| Dynamic secret | Generated per request, short TTL, auto-revoked |
| Secret lifecycle | Create → distribute → use → rotate → revoke |
| Rotation overlap | New credential works alongside old until old usage hits zero |
| Workload identity | The platform vouches for who a pod is — no key to steal |
| IRSA | EKS pod service account maps to an AWS IAM role |
| Workload Identity Federation | GCP's OIDC-based equivalent of IRSA |
| SPIFFE / SPIRE | Portable workload identity (SVID) across clouds and on-prem |
| Secrets Store CSI driver | Mounts vault secrets as files; nothing lands in etcd |
| External Secrets Operator | Syncs vault secrets into K8s Secrets on a schedule |
| Secrets sprawl | Secrets copied into Git, logs, chat, tickets, `.env` files |
| Break-glass | Emergency access path that is logged, time-boxed, and reviewed |
| gitleaks / trufflehog | Scan repos (including history) for leaked secrets |

## Static vs dynamic secrets

| | Static | Dynamic |
|---|---|---|
| Created | Once, by a human or script | On demand, per client |
| Lifetime | Until rotated (often never) | Minutes to hours, then revoked |
| Leak impact | Valid until someone notices | Dies on its own |
| Use when | SaaS API keys, legacy vendors | Databases, cloud creds, anything with an API |
| Cost | Rotation is manual and feared | Needs a broker like Vault |

## Rotation: the overlap window

| Step | Action |
|---|---|
| 1. Create | Second credential alongside the old one (dual-password or second user) |
| 2. Roll out | Deploy the new credential app by app, old one still valid |
| 3. Watch | Confirm old-credential usage hits zero (connection / audit logs) |
| 4. Delete | Only now revoke the old credential |
| Never | A single cutover moment where everything depends on one credential |

## Workload identity: choosing

| Option | Best when | Watch out |
|---|---|---|
| IRSA (EKS) | Mostly AWS, want nothing extra to run | AWS-only |
| Workload Identity Federation (GCP) | Mostly GCP, OIDC-native | GCP-only |
| SPIFFE / SPIRE | Multi-cloud, on-prem, or service-to-service mTLS | Another system to operate |
| OIDC federation (CI) | Pipelines need cloud access | Scope per repo and branch, never org-wide |

## Injecting secrets into pods

| Pattern | Secret lands in etcd? | Notes |
|---|---|---|
| K8s Secret as env var | Yes (base64, not encrypted) | Avoid — shows up in logs and dumps |
| External Secrets Operator | Yes (synced copy) | Better; cluster never holds the source of truth |
| Secrets Store CSI driver | No | Mounted file from the vault, can auto-rotate |
| Vault Agent Injector | No | Sidecar renders the secret as a file at pod start |

## Leak response: first hour

| Step | Action |
|---|---|
| 1. Revoke | Rotate or revoke the secret first — stop the bleeding |
| 2. Check logs | Vault audit log, CloudTrail, DB connection log — leak or breach? |
| 3. Expand scope | Same repo, channel, person — look for sibling leaks |
| 4. Tell the team | What leaked, what you rotated, what the logs showed |

## Commands to remember

```bash
# Scan a repo's full history for leaked secrets:
gitleaks detect --source . --verbose

# Ask Vault for a throwaway database credential (1h lease):
vault read database/creds/app-readonly

# Check who still uses the old database user (Postgres):
SELECT usename, count(*) FROM pg_stat_activity GROUP BY usename;

# Mount check: confirm no K8s Secret holds the credential:
kubectl get secrets | grep db-password   # expect nothing (CSI mount)

# Mask a value in GitHub Actions logs:
echo "::add-mask::$SECRET_VALUE"
```

## Interview traps

- "Vault solves secrets" — it solves storage, not lifecycle, rotation, or blast radius.
- Rotating by cutover — one deploy where everything flips at once is how you cause an outage.
- Forgetting the old credential's usage check — deleting before usage hits zero breaks the stragglers.
- Secrets in env vars — they leak into logs, debug dumps, and error reports.
- Scanning only HEAD — a secret removed from the code is still in git history.
- Investigating before revoking — every minute the credential stays valid is a minute it can be used.
- A scan without rotation — finding secrets nobody rotates is just a list of open doors.
