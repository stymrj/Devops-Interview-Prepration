# CI/CD Concepts & Pipeline Design — Cheat Sheet

Dense reference for pipelines, deploys, and rollbacks.

## Pipeline structure

| Stage | Purpose | Should fail when |
|---|---|---|
| Lint / format | Style, obvious bugs | Fast — first |
| Unit tests | Logic correctness | Fast — first |
| Build artifact | Compile / docker build | Build broken |
| SAST / dep scan | Vulnerabilities | Critical CVE |
| Integration tests | Service interaction | Contract broken |
| Deploy staging | Real-ish environment | Deploy fails |
| E2E / smoke | User flows | User flow broken |
| Deploy prod (gated) | Release | Manual approval |

## GitHub Actions essentials

```yaml
on:
  push:
    branches: [main]
    paths-ignore: ['docs/**', '*.md']   # skip docs-only changes
jobs:
  test:
    needs: lint                          # fail fast: lint runs first
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>      # pin to SHA, not @v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci && npm test
  deploy:
    needs: test
    environment: production               # triggers required-reviewer gate
```

## Jenkins essentials

```groovy
pipeline {
  agent any
  stages {
    stage('Lint')  { steps { sh 'npm run lint' } }
    stage('Test')  { steps { sh 'npm test' } }
    stage('Build') { steps { sh 'docker build -t myapp:${GIT_COMMIT} .' } }
  }
  post { failure { slackSend "build failed" } }
}
```

- `Jenkinsfile` in repo = pipeline-as-code. Shared logic → shared libraries (`vars/`).
- Secrets: `credentials('id')` binding or Jenkins credential store — never in the file.

## Artifact tagging

```bash
docker build -t myapp:$(git rev-parse --short HEAD) .   # traceable
docker tag myapp:$SHA registry.example.com/myapp:$SHA
docker push registry.example.com/myapp:$SHA
# NEVER use :latest for anything you deploy
```

## Deployment strategies at a glance

- **Recreate:** kill all, start new. Downtime. Dev only.
- **Rolling:** replace gradually, no downtime, mixed versions temporarily.
- **Blue-green:** two full envs, flip router, instant rollback.
- **Canary:** % of traffic to new, watch metrics, ramp up. Safest for risky changes.
- **A/B:** like canary, but measures user behavior, not just errors.

## kubectl rollout commands

```bash
kubectl set image deployment/myapp myapp=myapp:$SHA
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp              # roll back one revision
kubectl rollout undo deployment/myapp --to-revision=3
kubectl rollout pause deployment/myapp             # freeze mid-rollout
kubectl rollout resume deployment/myapp
```

## Speed wins

- Cache: deps (`npm`/`pip`/`go mod`), Docker layers, build outputs — key on lockfile.
- Parallelize: test shards (`--shard 1/4`), independent jobs concurrently.
- Skip: `paths-ignore` for docs; only run affected suites.
- Right-size runners; slim base images.

## Security checklist

- Pin actions/deps to SHA (`checkout@b4ffde65...`, lockfiles committed).
- Scan: `trivy image`, `npm audit` / `pip-audit`, SAST (semgrep/CodeQL).
- Sign: `cosign sign registry.example.com/myapp:$SHA`.
- Secrets via OIDC/IAM roles; mask in logs; scope per environment.
- Branch-protect workflow files; require reviews on pipeline changes.

## GitOps (pull-based) one-liner

CI builds + pushes image → updates image tag in git → ArgoCD/Flux agent in-cluster sees drift → syncs. Git is the source of truth; prod creds never leave the cluster.

*Pairs with the [CI/CD concepts & pipeline design guide](28-cicd-concepts-pipeline-design.md).*
