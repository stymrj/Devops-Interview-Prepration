# Terragrunt & OpenTofu — Hands-on Lab

*9 exercises to get Terragrunt and OpenTofu into your muscle memory. Run these locally (AWS examples use LocalStack or a sandbox account).*

---

## Exercise 1: Install and verify both binaries

**Goal:** Get `terragrunt` and `tofu` on your PATH.

**Commands:**

```bash
# OpenTofu
tofu version
# Terragrunt (install from GitHub releases)
terragrunt --version
```

**Expected output:** Version numbers for both, no "command not found".

**Why it matters:** Terragrunt shells out to the IaC binary — interviewers love asking what happens if the binary is missing.

---

## Exercise 2: Your first terragrunt.hcl

**Goal:** Wrap a tiny Terraform module with Terragrunt.

**Commands:**

```bash
mkdir -p lab/terragrunt-demo && cd lab/terragrunt-demo
cat > terragrunt.hcl << 'EOF'
terraform {
  source = "./modules/hello"
}
inputs = { name = "devops-lab" }
EOF
mkdir modules/hello && cat > modules/hello/main.tf << 'EOF'
variable "name" {}
output "greeting" { value = "hello ${var.name}" }
EOF
terragrunt init && terragrunt apply -auto-approve
```

**Expected output:** `greeting = "hello devops-lab"`.

**Why it matters:** This is the core Terragrunt shape — source + inputs — that every later exercise builds on.

---

## Exercise 3: DRY remote state with path_relative_to_include

**Goal:** Give two environments unique state keys from one root config.

**Commands:**

```bash
mkdir -p lab/root/{dev,prod} && cd lab/root
cat > terragrunt.hcl << 'EOF'
remote_state {
  backend = "local"
  config  = { path = "${path_relative_to_include()}/terraform.tfstate" }
}
EOF
echo 'include "root" { path = find_in_parent_folders() }' > dev/terragrunt.hcl
echo 'include "root" { path = find_in_parent_folders() }' > prod/terragrunt.hcl
cd dev && terragrunt init && cd ../prod && terragrunt init
ls ../dev/terraform.tfstate ../prod/terraform.tfstate
```

**Expected output:** Two separate `terraform.tfstate` files, one per folder, from one root block.

**Why it matters:** This is the headline Terragrunt win — unique state without copy-paste.

---

## Exercise 4: Generate a provider block with generate

**Goal:** Stamp out an identical provider file in every child.

**Commands:**

```bash
cd lab/root
cat >> terragrunt.hcl << 'EOF'
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = "terraform { required_version = \">= 1.6\" }"
}
EOF
cd dev && terragrunt init && cat provider.tf
```

**Expected output:** `provider.tf` appears in `dev/` with your generated contents.

**Why it matters:** `generate` kills the "same provider file in 12 folders" problem.

---

## Exercise 5: Wire two modules with a dependency block

**Goal:** Pass an output from module A into module B.

**Commands:**

```bash
mkdir -p lab/deps/{vpc,app} && cd lab/deps
cat > vpc/terragrunt.hcl << 'EOF'
terraform { source = "./modules/vpc" }
EOF
cat > app/terragrunt.hcl << 'EOF'
terraform { source = "./modules/app" }
dependency "vpc" {
  config_path  = "../vpc"
  mock_outputs = { vpc_id = "vpc-mock123" }
}
inputs = { vpc_id = dependency.vpc.outputs.vpc_id }
EOF
cd app && terragrunt plan   # works even though vpc/ was never applied
```

**Expected output:** A valid plan using the mocked `vpc_id`.

**Why it matters:** `mock_outputs` is how teams plan against dependencies that don't exist yet — classic interview topic.

---

## Exercise 6: Deploy in order with run-all

**Goal:** Apply the dependency tree in one command.

**Commands:**

```bash
cd lab/deps
terragrunt run-all plan --terragrunt-non-interactive
```

**Expected output:** Terragrunt discovers `vpc` and `app`, orders them, and plans vpc first.

**Why it matters:** `run-all` replaces hand-written CI ordering scripts.

---

## Exercise 7: Swap Terraform for OpenTofu on an existing project

**Goal:** Prove state compatibility between the two binaries.

**Commands:**

```bash
cd lab/terragrunt-demo
terraform state pull > state-backup.tfstate   # always backup first
tofu init && tofu plan
```

**Expected output:** `tofu plan` shows "No changes" — zero drift, same state.

**Why it matters:** An empty plan after a binary swap is the safest migration signal you can show an interviewer.

---

## Exercise 8: Try a before_hook for linting

**Goal:** Run a check automatically before every apply.

**Commands:**

```bash
cd lab/terragrunt-demo
cat >> terragrunt.hcl << 'EOF'
terraform {
  source = "./modules/hello"
  before_hook "echo_hello" {
    commands = ["apply"]
    execute  = ["echo", "lint would run here"]
  }
}
EOF
terragrunt apply -auto-approve
```

**Expected output:** Your hook message prints before the apply runs.

**Why it matters:** Hooks are the escape hatch for glue work Terraform can't do — know where they fit.

---

## Exercise 9: Explore an OpenTofu-only feature

**Goal:** See one thing OpenTofu can do that Terraform can't.

**Commands:**

```bash
mkdir -p lab/opentofu-only && cd lab/opentofu-only
tofu init
# Try tofu's encryption config or provider-defined functions per current docs
tofu version | head -1
```

**Expected output:** A working OpenTofu project using a feature you can name in interviews (e.g. state encryption, `.tofu` extensions).

**Why it matters:** "Name one OpenTofu-only feature" is a real interview question — now you can answer from experience.
