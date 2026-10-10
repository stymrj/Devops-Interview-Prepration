# Terraform State Management — Hands-On Lab

Nine exercises that make state feel real instead of theoretical. Run them in order — each builds on the last.

> Estimated time: 60–90 minutes. You'll need Terraform ≥ 1.10 installed and AWS credentials with S3 permissions. Nothing here creates billable resources — we use `null_resource` and locals so the lab is free.

---

## Exercise 1: Inspect a local state file

**Goal:** See what Terraform actually records after an apply.

**Commands:**

```bash
mkdir tf-state-lab && cd tf-state-lab
cat > main.tf <<'EOF'
terraform {
  required_version = ">= 1.5"
}
resource "null_resource" "demo" {}
EOF
terraform init && terraform apply -auto-approve
cat terraform.tfstate | head -40
```

**Expected output:** A JSON file with `terraform_version`, `serial: 1`, and a `resources` array holding `null_resource.demo`.

**Why it matters:** Most people answer state questions from theory. Having seen the raw JSON, you can talk about serials, attributes, and secrets-in-plaintext with authority.

---

## Exercise 2: Prove secrets land in state

**Goal:** Understand why state must never be committed or shared carelessly.

**Commands:**

```bash
cat >> main.tf <<'EOF'
variable "db_password" {
  type      = string
  sensitive = true
  default   = "super-secret-123"
}
output "pw" { value = var.db_password, sensitive = true }
EOF
terraform apply -auto-approve
grep -o 'super-secret-123' terraform.tfstate
```

**Expected output:** The "secret" appears in `terraform.tfstate` — `sensitive = true` only masks console output, never the state file.

**Why it matters:** This single experiment answers "why not commit state to Git" better than any definition. Interviewers love candidates who know sensitive ≠ encrypted.

---

## Exercise 3: Create an S3 backend bucket properly

**Goal:** Build the dedicated, versioned, encrypted bucket state deserves.

**Commands:**

```bash
BUCKET="tf-state-lab-$(date +%s)"
aws s3api create-bucket --bucket "$BUCKET" --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
aws s3api put-bucket-versioning --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket "$BUCKET" \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws s3api put-public-access-block --bucket "$BUCKET" \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
echo "$BUCKET" > .bucket-name
```

**Expected output:** The bucket name saved in `.bucket-name`; versioning and encryption confirmed via `aws s3api get-bucket-versioning` / `get-bucket-encryption`.

**Why it matters:** Versioning is your state backup. Encryption is your secrets story. Public-access-blocked is your "we take security seriously" line in the interview.

---

## Exercise 4: Migrate local state to the S3 backend

**Goal:** Feel the `-migrate-state` flow you'll describe in interviews.

**Commands:**

```bash
BUCKET=$(cat .bucket-name)
cat > backend.hcl <<EOF
bucket       = "$BUCKET"
key          = "lab/terraform.tfstate"
region       = "ap-south-1"
encrypt      = true
use_lockfile = true
EOF
cat >> main.tf <<'EOF'
terraform {
  backend "s3" {}
}
EOF
terraform init -backend-config=backend.hcl -migrate-state
aws s3 ls "s3://$BUCKET/lab/"
```

**Expected output:** Terraform asks to copy state to S3, confirms, and the object `lab/terraform.tfstate` appears in the bucket. Your local `terraform.tfstate` is backed up.

**Why it matters:** This is exactly the Q7 scenario. Doing it once means you describe it from memory, not from docs.

---

## Exercise 5: Watch locking happen

**Goal:** See the lock acquire and release around a real apply.

**Commands:**

```bash
terraform apply -auto-approve &
sleep 3
BUCKET=$(cat .bucket-name)
aws s3 ls "s3://$BUCKET/lab/terraform.tfstate.tflock" 2>&1
wait
aws s3 ls "s3://$BUCKET/lab/terraform.tfstate.tflock" 2>&1
```

**Expected output:** During the apply, the `.tflock` object exists in S3; after the apply finishes, it's gone.

**Why it matters:** Locking stops being abstract the moment you watch a lockfile appear and vanish. This is the "how does locking work" answer, demonstrated.

---

## Exercise 6: Simulate a stuck lock and force-unlock it

**Goal:** Practice the recovery flow from Q9 without breaking anything.

**Commands:**

```bash
BUCKET=$(cat .bucket-name)
echo '{"ID":"lab-simulated-lock","Operation":"apply"}' | \
  aws s3 cp - "s3://$BUCKET/lab/terraform.tfstate.tflock"
terraform plan 2>&1 | head -5
terraform force-unlock lab-simulated-lock -force
terraform plan 2>&1 | tail -3
```

**Expected output:** `terraform plan` fails with "Error acquiring the state lock" showing your fake ID; `force-unlock` releases it and the next plan succeeds.

**Why it matters:** Interviewers ask "what do you do with a stuck lock?" — now your answer starts with "last time I practiced it…"

---

## Exercise 7: Rename a resource with state mv

**Goal:** Refactor safely — the Q10 scenario.

**Commands:**

```bash
terraform state mv null_resource.demo null_resource.demo_renamed
sed -i 's/resource "null_resource" "demo"/resource "null_resource" "demo_renamed"/' main.tf
terraform plan
```

**Expected output:** `terraform plan` reports "No changes. Your infrastructure matches the configuration." — the object was re-addressed, not recreated.

**Why it matters:** This is the difference between a confident "I'd use state mv" and a candidate who'd destroy production with a rename.

---

## Exercise 8: Inspect state read-only

**Goal:** Learn the safe inspection commands from Q12.

**Commands:**

```bash
terraform state list
terraform state show null_resource.demo_renamed
terraform state pull | jq '.serial, (.resources | length)'
terraform show | head -20
```

**Expected output:** The address list, one resource's full attribute dump, the serial number and resource count from raw JSON, and the human-readable `terraform show` rendering.

**Why it matters:** These are the commands you run on production state without fear — and saying "all read-only, safe anywhere" in an interview signals operational maturity.

---

## Exercise 9: Restore a previous state version

**Goal:** Prove versioning is your backup strategy.

**Commands:**

```bash
BUCKET=$(cat .bucket-name)
aws s3api list-object-versions --bucket "$BUCKET" --prefix lab/terraform.tfstate \
  --query 'Versions[].{V:VersionId, M:LastModified}' --output table
```

**Expected output:** A table of state versions — one per apply — each restorable with `aws s3api get-object --version-id`.

**Why it matters:** This is the opening move of the Q14 recovery playbook. Seeing the version history makes "restore the last good state" a concrete action, not a vague plan.

---

## Cleanup

```bash
BUCKET=$(cat .bucket-name)
aws s3 rm "s3://$BUCKET" --recursive
aws s3api delete-bucket --bucket "$BUCKET"
cd .. && rm -rf tf-state-lab
```

State lab leaves nothing behind — including, fittingly, no state.
