# GitLab CI Interview Preparation Guide

*How to Answer GitLab CI Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Pipeline Basics](#pipeline-basics)
2. [Stages Jobs and Runners](#stages-jobs-and-runners)
3. [Variables Environments and Security](#variables-environments-and-security)
4. [Rules Workflows and Advanced Patterns](#rules-workflows-and-advanced-patterns)
5. [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)

---

## Pipeline Basics

### Q1: What is GitLab CI/CD, and how does it fit into GitLab's workflow?

**How to Answer:**

"GitLab CI/CD is the built-in automation engine that lives inside GitLab — no plugin, no separate server. You commit a `.gitlab-ci.yml` at the repo root, and GitLab automatically runs pipelines on pushes, merges, schedules, whatever you define. The big sell is it's one platform: code, issues, CI, deployments, and security scanning all live together, so there's no stitching Jenkins to a repo host. One trap I see: people treat it like Jenkins with YAML — but GitLab pushes a pipeline-per-commit model, which changes how you think about artifacts and environments."

**Key Point:** "It's CI/CD built into the repo platform itself — pipelines defined in `.gitlab-ci.yml`, run automatically per commit."

---

### Q2: Walk me through a basic `.gitlab-ci.yml` file.

**How to Answer:**

"Start with `stages:` — say `build`, `test`, `deploy` — that sets the execution order. Then you define jobs, and each job picks a stage with `stage: test`. The simplest useful job is three lines: a stage, an image if you want Docker, and a `script:` with your commands. For example, a test job on the `node:20` image running `npm ci && npm test`. What trips people up: if you never declare `stages:`, GitLab still gives you default `build`, `test`, `deploy` stages — so jobs land in stages you didn't define on purpose."

```yaml
stages: [build, test, deploy]

test-app:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm test
```

**Key Point:** "stages set the order, jobs pick a stage, script holds the commands — and default stages exist even if you don't declare them."

---

### Q3: What's the difference between a stage and a job?

**How to Answer:**

"A stage is a phase of the pipeline — build, test, deploy — and stages run in strict order. A job is one unit of work inside a stage — `lint`, `unit-tests`, `integration-tests` could all be jobs in the `test` stage. Jobs in the same stage run in parallel by default. The key mechanics: the pipeline moves to the next stage only when every job in the current stage finishes successfully. So if one test job fails, the whole pipeline blocks the deploy stage — unless you mark the job `allow_failure: true`."

**Key Point:** "Stages are sequential phases; jobs are parallel tasks inside a stage; one failed job blocks the next stage unless allow_failure is set."

---

### Q4: What is a GitLab Runner, and what executor types matter?

**How to Answer:**

"A Runner is the agent that actually executes your jobs — the pipeline is just a plan until a Runner picks it up. The executor decides where the job runs: `docker` spins up a container per job, `shell` runs directly on the host, `kubernetes` creates a pod per job. Most teams use Docker executors on auto-scaling compute. One thing interviewers love: Runners register with tags — like `docker` or `gpu` — and jobs pick Runners via `tags:`, so you route heavy jobs to the right machines."

**Key Point:** "Runners execute the jobs; the executor (docker, shell, kubernetes) decides where; tags route jobs to the right Runners."

---

### Q5: Artifacts vs cache — what's the difference?

**How to Answer:**

"Artifacts pass files *forward* — build output, test reports, binaries — from one job or stage to a later one, and they get uploaded to GitLab when the job finishes. Cache is for speeding up *repeated* runs: `node_modules`, Go module caches, things you don't want to download every time. Big trap: artifacts are per-pipeline and expire by default in 30 days; cache is per-branch and shared across pipelines on that branch. And cache isn't guaranteed — a Runner may miss it — so your job must still work from scratch."

```yaml
build-app:
  stage: build
  script: npm ci && npm run build
  artifacts:
    paths: [dist/]
    expire_in: 1 week
  cache:
    paths: [node_modules/]
```

**Key Point:** "Artifacts move files forward through the pipeline; cache speeds up repeated runs and is never guaranteed."

---

### Q6: How do you handle secrets and variables in GitLab CI?

**How to Answer:**

"CI/CD variables in Settings — marked masked and protected. Masked hides them from job logs, protected means they only inject on protected branches or tags — that's the combo I use for production secrets. You also get predefined variables like `CI_COMMIT_SHA` and `CI_ENVIRONMENT_NAME` for free. The classic mistake: echoing a masked variable can still leak it if it appears as a substring of a longer string — masking isn't magic. For real secret management I'd pull from Vault or AWS Secrets Manager at runtime instead of storing long-lived secrets in GitLab."

**Key Point:** "Use masked + protected variables for secrets; protected ties them to protected branches; prefer external secret stores for the real stuff."

---

### Q7: What are environments, and what's a review app?

**How to Answer:**

"An environment is a named deployment target — `staging`, `production` — declared with `environment:` on a deploy job. GitLab then tracks what's deployed where, shows it on merge requests, and lets you stop or roll back from the UI. Review apps are dynamic environments: GitLab spins one up per merge request using the branch name, so reviewers can click through a live version of the change. The gotcha: you need a stop job with `environment: { action: stop }` or you'll accumulate review apps and burn compute."

```yaml
deploy-review:
  stage: deploy
  script: ./deploy.sh $CI_ENVIRONMENT_NAME
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.example.com
    on_stop: stop-review
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**Key Point:** "Environments track deployments per target; review apps are dynamic per-MR environments that need a stop job to clean up."
