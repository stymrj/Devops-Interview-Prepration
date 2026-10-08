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

---

## Secrets, Variables and Security

### Q8: How do secrets and variables work in GitHub Actions?

**How to Answer:**

"Secrets are encrypted values — API keys, tokens, cloud credentials — referenced as `${{ secrets.DB_PASSWORD }}` and never shown in logs; GitHub masks them automatically. Variables are plain config via `${{ vars.ENV_NAME }}`, visible in logs. You scope both at repo, environment, or org level, and environments can add required reviewers. The interview trap: secrets aren't available to workflows triggered by `pull_request` from forks by default — that blocks PRs from exfiltrating secrets. That's why fork PRs often need `pull_request_target`, used carefully."

**Key Point:** "Secrets are encrypted and log-masked; variables are plain config. Fork PRs don't get secrets by default."

---

### Q9: What's OIDC, and why is it better than storing cloud credentials as secrets?

**How to Answer:**

"OIDC lets the workflow mint a short-lived token that the cloud provider trusts — no long-lived AWS keys sitting in GitHub secrets. You add `permissions: id-token: write`, the runner gets a JWT from GitHub, and AWS IAM validates it against an identity provider you registered. If the workflow finishes, the token dies. I prefer it because leaked static keys are how breaches happen — with OIDC there's nothing to leak or rotate. Interviewers love this question because it's the modern answer to 'how do you deploy from CI securely'."

```yaml
permissions:
  id-token: write
  contents: read
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-deploy
```

**Key Point:** "OIDC mints short-lived cloud tokens per run — no static keys to leak, rotate, or steal."

---

### Q10: How do environments and deployment protection rules work?

**How to Answer:**

"Environments like `staging` and `production` are named targets you attach to jobs with `environment: production`. Each environment can have its own secrets, required reviewers, and wait timers — so production deploys need a human approval while staging auto-deploys. You can also restrict which branches can deploy to an environment. The pattern I use: staging deploys on every main push automatically, production needs a reviewer click. Interviewers want to hear that environments are your blast-radius control, not just labels."

**Key Point:** "Environments bundle secrets, reviewer approvals, and branch rules — they're your blast-radius control for deploys."

---

## Expressions, Matrix and Reusable Workflows

### Q11: What are contexts and expressions? Give me a real example.

**How to Answer:**

"Contexts are objects GitHub injects — `github`, `env`, `secrets`, `matrix`, `steps` — and expressions `${{ ... }}` evaluate them. A real example: `${{ github.event_name == 'pull_request' && 'preview' || 'prod' }}` to pick a deploy target. Another: `${{ steps.build.outputs.image }}` to grab a step's output. The trap is forgetting that expressions in `if:` don't need the `${{ }}` wrapper — `if: github.ref == 'refs/heads/main'` works, wrapping it is harmless but redundant. Short-circuit logic in expressions is how you write branch-free conditional values."

**Key Point:** "Contexts like github, env, and matrix feed `${{ }}` expressions — they're how workflows stay branch-free and data-driven."

---

### Q12: How do matrix builds work, and what's the include/exclude trick?

**How to Answer:**

"A matrix fans one job out into many — say Node 18, 20, 22 across ubuntu and windows — so you test combinations without copy-pasting jobs. You write `strategy: matrix: node: [18, 20, 22]`, and reference `${{ matrix.node }}` in steps. `include` adds one-off combos like an experimental Node 23 on ubuntu only; `exclude` drops broken combos like a version that doesn't support windows. Add `fail-fast: false` so one failing combo doesn't kill the whole matrix. Interviewers ask this to check whether you've actually tested across versions or just run one happy path."

```yaml
strategy:
  fail-fast: false
  matrix:
    node: [18, 20, 22]
    os: [ubuntu-latest, windows-latest]
    exclude:
      - node: 22
        os: windows-latest
```

**Key Point:** "Matrix fans a job across version/OS combos; include and exclude fine-tune the grid, fail-fast: false keeps one bad combo from killing all."

---

### Q13: How do reusable workflows and composite actions reduce duplication?

**How to Answer:**

"Reusable workflows let you extract a whole job set — say a standard 'build, test, scan' pipeline — into one file triggered with `on: workflow_call`, then other repos call it with `uses: org/repo/.github/workflows/ci.yml@main`. Composite actions are smaller: a bundle of steps you call inside a job, like a custom 'setup my toolchain' step. Rule of thumb: composite action for shared steps, reusable workflow for shared pipelines. The governance win: update the deploy pipeline once and every repo inherits it. That's how platform teams keep hundreds of repos consistent."

**Key Point:** "Composite actions share steps; reusable workflows share whole pipelines — update once, every repo inherits."

---

## Troubleshooting and Best Practices

### Q14: A workflow is failing — how do you debug it?

**How to Answer:**

"First I re-run with debug logging enabled — set the `ACTIONS_STEP_DEBUG` secret to true for verbose step output. Then I check the obvious: did `actions/checkout` actually run, are secrets present (a masked empty secret fails silently), is the `if:` condition evaluating the way I think. For step outputs I `echo` values into `$GITHUB_OUTPUT` and print them. If it's flaky, I check runner image updates — `ubuntu-latest` moves and breaks things. And for really stubborn cases, I reproduce locally with `act` or nektos/act to run the workflow on my machine. The fastest wins are usually a wrong context name or a secret that doesn't exist."

**Key Point:** "Enable step-debug logging, verify checkout and secrets first, reproduce locally with act for stubborn failures."

---

### Q15: What are your non-negotiable GitHub Actions best practices?

**How to Answer:**

"Pin every third-party action to a full commit SHA, not just `@v4` — tags can be moved, SHAs can't. Give each job minimal `permissions:` instead of the default broad token. Never run untrusted code with write permissions — fork PRs get read-only and no secrets. Cache dependencies keyed on lockfiles, and use concurrency groups to cancel superseded runs so pushes don't queue up. And keep workflows small: extract shared logic into reusable workflows so a security fix lands everywhere at once. These are the answers that signal you've operated Actions at scale, not just written one tutorial workflow."

**Key Point:** "Pin actions to SHAs, least-privilege permissions, concurrency to cancel stale runs, and extract shared logic into reusable workflows."
