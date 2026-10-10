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

## Resources

### Q7: What is a resource, and how do you reference one resource from another?

**How to Answer:**

"A resource is one managed object — an EC2 instance, an S3 bucket — declared with its type and a local name. You reference its attributes with the address syntax `aws_instance.web.public_ip`, which creates an implicit dependency: Terraform knows to build the instance before anything that needs its IP. This dependency graph is Terraform's superpower — it plans the whole execution order itself instead of you sequencing steps. When there's a real ordering need with no attribute reference, you add an explicit `depends_on`."

```hcl
resource "aws_eip" "web" {
  instance = aws_instance.web.id
}
```

**Key Point:** "Reference attributes by address like aws_instance.web.id — Terraform builds the dependency graph from your references."

---

### Q8: What's the difference between a resource and a data source?

**How to Answer:**

"A resource is managed by Terraform — created, updated, destroyed. A data source is read-only information you fetch at plan time, like the latest Ubuntu AMI or a VPC someone else created. You declare it with a `data` block and reference it the same way as a resource attribute. Interviewers love asking why not hardcode the AMI — because data sources stay current, so your config keeps working as images get patched."

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-*"]
  }
}
```

**Key Point:** "Resources are managed, data sources are read-only lookups evaluated at plan time."

---

### Q9: How do count, for_each, and depends_on work?

**How to Answer:**

"`count` creates N identical copies, addressed as `aws_instance.web[0]`. `for_each` creates one copy per item in a map or set, addressed by key like `aws_instance.web["a"]` — I prefer it, because removing an item doesn't shift everyone else's index. `depends_on` is for the rare case with a real ordering need but no attribute reference, like a policy that must exist before a service starts. The trap is overusing `depends_on` where an attribute reference would do — explicit is fine, redundant is a smell."

```hcl
resource "aws_instance" "web" {
  for_each      = toset(["a", "b"])
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}
```

**Key Point:** "count for numbered copies, for_each for keyed copies, depends_on only when no attribute reference creates the dependency."

---

## The Terraform Workflow

### Q10: Walk me through the Terraform workflow.

**How to Answer:**

"`terraform init` first — downloads providers and modules, sets up the backend. Then `terraform plan` shows exactly what will change before anything happens. `terraform apply` executes — I always save the plan to a file and apply that file so nothing drifts between plan and apply. `terraform fmt` and `terraform validate` keep the code clean and syntactically sound. And `terraform destroy` tears it all down — dangerous enough that I'd gate it behind approval in any pipeline."

**Key Point:** "init, plan, apply — with fmt and validate in CI, and apply always running a saved plan file."

---

### Q11: Why do you save the plan to a file before applying?

**How to Answer:**

"Because a bare `terraform apply` re-plans at apply time, and anything that changed since your plan — a new AMI, a teammate's commit — silently enters the run. Saving with `terraform plan -out=tfplan` and applying with `terraform apply tfplan` freezes exactly what you reviewed. In CI this is the standard: plan on pull request, a human reviews it, apply the exact same file on merge. The plan file is also your audit artifact — it proves what ran."

**Key Point:** "A saved plan file freezes the reviewed changes — apply tfplan, never re-plan at apply time."

---

### Q12: What happens during terraform plan under the hood?

**How to Answer:**

"Terraform reads your configuration, refreshes its view of real-world state, and diffs desired vs actual to produce the create/update/destroy list. Since 1.x it does the refresh inside plan so the diff is current. What it can't know yet shows as `(known after apply)` — values only the provider can return at creation, like an instance's public IP. The trap answer people give is 'plan predicts everything' — no, anything marked known-after-apply is an honest placeholder."

**Key Point:** "Plan diffs desired vs actual state; provider-computed values show as known after apply."

---

## Interview Traps and Best Practices

### Q13: How do you force a resource to be recreated?

**How to Answer:**

"Two ways. The declarative way: a `lifecycle { replace_triggered_by = [aws_key_pair.deployer] }` block recreates the resource whenever that trigger changes — perfect for key rotation. The imperative escape hatch: `terraform apply -replace="aws_instance.web"` forces one replacement on that run. Older answers say `terraform taint` — it's deprecated in favor of `-replace`, so if the interviewer asks, mention both but lead with the current one."

**Key Point:** "lifecycle replace_triggered_by for declarative recreation; -replace for one-off — taint is deprecated."

---

### Q14: When would you use terraform import, and what's its gotcha?

**How to Answer:**

"Import pulls an existing real-world resource into Terraform management — the classic case is infra someone built in the console that you now want in code. You run `terraform import aws_instance.web i-0abcd1234`, which writes it into state, then you write the matching resource block and plan to confirm zero diff. The gotcha: import only updates state, it never generates the configuration — you write the HCL yourself. The newer `import {}` blocks make it declarative, but the same rule holds."

**Key Point:** "Import registers existing infra in state; you still have to write the config yourself to match."

---
