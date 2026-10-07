# CI/CD Concepts & Pipeline Design Interview Preparation Guide

*How to Answer CI/CD Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Foundations](#foundations)
2. [Pipeline Design](#pipeline-design)
3. [Deploying Safely](#deploying-safely)
4. [Speed and Reliability](#speed-and-reliability)
5. [Security](#security)

---

## Foundations

### Q1: What's the difference between CI, continuous delivery, and continuous deployment?

**How to Answer:**

"CI is just committing to main often and having an automated build plus tests run on every push. It catches integration bugs early, instead of a painful merge day once a month. Continuous delivery takes it further — every main commit is releasable to production, but a human still clicks the deploy button. Continuous deployment goes fully automatic — if everything passes, it ships to prod with no manual step. Most startups do CI plus manual delivery; only teams with really solid test coverage and monitoring can afford true continuous deployment."

**Key Point:** "CI builds and tests every push; delivery keeps it releasable, deployment ships it automatically."

---

### Q2: Walk me through a typical CI/CD pipeline from push to production.

**How to Answer:**

"You push code, and the pipeline triggers on that event. First it checks out the code and builds it — compiling, or building a Docker image. Then fast unit tests run, because they give the quickest feedback. If those pass, slower stages kick in: integration tests, security scans, maybe linting. Then it pushes the artifact to a registry with a unique tag — commit SHA, not `latest`. Staging gets deployed automatically, and after smoke tests or a manual approval, production deploys with a strategy like rolling or canary. The whole thing ends with a notification and metrics so you know exactly what shipped and when."

**Key Point:** "Push → build → fast tests → slow checks → artifact → staging → prod, with notifications at the end."

---

### Q3: What makes a pipeline design good versus bad?

**How to Answer:**

"A good pipeline fails fast — quick checks run first so a broken commit gets rejected in minutes, not after a 40-minute build. Each stage is independent and rerunnable, so you can retry a flaky test without rebuilding everything. Stages produce versioned artifacts so you always deploy the exact thing you tested. A bad pipeline is a slow monolith: it builds, tests, and deploys in one giant step, and when it fails nobody knows where. I've also seen pipelines that test code but deploy something different — that's not a pipeline, that's hope. And a good pipeline tells you what failed in one glance, not buried in 2000 lines of logs."

**Key Point:** "Fail fast, stages rerunnable, deploy exactly what you tested, and make failures obvious."

---

### Q4: How do you handle build artifacts in CI/CD?

**How to Answer:**

"You build once and promote the same artifact through every environment — rebuild for staging versus prod is a classic source of 'it worked in staging' bugs. For Docker workflows, the image is the artifact; tag it with the commit SHA so it's traceable back to source. Push it to a registry — ECR, GCR, Artifactory — and scan it before it can be deployed anywhere. Artifacts should be immutable once built: you don't patch an image, you build a new one. And keep a retention policy, because registries fill up fast and cost real money when you store every image forever."

```bash
# Build once, tag with commit SHA
docker build -t myapp:$(git rev-parse --short HEAD) .
```

**Key Point:** "Build once, tag immutably with the commit SHA, promote the same artifact everywhere."

---

### Q5: Pipeline-as-code versus click-ops GUI pipelines — which is better?

**How to Answer:**

"Pipeline-as-code every time — the pipeline definition lives in the repo, versioned right next to the code it builds. You get code review on pipeline changes, you can diff them, and spinning up an identical pipeline for a new service is trivial. GUI-built pipelines can't be reviewed, can't be diffed, and when someone fat-fingers a checkbox in Jenkins there's no audit trail. The only real trade-off is the learning curve for the DSL, but GitHub Actions YAML or a Jenkinsfile pays for itself fast. Honestly, if your pipeline isn't in git, you don't really know what it's doing."

**Key Point:** "Pipelines in code: reviewed, versioned, reproducible. GUI pipelines: none of that."

---

### Q6: How do you manage secrets in a CI/CD pipeline?

**How to Answer:**

"You never hardcode secrets in pipeline files or Docker images — that's the fastest way to leak an API key on GitHub. Use the platform's secret store — GitHub secrets, Jenkins credentials, Vault — and reference them as environment variables at runtime. The better approach is OIDC federation: the pipeline assumes an IAM role instead of holding long-lived credentials at all. And scope them tight — a secret should only be visible to the jobs and environments that actually need it. One thing people miss: mask secret values in logs, because the first time a secret prints in plain text in CI output, someone's getting paged."

**Key Point:** "No hardcoded secrets; prefer OIDC federation over long-lived keys, and scope everything tight."

---

### Q7: Explain the main deployment strategies — when would you pick each?

**How to Answer:**

"Recreate is the simplest — kill everything, start the new version. It's fine for dev, but there's downtime, so never for prod. Rolling updates replace instances gradually, so there's no downtime, but for a while old and new versions serve traffic together. Blue-green runs two full environments and flips traffic at the router — rollback is instant because the old version is still sitting there warm. Canary sends a small percentage of real traffic to the new version first, watches metrics, then ramps up — it's the safest for risky changes. I'd pick rolling for stateless services, blue-green when I need instant rollback, and canary when the change touches payments or something I really can't break."

**Key Point:** "Recreate for dev, rolling for everyday zero-downtime, blue-green for instant rollback, canary for risky changes."

---

## Deploying Safely

### Q8: How do you gate deployments between environments?

**How to Answer:**

"Between environments you put quality gates — automated checks that have to pass before promotion happens. That means all tests green, no critical CVEs from the image scan, and sometimes a performance baseline check. For production I usually add a manual approval step too, especially in regulated teams — automation handles the how, a human owns the when. Approvals should be tied to a specific person or role with a timeout, not an open button anyone can click. And the gate history matters: in an incident, the first question is always 'what changed?', and your pipeline should answer that instantly."

**Key Point:** "Automated quality gates between environments, manual approval for prod — and keep the audit trail."

---

### Q9: How do you handle rollbacks when a deploy goes wrong?

**How to Answer:**

"A rollback should be a first-class operation, not a panic redeploy. With blue-green or rolling strategies, you just switch traffic back to the previous version — that's why keeping the old version warm matters. In Kubernetes, `kubectl rollout undo` reverts a Deployment to the previous ReplicaSet in seconds. The key design point: rollbacks must not rebuild anything — you redeploy the exact previous artifact, which you only have if you kept it immutable and tagged. And database migrations are the real trap — rolling back code is easy, rolling back a schema change is not, so make migrations backward-compatible with expand-then-contract patterns."

```bash
# Roll back a deployment to the previous revision
kubectl rollout undo deployment/myapp
```

**Key Point:** "Redeploy the previous immutable artifact — never rebuild — and make schema migrations backward-compatible."

---

### Q10: Where do different types of tests belong in a pipeline?

**How to Answer:**

"Fast, cheap tests run early — unit tests and linting in the first stage, because they catch the most common mistakes in seconds. Integration tests come next, running against real-ish dependencies like a test database or localstack. Contract tests sit between services so a breaking API change fails the build before it hits staging. End-to-end tests go last, usually after deploying to staging, because they're slow and flaky by nature. The pyramid guides everything: lots of unit tests, fewer integration tests, very few E2E — and E2E tests that flake get quarantined or deleted, because a red pipeline everyone ignores is worse than no pipeline."

**Key Point:** "Fast tests first, E2E last in staging — and never let flaky tests train people to ignore red builds."

---

### Q11: Push-based versus pull-based deployment — what's the difference?

**How to Answer:**

"In push-based deployment, the CI server pushes the new version into the cluster or server — Jenkins SSHing into boxes or running `kubectl apply`. It's simple, but the pipeline needs direct credentials to production, which is a wide attack surface. Pull-based flips it: an agent inside the environment, like ArgoCD, watches a Git repo and pulls changes when the desired state drifts. That's the GitOps model — git is the single source of truth, and production credentials never leave the cluster. Pull-based is more secure and self-healing, which is why it's the standard for Kubernetes. Push still shows up for serverless or VM-based setups where there's no in-cluster agent."

**Key Point:** "Push: CI writes into prod. Pull (GitOps): an agent inside the environment syncs git's desired state."

---

## Speed and Reliability

### Q12: How do you make pipelines faster without losing reliability?

**How to Answer:**

"First, cache aggressively — dependencies, Docker layers, build outputs — but invalidate on lockfile or Dockerfile changes so the cache can't hide a broken build. Then parallelize: split tests into shards or run independent stages concurrently instead of one long serial chain. Use smaller runner images and the right-sized runners, because a 4GB base image and an underpowered runner waste minutes every run. And only run what's needed — path filters so a docs change doesn't trigger the full integration suite. The trap to avoid is over-caching: a green pipeline that passes because of stale cache is lying to you."

**Key Point:** "Cache dependencies and layers, parallelize independent stages, and skip work that the change doesn't need."

---

### Q13: How do you debug a pipeline that keeps failing?

**How to Answer:**

"I start by reproducing locally — if the pipeline runs tests in Docker, I run the same container and command on my machine. Then I check whether the failure is deterministic: if it passes on retry with no changes, it's flaky, and flaky tests get quarantined before they erode trust. Environment drift is the usual suspect — a runner with a different base image or an expired credential. I also look at what changed recently in the pipeline definition itself, because half of 'broken builds' are actually someone's YAML edit. And I keep pipeline logs structured and searchable, because grepping 2000 lines of raw logs in an incident is how you waste an hour."

**Key Point:** "Reproduce locally, separate flaky from deterministic, check the pipeline's own recent changes, keep logs searchable."

---

## Security

### Q14: How do you secure a CI/CD pipeline against supply-chain attacks?

**How to Answer:**

"The pipeline is production's front door, so treat it like one. Pin all actions and dependencies to specific SHAs or versions — a floating `latest` tag on a GitHub Action is an invitation for a compromise. Scan everything: source code with SAST, dependencies for known CVEs, and container images before they can be promoted. Sign artifacts with something like cosign so downstream knows the image actually came from your pipeline. And lock down who can change the pipeline itself — branch protection on the workflow files and required reviews, because whoever controls the pipeline controls what ships to prod."

**Key Point:** "Pin dependencies, scan code and images, sign artifacts, and protect the pipeline definition like production code."

---

*Day 28 of 58 — written for DevOps & SRE interviews at 1-3 YOE.*

