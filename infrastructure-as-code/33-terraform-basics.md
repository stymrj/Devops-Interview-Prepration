# Terraform Basics Interview Preparation Guide

*How to Answer Terraform Basics Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Terraform Fundamentals](#terraform-fundamentals)
2. [Providers](#providers)
3. [Resources](#resources)
4. [The Terraform Workflow](#terraform-workflow)
5. [Interview Traps and Best Practices](#interview-traps-and-best-practices)

---

## Terraform Fundamentals

### Q1: What is Terraform, and why do teams use it?

**How to Answer:**

"Terraform is HashiCorp's declarative infrastructure-as-code tool — you describe what you want in HCL, and Terraform makes the cloud match it. Teams use it because ClickOps doesn't scale: no audit trail, no review, and 'it worked in the console once' isn't a strategy. With Terraform, infra goes through code review, version control, and pipelines, just like app code. The honest pitch is repeatability — spin the same VPC and cluster in dev, staging, and prod without drift between them."

**Key Point:** "Terraform turns infrastructure into versioned, reviewable, repeatable code — no more ClickOps drift."

---

### Q2: How is Terraform different from configuration management tools like Ansible?

**How to Answer:**

"Terraform is declarative and mostly for provisioning — it manages the desired state of infrastructure and converges to it on its own. Ansible is procedural for configuration — it runs ordered steps like installing packages and templating config files. Terraform excels at creating VPCs, instances, and buckets; Ansible excels at what's inside the instance. In practice most teams use both: Terraform builds the VM, Ansible configures the app on it."

**Key Point:** "Terraform provisions infrastructure declaratively; Ansible configures machines procedurally — they complement each other."

---

### Q3: Walk me through the core Terraform language blocks.

**How to Answer:**

"Everything is a block. `terraform {}` holds version constraints for Terraform itself and the providers. `provider "aws"` configures credentials and region. `resource "aws_instance" "web"` is the thing you're creating — type plus a local name you reference elsewhere. Then `variable`, `output`, `data`, and `module` round it out. The thing interviewers check: can you name them and say what each does in one line."

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags          = { Name = "web-01" }
}
```

**Key Point:** "terraform, provider, resource, variable, output, data, module — know each block's job."

---

## Providers

### Q4: What is a Terraform provider?

**How to Answer:**

"A provider is the plugin that teaches Terraform how to talk to a platform — AWS, Azure, Kubernetes, Cloudflare, all of them. You declare it in a `required_providers` block with a source like `hashicorp/aws` and a version constraint. Running `terraform init` downloads the provider binary from the registry. The big idea is separation: Terraform core stays generic, providers hold all the API knowledge."

**Key Point:** "Providers are versioned plugins from the registry that let Terraform core speak any platform's API."

---

### Q5: How do you pin and upgrade provider versions safely?

**How to Answer:**

"I pin with a pessimistic constraint like `~> 5.0` in `required_providers`, which allows patches and minors but blocks a 6.0 surprise. `terraform init` writes the exact chosen version into `.terraform.lock.hcl`, so the whole team and CI install the identical binary. To upgrade, I bump the constraint, run `terraform init -upgrade`, and read the changelog for breaking changes before planning. Unpinned providers are a classic cause of 'it worked on my machine' pipeline failures."

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**Key Point:** "Pin with ~> constraints, let the lock file freeze exact versions, upgrade deliberately with -upgrade."

---

### Q6: How do you work with multiple regions or accounts?

**How to Answer:**

"You use provider aliases. Declare the default provider for your main region, then a second `provider "aws"` block with `alias = "eu"` and the region set to eu-west-1. Any resource that belongs there gets `provider = aws.eu` in its block. Same pattern for multi-account — one aliased provider per account, usually with different assumed roles. The follow-up I always get: how do you avoid hardcoding it — answer is variables, or separate state per account when isolation matters."

**Key Point:** "Aliased providers route resources to extra regions or accounts — one alias, one provider config."

---
