# Infrastructure Testing & Drift Detection Interview Preparation Guide

*How to Answer Infrastructure Testing & Drift Detection Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Test Infrastructure](#why-test-infrastructure)
2. [Testing Tools & Techniques](#testing-tools--techniques)
3. [Drift Detection & Remediation](#drift-detection--remediation)

---

## Why Test Infrastructure

### Q1: Why should infrastructure be tested at all — isn't `terraform plan` enough?

**How to Answer:**

"I'd say plan only tells you what Terraform *intends* to do, not whether the result is correct, secure, or even works. A plan can look perfect and still open port 22 to the world, or create a database with no backups. I test infrastructure for the same reason I test code: the cost of catching a mistake in CI is minutes, while catching it in production is an incident. And unlike app code, a bad infra change can delete a database or expose data — there's no rollback button for some of those. So I treat `plan` as the first check, not the only one."

**Key Point:** "`terraform plan` shows intent, not correctness — test infra like code, because a bad apply can delete data or open a hole that no rollback can undo."

---

### Q2: What does a testing pyramid for infrastructure as code look like?

**How to Answer:**

"I'd describe it in three layers, same shape as the app pyramid. At the bottom, static checks — linting, formatting, and policy scans that run in seconds on every commit. In the middle, plan-level tests — I run `terraform plan` against a test workspace and assert on the planned values, like 'this bucket must have versioning on'. At the top, live tests — I actually apply to a throwaway environment and verify the real resource behaves, like an EC2 instance you can SSH into or a load balancer that answers. I keep the top layer small because it's slow and costs money, and I let the bottom two layers catch most mistakes before anything gets applied anywhere."

**Key Point:** "Static checks at the bottom, plan assertions in the middle, a few real applies at the top — cheap fast checks catch most mistakes, expensive live tests prove the rest."

---

### Q3: What's the difference between static analysis, plan testing, and live testing of infrastructure?

**How to Answer:**

"I'd put it this way: static analysis reads the code without running it — tools like tflint or Checkov flagging a public S3 bucket or a missing encryption flag. Plan testing runs Terraform and checks the *proposed* state — for example, asserting the planned security group has no 0.0.0.0/0 ingress. Live testing actually creates the resources and pokes them — spinning up the VPC and confirming the instance really reaches the internet through the NAT gateway. Static is fastest and catches style and policy issues, plan testing catches logic errors in your modules, and live testing is the only one that catches the things the cloud provider does differently from what you assumed."

**Key Point:** "Static reads the code, plan testing checks the proposal, live testing proves the real thing — each layer catches a class of mistake the one below it can't see."

---

### Q4: How do you test a Terraform module before anyone else uses it?

**How to Answer:**

"I'd walk through my own checklist. First, `terraform fmt` and `terraform validate` in CI so broken syntax never merges. Then I write example usages in an examples folder and run `terraform plan` against them — if the example can't plan, the module is broken. Then I add a Terratest that applies the module to a real test account, checks the outputs, and destroys it all afterwards. And I version the module, so teams pin to a tested release instead of floating on main. The honest bit is that I also keep a tiny README with the inputs, outputs, and one working example — because a module nobody can figure out in five minutes won't get used, no matter how well tested it is."

**Key Point:** "Validate and format in CI, plan the bundled examples, Terratest a real apply-and-destroy, and version it — then document it so people actually use the tested module."

---

## Testing Tools & Techniques

### Q5: What is Terratest, and when is it worth the effort?

**How to Answer:**

"I'd explain Terratest as a Go library that lets you write real tests around Terraform — apply, check, destroy — in code instead of shell scripts. I reach for it when a module is shared across teams or when the infrastructure is critical enough that a broken apply is an incident. A typical test applies the module, reads the outputs, and asserts things like 'the S3 bucket exists and versioning is enabled', then destroys everything in a defer so nothing is left running. I wouldn't write Terratest for a one-off personal project — it's heavy for that. But for a shared VPC or database module, it's the difference between 'I think it works' and 'CI proved it works this morning.'"

```go
// A Terratest test: apply, assert, always destroy.
terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
    TerraformDir: "../examples/vpc",
})
defer terraform.Destroy(t, terraformOptions)
terraform.InitAndApply(t, terraformOptions)
vpcID := terraform.Output(t, terraformOptions, "vpc_id")
assert.NotEmpty(t, vpcID)
```

**Key Point:** "Terratest applies your module for real, asserts on the live resources, and destroys them — worth it for shared or critical modules, overkill for one-offs."

---

### Q6: How do tflint, tfsec, and Checkov differ — and do you need all three?

**How to Answer:**

"I'd say they overlap but each catches a different flavour of mistake. tflint is a linter — it checks Terraform-specific things like deprecated syntax, missing required variables, and provider rules. tfsec and Checkov are security scanners — they flag misconfigurations like unencrypted storage or public buckets, and Checkov covers more than Terraform, including CloudFormation and Kubernetes manifests. I don't run all three religiously in every repo — I'd pick tflint for linting plus one security scanner, usually Checkov, and wire both into CI as a failing check. The point I make in interviews is that these tools are the seatbelt: they don't make the design good, they just stop the obviously dangerous stuff from ever reaching apply."

**Key Point:** "tflint lints Terraform itself, tfsec and Checkov scan for insecure configs — run a linter plus one security scanner in CI and let them block the obviously dangerous applies."

---

### Q7: How do you test infrastructure policies with Conftest and OPA?

**How to Answer:**

"I'd explain it as testing the rules before they enforce anything. I write the policy in Rego — say, 'no security group may allow SSH from anywhere' — then I run Conftest against the Terraform plan JSON, and it tells me pass or fail before anything is applied. The trick I always mention is testing the policy itself: I keep a small set of good and bad plan fixtures and run the policy against both, so I know it actually blocks the bad case and doesn't block the good one. That matters because a policy with a bug is worse than no policy — it either lets the bad thing through or blocks a deploy at 2 a.m. and nobody can tell why."

```bash
# Test a Rego policy against a Terraform plan, before applying:
terraform plan -out=tfplan.binary && terraform show -json tfplan.binary > plan.json
conftest test plan.json --policy policy/ --namespace main
```

**Key Point:** "Conftest runs your Rego policy against the plan JSON before apply — and always test the policy itself against good and bad fixtures, because a buggy policy is worse than none."

---

### Q8: What are ephemeral environments, and how do they change infrastructure testing?

**How to Answer:**

"I'd describe an ephemeral environment as a full copy of the stack that lives only as long as a pull request — spun up when the PR opens, torn down when it merges. It changes testing because you can run the real thing instead of guessing from a plan: the app boots, the database migrates, the health checks pass, all against the exact Terraform from that branch. I usually build them with the same modules as production but smaller instance sizes and a namespace or prefix per PR so nothing collides. The honest trade-off is cost and speed — they take minutes to build and you pay for them while they run — so I keep them for services where a bad deploy is expensive, and I auto-destroy them on merge or after a TTL."

**Key Point:** "An ephemeral environment is a per-PR copy of the stack you can actually test against — expensive but worth it where a bad deploy hurts, and always auto-destroyed."

---

## Drift Detection & Remediation

### Q9: What is configuration drift, and how does it happen?

**How to Answer:**

"I'd define drift simply: the real infrastructure no longer matches what your code says it should be. It happens in three ways I see constantly. One, someone changes something in the console during an incident and never writes it back to Terraform. Two, the cloud provider changes a default or a managed service updates itself. Three, two tools fight over the same resource — say an autoscaler changing a setting Terraform owns. I always add that drift isn't a people problem, it's a process problem — if the only way to fix production at 3 a.m. is the console, drift is guaranteed, so the fix has to make the code path as fast as the console path."

**Key Point:** "Drift is real infra diverging from code — console hotfixes, provider-side changes, or two tools owning one resource — and it's a process problem, not a people problem."

---

### Q10: How do you detect drift in practice?

**How to Answer:**

"I'd give the practical stack, cheapest first. `terraform plan` with no changes is the simplest drift check — if it wants to change something nobody changed in code, that's drift. I run that on a schedule, not just on PRs, and I alert on any non-empty plan. For bigger estates I add a dedicated tool like driftctl, which compares Terraform state against the live cloud account and reports what moved. And for resources Terraform doesn't manage at all, I use AWS Config rules or CloudQuery to inventory everything and flag anything outside the baseline. The key point I make is that detection has to be continuous — a drift check you run once a quarter finds drift three months late."

```bash
# A scheduled drift check: a non-empty plan means something moved.
terraform plan -detailed-exitcode -out=/tmp/drift.plan
# Exit code 2 = changes present, i.e. drift (or a pending code change).
```

**Key Point:** "Scheduled `terraform plan` is the cheapest drift detector, driftctl compares state to the live account, and AWS Config covers what Terraform doesn't manage — run them continuously, not quarterly."

---

### Q11: When you find drift, do you fix it by hand or let Terraform overwrite it?

**How to Answer:**

"I'd say it depends on which side is right, and I never blindly overwrite. First I look at what drifted and why — if it's a console hotfix from an incident, I ask whether the change was correct and just missing from code. If it was correct, I import it into Terraform and commit the code, so the fix becomes the documented state. If it was a mistake or a temporary hack, I let the next `terraform apply` revert it — but only after telling the team, because reverting something someone depends on can cause a second incident. And if the drift is a provider-side change I didn't ask for, I update the code to match reality and pin the version so it stops moving. The rule I state in interviews: drift is a decision, not a command — you choose which side wins, and you write the reason down."

**Key Point:** "Never blindly overwrite drift — import the change if it was right, revert it if it was a mistake, and pin the version if the provider moved it; drift is a decision you document."

---

### Q12: Walk me through a real drift incident you would describe in an interview.

**How to Answer:**

"I'd tell it as a short story with a lesson. Our scheduled plan started showing a security group change nobody had committed, and the diff showed port 5432 open to a single IP. It turned out an engineer had opened it from the console during a late-night database debug and never closed it. I didn't just revert it — I first checked whether anything was still using that access, then I closed it in code, applied, and added a Conftest policy that blocks database ports from public ranges so it can't happen again. The interview point is the ending: the fix wasn't the revert, it was the policy and the scheduled drift check that caught it in the first place. That's the answer that shows I think in systems, not just commands."

**Key Point:** "Tell drift as a story — what the scheduled check found, why it happened, the safe revert, and the policy you added so it can't recur; the system fix is the real answer."
