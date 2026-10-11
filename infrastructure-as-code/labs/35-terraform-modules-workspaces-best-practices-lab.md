# Terraform Modules, Workspaces & Best Practices — Hands-On Lab

Nine exercises that make modules and workspaces click. Run them in order — each builds on the last.

> Estimated time: 60–90 minutes. You'll need Terraform ≥ 1.10 installed. Nothing here touches the cloud or creates billable resources — everything uses `null_resource` and locals.

---

## Exercise 1: Turn a config into a module

**Goal:** Feel the root-module vs child-module split.

**Commands:**

```bash
mkdir tf-modules-lab && cd tf-modules-lab
mkdir -p modules/greeter
cat > modules/greeter/main.tf <<'EOF'
variable "name" { type = string }
resource "null_resource" "greet" {
  provisioner "local-exec" { command = "echo Hello, ${var.name}" }
}
output "who" { value = var.name }
EOF
cat > main.tf <<'EOF'
module "greeter" { source = "./modules/greeter" name = "devops" }
output "greeted" { value = module.greeter.who }
EOF
terraform init && terraform apply -auto-approve
```

**Expected output:** `Hello, devops` printed by the provisioner, and output `greeted = "devops"`.

**Why it matters:** The "any directory with .tf files is a module" line from the guide only lands once you've done it — the root is a module too.

---

## Exercise 2: Fail init by forgetting to re-run it

**Goal:** Learn when init is required.

**Commands:**

```bash
cat > main.tf <<'EOF'
module "greeter" { source = "./modules/greeter" name = "devops" }
module "greeter2" { source = "./modules/greeter" name = "sre" }
EOF
terraform apply -auto-approve
terraform init && terraform apply -auto-approve
```

**Expected output:** First apply errors (`Module not installed` / uninitialized). After init, both modules apply and print `Hello, devops` and `Hello, sre`.

**Why it matters:** Any change to module sources needs `terraform init` again — the most common junior mistake in module-heavy repos.

---

## Exercise 3: Address a module resource with for_each

**Goal:** See how module instances are keyed.

**Commands:**

```bash
cat > main.tf <<'EOF'
module "teams" {
  for_each = toset(["frontend", "backend"])
  source   = "./modules/greeter"
  name     = each.key
}
EOF
terraform init && terraform apply -auto-approve
terraform state list | grep null_resource
```

**Expected output:** Addresses like `module.teams["frontend"].null_resource.greet` — keyed by name, not index.

**Why it matters:** This is the proof for the for_each-over-count answer: removing one key doesn't re-key the others, so plans stay clean.

---

## Exercise 4: Pass a typed object into a module

**Goal:** Practice real-world inputs instead of loose strings.

**Commands:**

```bash
cat > modules/greeter/variables.tf <<'EOF'
variable "config" {
  type = object({ name = string, loud = bool })
}
EOF
cat > main.tf <<'EOF'
module "teams" {
  source = "./modules/greeter"
  config = { name = "platform", loud = true }
}
EOF
terraform init && terraform validate
```

**Expected output:** `validate` succeeds. Try passing `loud = "yes"` and watch it reject a string where a bool was declared.

**Why it matters:** Typed objects catch config bugs at validate time instead of mid-apply — this is why interviewers ask about variable types.

---

## Exercise 5: Mark a value sensitive

**Goal:** See how sensitive metadata hides output.

**Commands:**

```bash
cat >> modules/greeter/main.tf <<'EOF'
variable "secret" { type = string, sensitive = true, default = "hunter2" }
output "masked" { value = var.secret, sensitive = true }
EOF
terraform init && terraform apply -auto-approve 2>&1 | tail -8
```

**Expected output:** The `masked` output prints as `(sensitive value)` instead of `hunter2`.

**Why it matters:** Shows what `sensitive = true` actually protects — output display — and reminds you secrets still live in plaintext inside state (topic 34).

---

## Exercise 6: Create and switch workspaces

**Goal:** Understand workspace state isolation.

**Commands:**

```bash
terraform workspace new dev
terraform workspace list
terraform workspace select default
terraform workspace list
```

**Expected output:** A list showing `default` and `dev`, with `*` marking the active one after each switch.

**Why it matters:** Workspaces are cheap state partitions — you'll want this muscle memory to explain when they're appropriate and when they aren't.

---

## Exercise 7: Prove workspaces keep separate state

**Goal:** Watch the same config track two different states.

**Commands:**

```bash
terraform workspace select dev
terraform apply -auto-approve
terraform state list > /tmp/dev-state.txt
terraform workspace select default
terraform state list > /tmp/default-state.txt
diff /tmp/dev-state.txt /tmp/default-state.txt && echo "identical" || echo "different"
ls terraform.tfstate.d/
```

**Expected output:** The two state listings differ (`module.teams` only in dev), and `terraform.tfstate.d/` holds a `dev/` directory with its own state file.

**Why it matters:** This is the physical picture behind the interview answer — separate state files under one backend, one bucket, no access boundary.

---

## Exercise 8: Branch behavior on terraform.workspace

**Goal:** Try the workspace conditional — then see why it smells.

**Commands:**

```bash
cat > main.tf <<'EOF'
module "single" {
  source = "./modules/greeter"
  name   = terraform.workspace
}
EOF
terraform init
terraform workspace select dev && terraform apply -auto-approve | grep Hello
terraform workspace select default && terraform apply -auto-approve | grep Hello
```

**Expected output:** `Hello, dev` in the dev workspace, `Hello, default` in default — same config, workspace-dependent behavior.

**Why it matters:** The conditional works, but imagine 30 of these scattered through prod config — this is the anti-pattern from Q9 you can now speak about from experience.

---

## Exercise 9: Refactor with a moved block

**Goal:** Rename a module without destroying anything.

**Commands:**

```bash
cat >> main.tf <<'EOF'
moved {
  from = module.single
  to   = module.renamed
}
EOF
sed -i 's/module "single"/module "renamed"/' main.tf
terraform apply -auto-approve
```

**Expected output:** Terraform reports the move (no resources created or destroyed) and the address is now `module.renamed`.

**Why it matters:** The safe-refactor story from Q12 only works if you've seen a zero-change move — this is the exact motion interviewers want described.

---

## Cleanup

```bash
terraform workspace select dev && terraform destroy -auto-approve
terraform workspace select default && terraform destroy -auto-approve
terraform workspace delete dev
cd .. && rm -rf tf-modules-lab
```
