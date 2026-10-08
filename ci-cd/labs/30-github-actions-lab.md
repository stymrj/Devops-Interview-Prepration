# GitHub Actions Hands-On Lab

Practical exercises for Topic 30 — GitHub Actions. You'll need a GitHub account and any repo you can push to. Each exercise builds on the last.

## Exercise 1: Your first workflow

**Goal:** Create a workflow that runs on every push and prints a greeting.

1. In your repo, create `.github/workflows/hello.yml`:

```yaml
name: Hello
on: [push]
jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello from GitHub Actions"
```

2. Commit and push it. Go to the **Actions** tab — your run should appear within seconds.

**Expected output:** A green check with "Hello from GitHub Actions" in the logs.

**Why it matters:** This is the skeleton of every CI pipeline you'll ever write. The Actions tab is where you'll live.

---

## Exercise 2: Scope your triggers

**Goal:** Stop the workflow from running on every branch.

1. Change `on: [push]` to:

```yaml
on:
  push:
    branches: [main]
  pull_request:
```

2. Push a commit to a feature branch. Watch what triggers.

**Expected output:** Runs fire only on main pushes and on PR activity.

**Why it matters:** Unscoped triggers are the #1 way teams burn through their free minutes.

---

## Exercise 3: Build a real CI pipeline

**Goal:** Run lint and tests for a Node (or Python) project.

1. Replace the steps with a checkout + setup + test flow:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'
  - run: npm ci
  - run: npm test
```

2. Push and watch it run.

**Expected output:** Dependencies install from cache, tests run, run goes green.

**Why it matters:** Checkout + setup + test is the shape of 90% of CI jobs. The built-in `cache: 'npm'` option saves minutes per run.

---

## Exercise 4: Wire jobs with needs

**Goal:** Add a deploy job that only runs after build and test pass, and only on main.

1. Add a second job:

```yaml
deploy:
  needs: [build]
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  steps:
    - run: echo "Deploying ${{ github.sha }}"
```

2. Push to main, then push to a branch and open a PR.

**Expected output:** On main: build → deploy. On PR: build only, deploy skipped.

**Why it matters:** `needs` + `if` is how you build safe pipelines — deploys gated on green builds and the right branch.

---

## Exercise 5: Pass data between jobs with artifacts

**Goal:** Build in one job, consume the output in another.

1. In the build job, add:

```yaml
- run: echo "artifact-demo" > dist.txt
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist.txt
```

2. In the deploy job, add:

```yaml
- uses: actions/download-artifact@v4
  with:
    name: build-output
- run: cat dist.txt
```

**Expected output:** The deploy job logs print `artifact-demo`.

**Why it matters:** Jobs don't share filesystems. Artifacts are the bridge — this is how real pipelines hand build output to deploy stages.

---

## Exercise 6: Fan out with a matrix

**Goal:** Test across Node 18 and 20 in parallel.

1. Add a matrix to your build job:

```yaml
strategy:
  fail-fast: false
  matrix:
    node: [18, 20]
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node }}
```

**Expected output:** Two parallel job runs, one per Node version, both green.

**Why it matters:** Matrix builds are how you catch "works on my version" bugs before your users do.

---

## Exercise 7: Use a secret safely

**Goal:** Store a fake token and reference it without leaking it.

1. In repo **Settings → Secrets and variables → Actions**, add a secret named `DEMO_TOKEN` with value `fake-token-123`.
2. Add a step: `- run: echo "token length is ${#TOKEN}"` with `env: TOKEN: ${{ secrets.DEMO_TOKEN }}` — never echo the raw value.
3. Check the logs.

**Expected output:** Logs show the length, never the token. If you echo it raw, GitHub masks it as `***`.

**Why it matters:** Secrets handling is a favorite interview topic — and leaking one in logs is a real-world incident.

---

## Exercise 8: Manual runs and reusable pieces

**Goal:** Trigger a workflow on demand and extract a composite step.

1. Add `workflow_dispatch:` to your `on:` block. Go to the Actions tab and run it manually with the **Run workflow** button.
2. Extract your setup steps into a composite action at `.github/actions/setup/action.yml`:

```yaml
name: setup
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
```

3. Call it with `- uses: ./.github/actions/setup`.

**Expected output:** Manual run succeeds; the composite action works in your job.

**Why it matters:** `workflow_dispatch` is your "run it now" escape hatch. Composite actions are how teams stop copy-pasting setup steps across repos.

---

## Exercise 9: Debug a broken workflow

**Goal:** Practice the debugging loop on a deliberately broken run.

1. Introduce a bug: reference `${{ secrets.DOES_NOT_EXIST }}` or typo a context like `github.brnach`.
2. Push, watch it fail, read the logs.
3. Add the `ACTIONS_STEP_DEBUG` secret set to `true`, re-run, and compare the log verbosity.

**Expected output:** You can point to the exact failing step and explain why.

**Why it matters:** "How do you debug a failing workflow?" is asked in almost every DevOps interview. Now you have a real answer.
