# Terragrunt & OpenTofu Interview Preparation Guide

*How to Answer Terragrunt & OpenTofu Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Terragrunt Basics](#terragrunt-basics)
2. [Terragrunt Configuration](#terragrunt-configuration)
3. [DRY and Reuse with Terragrunt](#dry-and-reuse-with-terragrunt)
4. [OpenTofu Basics](#opentofu-basics)
5. [Terraform vs OpenTofu Migration](#terraform-vs-opentofu-migration)
6. [Production Workflows](#production-workflows)
7. [Interview Traps and Real Scenarios](#interview-traps-and-real-scenarios)

---

## Terragrunt Basics

### Q1: What is Terragrunt, and why would you use it instead of plain Terraform?

**How to Answer:**

"Terragrunt is a thin wrapper around Terraform that keeps your configurations DRY — don't repeat yourself. Plain Terraform forces you to copy the same backend, provider, and variable blocks into every environment folder. Terragrunt lets you define all of that once in a root config and inherit it everywhere. I use it when I have multiple environments and modules that would otherwise drift apart through copy-paste."

**Key Point:** "Terragrunt wraps Terraform to kill copy-paste across environments."

---

### Q2: How does Terragrunt actually work under the hood?

**How to Answer:**

"Terragrunt doesn't replace Terraform — it generates Terraform code and then calls the Terraform binary for you. When you run `terragrunt apply`, it merges your `terragrunt.hcl` with parent configs, generates backend and provider files on the fly, then shells out to `terraform apply`. That means everything Terraform does — plan, apply, state — still works exactly the same. It's orchestration on top, not a different engine."

**Key Point:** "Terragrunt generates config, then delegates to the real Terraform binary."

---

## Terragrunt Configuration

### Q3: What does a typical terragrunt.hcl file look like?

**How to Answer:**

"A terragrunt.hcl has three main jobs: point at a Terraform module, pass inputs, and declare backend config. I write one per environment, and each one is usually under 20 lines because the root config carries the shared stuff."

```hcl
terraform {
  source = "../../modules/vpc"
}
inputs = {
  cidr_block = "10.0.0.0/16"
  env        = "prod"
}
```

**Key Point:** "terragrunt.hcl is three things: source, inputs, and inherited backend."

---

### Q4: How does Terragrunt handle remote state better than raw Terraform?

**How to Answer:**

"In raw Terraform, every folder needs its own backend block with a unique state key — that's pure copy-paste. Terragrunt lets me define the backend once in the root config and auto-generates a unique key per folder using `path_relative_to_include()`. Each environment gets its own state file automatically, and the backend code never lives in my working directory. It deletes the repetition without changing how state actually works."

**Key Point:** "One root backend block, unique state keys generated per folder."

---

## DRY and Reuse with Terragrunt

### Q5: How do you keep Terraform code truly DRY with Terragrunt?

**How to Answer:**

"The pattern is: write reusable modules once, then reference them from dozens of terragrunt.hcl files with different inputs. My root config carries the backend, provider versions, and common locals, and child configs just include it and pass their own inputs. I also use `generate` blocks to stamp out identical provider files everywhere. The module code itself never gets duplicated — only the thin wrapper per environment."

```hcl
include "root" {
  path = find_in_parent_folders()
}
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
provider "aws" { region = "ap-south-1" }
EOF
}
```

**Key Point:** "Modules are written once, referenced from many thin terragrunt wrappers."

---

### Q6: What are dependency blocks and run-all in Terragrunt?

**How to Answer:**

"Dependency blocks let one Terragrunt config read outputs from another, like a VPC module feeding its ID into an app module. I can use `mock_outputs` so a plan works even before the dependency exists. `run-all` commands — `terragrunt run-all apply` — walk the whole folder tree, figure out the dependency order, and apply everything in sequence. It's how I deploy a full stack in one command without hardcoding ordering in CI."

**Key Point:** "Dependency blocks wire module outputs together; run-all deploys them in order."

---

### Q7: What are hooks in Terragrunt, and when do you actually use them?

**How to Answer:**

"Hooks run custom commands before or after Terraform commands — things like `before_hook` and `after_hook` in the terragrunt.hcl. I use them for stuff Terraform can't do natively, like running `tflint` before apply or invalidating a CloudFront distribution after. They're escape hatches, not everyday tools — if my logic fits in a module, it stays in the module. Hooks are for the glue work around Terraform."

**Key Point:** "Hooks run shell commands around Terraform — linting, notifications, cleanup."

---

## OpenTofu Basics

### Q8: What is OpenTofu, and why was it created?

**How to Answer:**

"OpenTofu is the open-source fork of Terraform, created after HashiCorp switched Terraform from open-source to the Business Source License in 2023. The Linux Foundation picked it up so the community has a truly open IaC engine that stays free forever. It's fully compatible with Terraform configs — my existing HCL works with almost no changes. I treat it as Terraform's open future, not a different product."

**Key Point:** "OpenTofu is Terraform's Linux Foundation fork — open forever, same HCL."

---

### Q9: What's actually different between Terraform and OpenTofu today?

**How to Answer:**

"Feature-wise they're very close, but OpenTofu has added things Terraform doesn't have. It supports dynamic provider-defined functions, state encryption at rest, and `.tofu` file extensions. The `tofu test` framework works the same as Terraform's. The biggest practical difference is philosophical: OpenTofu is community-governed with no license risk. Under the hood the syntax and workflow are near-identical, which is why switching is so easy."

**Key Point:** "Near-identical workflow, plus state encryption and no license lock-in."

---

## Terraform vs OpenTofu Migration

### Q10: How would you migrate a production Terraform setup to OpenTofu?

**How to Answer:**

"I'd do it in stages, never in one big bang. First, swap the binary — `tofu` reads Terraform state files directly, so I run `tofu init` and `tofu plan` and compare the plan against Terraform's. If the plan is empty, I know it's safe. Then I migrate the team: update CI pipelines, rename commands in docs, and gradually switch providers. State backups before every step, obviously — I never touch production state without a rollback copy."

```bash
terraform state pull > state-backup.tfstate
tofu init && tofu plan   # plan must be empty = zero drift
```

**Key Point:** "Swap the binary, compare plans, migrate in stages — backups first."

---

### Q11: Is OpenTofu's state format compatible with Terraform's?

**How to Answer:**

"Yes — OpenTofu reads Terraform state files directly because it's a fork of the same codebase. My existing `.tfstate` files work with the `tofu` binary without conversion. Going backward can be trickier: if OpenTofu writes state with a newer format version, old Terraform may refuse to read it. So I only move one direction at a time in production, and I always keep a state backup before the switch."

**Key Point:** "OpenTofu reads Terraform state directly — migrate forward, keep backups."

---

## Production Workflows

### Q12: When would you pick Terragrunt, OpenTofu, or plain Terraform in 2026?

**How to Answer:**

"I'd pick plain Terraform or OpenTofu when the setup is small — one or two environments, no repetition problem. I'd add Terragrunt when I have many environments or accounts and the copy-paste is becoming a real risk. For a fresh greenfield project in 2026, I'd go OpenTofu for the license safety unless the company is already invested in Terraform Cloud. The decision is about scale and licensing, not about which tool is 'better' in the abstract."

**Key Point:** "Small: plain Terraform. Many envs: Terragrunt. Greenfield 2026: OpenTofu."

---

### Q13: How do you run Terragrunt or OpenTofu in a CI/CD pipeline?

**How to Answer:**

"My pipeline runs `run-all plan` on pull requests and posts the plan as a comment, so reviewers see the blast radius. On merge to main, `run-all apply` runs with the dependency graph so resources deploy in the right order. State lives in S3 with DynamoDB locking, and the pipeline assumes an IAM role — no long-lived keys. For OpenTofu it's the same shape, just `tofu plan` instead of `terraform plan`."

**Key Point:** "Plan on PR, apply on merge, remote state with locking, IAM roles."

---

## Interview Traps and Real Scenarios

### Q14: What are the common traps interviewers set around Terragrunt and OpenTofu?

**How to Answer:**

"One classic: 'Does Terragrunt lock state?' No — it inherits Terraform's locking; Terragrunt is just a wrapper. Another: 'Is Terragrunt a replacement for Terraform?' No — you still need the Terraform binary installed. For OpenTofu, they'll ask 'Can Terraform read OpenTofu-written state?' Maybe not, so backups matter. And the scenario one: 'Your run-all apply failed halfway — what now?' I fix the root cause and re-run, because each unit has its own state, so reruns are safe and idempotent."

**Key Point:** "Terragrunt doesn't lock or replace Terraform; reruns are safe per-unit."
