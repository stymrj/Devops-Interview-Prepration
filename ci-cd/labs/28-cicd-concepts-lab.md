# CI/CD Concepts & Pipeline Design — Hands-On Lab

Build, break, and fix a real CI/CD pipeline. You'll write a GitHub Actions workflow, add caching, gate a staging deploy, and practice a rollback.

**Prereqs:** GitHub account, a scratch repo, Docker installed locally (for exercises 6–7).

---

## Exercise 1: Build a minimal CI pipeline

**Goal:** Get a pipeline running on every push.

Create `.github/workflows/ci.yml` that checks out the code, sets up Node (or Python), installs dependencies, and runs your test command:

```yaml
name: CI
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci && npm test
```

**Expected output:** Push a commit → the Actions tab shows a green run within a minute or two.

**Why it matters:** This is the smallest useful pipeline. Everything else — gates, deploys, scans — bolts onto this skeleton.

---

## Exercise 2: Make it fail fast with a lint stage

**Goal:** Catch trivial failures before expensive tests run.

Add a `lint` job that runs first, and make the `test` job depend on it with `needs: lint`:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run lint
  test:
    needs: lint
    runs-on: ubuntu-latest
    steps: [ ... ]
```

**Expected output:** A lint error fails the pipeline in seconds; the test job never starts (shows "skipped").

**Why it matters:** Fail-fast ordering is the cheapest pipeline optimization there is — quick checks first, slow checks later.

---

## Exercise 3: Add dependency caching

**Goal:** Cut install time with cache.

Add a cache step keyed on the lockfile, or use the built-in cache in `setup-node`:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: npm
```

**Expected output:** Second run shows "Cache restored" and `npm ci` finishes noticeably faster.

**Why it matters:** Dependency installs are pure repeated work. Caching them is the highest-ROI speed win, but always key on the lockfile so a changed dependency invalidates it.

---

## Exercise 4: Build and tag a Docker image with the commit SHA

**Goal:** Produce a traceable, immutable artifact.

Add a job that builds and pushes the image tagged with the commit SHA:

```yaml
- run: |
    docker build -t myapp:${{ github.sha }} .
    docker tag myapp:${{ github.sha }} registry.example.com/myapp:${{ github.sha }}
    docker push registry.example.com/myapp:${{ github.sha }}
```

**Expected output:** The registry shows an image tagged with a 40-char SHA matching the commit that built it.

**Why it matters:** SHA tags make every artifact traceable to source. Building once here means staging and prod deploy the exact same bytes you tested.

---

## Exercise 5: Gate promotion with an environment approval

**Goal:** Require a human click before production.

In GitHub: Settings → Environments → create `production`, add a required reviewer. Then reference it:

```yaml
deploy-prod:
  needs: [build, test]
  environment: production
  runs-on: ubuntu-latest
  steps:
    - run: echo "deploying ${{ github.sha }} to production"
```

**Expected output:** The run pauses at `deploy-prod` showing "Waiting for review" until you approve it.

**Why it matters:** This is continuous delivery in action — everything automated up to the prod gate, a human owning the final decision.

---

## Exercise 6: Practice a zero-downtime rolling update

**Goal:** Deploy without dropping traffic.

With a local kind/k3s cluster or minikube, update an image and watch the rollout:

```bash
kubectl set image deployment/myapp myapp=myapp:<new-sha>
kubectl rollout status deployment/myapp
```

**Expected output:** Pods cycle one by one; `rollout status` ends with "successfully rolled out". A `curl` loop against the service shows no failed requests.

**Why it matters:** Rolling updates are the everyday prod strategy — old and new versions overlap so traffic never drops.

---

## Exercise 7: Roll it back

**Goal:** Recover from a bad deploy in seconds.

Deploy a deliberately broken image, then undo:

```bash
kubectl set image deployment/myapp myapp=myapp:broken-sha
kubectl rollout undo deployment/myapp
kubectl rollout history deployment/myapp
```

**Expected output:** The broken version serves errors briefly, then `rollout undo` restores the previous ReplicaSet. History shows both revisions.

**Why it matters:** Rollback is a first-class operation — redeploy the previous artifact, never rebuild. Practice it before an incident forces you to.

---

## Exercise 8: Pin and scan your pipeline's dependencies

**Goal:** Close the easy supply-chain gaps.

Pin every third-party action to a full SHA (`actions/checkout@b4ffde65...` instead of `@v4`), then run an image scan:

```bash
# Scan the image you built in exercise 4
trivy image myapp:${SHA}
```

**Expected output:** Trivy lists CVEs by severity. Pinning shows as exact SHAs in the workflow file.

**Why it matters:** Floating tags on pipeline actions are a supply-chain risk, and un-scanned images are how known CVEs reach production.

---

## Exercise 9: Skip the pipeline for docs-only changes

**Goal:** Stop wasting compute on changes that need no tests.

Add path filters so the workflow only runs for relevant changes:

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
      - '*.md'
```

**Expected output:** A push touching only `README.md` shows no workflow run (or a skipped one).

**Why it matters:** Pipelines should do work proportional to the change. Docs edits triggering full integration suites is pure waste.

---

## Exercise 10: Review your pipeline like a production system

**Goal:** Audit what you built.

Answer honestly about your scratch pipeline: (1) Could someone push a malicious workflow change without review? (2) Are secrets scoped to the jobs that need them? (3) Can you trace any deployed image back to its commit? (4) What fails if the registry is down?

**Expected output:** A short written list of gaps — e.g. "no branch protection on main", "secrets are repo-wide", "no retention policy on images".

**Why it matters:** This audit mindset is exactly what interviewers probe with "how would you secure your pipeline?" — and it's how real pipeline reviews work.

---

*Pairs with the [CI/CD concepts & pipeline design guide](28-cicd-concepts-pipeline-design.md).*
