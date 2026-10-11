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
