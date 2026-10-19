# Lab: Secrets Management (Vault + AWS Secrets Manager)

Nine hands-on exercises. You need Docker for the Vault parts and an AWS CLI (any
role with Secrets Manager read is fine) for the AWS parts. Everything here is
safe to run locally — Vault runs in dev mode.

## Exercise 1: Start a Vault dev server

**Goal:** Get a working Vault in 30 seconds and understand dev vs prod mode.

**Commands:**

```bash
docker run -d --name vault -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=root -e VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200 \
  hashicorp/vault:latest
export VAULT_ADDR=http://127.0.0.1:8200 VAULT_TOKEN=root
vault status
```

**Expected output:** `vault status` shows `Sealed: false`, `Storage Type: inmem`.
The sealed output from Q3 doesn't apply here — dev mode auto-unseals.

**Why it matters:** Dev mode is for learning only — single in-memory backend, no
unseal, one hardcoded root token. In production you'd use Raft storage,
auto-unseal via KMS, and revoke that root token on day one.

## Exercise 2: Write and read a KV secret

**Goal:** Feel the versioned KV secrets engine.

**Commands:**

```bash
vault kv put secret/myapp/db username=appuser password='Sup3rS3cret!'
vault kv get secret/myapp/db
vault kv put secret/myapp/db username=appuser password='Rotated1!' && \
  vault kv get -version=1 secret/myapp/db
```

**Expected output:** The `-version=1` read returns the OLD password while the
current version shows the new one. KV v2 keeps full version history.

**Why it matters:** Version history is your safety net during rotation — if the
new password breaks the app, the old version is one command away.

## Exercise 3: Create a least-privilege policy and an AppRole

**Goal:** See how identity + policy scope access (the Q5 model).

**Commands:**

```bash
vault policy write myapp - <<'EOF'
path "secret/data/myapp/*" { capabilities = ["read"] }
EOF
vault write auth/approle/role/myapp token_policies="myapp" token_ttl="10m"
vault write -f auth/approle/role/myapp/secret-id | grep secret_id
ROLE_ID=$(vault read -field=role_id auth/approle/role/myapp/role-id)
echo "role_id=$ROLE_ID"
```

**Expected output:** A `secret_id` value and a `role_id` printed. Try reading
`secret/other-app/db` with a token from this role later — it will be denied.

**Why it matters:** This is the minimum viable secure pattern: a service identity
(role_id + secret_id) scoped to exactly its paths, with a 10-minute token TTL.
No long-lived creds, no shared secrets.

## Exercise 4: Simulate a dynamic database credential

**Goal:** Watch a credential be born and die on a lease.

**Commands:** (Full DB engine needs Postgres; this shows the lease mechanics with
a generic lease concept using a short-lived token instead.)

```bash
APP_TOKEN=$(vault token create -policy=myapp -ttl=60s -format=json | jq -r .auth.client_token)
VAULT_TOKEN=$APP_TOKEN vault kv get secret/myapp/db   # works
sleep 65
VAULT_TOKEN=$APP_TOKEN vault kv get secret/myapp/db   # fails
```

**Expected output:** First read succeeds, second fails with
`permission denied` — the token expired after 60 seconds.

**Why it matters:** This is the same machinery dynamic secrets use. A leaked
credential with a 1-hour TTL is a self-healing incident; a static one is a
breach investigation.

## Exercise 5: Enable the audit device

**Goal:** Prove every secret read is logged.

**Commands:**

```bash
vault audit enable file file_path=/tmp/vault-audit.log
vault kv get secret/myapp/db >/dev/null
tail -1 /tmp/vault-audit.log | python3 -m json.tool | head -20
```

**Expected output:** A JSON audit record showing `"path": "secret/data/myapp/db"`,
the client token's accessor, and a timestamp.

**Why it matters:** Audit logs are how you answer "who read the prod DB password
and when" — the Q13 interview answer, now in your terminal.

## Exercise 6: Create a secret in AWS Secrets Manager

**Goal:** Get comfortable with the AWS-side API.

**Commands:**

```bash
aws secretsmanager create-secret \
  --name lab/db/password \
  --description "Lab only" \
  --secret-string '{"username":"appuser","password":"TempPass123!"}'
aws secretsmanager get-secret-value --secret-id lab/db/password \
  --query SecretString --output text
```

**Expected output:** The JSON string echoes back. Note the response includes
`VersionStages: ["AWSCURRENT"]` — rotation moves versions through AWSCURRENT /
AWSPENDING stages.

**Why it matters:** `get-secret-value` is the primitive behind ECS task secret
injection, SDK calls, and the ESO/CSI flows. Knowing the version-stage model
answers half the rotation questions.

## Exercise 7: Read a secret the ECS way (no ECS needed)

**Goal:** Understand how IAM scopes secret access for containers.

**Commands:**

```bash
aws secretsmanager get-resource-policy --secret-id lab/db/password || echo "no resource policy (default: IAM identity policy)"
```

**Expected output:** An error like `ResourceNotFoundException` — meaning access
is controlled by the caller's IAM identity policy, not a resource policy.

**Why it matters:** In ECS, the task role's IAM policy lists the exact secret ARNs
it may read. That's how you get per-service scoping: the payments task role can
read `prod/payments/*`, nothing else.

## Exercise 8: Scan a repo for leaked secrets

**Goal:** Build the shift-left habit from Q14's playbook.

**Commands:**

```bash
pip install -q trufflehog 2>/dev/null || true
echo 'AWS_KEY=AKIAIOSFODNN7EXAMPLE' > /tmp/leaky-repo/config.env
trufflehog filesystem /tmp/leaky-repo 2>/dev/null | head -5 || \
  grep -rEn 'AKIA[0-9A-Z]{16}' /tmp/leaky-repo && echo "LEAK FOUND"
```

**Expected output:** Either trufflehog flags the fake key, or the grep finds it.
Either way: a scanner would have caught this before the PR merged.

**Why it matters:** Pre-commit hooks and CI secret scanning are the cheapest
security control in DevOps. Run them on every commit and most leak incidents
never start.

## Exercise 9: Clean up and write the incident one-pager

**Goal:** Practice the leak-response muscle from Q14.

**Commands:**

```bash
aws secretsmanager delete-secret --secret-id lab/db/password --force-delete-without-recovery
docker stop vault && docker rm vault
rm -rf /tmp/leaky-repo
```

Then write (in your own words, 5 lines max): what leaked, blast radius, what
you rotated, what systemic fix you added (e.g. trufflehog pre-commit hook).

**Expected output:** All lab resources gone. The one-pager is the deliverable —
interviewers love candidates who can describe a past leak response calmly.

**Why it matters:** Cleanup prevents lab creds becoming real leaks, and the
write-up is exactly the story format Q14 expects you to tell.
