# DevOps System Design Round — Hands-On Lab

Ten drills that make whiteboard answers sound like real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Draw then build a pipeline skeleton

**Goal:** Turn a whiteboard pipeline into a real GitHub Actions file.

**Commands:**

```bash
mkdir -p pipeline-lab && cd pipeline-lab && git init -b main
mkdir -p .github/workflows
cat > .github/workflows/pipeline.yml <<'EOF'
name: pipeline
on: { push: { branches: [main] }, pull_request: {} }
jobs:
  build: { runs-on: ubuntu-latest, steps: [{ run: echo "build + unit tests" }] }
EOF
git add . && git commit -m "pipeline skeleton"
```

**Expected output:** A commit on `main` with `.github/workflows/pipeline.yml` in it.

**Why it matters:** Saying "pipeline as code" is cheap — having built one lets you describe stage boundaries from memory.

---

## Exercise 2: Add a canary stage to that pipeline

**Goal:** Practice describing progressive delivery concretely.

**Commands:**

```bash
# edit pipeline.yml: add a deploy job gated on metrics
grep -n "jobs" .github/workflows/pipeline.yml
echo "canary: 5% traffic -> check error rate -> promote or rollback" >> DESIGN-NOTES.md
```

**Expected output:** `DESIGN-NOTES.md` with a written promotion/rollback policy in your own words.

**Why it matters:** Interviewers probe canary logic — "what metric promotes?" You need an answer, not a diagram box.

---

## Exercise 3: Terraform a two-AZ VPC on paper, then for real

**Goal:** Be able to sketch network topology without hesitation.

**Commands:**

```bash
mkdir vpc-lab && cd vpc-lab
terraform init -backend=false
# write main.tf: 1 VPC, 2 public + 2 private subnets, 1 NAT gateway
terraform fmt && terraform validate
```

**Expected output:** `terraform validate` prints `Success! The configuration is valid.`

**Why it matters:** "Two AZs with NAT" is the opening move of every HA design — fumble it and the round wobbles.

---

## Exercise 4: Load-test a local app and find the first bottleneck

**Goal:** Practice the "what breaks next?" reasoning out loud.

**Commands:**

```bash
docker run -d -p 8080:80 --name webapp nginx:alpine
# install k6, then: k6 run --vus 50 --duration 30s -e URL=http://localhost:8080 load.js
docker stats webapp --no-stream
```

**Expected output:** `docker stats` shows CPU climbing toward 100% as k6 ramps up.

**Why it matters:** Scaling answers are judged on whether you can name the bottleneck and the fix in order.

---

## Exercise 5: Put a cache in front of the "database"

**Goal:** Feel the latency difference caching makes.

**Commands:**

```bash
docker run -d -p 6379:6379 --name cache redis:alpine
redis-cli -p 6379 SET session:abc '{"user":"satyam"}' EX 300
redis-cli -p 6379 GET session:abc
```

**Expected output:** The `GET` returns the JSON instantly; `TTL session:abc` counts down from ~300.

**Why it matters:** "Add caching" is step two of every scale-up answer — now you can say exactly what you'd cache and for how long.

---

## Exercise 6: Run a tiny chaos drill with a kill switch

**Goal:** Practice designing for failure, not just drawing it.

**Commands:**

```bash
docker run -d --name fragile nginx:alpine
docker kill fragile          # simulate a node dying
docker ps -a | grep fragile # confirm it's dead
docker start fragile && curl -s -o /dev/null -w "%{http_code}\n" localhost:8080
```

**Expected output:** `curl` returns `200` only after `docker start` — proving a single instance has no redundancy.

**Why it matters:** You can now argue for replicas from a lived demo, not a textbook.

---

## Exercise 7: Price two architectures on the back of an envelope

**Goal:** Answer cost questions with numbers, not vibes.

**Commands:**

```bash
cat > cost.md <<'EOF'
Option A: 4x t3.medium on-demand, 24/7     -> ~4 x $30 = $120/mo
Option B: same, but 3 on spot + ASG scale   -> ~$40-60/mo
Data transfer out 1TB                       -> ~$90/mo (the real killer)
EOF
cat cost.md
```

**Expected output:** A written comparison naming the most expensive line item (hint: it's usually transfer).

**Why it matters:** "Too expensive" follow-ups are easy when you've already done the arithmetic once.

---

## Exercise 8: Write SLOs for a service you actually use

**Goal:** Practice defining SLIs/SLOs from user pain, not infrastructure metrics.

**Commands:**

```bash
cat > slos.md <<'EOF'
Service: personal blog
SLI: % of page loads < 2s (measured at the edge)
SLO: 99% over 30 days
Burn alert: 2% budget in 1h -> page; 10% in 6h -> ticket
EOF
cat slos.md
```

**Expected output:** One page with an SLI, an SLO, and a two-tier alert policy.

**Why it matters:** "Page only on SLO burn" is a line interviewers remember — you need to be able to write one on demand.

---

## Exercise 9: Do a timed mock — 45 minutes, one prompt

**Goal:** Simulate the real round under pressure.

**Commands:**

```bash
# set a timer, pick ONE prompt, talk out loud while drawing:
# "Design a CI/CD pipeline for a 20-person team shipping a Node.js monolith."
# Rules: 5 min questions, 10 min sketch, 25 min deep dives, 5 min wrap-up.
date && echo "timer started"
```

**Expected output:** A photo of your whiteboard/paper sketch plus three things you'd improve next time.

**Why it matters:** The framework only works if you've run it before the interview — timing collapses under pressure otherwise.

---

## Exercise 10: Write the tradeoffs table for your mock design

**Goal:** Train the habit of naming what you gave up.

**Commands:**

```bash
cat > tradeoffs.md <<'EOF'
| Choice            | Gained            | Gave up              |
|-------------------|-------------------|----------------------|
| Rolling deploys   | Zero downtime     | Slow bad-version kill|
| Managed RDS       | No ops burden     | 3x cost vs self-host |
| Single region     | Simple + cheap    | Region-level outage  |
EOF
cat tradeoffs.md
```

**Expected output:** A filled tradeoffs table for your Exercise 9 design.

**Why it matters:** Ending a design answer with "here's what I'd revisit at 10x scale" is what separates senior-sounding answers from junior ones.

---
