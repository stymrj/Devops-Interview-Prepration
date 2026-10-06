# Helm Charts Interview Preparation Guide

*How to Answer Helm Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Helm Basics Charts Releases and the Why](#helm-basics-charts-releases-and-the-why)
2. [Chart Anatomy Templates and Values](#chart-anatomy-templates-and-values)
3. [Releases Upgrades Rollbacks and Lifecycle](#releases-upgrades-rollbacks-and-lifecycle)
4. [Dependencies Libraries and Production Charts](#dependencies-libraries-and-production-charts)

---

## Helm Basics Charts Releases and the Why

### Q1: Why does Kubernetes even need Helm? What problem does it solve?

**How to Answer:**

"Raw Kubernetes manifests don't scale as a deployment story. Once you have a deployment, a service, an ingress, a configmap, secrets, and an HPA for one app, you're maintaining hundreds of lines of YAML — and then you need a second copy for staging and a third for prod, differing only in image tag and replica count."

"Helm is the package manager for Kubernetes. A chart packages all those resources together and templatizes the parts that change per environment. I install the whole app with one command, and a values file decides the environment-specific bits."

"So the interview answer is simple: Helm turns copy-pasted YAML into a reusable, versioned, parameterized package."

**Key Point:** "Helm turns a folder of copy-pasted YAML into a versioned package you install with one command."

---

### Q2: What exactly is a chart, a release, and a repository?

**How to Answer:**

"A chart is the package itself — a directory with templates and metadata. Think of it as the installer, like an npm package or a Docker image. A release is a running instance of that chart in a cluster: same chart, but release A might be myapp in staging with tag 1.2.3 and release B is myapp in prod with tag 1.2.4."

"A repository is just a place charts are stored and shared — a Helm repo is basically a web server hosting chart archives plus an index.yaml that lists versions. `helm repo add` registers it, `helm search` finds charts, `helm install` pulls them down."

"The vocabulary trap is mixing chart and release up. Chart is the template, release is the deployed instance — and each release gets tracked separately, so upgrading staging never touches prod."

**Key Point:** "Chart is the package, release is the running instance of it, repository is where charts live."

---

### Q3: Walk me through a chart's directory structure — what matters?

**How to Answer:**

"`helm create mychart` gives you a scaffold. The pieces I actually care about are: `Chart.yaml` with the chart name and version, `values.yaml` with the defaults, and the `templates/` directory with the actual Kubernetes manifests that get rendered. There's also a `charts/` folder for bundled dependencies and an optional `templates/NOTES.txt` that prints usage hints after install."

"The stuff beginners ignore is usually the interesting part. `templates/_helpers.tpl` holds shared template functions so you're not repeating the same fullname logic everywhere. And the tests directory, if it exists, is where people are supposed to put smoke tests — though in practice most charts don't."

"When I read someone else's chart, I start at values.yaml to see what's configurable, then look at the templates to see how those values are used."

**Key Point:** "Chart.yaml, values.yaml, and templates/ are the core — everything else is scaffolding."

---

### Q4: How do templates and values.yaml actually work together?

**How to Answer:**

"Templates are Kubernetes YAML with Go templating sprinkled in. Values.yaml is the default configuration. When I run `helm install`, Helm merges my values — defaults from values.yaml, overridden by `-f` files and `--set` flags — and renders each template into a complete manifest."

"So a template might say `replicas: {{ .Values.replicaCount }}` and values.yaml has `replicaCount: 2`. If I deploy with `-f prod.yaml` where replicaCount is 5, the rendered Deployment gets 5. The chart author decides what's parameterizable; values.yaml is the contract with the operator."

"One trap: Helm renders ALL templates in the directory, then figures out what to install. You can't conditionally skip rendering a broken template — you use `{{- if .Values.something }}` inside it to control whether the resource is emitted."

**Key Point:** "Values are the inputs, templates are the code — Helm merges and renders them into plain Kubernetes YAML."

---

### Q5: What are the template gotchas — quote, nindent, toYaml?

**How to Answer:**

"The three functions I use daily are `quote`, `default`, and `nindent`. `quote` wraps a value in quotes so YAML doesn't misinterpret it — a password like `true123` would parse as a boolean without it. `default` gives a fallback: `{{ .Values.image.tag | default "latest" }}`."

"`nindent` is the one everyone gets bitten by. When you inject a multi-line block — like labels from a helper — into indented YAML, `nindent 4` adds a newline plus four spaces so the block lands at the right depth. Get the indentation wrong and the manifest is invalid YAML, but Helm will happily try to apply it and the API server rejects it."

"`helm template` is my debugging weapon. It renders locally without touching a cluster, so I run it and pipe through `kubectl apply --dry-run=client` when a chart isn't rendering right."

**Key Point:** "quote prevents YAML type surprises, nindent fixes multi-line indentation, and helm template lets you debug rendering locally."

---

### Q6: How does the values precedence actually work — which file wins?

**How to Answer:**

"The order is: chart's values.yaml is the base, then each `-f` file overrides the previous one left to right, and `--set` flags beat everything. So `helm install -f base.yaml -f prod.yaml --set image.tag=v2` means base values first, prod overrides them, and the --set wins on the tag."

"The interview trap is forgetting that `-f` order matters. Later files win. I keep a base values.yaml in the chart, an environment values file per stage, and use --set only for truly one-off things like an image tag from CI."

"I'd rather have `-f prod-values.yaml` checked into git than a long string of --set flags in a pipeline — it's reviewable and auditable."

**Key Point:** "values.yaml is the base, -f files override left to right, and --set always wins."

---

## Releases Upgrades Rollbacks and Lifecycle

### Q7: What actually happens when I run helm install or helm upgrade?

**How to Answer:**

"On install, Helm renders the templates with my values, sends the manifests to the API server, and stores the whole release — the rendered manifest and metadata — as a Secret in the namespace. On upgrade, it renders again with the new values, computes a three-way merge between the old chart, the new chart, and the live state, then applies only what changed."

"The three-way merge is the clever part. If I hand-edited a replica count with kubectl after install, an upgrade won't clobber it unless the new chart changed that field too — Helm respects live-state drift for fields the chart doesn't manage. Helm 2's Tiller is long gone; Helm 3 talks to the API server directly using my kubeconfig, so RBAC applies to the user running the command."

"Releases are versioned as revisions, so `helm history` shows every deploy and I can roll back to any of them."

**Key Point:** "Helm 3 has no server component — it renders locally, applies via my kubeconfig, and stores releases as Secrets."

---

### Q8: How do you roll back a bad release, and what are the traps?

**How to Answer:**

"`helm rollback myapp 2` reverts to revision 2. It's instant in most cases because Helm just re-applies the stored manifest from that revision. But rollback doesn't undo everything — it won't delete resources that were created outside the chart, and it doesn't handle data migrations backwards."

"The traps are the stuck states. If an upgrade fails halfway, the release can sit in PENDING_UPGRADE and every further command is blocked. The fix is `helm rollback` to a good revision or `helm uninstall` and start over. That's why I use `--atomic` on deploys that matter: it auto-rolls-back on failure and cleans up the pending state."

"I also always use `--wait --timeout` so Helm blocks until resources are actually ready, not just accepted by the API server. Deploying and walking away without --wait is how you find out at 2am that the rollout was broken."

```bash
helm upgrade myapp ./mychart -f prod-values.yaml \
  --wait --timeout 5m --atomic
```

**Key Point:** "helm rollback reverts to a stored revision — and --atomic is what saves you from PENDING_UPGRADE hell."

---

### Q9: How do you handle secrets in Helm charts?

**How to Answer:**

"Never plain text in values.yaml checked into git — that's the trap answer. The chart itself should reference secrets that already exist, usually via `envFrom` on an existing Secret or an external secret store, rather than embedding the secret values."

"In practice I use one of three patterns. The helm-secrets plugin with sops encrypts secret values in the values file — decrypted at deploy time with a KMS or age key. Or I skip Helm entirely for secrets and use External Secrets Operator, where the chart creates an ExternalSecret CR and the operator syncs real values from AWS Secrets Manager."

"The worst thing I see is people `--set`ting passwords on the command line — it lands in shell history and CI logs. Environment files encrypted with sops, or ESO pulling from a real secret store, keeps secrets out of git and out of logs."

**Key Point:** "Charts reference secrets, they don't contain them — encrypt with sops or pull them with External Secrets Operator."

---

### Q10: How do you manage dev, staging, and prod config with one chart?

**How to Answer:**

"One chart, one values file per environment, all in git. `values.yaml` holds sane defaults, then `values-dev.yaml`, `values-staging.yaml`, and `values-prod.yaml` override what differs — replica counts, resource limits, ingress hosts, feature flags. The deploy pipeline picks the file: `helm upgrade -f values-prod.yaml`."

"For shared patterns across many charts, I use umbrella charts or library charts so every service gets the same deployment template. And the environment values files themselves live next to the chart in git, so config changes go through pull requests like code."

"What I don't do is maintain three separate charts, or template on hostname inside the chart with if-chains. One chart parameterized by values files — that's the GitOps-friendly shape."

**Key Point:** "One chart, one values file per environment in git — the pipeline just picks which -f file to use."

---

### Q11: How do you debug a release that's not working?

**How to Answer:**

"First I figure out which layer is broken. `helm status myapp` tells me if the release deployed cleanly. `helm history` shows whether we're on the revision I think we're on. If a pod isn't coming up, it's a Kubernetes problem, not a Helm problem — I drop to kubectl and debug like any workload."

"For chart problems specifically: `helm template` renders without deploying, which catches 80% of templating bugs. `helm lint` catches Chart.yaml and values mistakes. `kubectl get events` in the namespace shows the scheduler and kubelet side of the story."

"The classic one is a release that deployed but the app can't connect — usually a Service selector or Ingress host that got templated wrong. Rendering the chart and eyeballing the selector labels versus the pod labels finds it fast."

**Key Point:** "helm status and history first, then debug it like any Kubernetes workload — the chart is just the delivery mechanism."

---

### Q12: What are Helm hooks, and when do you actually need them?

**How to Answer:**

"Hooks are resources in the chart that run once at a specific lifecycle point instead of being managed as part of the release — annotated with `helm.sh/hook`. The common ones are `pre-install` for things like running database migrations before the new pods start, and `post-upgrade` for smoke tests or cache clears."

"The key thing interviewers want is knowing that hooks are NOT managed like normal resources. Helm doesn't track or upgrade them — they fire and Helm moves on. So a pre-upgrade migration job runs once per upgrade, and if it fails, the upgrade stops."

"I use hooks sparingly: migrations before rollout, cleanup jobs on delete. Anything that needs to be idempotent and tracked belongs in a normal template, not a hook."

```yaml
annotations:
  "helm.sh/hook": pre-install,pre-upgrade
  "helm.sh/hook-weight": "-5"
  "helm.sh/hook-delete-policy": hook-succeeded
```

**Key Point:** "Hooks fire once at a lifecycle point and aren't managed like normal resources — perfect for migrations, bad for everything else."

---

## Dependencies Libraries and Production Charts

### Q13: How do chart dependencies and library charts work?

**How to Answer:**

"Dependencies let a chart pull in other charts — my app chart can depend on the bitnami postgres chart so one `helm install` deploys app plus database. They're declared in Chart.yaml under `dependencies`, and `helm dependency update` fetches them into the `charts/` directory as packaged archives."

"Library charts are the other half: charts with `type: library` that contain only templates and helpers, no resources. Teams build a library chart with their standard deployment, service, and ingress templates, and every service chart depends on it — so all 40 microservices render from one canonical template instead of 40 copy-pasted ones."

"The interview trap is dependency versioning: pin versions in Chart.yaml, because an unpinned dependency that publishes a breaking major will silently wreck your deploys. And vendored charts/ archives get committed, so the deploy doesn't depend on a repo being reachable at apply time."

**Key Point:** "Dependencies bundle other charts, library charts share template logic — both pinned and vendored so deploys are reproducible."

---

### Q14: How do you version and publish a chart? What's version vs appVersion?

**How to Answer:**

"In Chart.yaml there are two versions and they mean different things. `version` is the chart's own version — bump it every time the chart changes, because Helm treats chart version as the identity of the package. `appVersion` is the version of the application the chart deploys — like the default image tag."

"The rule is SemVer for the chart version, and any chart change — even a template-only fix with no app change — needs a version bump or package managers can't tell releases apart. Publishing is just packaging with `helm package` and pushing to an OCI registry with `helm push oci://...` or a traditional repo index.yaml."

"CI should enforce the bump: a pipeline check that fails the PR if templates changed but the chart version didn't."

**Key Point:** "version tracks the chart, appVersion tracks the app — bump chart version on every chart change."
