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

<!-- PART 2 CONTINUES BELOW -->
