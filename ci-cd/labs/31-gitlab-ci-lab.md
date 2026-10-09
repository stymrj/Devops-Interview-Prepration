# GitLab CI — Hands-On Lab

Work through these exercises in order. Each one has a **goal**, the **commands** to run, the **expected output**, and **why it matters** for interviews.

> You need a GitLab account and a test project. Use a shared runner or your own `gitlab-runner` with the Docker executor. Never use production credentials.

---

## Exercise 1: Your first pipeline

**Goal:** Trigger a pipeline with a single job and read the pipeline graph.

**Commands:**
```bash
# In your test project root:
cat > .gitlab-ci.yml <<'EOF'
hello-job:
  script:
    - echo "Hello from $CI_RUNNER_DESCRIPTION"
EOF
git add .gitlab-ci.yml && git commit -m "first pipeline" && git push
```

**Expected output:** A green pipeline in CI/CD → Pipelines with one job; the job log shows `Hello from ...`.

**Why it matters:** Proves you know the minimal unit — every pipeline is just jobs with scripts, and it gives you the CI/CD UI muscle memory.

---

## Exercise 2: Stages in order

**Goal:** See stages execute sequentially and jobs run in parallel inside a stage.

**Commands:**
```bash
cat > .gitlab-ci.yml <<'EOF'
stages: [build, test]

build-a:
  stage: build
  script: [echo "building A", sleep 5]

build-b:
  stage: build
  script: [echo "building B", sleep 5]

smoke-test:
  stage: test
  script: [echo "tests run after BOTH builds finish"]
EOF
git add -A && git commit -m "stages demo" && git push
```

**Expected output:** `build-a` and `build-b` run at the same time; `smoke-test` starts only after both finish.

**Why it matters:** This is the stage-vs-job mental model every interview question builds on.

---

## Exercise 3: Pass files with artifacts

**Goal:** Build in one job, consume the output in a later job.

**Commands:**
```bash
cat > .gitlab-ci.yml <<'EOF'
stages: [build, test]

make-file:
  stage: build
  script: [echo "build-output" > result.txt]
  artifacts:
    paths: [result.txt]

check-file:
  stage: test
  script: [cat result.txt]
EOF
git add -A && git commit -m "artifacts demo" && git push
```

**Expected output:** `check-file` prints `build-output` — the file traveled between jobs via the artifact.

**Why it matters:** Artifacts are how real pipelines move binaries, reports, and Docker contexts between stages.

---

## Exercise 4: Cache dependencies

**Goal:** Cache `node_modules` and observe a cache hit on the second run.

**Commands:**
```bash
cat > .gitlab-ci.yml <<'EOF'
image: node:20

install-deps:
  script:
    - npm ci --prefer-offline
  cache:
    key: npm-$CI_COMMIT_REF_SLUG
    paths: [node_modules/]
EOF
echo '{"name":"demo","dependencies":{"lodash":"4.17.21"}}' > package.json
git add -A && git commit -m "cache demo" && git push
# push an empty commit to trigger a second pipeline
git commit --allow-empty -m "second run" && git push
```

**Expected output:** Second pipeline's job log shows cache extraction (`node_modules/` restored) and a faster `npm ci`.

**Why it matters:** Caching is the #1 pipeline speed win; interviewers expect you to have seen a cache hit in a log.

---

## Exercise 5: rules with changes

**Goal:** Skip the test job when only docs change.

**Commands:**
```bash
cat > .gitlab-ci.yml <<'EOF'
unit-tests:
  image: node:20
  script: [npm test]
  rules:
    - changes: ["docs/**/*"]
      when: never
    - when: on_success
EOF
git add -A && git commit -m "rules demo" && git push
# now change only docs and push
echo "# docs" > docs/readme.md && git add -A && git commit -m "docs only" && git push
```

**Expected output:** The `docs only` pipeline shows the test job skipped (gray), not failed.

**Why it matters:** `rules: changes:` is the standard answer to "how do you avoid wasting CI minutes."

---

## Exercise 6: Masked and protected variables

**Goal:** Add a secret variable and confirm it is masked in logs.

**Commands:** In GitLab UI: Settings → CI/CD → Variables → add `DEMO_TOKEN` = `supersecret123`, check **Masked** (and **Protected** if your branch is protected).
```bash
cat > .gitlab-ci.yml <<'EOF'
show-secret:
  script:
    - echo "token length is ${#DEMO_TOKEN}"
    - echo "token is $DEMO_TOKEN"
EOF
git add -A && git commit -m "variables demo" && git push
```

**Expected output:** The log shows the length but prints `[MASKED]` where the raw token would appear.

**Why it matters:** You'll be asked exactly how to handle secrets in CI — this is the hands-on proof.

---

## Exercise 7: needs for parallel speed

**Goal:** Break strict stage ordering with a DAG.

**Commands:**
```bash
cat > .gitlab-ci.yml <<'EOF'
stages: [build, test, slow-test]

build-app:
  stage: build
  script: [echo building, sleep 10]
  artifacts: {paths: [result.txt]}

fast-unit:
  stage: test
  needs: [build-app]
  script: [echo "unit tests start early"]

slow-e2e:
  stage: slow-test
  needs: [build-app]
  script: [echo "e2e starts without waiting for unit tests"]
EOF
git add -A && git commit -m "needs demo" && git push
```

**Expected output:** `fast-unit` and `slow-e2e` both start as soon as `build-app` finishes — no waiting on each other's stages.

**Why it matters:** `needs:` is the go-to answer for "how do you speed up a slow pipeline."

---

## Exercise 8: Review app on a merge request

**Goal:** Spin up a dynamic environment per MR.

**Commands:**
```bash
cat > .gitlab-ci.yml <<'EOF'
stages: [deploy]

deploy-review:
  stage: deploy
  script: [echo "deploying to review/$CI_COMMIT_REF_SLUG"]
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.example.com
    on_stop: stop-review
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop-review:
  stage: deploy
  script: [echo "tearing down review/$CI_COMMIT_REF_SLUG"]
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  when: manual
EOF
git checkout -b review-demo && git add -A && git commit -m "review app" && git push -u origin review-demo
# open an MR in the GitLab UI
```

**Expected output:** An environment `review/review-demo` appears under Deployments → Environments, linked from the MR.

**Why it matters:** Review apps are a favorite "how do you do preview environments" answer, and the stop job is the detail interviewers check.

---

## Exercise 9: Pipeline failure drill

**Goal:** Practice the debugging workflow on a broken job.

**Commands:**
```bash
cat > .gitlab-ci.yml <<'EOF'
broken-job:
  image: node:18
  script:
    - node -e "console.log(process.version)"
    - npm ci   # fails: no package.json committed
EOF
git add -A && git commit -m "broken demo" && git push
# Then: open the failed job, expand collapsed sections, find the real error line,
# fix it, push again, and confirm green.
```

**Expected output:** A red pipeline; you identify the root cause from the expanded log sections, fix, and get green.

**Why it matters:** "Walk me through debugging a failing pipeline" is a guaranteed interview question — this is the rehearsal.

---

## Cleanup

Delete the test project or remove `.gitlab-ci.yml` so no stray pipelines keep running. If you registered a personal runner for the lab, unregister it.
