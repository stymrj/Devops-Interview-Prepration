# Infrastructure Testing & Drift Detection — Hands-On Lab

Nine drills that turn infrastructure testing answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Lint and validate a Terraform module in CI order

**Goal:** A module that fails fast on syntax and style before anything plans.

**Steps:**

```bash
# In a small Terraform module directory, run the checks in CI order:
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
# Now add a deliberate mistake — an unclosed brace or a bad variable type —
# and run the same three commands again to see which one catches it first.
```

**Expected output:** Clean output on the good module; `validate` (or `fmt`) failing clearly on the broken one, with a line number you can point to.

**Why it matters:** This is Q4 made concrete — "validate and format in CI" is an answer you can only give confidently once you've watched it catch a real mistake.

---

## Exercise 2: Write your first Checkov scan

**Goal:** A security scan that fails a build on a genuinely dangerous config.

**Steps:**

```bash
# Install Checkov, then point it at a Terraform directory:
pip install checkov
checkov -d ./terraform --framework terraform
# Now add a public S3 bucket (acl = "public-read") and re-run — watch it fail.
# Fix it with a bucket policy / block-public-access block and re-run to green.
```

**Expected output:** A pass/fail report naming the check (e.g. CKV_AWS_20) and the resource; red on the public bucket, green after the fix.

**Why it matters:** This is Q6 — you can now name a real check ID in an interview instead of saying "we scan for security issues".

---

## Exercise 3: Assert on a Terraform plan with `terraform show -json`

**Goal:** A plan-level test that proves a security group has no open SSH.

**Steps:**

```bash
# Plan your module, then dump the plan as JSON and query it:
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > plan.json
jq -r '.planned_values.root_module.resources[]
  | select(.type=="aws_security_group_rule")
  | [.address, .values.cidr_blocks] | @tsv' plan.json
# Confirm no rule contains 0.0.0.0/0 on port 22. Try adding one and re-running.
```

**Expected output:** A list of security group rules with their CIDR blocks — and an empty result when you grep for `0.0.0.0/0` on the clean config.

**Why it matters:** This is Q3's plan-testing layer — the exact technique Conftest automates, and the one interviewers love to hear described step by step.

---

## Exercise 4: Test a Rego policy with Conftest

**Goal:** A policy that blocks public SSH — and proof it blocks the bad case only.

**Steps:**

```bash
# Write a tiny Rego policy (deny ingress 0.0.0.0/0 on port 22), then:
conftest test plan.json --policy policy/ --namespace main
# Test both fixtures: the clean plan from Exercise 3 (should pass)
# and a plan with an open SSH rule (should fail with your deny message).
```

**Expected output:** `PASS` on the clean plan, `FAIL` with your deny message on the bad plan — proof the policy works both ways.

**Why it matters:** This is Q7 — testing the policy itself against good and bad fixtures is the detail that separates a memorised answer from a real one.

---

## Exercise 5: Terratest a module end to end

**Goal:** A Go test that applies a module, asserts on it, and destroys it.

**Steps:**

```bash
# In a module with an examples/ folder, write a Terratest test that:
# 1. applies the example, 2. reads an output (e.g. bucket name or VPC id),
# 3. asserts it is non-empty, 4. destroys everything in a defer.
go test -v -run TestModule -timeout 30m ./test/
# Check your cloud console afterwards — nothing from the test should remain.
```

**Expected output:** `PASS`, plus a clean cloud account — the destroy in the defer ran even if an assertion failed.

**Why it matters:** This is Q5 — the single most convincing sentence in a testing interview is "I wrote the Terratest that proves it".

---

## Exercise 6: Simulate drift and catch it with a scheduled plan

**Goal:** Drift you created yourself, caught by the cheapest detector.

**Steps:**

```bash
# Apply a small module (e.g. an S3 bucket with versioning enabled).
# Now change something by hand in the console or CLI — disable versioning:
aws s3api put-bucket-versioning --bucket <name> --versioning-configuration Status=Suspended
# Run the drift check:
terraform plan -detailed-exitcode
echo "exit code: $?"
# Then run terraform apply and watch it restore the coded state.
```

**Expected output:** Exit code 2 from the plan (changes present), a diff showing versioning flipping back to Enabled, and a clean plan after the apply.

**Why it matters:** This is Q10 — you've now seen drift appear and disappear with your own hands, and you can describe the exit code from memory.

---

## Exercise 7: Compare state to live infra with driftctl

**Goal:** A drift report that covers resources Terraform doesn't even manage.

**Steps:**

```bash
# Install driftctl and run it against your state file:
driftctl scan --from tfstate://terraform.tfstate
# Note the resources it flags as unmanaged or drifted.
# Pick one unmanaged resource and decide: import it, or leave it and document why.
```

**Expected output:** A drift report listing managed, unmanaged, and drifted resources with percentages — and a written decision for at least one unmanaged resource.

**Why it matters:** This is Q10's second tool — interviewers ask how you handle resources outside Terraform, and this is the honest answer.

---

## Exercise 8: Build a throwaway ephemeral environment

**Goal:** A full stack from a branch, tested live, then destroyed.

**Steps:**

```bash
# Create a workspace per PR/branch and apply the same modules with small sizes:
terraform workspace new pr-123
terraform apply -var="environment=pr-123" -var="instance_type=t3.micro" -auto-approve
# Hit the app's health endpoint, run a smoke test, then destroy:
terraform destroy -auto-approve
terraform workspace select default && terraform workspace delete pr-123
```

**Expected output:** A working stack under the pr-123 name, a passing smoke test, and nothing left running after the destroy — verify in the console.

**Why it matters:** This is Q8 — ephemeral environments are easy to describe and hard to believe until you've built and torn one down yourself.

---

## Exercise 9: Write the drift postmortem

**Goal:** A one-page write-up of a drift incident, with the system fix.

**Steps:**

```bash
# Take the drift from Exercise 6 and write it up as a postmortem:
# What the scheduled check found, why it happened, what you reverted,
# and which policy or check you added so it can't recur.
# Keep it to one page: timeline, root cause, fix, prevention.
# Read it aloud — if it takes more than 90 seconds, cut it down.
```

**Expected output:** A one-page postmortem you could paste into an interview answer, ending with the prevention (policy or scheduled check), not the revert.

**Why it matters:** This is Q12 — the story is the interview answer, and writing it once means you tell it cleanly under pressure.
