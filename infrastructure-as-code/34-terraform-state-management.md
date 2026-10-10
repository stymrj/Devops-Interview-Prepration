# Terraform State Management Interview Preparation Guide

*How to Answer Terraform State Management Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why State Matters](#why-state-matters)
2. [Local vs Remote State](#local-vs-remote-state)
3. [S3 Backend Deep Dive](#s3-backend-deep-dive)
4. [State Locking](#state-locking)
5. [State Operations](#state-operations)
6. [Interview Traps and Best Practices](#interview-traps-and-best-practices)

---

## Why State Matters

### Q1: What is Terraform state, and why does Terraform need it?

**How to Answer:**

"State is Terraform's memory — a JSON file mapping every resource in your config to its real-world object and attributes. Without it, Terraform couldn't know an `aws_instance.web` already exists, so every plan would try to create duplicates. It also stores metadata like resource dependencies and provider versions, which is how plan diffs stay fast and correct. Think of it as the source of truth Terraform trusts about the world, and that's exactly why it's so sensitive."

**Key Point:** "State is Terraform's memory — it maps config to real-world objects so plans can diff instead of blindly creating."

---

### Q2: What actually lives inside a state file?

**How to Answer:**

"Each resource's full attribute set as the provider returned it — IDs, IPs, ARNs, tags — plus the configuration that produced it. There's a `serial` number that increments on every write, a `terraform_version`, and the outputs block. Importantly, it holds secrets in plain text — database passwords, private keys — which is the whole argument for encrypted remote state and never committing it to Git. One real-world gotcha: the state can hold values from a provider version older than what's installed, and plan will flag the drift."

```hcl
# Outputs also live in state — this is how remote_state data lookups work
output "db_endpoint" {
  value = aws_db_instance.main.endpoint
}
```

**Key Point:** "State holds every resource attribute, outputs, and the serial — including secrets in plain text, so encryption and access control are non-negotiable."

---

### Q3: What's the difference between desired, actual, and recorded state?

**How to Answer:**

"Desired is your `.tf` files — what you want. Actual is what's really in the cloud right now. Recorded is the state file — what Terraform last saw. Plan's job is reconciling all three: it refreshes recorded against actual, then diffs desired against that. Drift is when actual moved away from recorded because someone clicked in the console. The interview trap here is saying 'plan shows what's in state' — no, plan refreshes state first, then compares to config."

**Key Point:** "Desired = config, actual = cloud, recorded = state file — plan reconciles all three, and drift is actual diverging from recorded."

---

## Local vs Remote State

### Q4: What's wrong with the default local backend?

**How to Answer:**

"Three things, and they all bite in teams. First, it's a local file — nobody else can see it, so collaboration is dead on arrival. Second, there's no locking, so two people running apply at the same time corrupt it. Third, it sits unencrypted on your disk with every secret in plain text. Local state is fine for learning Terraform, but the first question a good interviewer asks about any real setup is 'where does state live and how is it locked?'"

**Key Point:** "Local state can't be shared, can't be locked, and holds secrets unencrypted — fine for learning, disqualifying for teams."

---

### Q5: What is a backend, and what does it do?

**How to Answer:**

"A backend is where Terraform stores state and how it handles locking — it's configured in a `backend` block inside `terraform {}`, and it's set at `init` time. Backends do two jobs: store the state file somewhere shared and durable like S3 or Terraform Cloud, and provide a locking mechanism so concurrent runs can't corrupt it. The key detail people miss: you can't use variables in a backend block, because the backend must be configured before Terraform even loads variables. So you pass values via `-backend-config` flags or a separate `.hcl` file."

```hcl
terraform {
  backend "s3" {
    bucket = "my-tf-state"
    key    = "prod/network/terraform.tfstate"
    region = "ap-south-1"
  }
}
```

**Key Point:** "A backend defines where state lives and how locking works — configured at init, and it can't use variables, so feed it with -backend-config."

---

## S3 Backend Deep Dive

### Q6: Walk me through a production S3 backend setup.

**How to Answer:**

"One dedicated bucket, never shared with application data — versioning on, default SSE encryption on, public access fully blocked, and a lifecycle rule to keep old state versions. In the backend block I point at a namespaced key like `prod/network/terraform.tfstate` so one bucket holds many environments. Since Terraform 1.10, S3 has native locking via `use_lockfile = true` — no DynamoDB table needed anymore, though lots of setups still use DynamoDB. And I always use a separate backend config file per environment rather than hardcoding values."

```hcl
terraform {
  backend "s3" {
    bucket       = "my-tf-state"
    key          = "prod/network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

**Key Point:** "Dedicated bucket, versioning, encryption, namespaced keys, and S3-native locking — environment values come from backend config files, never hardcoded."

---

### Q7: What does terraform init -migrate-state do, and when do you use it?

**How to Answer:**

"You use it when the backend changes — say you wrote the backend block pointing at S3 while state is still local. `terraform init` detects the mismatch and asks whether to migrate; `-migrate-state` answers yes non-interactively. It copies the local state up to the new backend and then you're done — the old local file gets backed up. The dangerous variant is `-reconfigure`, which skips migration and starts fresh — use it only when you intentionally want a clean slate, never by accident on a project with real resources."

**Key Point:** "Backend changed → init offers to migrate; -migrate-state copies state to the new backend, -reconfigure skips it and starts empty — know which you meant."

---

## State Locking

### Q8: How does state locking work, and what does a stuck lock look like?

**How to Answer:**

"Before any write, Terraform acquires a lock on the state so no two runs write simultaneously — with the S3 backend that means creating a lockfile object, or a conditional write in DynamoDB on older setups. Every other command that needs to write then fails fast with an error showing the lock ID, who holds it, and when it was created. A stuck lock looks exactly like that error but nobody is actually running — it happens when a process dies mid-apply, CI gets killed, or a laptop lid closes. The rule: never unlock blindly, always verify no run is actually in progress first."

**Key Point:** "Locking serializes state writes; a stuck lock is a leftover from a dead process — verify it's truly abandoned before touching it."

---

### Q9: When and how do you use force-unlock?

**How to Answer:**

"Only when you've confirmed the lock holder is dead — checked CI, asked the team, verified no apply is running. Then `terraform force-unlock <lock-id>` releases it, using the ID from the error message. It's safe when the process is genuinely gone, because there's no partial state to reconcile — Terraform writes state atomically. But force-unlocking a live run is how you corrupt state and end up with two applies fighting over the same resources. If I'm ever unsure, I'd rather wait ten minutes than unlock."

**Key Point:** "force-unlock <lock-id> only after confirming the holder is dead — unlocking a live run is how state gets corrupted."

---

## State Operations

### Q10: When do you use terraform state mv?

**How to Answer:**

"When you refactor the config — rename a resource, move it into a module, split one file into two — without wanting to destroy and recreate the real thing. `terraform state mv aws_instance.web aws_instance.web_new` tells state 'this existing object now lives at the new address', so the next plan shows zero changes. It's the standard answer to 'how do you rename a resource safely.' The modern alternative is a `moved {}` block in config, which is declarative and reviewable in a PR — I prefer that in teams, and use the CLI only for one-off fixes."

```hcl
moved {
  from = aws_instance.web
  to   = aws_instance.web_new
}
```

**Key Point:** "state mv (or the moved block) re-addresses a real object so refactors don't trigger destroy-and-recreate."

---

### Q11: What does terraform state rm do, and why is it dangerous but useful?

**How to Answer:**

"It removes a resource from state without touching the real infrastructure — Terraform just forgets it manages that object. Useful when you want to stop managing something but keep it running, like handing a manually-tuned database back to the DBAs. Dangerous because the next apply won't show a destroy — the object becomes invisible to Terraform, and if someone later adds a matching resource block, Terraform creates a duplicate. Always pair it with a plan review and a note in the PR explaining why the resource is leaving management."

**Key Point:** "state rm forgets a resource without destroying it — the object goes invisible to Terraform, so document why and watch for duplicates."

---

### Q12: How do you inspect state without changing anything?

**How to Answer:**

"`terraform state list` for the address inventory, `terraform state show aws_instance.web` for one resource's full attributes. `terraform show` dumps the whole state readably, and `terraform state pull > state.json` downloads the raw JSON for scripting — handy for greps and audits. All read-only, all safe on production. For cross-stack references, the `terraform_remote_state` data source reads another state file's outputs — though I flag that it creates tight coupling, and prefer explicit outputs passed via variables or SSM parameters."

```bash
terraform state list
terraform state show aws_db_instance.main
terraform state pull | jq '.outputs'
```

**Key Point:** "state list, state show, and state pull are read-only windows into state — safe to run anywhere, any time."

---

## Interview Traps and Best Practices

### Q13: Should you commit state files to Git?

**How to Answer:**

"Never. State holds secrets in plain text, it's a constantly rewritten binary-ish JSON that would murder your Git history with merge conflicts, and there's no locking — two branches applying would silently overwrite each other. What goes in Git: the config, the backend configuration, and `.terraform.lock.hcl` for provider pinning. What doesn't: `terraform.tfstate`, backups, and `.terraform/`. The interviewer is checking you can separate what's versioned (code) from what's operational data (state)."

**Key Point:** "State never goes in Git — secrets, churn, and no locking. Version the config and lock file; keep state in an encrypted remote backend."

---

### Q14: The state file is corrupted, or someone deleted resources in the console — what's your recovery playbook?

**How to Answer:**

"First, don't panic and don't apply — a corrupted state plus a blind apply is how things get worse. If the S3 backend has versioning, I restore the last good state version and run plan to confirm the diff is sane. If resources were deleted in the console, plan shows them as needing recreation — that's drift, and the fix is reviewing whether to recreate or `state rm` what shouldn't be managed. If state is truly gone with no backup, `terraform import` rebuilds it resource by resource — painful, which is exactly why versioning and a replicated bucket are table stakes."

**Key Point:** "Restore the last versioned state, review plan before touching anything, import as the last resort — this is why state backups are table stakes."

---
