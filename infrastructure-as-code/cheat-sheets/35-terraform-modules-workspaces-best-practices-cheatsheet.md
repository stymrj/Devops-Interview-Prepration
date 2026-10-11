# Terraform Modules, Workspaces & Best Practices — Cheat Sheet

Dense reference. One page, everything you reach for daily.

## Module block

| Snippet | What it does |
|---|---|
| `module "vpc" { source = "./modules/vpc" }` | Call a local module |
| `source = "terraform-aws-modules/vpc/aws"` | Public registry module |
| `version = "~> 5.0"` | Pessimistic version constraint on registry modules |
| `source = "git::https://host/org/repo.git?ref=v1.2.0"` | Git module pinned to a tag |
| `for_each = toset(["a", "b"])` | One module instance per key (preferred over count) |
| `module.vpc.vpc_id` | Read a module output in root config |

## Module structure

| File | Contents |
|---|---|
| `main.tf` | The resources the module manages |
| `variables.tf` | Typed inputs with descriptions and safe defaults |
| `outputs.tf` | Values callers need (IDs, endpoints) |
| `versions.tf` | `required_version` + `required_providers` pins |
| `README.md` | Usage example — modules without docs rot |
| `examples/` | Working sample configs (double as test fixtures) |

## Variables & outputs

| Snippet | What it does |
|---|---|
| `variable "x" { type = string }` | Always type variables — string, list(string), map, object |
| `sensitive = true` | Hide value in plan/apply output (still plaintext in state) |
| `validation { ... }` | Fail fast on bad input, e.g. `condition = length(var.cidr) > 0` |
| `output "id" { value = aws_vpc.this.id }` | Expose computed values to the caller |
| `terraform.workspace` | Current workspace name — use sparingly in config |

## Workspaces

| Command | What it does |
|---|---|
| `terraform workspace new dev` | Create and switch to a new workspace |
| `terraform workspace list` | Show all workspaces (`*` marks active) |
| `terraform workspace select prod` | Switch workspace |
| `terraform workspace show` | Print current workspace name |
| `terraform workspace delete dev` | Delete a workspace (state must be empty) |
| `terraform.tfstate.d/dev/` | Where workspace states live (local backend) |

## Safe refactors

| Snippet | What it does |
|---|---|
| `moved { from = a, to = b }` | Rename a resource/module in code without recreating |
| `terraform state mv module.old module.new` | CLI equivalent of moved (imperative, not reviewable) |
| `terraform plan` after refactor | Must show zero changes — the refactor test |

## Production best practices (checklist)

| Rule | Why |
|---|---|
| Remote state with locking | No local state in teams, ever |
| Pin providers + module versions | Repeatable applies, no surprise upgrades |
| One directory per environment | Real separation; workspaces are not prod separation |
| `terraform plan` in CI on every PR | Review infra diffs like code diffs |
| `.gitignore` state + `.terraform/` | State never enters Git |
| Modules do one thing | Split at ~200 lines; no 40-variable god modules |
| Registry or tagged Git sources | Never point at `main` branch |
