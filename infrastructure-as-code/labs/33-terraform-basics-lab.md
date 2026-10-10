# Terraform Basics — Hands-On Lab

Nine exercises that take the theory in the guide and make it muscle memory. Run them in order — each builds on the last. All examples use the AWS provider, but the patterns are provider-agnostic.

> Estimated time: 60–90 minutes. You'll need Terraform ≥ 1.5 installed and AWS credentials with EC2 permissions.

---

## Exercise 1: Install and verify Terraform

**Goal:** Get a working Terraform install and confirm the provider download flow.

**Commands:**

```bash
terraform version
```

**Expected output:** Something like `Terraform v1.9.x` — no errors.

**Why it matters:** Interviewers expect you to have run Terraform, not just read about it. Version matters because provider behavior and language features differ across releases.

---

## Exercise 2: Initialize a project and inspect what init creates

**Goal:** Understand `terraform init` and the `.terraform` directory plus the lock file.

**Commands:**

```bash
mkdir tf-lab && cd tf-lab
cat > main.tf <<'EOF'
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" { region = "ap-south-1" }
EOF
terraform init
ls .terraform/providers && cat .terraform.lock.hcl
```

**Expected output:** Provider plugins under `.terraform/providers/registry.terraform.io/hashicorp/aws/`, and a lock file pinning an exact AWS provider version.

**Why it matters:** The lock file is how teams guarantee identical provider binaries across machines and CI — this is the first thing you explain when asked about reproducible runs.

---

## Exercise 3: Write your first resource and plan it

**Goal:** Declare a resource, then read a plan before anything is created.

**Commands:**

```bash
cat >> main.tf <<'EOF'
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags          = { Name = "tf-lab-web" }
}
EOF
terraform fmt
terraform validate
terraform plan
```

**Expected output:** `validate` says "Success! The configuration is valid." `plan` shows `Plan: 1 to add, 0 to change, 0 to destroy.`

**Why it matters:** Plan-before-apply is the habit every interviewer probes. `fmt` + `validate` in this flow is exactly what your CI pipeline should run on every pull request.

---

## Exercise 4: Apply, then reference an attribute from another resource

**Goal:** See implicit dependencies and attribute references in action.

**Commands:**

```bash
terraform apply -auto-approve
cat >> main.tf <<'EOF'
resource "aws_eip" "web" {
  instance = aws_instance.web.id
}
EOF
terraform plan
```

**Expected output:** The new plan shows 1 Elastic IP to add, referencing `aws_instance.web.id`. Terraform plans the EIP after the instance automatically.

**Why it matters:** Attribute references are how Terraform builds its dependency graph. This is the answer to "how does Terraform know what order to create things in?"

---

## Exercise 5: Use a data source instead of a hardcoded AMI

**Goal:** Replace the hardcoded AMI with a live lookup.

**Commands:**

```bash
# edit main.tf: add a data block, swap the AMI line
terraform plan
```

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

**Expected output:** Plan runs with the resolved AMI ID from the data lookup — no hardcoded value in config.

**Why it matters:** Hardcoded AMIs go stale; data sources stay current. This is the textbook answer to "why use data sources?"

---

## Exercise 6: Save the plan to a file and apply it

**Goal:** Practice the plan-file workflow used in real pipelines.

**Commands:**

```bash
terraform plan -out=tfplan
terraform show tfplan | head -30
terraform apply tfplan
rm tfplan
```

**Expected output:** `terraform show` renders a human-readable summary of the frozen plan; apply runs exactly that — no re-planning.

**Why it matters:** Bare `apply` re-plans at apply time. The saved plan file is what gets reviewed on a PR and executed on merge — this is the standard CI/CD pattern.

---

## Exercise 7: for_each two instances, then remove one

**Goal:** Feel why `for_each` beats `count` for real fleets.

**Commands:**

```bash
# edit the instance block to:
#   for_each = toset(["a", "b"])
terraform apply -auto-approve
terraform state list
# remove "b" from the set, then:
terraform plan
```

**Expected output:** After the edit, the plan destroys only `aws_instance.web["b"]` — `["a"]` is untouched.

**Why it matters:** With `count`, removing index 0 shifts everything and recreates the fleet. `for_each` keys resources by identity — this is the follow-up every interviewer asks after you mention `count`.

---

## Exercise 8: Force a replacement with -replace

**Goal:** Recreate a resource without changing any config.

**Commands:**

```bash
terraform apply -replace="aws_instance.web[\"a\"]" -auto-approve
```

**Expected output:** Plan shows 1 to add, 1 to destroy — same config, new instance.

**Why it matters:** Replacement is the answer when something's unhealthy and config is fine. Know `-replace` by name, and that `terraform taint` is the deprecated predecessor.

---

## Exercise 9: Destroy and review what Terraform leaves behind

**Goal:** Tear down cleanly and confirm what state tracks.

**Commands:**

```bash
terraform destroy -auto-approve
ls -la
```

**Expected output:** All resources destroyed. `terraform.tfstate` records everything as gone; the config files remain.

**Why it matters:** Destroy is how you verify your config actually describes everything — if manual resources survive, they were never in code. It also ends your lab spend.

---

## Checklist

- [ ] `terraform init` downloads providers into `.terraform/`
- [ ] `.terraform.lock.hcl` pins the exact provider version
- [ ] `plan` shows a diff before anything is created
- [ ] Attribute references create implicit dependencies
- [ ] Data sources replace hardcoded values
- [ ] Plan files freeze what gets applied (`plan -out` / `apply tfplan`)
- [ ] `for_each` removal destroys only the removed key
- [ ] `-replace` recreates without config changes
- [ ] `destroy` returns the account to zero

*Keep this lab project — you'll extend it in the Terraform state management guide.*
