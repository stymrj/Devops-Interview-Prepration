# Secrets Management Cheat Sheet

Dense reference for Vault CLI, AWS Secrets Manager, K8s secret flows, and
the interview one-liners that pair with the [full guide](../51-secrets-management.md).

## HashiCorp Vault CLI

| Task | Command |
|---|---|
| Status / seal state | `vault status` |
| Unseal (shard) | `vault operator unseal` |
| Init (one time) | `vault operator init -key-shares=5 -key-threshold=3` |
| Write KV v2 secret | `vault kv put secret/myapp/db user=x pass=y` |
| Read / old version | `vault kv get secret/myapp/db`, `-version=1` |
| List secrets | `vault kv list secret/myapp` |
| Delete (soft) / destroy | `vault kv delete secret/myapp/db` / `vault kv destroy -versions=1 ...` |
| Write policy | `vault policy write myapp policy.hcl` |
| Enable auth method | `vault auth enable approle` |
| Create role | `vault write auth/approle/role/myapp token_policies=myapp token_ttl=1h` |
| Get role-id / secret-id | `vault read auth/approle/role/myapp/role-id`, `vault write -f .../secret-id` |
| Login via AppRole | `vault write auth/approle/login role_id=.. secret_id=..` |
| Mint short token | `vault token create -policy=myapp -ttl=10m` |
| Renew / revoke token | `vault token renew`, `vault token revoke <id>` |
| Enable audit | `vault audit enable file file_path=/var/log/vault-audit.log` |
| Enable DB dynamic secrets | `vault secrets enable database` |
| Configure DB engine | `vault write database/config/pg plugin_name=postgresql-database-plugin connection_url=... allowed_roles=myapp-readonly` |
| Create DB role (dynamic creds) | `vault write database/roles/myapp-readonly db_name=pg creation_statements="..." default_ttl=1h max_ttl=24h` |
| Get dynamic creds | `vault read database/creds/myapp-readonly` |
| Lease: renew / revoke | `vault lease renew <lease-id>`, `vault lease revoke -prefix=database/creds` |

## Policy snippet (least privilege)

```hcl
path "secret/data/myapp/*" { capabilities = ["read"] }
path "database/creds/myapp-*" { capabilities = ["read"] }
path "sys/leases/renew"        { capabilities = ["update"] }
```

## AWS Secrets Manager CLI

| Task | Command |
|---|---|
| Create | `aws secretsmanager create-secret --name prod/db/password --secret-string '{"user":"x","pass":"y"}'` |
| Read | `aws secretsmanager get-secret-value --secret-id prod/db/password` |
| Read specific version | `--version-stage AWSPENDING` |
| Rotate now | `aws secretsmanager rotate-secret --secret-id prod/db/password --rotation-lambda-arn <arn> --rotation-rules '{"AutomaticallyAfterDays":30}'` |
| Schedule rotation | same command; rules drive cadence (1–365 days) |
| Update value | `aws secretsmanager put-secret-value --secret-id ... --secret-string '...'` |
| List versions | `aws secretsmanager list-secret-version-ids --secret-id ...` |
| Delete (recoverable) | `aws secretsmanager delete-secret --secret-id ... --recovery-window-in-days 7` |
| Random password | `aws secretsmanager get-random-password --password-length 32 --exclude-punctuation` |

## SSM Parameter Store (quick compare)

```bash
aws ssm put-parameter --name /myapp/db/pass --type SecureString --value 'x' --key-id alias/my-kms
aws ssm get-parameter --name /myapp/db/pass --with-decryption
```
Use Parameter Store for config; Secrets Manager for rotating credentials.

## ECS / EKS injection

```json
// ECS task definition: secrets block (ARN + env name)
"secrets": [{ "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:...:secret:prod/db/password:password::" }]
```

```yaml
# EKS: Secrets Store CSI Driver + IRSA
annotations:
  secrets-store.csi.k8s.io/usedAs: "env"   # or file mount
# ServiceAccount annotated with eks.amazonaws.com/role-arn
```

## External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata: { name: aws-backend }
spec:
  provider:
    aws: { service: SecretsManager, region: ap-south-1, auth: { jwt: { serviceAccountRef: { name: eso-sa } } } }
---
kind: ExternalSecret
metadata: { name: db-creds }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-backend, kind: ClusterSecretStore }
  target: { name: db-creds }
  dataFrom: [ { extract: { key: prod/db/password } } ]
```

## Vault Agent injector annotations

```yaml
vault.hashicorp.com/agent-inject: "true"
vault.hashicorp.com/role: "myapp"
vault.hashicorp.com/agent-inject-secret-db: "database/creds/myapp-readonly"
vault.hashicorp.com/agent-inject-template-db: |
  {{- with secret "database/creds/myapp-readonly" -}}
  DB_USER={{ .Data.username }}
  DB_PASS={{ .Data.password }}
  {{- end }}
```

## Interview one-liners

- Env vars are transport, not storage — visible in `/proc`, logs, crash dumps.
- Vault: sealed by default → Shamir quorum or KMS auto-unseal; root token revoked after setup.
- Dynamic secrets: per-client creds with TTL; leaks self-heal on expiry.
- Auth method = identity, policy = permissions, tokens short-lived.
- Secrets Manager for rotating creds (RDS templates), Parameter Store for config.
- Rotation runs AWSCURRENT → AWSPENDING stages; apps must re-read or restart.
- K8s Secrets are base64; enable etcd encryption at rest, use ESO/CSI for real creds.
- Kill static creds: workload identity (IRSA/OIDC) + short-lived tokens; only break-glass stays.
- Leaked secret = rotate + revoke immediately, check audit window, add scanning.
