# Infrastructure Testing & Drift Detection — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Infrastructure testing | Prove the infra works before apply, not after an incident |
| Testing pyramid | Static checks → plan assertions → a few real applies |
| Static analysis | Read the code without running it (tflint, Checkov) |
| Plan testing | Assert on proposed state via `terraform show -json` |
| Live testing | Apply for real and poke the resource (Terratest) |
| Terratest | Go library: apply, assert on live resources, destroy |
| tflint | Lints Terraform syntax, variables, provider rules |
| Checkov / tfsec | Scan configs for insecure settings (public buckets, no encryption) |
| Conftest | Runs Rego policies against plan JSON before apply |
| OPA / Rego | Policy language — "no SSH from 0.0.0.0/0" as code |
| Ephemeral environment | Per-PR copy of the stack, destroyed on merge |
| Drift | Real infra no longer matches what the code says |
| Drift detection | Scheduled `plan`, driftctl, AWS Config |
| Drift remediation | Import it if right, revert it if wrong, pin it if the provider moved |

## Testing pyramid, at a glance

| Layer | Tool examples | Speed | Catches |
|---|---|---|---|
| Static | `terraform fmt`, `validate`, tflint, Checkov | Seconds | Syntax, style, insecure defaults |
| Plan | `terraform show -json`, Conftest, OPA | Seconds–minutes | Logic errors, policy violations |
| Live | Terratest, ephemeral environments | Minutes | Real provider behaviour, wiring mistakes |

## The tools, at a glance

| Tool | Job |
|---|---|
| tflint | Terraform-specific linting |
| Checkov | Security scanning across IaC formats |
| tfsec | Terraform security scanner |
| Conftest | Test any JSON/YAML against Rego policies |
| Terratest | Apply → assert → destroy in Go |
| driftctl | Compare Terraform state to the live cloud |
| AWS Config | Rules and inventory for unmanaged resources |
| CloudQuery | Inventory everything as SQL-queryable data |

## Drift: detect → decide → fix

| Step | Action |
|---|---|
| Detect | Scheduled `terraform plan -detailed-exitcode` (exit 2 = changes); driftctl scan; AWS Config rules |
| Decide | Was the change correct? Import it. A mistake? Revert it. Provider-side? Pin the version |
| Fix the system | Add the Conftest policy + the scheduled check so it can't recur silently |

## Commands to remember

```bash
# Format, validate, plan — the CI order:
terraform fmt -check -recursive && terraform validate && terraform plan

# Dump a plan as JSON and query it:
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > plan.json
jq '.planned_values.root_module.resources[] | .address' plan.json

# Test a policy against the plan:
conftest test plan.json --policy policy/ --namespace main

# Scheduled drift check (exit code 2 = drift present):
terraform plan -detailed-exitcode

# Compare state to live infra:
driftctl scan --from tfstate://terraform.tfstate
```

## Interview traps

- "Plan is enough" — plan shows intent, not correctness; it never proves the real resource works.
- Testing the policy itself — a Rego rule you never tested against a bad fixture may block nothing.
- Blindly reverting drift — check first whether the console change was a correct hotfix that should be imported.
- Forgetting the destroy — a Terratest without `defer terraform.Destroy` leaves paid resources running.
- Ephemeral envs without a TTL — a per-PR stack nobody destroys becomes permanent shadow infra.
