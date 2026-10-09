# ArgoCD & GitOps Principles Interview Preparation Guide

*How to Answer ArgoCD & GitOps Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [GitOps Principles](#gitops-principles)
2. [ArgoCD Core Concepts](#argocd-core-concepts)
3. [Applications and Sync](#applications-and-sync)
4. [Advanced ArgoCD Patterns](#advanced-argocd-patterns)
5. [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)

---

## GitOps Principles

### Q1: What is GitOps, and how does it differ from classic CI/CD?

**How to Answer:**

"GitOps means Git is the single source of truth for your infrastructure and app deployments. You describe the desired state in YAML, and an agent — like ArgoCD — makes the cluster match it. In classic CI/CD, your pipeline pushes changes into the cluster with `kubectl apply` or helm from outside; in GitOps, the agent inside the cluster pulls the state in. I tell people: CI builds and publishes images, GitOps deploys what's in Git. The killer benefit is everything is audited — every change to prod is a Git commit, so rollbacks are just `git revert`."

**Key Point:** "GitOps = Git as the source of truth, cluster state reconciled to it automatically; CI pushes images, GitOps pulls manifests."

---

### Q2: What are the core principles of GitOps?

**How to Answer:**

"There are four, from the OpenGitOps working group. First: declarative — everything is YAML that describes the end state. Second: versioned and immutable — the desired state lives in Git, so it's reviewed, audited, timestamped. Third: pulled automatically — software agents sync the cluster to match, nobody hand-applies things. Fourth: continuously reconciled — the agent watches for drift and corrects it, so the system heals itself. The interview trap is people recite the words but can't explain reconciliation — that's the loop where ArgoCD compares Git's desired state to the live cluster state every few minutes and fixes differences."

**Key Point:** "Declarative, versioned in Git, pulled automatically, continuously reconciled — and reconciliation is the drift-correction loop."

---

### Q3: What does a typical GitOps repo layout look like?

**How to Answer:**

"Two schools, but I like app-of-apps or per-environment folders. You usually have a repo with `environments/dev`, `environments/staging`, `environments/prod`, each pointing at pinned versions. Inside, either Kustomize overlays or Helm values per environment. The golden rule: the Git repo must contain only the desired state — never the live state, never secrets in plaintext. I keep image tags out of the shared manifests too — one file or Kustomize patch holds the tag per environment, updated by the CI pipeline or Image Updater."

**Key Point:** "Per-environment folders or overlays holding only desired state — no live state, no plaintext secrets, image tags isolated for easy promotion."

---

## ArgoCD Core Concepts

### Q4: What is ArgoCD, and where does it sit in the stack?

**How to Answer:**

"ArgoCD is the GitOps continuous-delivery agent for Kubernetes — it runs inside your cluster and keeps your apps synced to what's in Git. Your CI pipeline builds the image and updates the Git repo; ArgoCD notices the new commit and rolls it into the cluster. It natively understands Helm, Kustomize, and plain YAML, plus Jsonnet. I always clarify this in interviews: ArgoCD is CD only — it doesn't build images or run tests, that's still CI's job."

**Key Point:** "ArgoCD is the in-cluster GitOps agent that syncs Kubernetes apps to Git; CI builds images, ArgoCD deploys them."

---

### Q5: Walk me through ArgoCD's architecture — the main components.

**How to Answer:**

"Three main pieces. The API server is the brain and the frontend — it serves the UI, the CLI, and enforces RBAC. The repo server clones your Git repos and renders the templates — it turns Helm charts and Kustomize into plain manifests, and it caches aggressively so it's not re-cloning constantly. The application controller is the muscle — it compares the desired state from Git against the live cluster state and performs the sync. The controller is the part that scales; in big setups you shard it across clusters so one controller isn't watching everything."

**Key Point:** "API server (UI/CLI/RBAC), repo server (renders templates from Git), application controller (compares and syncs state — the piece you scale)."

---

### Q6: What's the difference between an Application and an ApplicationSet?

**How to Answer:**

"An Application is ArgoCD's core resource — it says 'this Git repo path should be live in this cluster namespace.' It's one app, one destination, defined in YAML. An ApplicationSet is a generator that creates many Applications from a template — say one app per cluster, per environment, per tenant. I use it when we add a new region: instead of writing ten Application manifests by hand, the ApplicationSet's cluster generator spawns them automatically. The trap people hit is treating ApplicationSet as just templating — its real power is the generators reacting to external lists like cluster inventories or Git directory trees."

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo
    path: apps/payments/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
```

**Key Point:** "Application = one app's Git-to-cluster mapping; ApplicationSet = a template + generators that create many Applications automatically."

---

### Q7: Explain ArgoCD's health model — how does it know an app is 'healthy'?

**How to Answer:**

"ArgoCD doesn't just check if pods exist — it evaluates each resource with health checks, and Deployments get special treatment. A Deployment is Healthy only when its rollout completes: updated replicas match, no old pods hanging around. It'll show Progressing during a rollout, Degraded if pods crashloop. This matters because sync status and health are different axes — an app can be Synced but Degraded, meaning Git and the cluster match but the thing is broken. In interviews I always name both: sync = Git vs live match, health = is it actually running right."

**Key Point:** "Sync status = Git vs live match; health = actual runtime state — an app can be Synced yet Degraded."

---
