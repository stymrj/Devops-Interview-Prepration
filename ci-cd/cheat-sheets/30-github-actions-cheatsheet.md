# GitHub Actions Cheat Sheet

Dense command reference for Topic 30 — GitHub Actions. Keep this open while writing workflows.

## Workflow skeleton

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo hi
```

## Triggers

| Trigger | Fires when |
|---|---|
| `push` / `pull_request` | Commits land / PR lifecycle |
| `schedule: [{cron: '0 2 * * *'}]` | On a cron (UTC) |
| `workflow_dispatch` | Manual run from Actions tab |
| `workflow_call` | Called as reusable workflow |
| `release: {types: [published]}` | A release is published |
| `paths: ['src/**']` | Only when matching files change |

## Jobs & control flow

```yaml
deploy:
  needs: [build, test]          # wait for these jobs
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  concurrency:                  # cancel superseded runs
    group: deploy-${{ github.ref }}
    cancel-in-progress: true
  environment: production       # protection rules + secrets
  steps: [...]
```

- `needs` → dependency graph (parallel by default without it)
- `if:` conditions: `success()`, `always()`, `failure()`, `cancelled()`
- No `${{ }}` needed inside `if:` — it's already an expression

## Runners

- `runs-on: ubuntu-latest` / `windows-latest` / `macos-latest` — GitHub-hosted
- `runs-on: [self-hosted, linux]` — your own registered runner
- Self-hosted = VPC access, GPUs, persistent caches; you own patching + security

## Contexts & expressions

| Context | Contains |
|---|---|
| `${{ github.sha }}` / `github.ref` / `github.event_name` | Run metadata |
| `${{ secrets.MY_SECRET }}` | Encrypted secrets (masked in logs) |
| `${{ vars.ENV_NAME }}` | Plain config variables |
| `${{ matrix.node }}` | Current matrix value |
| `${{ steps.build.outputs.image }}` | A step's outputs |
| `${{ env.MY_VAR }}` | `env:` values |

Set outputs: `echo "image=abc" >> "$GITHUB_OUTPUT"` · Set env: `echo "X=1" >> "$GITHUB_ENV"`

## Matrix

```yaml
strategy:
  fail-fast: false
  matrix:
    node: [18, 20]
    os: [ubuntu-latest]
    include: [{node: 22, os: ubuntu-latest, experimental: true}]
    exclude: [{node: 18, os: windows-latest}]
```

## Artifacts & caching

```yaml
- uses: actions/upload-artifact@v4
  with: {name: dist, path: dist/}
- uses: actions/download-artifact@v4
  with: {name: dist}
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
```

Artifacts = outputs passed between jobs · Cache = deps to speed up repeats

## Secrets & OIDC

```yaml
permissions:
  id-token: write    # OIDC: short-lived cloud tokens, no static keys
  contents: read
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/github-deploy
```

- Fork PRs get **no secrets** by default (`pull_request` trigger)
- Pin third-party actions to full commit SHAs, not just `@v4`

## Reuse

```yaml
# composite action (.github/actions/setup/action.yml)
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
# reusable workflow
on: {workflow_call: {inputs: {env: {type: string}}}}
jobs:
  call:
    uses: org/repo/.github/workflows/ci.yml@main
    with: {env: prod}
```

## Debugging

- Add secret `ACTIONS_STEP_DEBUG=true` for verbose step logs
- Reproduce locally: `act` (nektos/act)
- Common failures: missing `checkout`, empty/missing secret, wrong context name (`github.brnach`), stale `ubuntu-latest` image
