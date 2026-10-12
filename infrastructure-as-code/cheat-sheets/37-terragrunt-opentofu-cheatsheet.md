# Terragrunt & OpenTofu — Cheat Sheet

*One page. Print it, pin it, glance before interviews.*

## Terragrunt commands

| Command | What it does |
|---|---|
| `terragrunt init` | Init Terraform with generated config |
| `terragrunt plan` | Plan one unit |
| `terragrunt apply` | Apply one unit |
| `terragrunt run-all plan` | Plan every unit in dependency order |
| `terragrunt run-all apply` | Apply every unit in dependency order |
| `terragrunt run-all destroy` | Destroy in reverse order |
| `terragrunt hcl fmt` | Format all hcl files |
| `terragrunt --terragrunt-config <f>` | Point at a specific config |

## terragrunt.hcl blocks

| Block | Purpose |
|---|---|
| `terraform { source = "..." }` | Point at a Terraform module |
| `include "root" { path = find_in_parent_folders() }` | Inherit parent config |
| `inputs = { ... }` | Pass variables to the module |
| `locals { ... }` | Reusable values, shared via include |
| `remote_state { backend = "s3" ... }` | One backend def for all children |
| `generate "x" { path, if_exists, contents }` | Stamp out shared files |
| `dependency "vpc" { config_path, mock_outputs }` | Read another unit's outputs |
| `before_hook "name" { commands, execute }` | Run shell before a command |
| `after_hook "name" { commands, execute }` | Run shell after a command |

**Unique state key trick:** `path = "${path_relative_to_include()}/terraform.tfstate"`

## OpenTofu commands

| Command | Terraform equivalent |
|---|---|
| `tofu init` | `terraform init` |
| `tofu plan` | `terraform plan` |
| `tofu apply` | `terraform apply` |
| `tofu test` | `terraform test` |
| `tofu fmt` | `terraform fmt` |
| `tofu state pull > backup.tfstate` | state backup |

## OpenTofu facts to drop in interviews

- Linux Foundation fork after Terraform's 2023 BSL license change — open forever
- Reads Terraform state directly; `.tofu` file extensions supported
- OpenTofu-only: state encryption at rest, provider-defined functions
- New providers published to OpenTofu registry from day one
- Migration: swap binary → empty plan → move CI → one direction at a time

## Terraform vs OpenTofu vs Terragrunt

| Tool | Role |
|---|---|
| Terraform | The original IaC engine (BSL license) |
| OpenTofu | Open fork, same HCL, no license risk |
| Terragrunt | Thin wrapper for DRY multi-env orchestration (works with both) |

## Traps

- Terragrunt does NOT lock state or replace Terraform — it wraps it
- `run-all` needs `dependency` blocks to order correctly
- Backward state reads (Tofu → old Terraform) can break — always backup
- Hooks are glue, not logic — keep real logic in modules
