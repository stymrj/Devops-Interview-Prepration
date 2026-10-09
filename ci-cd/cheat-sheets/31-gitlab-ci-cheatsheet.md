# GitLab CI — Cheat Sheet

## Pipeline structure

```yaml
stages: [build, test, deploy]   # execution order; default is [build, test, deploy]
image: node:20                  # default image for all jobs

job-name:
  stage: test                   # which stage it belongs to
  image: alpine:latest          # override per job
  tags: [docker]                # pick runners with this tag
  script:                       # the actual commands
    - npm ci
    - npm test
  before_script: [echo setup]   # runs before every job's script
  after_script: [echo cleanup]  # runs even if job fails
  allow_failure: true           # failure won't block pipeline (orange)
  timeout: 30m                  # job-level timeout (default 60m)
```

## rules (modern conditional control)

```yaml
job:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"            # MRs, pushes, schedules...
      changes: [src/**/*]                       # ...combined with path filters
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_TAG                         # tags
    - exists: [Dockerfile]                       # file exists in repo
    - when: manual                               # play button in UI
    - when: never                                # skip
```

`only:`/`except:` is legacy — don't mix with `rules` in one job.

## DAG with needs

```yaml
unit-tests:
  stage: test
  needs: [build-app]        # start once build-app finishes; skip stage gate
  # needs: {job: build-app, artifacts: false, optional: true}
```

## Artifacts & cache

```yaml
job:
  artifacts:
    paths: [dist/, report.xml]   # passed to later jobs
    expire_in: 1 week            # default 30 days; "never" to keep
    when: always                 # also upload on failure (default on_success)
    reports: {junit: report.xml} # test reports render in MR UI
  cache:
    key: npm-$CI_COMMIT_REF_SLUG # per-branch cache; use $CI_COMMIT_SHA for exact
    paths: [node_modules/]       # cache = speed; artifacts = handoff
    policy: pull-push            # or pull (don't update) / push (only update)
```

## Variables & secrets

| Where | Example |
|---|---|
| `.gitlab-ci.yml` | `variables: {NODE_ENV: test}` |
| Project settings | Settings → CI/CD → Variables (masked, protected) |
| File-type variable | `MY_KEY_FILE` → job sees `$MY_KEY_FILE` as a path |
| Predefined | `$CI_COMMIT_SHA` `$CI_COMMIT_BRANCH` `$CI_COMMIT_TAG` `$CI_PIPELINE_SOURCE` `$CI_ENVIRONMENT_NAME` `$CI_PROJECT_DIR` `$CI_REGISTRY_IMAGE` |

Masked hides from logs; Protected injects only on protected branches/tags.

## Reuse patterns

```yaml
include:
  - local: ci/common.yml
  - project: team/ci-templates
    file: [node.yml]
  - remote: https://example.com/template.yml
  - component: $CI_SERVER_FQDN/team/php/[email protected]

.base-job:                       # hidden job (dot prefix = not run)
  before_script: [echo common]

my-job:
  extends: .base-job             # inherit everything, override as needed

other-job:
  script:
    - !reference [.base-job, before_script]  # reuse one key only
```

## Environments & review apps

```yaml
deploy-staging:
  stage: deploy
  script: [./deploy.sh staging]
  environment:
    name: staging
    url: https://staging.example.com

deploy-review:
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    on_stop: stop-review        # needs a stop job with action: stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

## Multi-project / child pipelines

```yaml
trigger-child:                  # child pipeline, same project
  trigger:
    include: ci/child-pipeline.yml
  rules: [{if: $CI_COMMIT_BRANCH == "main"}]

trigger-infra:                  # downstream pipeline, another project
  trigger:
    project: team/infra
    branch: main
```

## Services (sidecar containers)

```yaml
integration-tests:
  services:
    - name: postgres:15
      alias: db
  variables:
    POSTGRES_PASSWORD: secret   # service reads same variables
  script: [psql -h db -U postgres -c "select 1"]
```

## Debugging quickies

- **See full log:** expand collapsed sections in the failed job view.
- **Lint YAML:** CI/CD → Editor → "Validate" tab, or `CI Lint` API.
- **Run locally:** `gitlab-runner exec docker my-job` (single job, no services).
- **Which runner?** Job page shows runner name + tags; check `tags:` match.
- **Cache miss?** Cache is best-effort — job must work without it.
- **Pipeline stuck?** No matching runner (check tags), or `when: manual` waiting.

## Security checklist

- [ ] Protected branches + protected variables for prod secrets
- [ ] OIDC/`CI_JOB_JWT` for cloud auth instead of long-lived keys
- [ ] MR approvals + "pipeline must succeed" before merge
- [ ] Pin `include:` remote templates to a commit SHA
- [ ] Treat `.gitlab-ci.yml` edits like production code (CODEOWNERS)
