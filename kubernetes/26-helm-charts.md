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
