# Terraform State Management — Cheat Sheet

Dense reference. One page, everything you reach for daily.

## Backend configuration

| Snippet | What it does |
|---|---|
| `backend "s3" { bucket, key, region, encrypt, use_lockfile = true }` | S3 backend with native locking (TF ≥ 1.10) |
| `terraform init -backend-config=backend.hcl` | Feed backend values from a file (backends can't use variables) |
| `terraform init -migrate-state` | Copy state to a new/changed backend |
| `terraform init -reconfigure` | Skip migration, start with a fresh empty state |
| `terraform init -backend=false` | Init without touching any backend |

## State inspection (read-only, always safe)

| Command | What it does |
|---|---|
| `terraform state list` | All resource addresses in state |
| `terraform state list 'aws_instance.*'` | Filter addresses by pattern |
| `terraform state show aws_instance.web` | Full attributes of one resource |
| `terraform state pull` | Download raw state JSON |
| `terraform state pull \| jq '.serial'` | Current state serial number |
| `terraform show` | Human-readable rendering of state + config |
| `terraform show -json tfplan` | Machine-readable plan for tooling |

## State surgery

| Command | What it does |
|---|---|
| `terraform state mv old new` | Re-address a resource (safe renames/refactors) |
| `moved { from = a, to = b }` | Declarative version of state mv — reviewable in PRs |
| `terraform state rm aws_instance.web` | Stop managing a resource without destroying it |
| `terraform import aws_instance.web i-0abcd` | Register existing infra into state |
| `import { to = aws_instance.web, id = "i-0abcd" }` | Declarative import block (TF ≥ 1.5) |
| `terraform refresh` | Deprecated standalone — refresh happens inside plan |

## Locking

| Command / Setting | What it does |
|---|---|
| `use_lockfile = true` | S3-native locking (no DynamoDB needed, TF ≥ 1.10) |
| `dynamodb_table = "tf-locks"` | Legacy DynamoDB lock table (pre-1.10 pattern) |
| `terraform force-unlock <lock-id>` | Release a stuck lock after confirming holder is dead |
| `terraform plan -lock=false` | Skip locking (rescue/debugging only — never in prod) |
| `terraform apply -lock-timeout=5m` | Wait up to 5 min for the lock instead of failing fast |

## S3 backend essentials

| Item | Production setting |
|---|---|
| Bucket | Dedicated, one purpose only |
| Versioning | Enabled — your state backup |
| Encryption | SSE on (`encrypt = true`) |
| Public access | Fully blocked |
| Key layout | `env/service/terraform.tfstate` per stack |
| Locking | `use_lockfile = true` (or DynamoDB on old setups) |

## Never / always

| ❌ Never | ✅ Always |
|---|---|
| Commit `*.tfstate` / `*.tfstate.backup` to Git | Add them to `.gitignore` |
| `force-unlock` a lock you haven't verified as dead | Check CI + team first, then unlock |
| `state rm` without a PR note explaining why | Document why a resource leaves management |
| Apply against local state in a team | Remote backend + locking before first deploy |
| Reuse one state file for everything | Split state by blast radius (env/service) |

## One-liner triage

- Lock error, nobody running → verify CI is idle, then `terraform force-unlock <id>`
- Rename refactor, no recreation → `state mv` or a `moved {}` block
- Console drift on next plan → decide: recreate, or `state rm` what leaves management
- Corrupted state → restore previous S3 version (`get-object --version-id`), plan, review
- Lost state, no backup → `terraform import` resource by resource (the painful lesson)
