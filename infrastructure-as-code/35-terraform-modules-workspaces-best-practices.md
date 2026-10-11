# Terraform Modules, Workspaces & Best Practices Interview Preparation Guide

*How to Answer Terraform Modules, Workspaces & Best Practices Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Module Basics](#module-basics)
2. [Writing a Module](#writing-a-module)
3. [Module Sources and Versioning](#module-sources-and-versioning)
4. [Workspaces](#workspaces)
5. [Best Practices and Interview Traps](#best-practices-and-interview-traps)

---

## Module Basics

### Q1: What is a Terraform module, and why should I use one?

**How to Answer:**

"A module is just a reusable package of Terraform configuration — any directory with `.tf` files is a module, including your root config. You use them to stop copy-pasting the same VPC or EC2 setup across environments and teams. A good module takes inputs as variables, does one thing well, and exposes outputs. The payoff is consistency: every team spins up infra the same way, and a bug fix in the module fixes it everywhere at once."

**Key Point:** "A module is a reusable Terraform package — inputs in, outputs out — that kills copy-paste and makes infra consistent across teams."

---

### Q2: How do you actually call a module in your config?

**How to Answer:**

"With a `module` block — you give it a label, a `source`, and the variable values it needs. The source can be a local path, the Terraform Registry, or a Git URL. After `terraform init` downloads it, you reference its outputs like `module.vpc.vpc_id`. One trap: the label is how you address everything inside, so pick names that say what it is, not what it calls."

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"
}
```

**Key Point:** "A module block needs a label, a source, and inputs — init pulls it, and outputs are reached through the label."

---

## Writing a Module

### Q3: How should a module be structured?

**How to Answer:**

"Small and predictable: `main.tf` for resources, `variables.tf` for inputs, `outputs.tf` for what it exposes, and a `README.md` so people know how to use it. Versions of everything go in `versions.tf` with `required_version` and `required_providers` pinned. Keep it to one concern — a module that makes a VPC shouldn't also deploy your app. If the directory gets over a few hundred lines, it's probably two modules hiding as one."

**Key Point:** "main, variables, outputs, versions, README — one concern per module, split it if it grows past a couple hundred lines."

---

### Q4: How do inputs and outputs work in modules?

**How to Answer:**

"Inputs are `variable` blocks — you declare a type, a description, and sensible defaults where they're safe. Outputs are `output` blocks exposing computed values like IDs or endpoints the caller needs. Always type your variables — `string`, `list(string)`, `map(string)` — because untyped variables become a debugging nightmare. And mark sensitive values with `sensitive = true` so they don't print in plan output."

```hcl
variable "cidr" {
  type        = string
  description = "CIDR block for the VPC"
}

output "vpc_id" {
  value       = aws_vpc.this.id
  description = "ID of the created VPC"
}
```

**Key Point:** "Variables typed with descriptions go in, outputs come out — typed inputs and marked-sensitives keep modules safe to share."

---

### Q5: Can you use count or for_each on a module?

**How to Answer:**

"Yes — a module block supports both, which is handy for spinning up, say, one module instance per environment. `for_each` is almost always the better pick because instances are keyed by something meaningful, so adding one later doesn't shuffle every address. With `count`, removing index 1 re-keys everything after it and Terraform wants to recreate things. Inside the module, resources see `each.key` and `each.value` just like anywhere else."

**Key Point:** "for_each on a module keys instances by name, count re-keys by index — for_each is the safe choice for anything you might add to later."

---

## Module Sources and Versioning

### Q6: Where can a module's source point, and how do you pick?

**How to Answer:**

"Four main places: a local path for modules you own in the same repo, the public Terraform Registry for battle-tested community modules, a private registry for your org's shared modules, or a Git URL — with a `?ref=` tag — for anything you keep in a repo. Registry sources get a `version` constraint; Git gets a ref. The interview trap is pointing at `main` branch in Git — that's a moving target, so every apply can surprise you."

```hcl
module "s3" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "~> 4.0"   # semver constraint — patches and minors, not majors
}
```

**Key Point:** "Local, registry, private registry, or Git — pin every remote source to a version or ref so applies never surprise you."

---

### Q7: How do you version and test a module before the whole org depends on it?

**How to Answer:**

"Version it with Git tags and semver, and treat breaking changes like a contract — a major bump means the inputs changed. For testing, the standard is Go-based Terratest for real apply-and-verify tests, plus `terraform validate` and `tflint` in CI on every PR. A good pattern is an `examples/` folder with working configs that double as your test fixtures. Publish internally to a private registry or a tagged Git repo, never a mutable branch."

**Key Point:** "Git tags plus semver for versioning, Terratest and linters in CI for testing, examples as fixtures — never ship a module on a mutable branch."

---

## Workspaces

### Q8: What are Terraform workspaces?

**How to Answer:**

"Workspaces are separate state environments inside one backend — `terraform workspace new staging` gives you an isolated state keyed by the workspace name. You branch on `terraform.workspace` in your config to tweak things per environment. They're fine for quick personal experiments, but I'd never use them for prod separation — one backend, one bucket, one pair of hands away from a mistake. Serious env separation means separate directories or separate backends."

**Key Point:** "Workspaces are multiple states inside one backend — handy for experiments, not a substitute for real environment separation."

---

### Q9: Why wouldn't you use workspaces to separate dev, staging, and prod?

**How to Answer:**

"Because workspaces share the backend and the same credentials — there's no blast-radius boundary. You can't give devs access to dev state without also giving them prod, since it's all in one S3 bucket. And config differences have to be hacked through `terraform.workspace` conditionals, which get ugly fast. The production-grade pattern is separate backend configs per environment — same modules, different state files, different access."

```hcl
# The classic anti-pattern — conditionals branching on workspace name
instance_type = terraform.workspace == "prod" ? "m5.large" : "t3.micro"
```

**Key Point:** "Workspaces share credentials and backends with no blast-radius boundary — real prod separation is separate backends and separate state files."

---

## Best Practices and Interview Traps

### Q10: What are your top best practices for production Terraform?

**How to Answer:**

"Remote state with locking — always, no local state in a team. Pin provider versions and module versions so applies are repeatable. One environment per directory with shared modules, not one config copied three ways. Every change through a PR with `terraform plan` in CI so the diff is reviewed like code. And a `.gitignore` that keeps state files, `.terraform/`, and crash logs out of Git — that's the one interviewers always check."

**Key Point:** "Remote locked state, pinned versions, one directory per env, plan in CI on every PR — and state never in Git."

---

### Q11: What's the biggest trap people fall into with modules?

**How to Answer:**

"Over-generalizing — writing a 'god module' with 40 variables and flags for every cloud quirk, which becomes worse than the code it replaced. The flip side is modules that leak secrets: passing passwords as plain variables without `sensitive = true`, or hardcoding credentials in examples. Another classic is burying real logic inside a module so `plan` output is unreadable — if a reviewer can't tell what changed from the plan, the module is hiding too much."

**Key Point:** "God modules with 40 flags, unmarked secrets, and logic hidden so deep the plan is unreadable — abstraction should clarify, not obscure."

---

### Q12: Walk me through refactoring a 300-line main.tf into modules.

**How to Answer:**

"First I'd draw the seams — grouping resources by what they deploy together, like VPC stuff, app instances, and the database. Then I'd carve out one module at a time, starting with the most self-contained piece, and use `moved` blocks so Terraform renames addresses instead of destroying things. After each extraction I run plan to confirm zero changes before moving on. The test that it worked is a clean plan — same infra, new structure, nothing recreated."

**Key Point:** "Carve seams around resources that deploy together, move addresses with moved blocks, and demand a zero-change plan after each step."
