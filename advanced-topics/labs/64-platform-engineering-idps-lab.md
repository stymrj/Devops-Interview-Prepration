# Platform Engineering & IDPs — Hands-On Lab

Nine drills that turn platform answers into real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Install Backstage locally and explore the catalogue

**Goal:** Get a developer portal running and see what an IDP frontend actually looks like.

**Commands:**

```bash
npx -y @backstage/create-app@latest
cd my-backstage-app
yarn install && yarn dev
# Open http://localhost:3000 — explore the software catalogue and templates.
```

**Expected output:** Backstage runs locally, with the default catalogue and example templates visible.

**Why it matters:** This is the portal layer from Q6 — you've seen it, not just named it.

---

## Exercise 2: Write a software template that scaffolds a service

**Goal:** Build the golden-path entry point — one form, one repo.

**Commands:**

```bash
mkdir -p templates/node-service/content
cat > templates/node-service/template.yaml <<'EOF'
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata: { name: node-service, title: New Node.js Service }
spec:
  owner: platform-team
  type: service
  parameters:
    - title: Service details
      properties:
        name: { type: string, title: Service name }
  steps:
    - id: fetch
      action: fetch:template
      input: { url: ./content, values: { name: ${{ parameters.name }} } }
EOF
echo '{"name":"{{ values.name }}","version":"1.0.0"}' > templates/node-service/content/package.json
```

**Expected output:** A valid template file that Backstage can load and render as a form.

**Why it matters:** This is the golden path from Q4, in its simplest working form.

---

## Exercise 3: Create a golden-path pipeline template

**Goal:** Make the paved road include CI, not just scaffolding.

**Commands:**

```bash
mkdir -p templates/node-service/content/.github/workflows
cat > templates/node-service/content/.github/workflows/ci.yaml <<'EOF'
name: CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci && npm test
      - run: docker build -t myorg/${{ values.name }}:${{ github.sha }} .
EOF
```

**Expected output:** Every service scaffolded from the template ships with a working CI pipeline.

**Why it matters:** A golden path that stops at scaffolding still leaves teams copying pipelines by hand.

---

## Exercise 4: Provision a namespace and database with Terraform, triggered by a template

**Goal:** Wire self-service to real provisioning, not a ticket.

**Commands:**

```bash
cat > main.tf <<'EOF'
variable "team" { type = string }
resource "kubernetes_namespace" "team" {
  metadata: { name = var.team, labels = { team = var.team } }
}
resource "kubernetes_network_policy" "default_deny" {
  metadata: { name = "default-deny", namespace = var.team }
  spec: {
    pod_selector = {}
    policy_types = ["Ingress", "Egress"]
  }
}
EOF
terraform init && terraform apply -var team=checkout -auto-approve
kubectl get namespace checkout
```

**Expected output:** A team namespace exists with a default-deny network policy applied.

**Why it matters:** This is the self-service provisioning from Q7 — automation behind the form, landing in Git.

---

## Exercise 5: Test the abstraction level — one override, not a fork

**Goal:** Prove the escape hatch works before a team needs it in anger.

**Commands:**

```bash
# The golden path defaults to a small database; override it for one service:
cat > values-override.yaml <<'EOF'
database:
  instanceClass: db.t3.medium
  backupRetentionDays: 14
EOF
# Apply the override on top of the golden-path defaults (helm-style merge):
helm upgrade --install checkout-db ./charts/db -f values-default.yaml -f values-override.yaml
helm get values checkout-db
```

**Expected output:** The service runs with the overridden instance class while everything else stays on defaults.

**Why it matters:** This is the Q8 rule in action — sensible defaults plus a documented override, not a fork.

---

## Exercise 6: Measure lead time for a service built on the golden path

**Goal:** Get a real platform metric instead of a vanity one.

**Commands:**

```bash
# Time from first commit to production for a golden-path service:
gh api repos/myorg/checkout-service/commits --jq '.[-1].commit.committer.date'
gh api repos/myorg/checkout-service/deployments --jq '.[0].created_at'
# Lead time = deployment time minus first-commit time; compare it with a
# hand-rolled service's lead time from the same API calls.
```

**Expected output:** A lead-time number for the golden-path service that you can compare against a hand-rolled one.

**Why it matters:** This is the Q10 measurement — lead time and adoption, not templates shipped.

---

## Exercise 7: Run a canary upgrade of a platform template

**Goal:** Practise day-2 platform work — upgrade one team first, not everyone at once.

**Commands:**

```bash
# Bump the base image in the template, then migrate one volunteer team:
sed -i 's/node:20-alpine/node:22-alpine/' templates/node-service/content/Dockerfile
# Volunteer team opens a PR bumping their service; review, merge, watch CI:
gh pr create --repo myorg/checkout-service --title "chore: adopt platform base image v2" --body "Canary upgrade of the golden path."
gh pr checks --repo myorg/checkout-service --watch
```

**Expected output:** The volunteer team's service passes CI on the new base image before any other team migrates.

**Why it matters:** This is the Q11 versioning discipline — canary upgrades with a migration path, not surprise breakage.

---

## Exercise 8: Add per-team cost visibility

**Goal:** Give teams the bill view that makes platform costs real.

**Commands:**

```bash
kubectl label namespace checkout cost-center=checkout-team --overwrite
kubectl create namespace platform-costs
# In your cost tool (e.g. OpenCost or a cloud cost report), group by the
# cost-center label and export one team's weekly spend:
kubectl -n platform-costs run cost-report --image=curlimages/curl --restart=Never -- \
  curl -s "http://opencost:9003/allocation/compute?aggregate=label:cost-center&window=7d"
```

**Expected output:** A per-team cost breakdown, grouped by the cost-center label.

**Why it matters:** Cost showback is part of the Q11 day-2 story — teams manage what they can see.

---

## Exercise 9: Interview a developer and write the platform backlog

**Goal:** Practise the user research that keeps a platform team alive.

**Commands:**

```bash
# Ask one developer these three questions and write the answers down:
cat <<'EOF'
Platform user interview:
1. What took longest between "I want to ship" and "it's live" this month?
2. What did you copy-paste from another repo because no golden path existed?
3. What would you stop doing if the platform handled it for you?
EOF
# Turn the top answer into ONE backlog item: "Golden path for ___ — fixes pain ___."
```

**Expected output:** A backlog item tied to a real, named developer pain — not a guessed feature.

**Why it matters:** This is the Q9/Q12 antidote — ship from real user pain, not from a roadmap written in isolation.
