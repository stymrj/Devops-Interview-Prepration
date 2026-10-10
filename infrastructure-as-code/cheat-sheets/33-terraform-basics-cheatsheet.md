# Terraform Basics — Cheat Sheet

Dense reference. One page, everything you reach for daily.

## Core workflow

| Command | What it does |
|---|---|
| `terraform init` | Download providers/modules, configure backend |
| `terraform init -upgrade` | Upgrade providers/modules to newest allowed versions |
| `terraform plan` | Diff desired vs actual, print the change list |
| `terraform plan -out=tfplan` | Save the plan to a file for review |
| `terraform show tfplan` | Render a saved plan in human-readable form |
| `terraform apply` | Execute the plan (re-plans at apply time) |
| `terraform apply tfplan` | Apply the exact saved plan — the CI-safe way |
| `terraform apply -auto-approve` | Skip the confirmation prompt (pipelines only) |
| `terraform destroy` | Tear down everything in state |
| `terraform fmt` | Canonical formatting of `.tf` files |
| `terraform fmt -check -recursive` | CI gate: fail if anything is unformatted |
| `terraform validate` | Syntax + internal consistency check (no cloud calls) |

## State inspection

| Command | What it does |
|---|---|
| `terraform state list` | All resource addresses in state |
| `terraform state show aws_instance.web` | Full attributes of one resource |
| `terraform state mv old new` | Rename/move a resource in state (refactors) |
| `terraform state rm aws_instance.web` | Untrack a resource without destroying it |
| `terraform refresh` | (Deprecated standalone) — refresh happens inside plan now |

## Targeting and replacement

| Command | What it does |
|---|---|
| `terraform plan -target=aws_instance.web` | Plan only this resource (rescue tool, not a workflow) |
| `terraform apply -replace="aws_instance.web"` | Force replacement on this run |
| `terraform import aws_instance.web i-0abcd` | Register existing infra into state |
| `terraform console` | REPL for testing expressions/interpolation |

## Language blocks (minimal syntax)

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "ap-south-1"
  alias  = "eu"          # extra region/account: provider = aws.eu
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  for_each      = toset(["a", "b"])      # or: count = 2
  depends_on    = [aws_iam_role_policy.app]  # last resort ordering
  lifecycle {
    prevent_destroy       = true          # refuse to destroy
    replace_triggered_by  = [aws_key_pair.deployer]
    ignore_changes        = [tags]        # drift you accept
  }
  tags = { Name = "web-${each.key}" }
}

data "aws_ami" "ubuntu" {                # read-only lookup
  most_recent = true
  owners      = ["099720109477"]
}

output "web_ip" {
  value       = aws_instance.web["a"].public_ip
  description = "Public IP of web-a"
}
```

## Version constraints

| Constraint | Meaning |
|---|---|
| `= 5.0.0` | Exactly this version |
| `~> 5.0` | `>= 5.0, < 6.0` — minor+patch OK (the sane default) |
| `~> 5.2.1` | `>= 5.2.1, < 5.3.0` — patch only |
| `>= 1.5, < 2.0` | Range |
| `.terraform.lock.hcl` | Exact pinned versions — commit it |

## Plan output shorthand

| Marker | Meaning |
|---|---|
| `+` | Create |
| `-` | Destroy |
| `~` | Update in place |
| `-/+` | Destroy then recreate (replacement) |
| `(known after apply)` | Provider-computed, unknowable until created |

## Gotchas

- Bare `apply` re-plans — always apply a saved plan file in pipelines.
- `count` removal shifts indexes and recreates neighbors — prefer `for_each`.
- `import` updates state only — you write the config yourself, then plan for zero diff.
- `taint` is deprecated — use `apply -replace="..."`.
- Commit `.terraform.lock.hcl`; never commit `terraform.tfstate` to Git.
