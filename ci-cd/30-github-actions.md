# GitHub Actions Interview Preparation Guide

*How to Answer GitHub Actions Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Workflow Basics](#workflow-basics)
2. [Events, Jobs and Runners](#events-jobs-and-runners)
3. [Secrets, Variables and Security](#secrets-variables-and-security)
4. [Expressions, Matrix and Reusable Workflows](#expressions-matrix-and-reusable-workflows)
5. [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)

---

## Workflow Basics

### Q1: What is GitHub Actions, and how do workflows, jobs, and steps fit together?

**How to Answer:**

"GitHub Actions is GitHub's built-in CI/CD engine — you define automation as YAML in `.github/workflows/` and it runs on pushes, PRs, schedules, whatever you trigger on. A workflow is the whole automation. Inside it you have jobs, which run in parallel by default on separate runners. Each job is a sequence of steps — either `run` commands or `uses` actions. The gotcha interviewers want: steps inside a job share a filesystem, but jobs don't share anything unless you pass artifacts explicitly."

**Key Point:** "Workflows contain jobs that run in parallel; steps inside a job run sequentially and share the filesystem."

---

### Q2: What's the difference between an action and a workflow?

**How to Answer:**

"A workflow is your automation defined end-to-end — the triggers, jobs, and steps for a specific repo. An action is a reusable building block a step calls with `uses:` — like `actions/checkout@v4` or `aws-actions/configure-aws-credentials`. Think of it like: workflows are the program, actions are the functions. You can write actions in composite steps or Docker/JS, but in interviews the key point is you don't reinvent checkout, caching, or cloud login — you reuse community actions."

**Key Point:** "Workflows are the full automation; actions are reusable steps you call with `uses:`."

---

### Q3: How does a basic workflow file look? Walk me through one.

**How to Answer:**

"It starts with `name:` and `on:` for triggers — say `push` to main and `pull_request`. Then `jobs:` — I'd define a `build` job with `runs-on: ubuntu-latest`. Inside, steps: first `actions/checkout@v4` to get the code, then `actions/setup-node@v4` with a version, then `run: npm ci && npm test`. One trap: `on: push` with no branch filter fires on every branch, which burns minutes fast. I'd always scope triggers early."

```yaml
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
```

**Key Point:** "name, on, jobs, steps — and always scope your triggers so you don't burn minutes on every branch."

---

## Events, Jobs and Runners

### Q4: What triggers can start a workflow, and how do pull_request and push differ?

**How to Answer:**

"The big ones are `push`, `pull_request`, `schedule` for cron, `workflow_dispatch` for manual runs, and `workflow_call` for reusable workflows. `push` fires when commits land — on main that's after merge. `pull_request` fires on the PR lifecycle: opened, synchronized, reopened. The difference matters because `pull_request` runs in the context of the PR head and gets PR info, while `push` just runs against the branch. For PR checks I always use `pull_request` so the checks show up as status checks blocking the merge."

**Key Point:** "push fires on commits landing; pull_request fires on the PR lifecycle and powers merge-blocking status checks."

---

### Q5: How do you control job order — needs, if conditions, and dependencies?

**How to Answer:**

"Jobs run in parallel unless you say `needs:`. So a deploy job would have `needs: [build, test]` and it only runs when both pass. If a needed job fails, dependents get skipped by default. You can add `if:` conditions too — like `if: github.ref == 'refs/heads/main'` so deploy only runs on main. There's also `always()` and `failure()` functions for cleanup jobs that must run regardless. The classic interview trap: `needs` creates the dependency graph, `if` gates whether a job runs — they're different levers."

```yaml
deploy:
  needs: [build, test]
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  steps:
    - run: ./deploy.sh
```

**Key Point:** "needs wires the dependency graph; if decides whether a job runs at all."

---

### Q6: GitHub-hosted vs self-hosted runners — when do you pick which?

**How to Answer:**

"GitHub-hosted runners are the default VMs — ubuntu, windows, macOS — maintained by GitHub, and you just say `runs-on: ubuntu-latest`. Zero maintenance, but you pay per minute and you can't reach private networks. Self-hosted runners are machines you register — a VM, an EC2 box, even on-prem hardware. I pick self-hosted when builds need VPC access, special hardware like GPUs, or long-lived caches that GitHub-hosted wipes. The tradeoff interviewers probe: self-hosted means YOU own the patching, scaling, and the security risk of arbitrary code running on your box from PRs."

**Key Point:** "GitHub-hosted is zero-maintenance and public-network; self-hosted wins for VPC access, GPUs, and heavy caching — but you own its security."

---

### Q7: How do artifacts and caches work, and when do you use each?

**How to Answer:**

"Artifacts pass files between jobs or keep them after a run — build outputs, test reports, deployment packages. You upload with `actions/upload-artifact` and download with `actions/download-artifact`. Caching is different: it's for dependencies like npm modules or pip packages to speed up runs, using `actions/cache` or the built-in cache in setup actions. The rule of thumb: artifacts carry build outputs downstream; caches carry dependencies to make the same job faster next time. And caches are immutable by key — if the key matches, you get the old cache, so I always key on the lockfile hash."

**Key Point:** "Artifacts move build outputs between jobs; caches speed up repeated dependency installs and are keyed on lockfile hashes."
