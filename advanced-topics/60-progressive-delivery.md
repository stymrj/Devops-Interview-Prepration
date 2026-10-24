# Progressive Delivery Interview Preparation Guide

*How to Answer Progressive Delivery Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Progressive Delivery](#why-progressive-delivery)
2. [Canary and Blue-Green in Practice](#canary-and-blue-green-in-practice)
3. [Argo Rollouts](#argo-rollouts)
4. [Flagger](#flagger)
5. [Analysis, Rollback and Debugging](#analysis-rollback-and-debugging)

---

## Why Progressive Delivery

### Q1: What is progressive delivery, in one simple answer?

**How to Answer:**

"I think of it as shipping to a few users first and only expanding when the evidence says it's safe. CI/CD gets the new version built and deployed, but progressive delivery controls who actually sees it and how fast it spreads. If the canary misbehaves, a small slice of users is affected and the rollout stops itself. It's deployment with a safety net built in."

**Key Point:** "CI/CD ships the version. Progressive delivery controls exposure — a few users first, everyone only when the metrics say so."

---

### Q2: Canary vs blue-green vs rolling — how do you pick?

**How to Answer:**

"I pick rolling when I want simple and cheap — pods swap in gradually, but I can't compare old and new side by side. I pick blue-green when I need instant rollback by flipping traffic back — but it costs double capacity during the switch. I pick canary when I want real evidence: a small slice of real traffic on the new version, watched against the old. For anything user-facing and risky, canary is my default answer."

**Key Point:** "Rolling = simple. Blue-green = instant rollback, double capacity. Canary = real traffic evidence, my default for risky releases."

---

### Q3: Walk me through a canary step by step.

**How to Answer:**

"I deploy the new version next to the old one and send it a small share of traffic — say 10 percent. I watch error rate and latency for the canary specifically, not the blended average. If it's clean, I step to 25, then 50, then 100 percent, pausing between steps. If a step fails its checks, the rollout aborts and traffic snaps back to the old version automatically. Nobody pages me at 2am to flip traffic by hand."

**Key Point:** "Small slice, measured steps, automatic abort — a canary is a controlled experiment, not a slow deploy."

---

## Canary and Blue-Green in Practice

### Q4: What metrics actually decide promote vs abort?

**How to Answer:**

"I watch the golden signals for the canary only: error rate, p99 latency, and request success rate compared against the stable version. I also watch business signals when they exist — checkout success, not just HTTP 200s. A canary can pass all HTTP checks and still be broken if, say, the payment call silently fails. Thresholds are set before the rollout starts, not invented after it looks bad."

**Key Point:** "Compare the canary against the stable version on errors, p99 and business success — thresholds agreed before the rollout, not after."

---

### Q5: What's the difference between a deployment and a progressive rollout?

**How to Answer:**

"A Deployment just replaces pods — it knows nothing about traffic or metrics. A progressive rollout is a controller that owns the traffic split and the decision to continue. It can pause at a weight, run an analysis, and abort on bad metrics. In interviews I say the Deployment moves containers, the rollout controller moves risk."

**Key Point:** "Deployment = replace pods. Rollout controller = traffic split + analysis + automatic abort."

---

## Argo Rollouts

### Q6: What is Argo Rollouts and why do teams pick it?

**How to Answer:**

"I use Argo Rollouts when I want canary and blue-green inside Kubernetes without leaving the GitOps flow. It's a custom resource — a Rollout instead of a Deployment — that works with the same ingress or service mesh I already run. Because it plugs into ArgoCD, the rollout state shows up in the same dashboard as everything else. It's the natural choice if the team is already on Argo."

```bash
kubectl argo rollouts get rollout web -n prod   # live step, weight, analysis state
```

**Key Point:** "Argo Rollouts = canary/blue-green as a Kubernetes resource, native to the ArgoCD workflow."

---

### Q7: Explain a Rollout canary spec like you'd explain it to a teammate.

**How to Answer:**

"I define steps in the spec: set weight 10, pause, run an analysis, then step to 50 and 100. The canary service carries the new version, the stable service carries the old one, and the ingress or mesh splits traffic between them by weight. When the rollout finishes, the stable service points at the new version and the old pods are gone. The spec is the whole runbook — I never click through steps by hand."

```yaml
spec:
  strategy:
    canary:
      canaryService: web-canary
      stableService: web-stable
      steps:
      - setWeight: 10
      - pause: { duration: 5m }
      - analysis:
          templates: [{ templateName: success-rate }]
      - setWeight: 50
      - setWeight: 100
```

**Key Point:** "Canary service + stable service + stepped weights in the spec = the runbook, written once, run automatically."

---

### Q8: How do AnalysisTemplate and AnalysisRun work?

**How to Answer:**

"An AnalysisTemplate says what to measure and what counts as passing — for example, query Prometheus for the canary's error rate and require it under 1 percent. When the rollout reaches an analysis step, it creates an AnalysisRun that actually executes those queries over a time window. If a metric fails, the run fails and the rollout aborts. I keep templates in Git next to the Rollout so the quality bar is reviewed like code."

**Key Point:** "Template = the pass/fail bar in Git. Run = the live measurement. A failed run aborts the rollout."

---

### Q9: How does automatic rollback actually happen?

**How to Answer:**

"When an analysis fails or a step times out, the controller scales the canary down and routes all traffic back to the stable version — no human step. The old version was never deleted during the rollout, so rollback is just traffic, not a redeploy. I verify it after the fact with the rollout status and the stable service's endpoints. The thing I always say in interviews: rollback must be boring and instant, not a second deployment."

```bash
kubectl argo rollouts abort web -n prod        # manual abort, same safe path
kubectl argo rollouts get rollout web -n prod  # confirm stable is serving 100%
```

**Key Point:** "Rollback is traffic, not a redeploy — the old version stays alive until the canary fully wins."

---

## Flagger

### Q10: What is Flagger, and how is it different from Argo Rollouts?

**How to Answer:**

"I reach for Flagger when the team runs a service mesh or ingress like Istio, Linkerd or NGINX and wants the rollout driven by that layer. Flagger watches a Deployment and progressively shifts traffic at the mesh, running its own canary analysis with Prometheus metrics. Argo Rollouts replaces the Deployment with a Rollout resource; Flagger leaves the Deployment alone and drives traffic externally. Same goal, different control point — I pick based on what the cluster already runs."

**Key Point:** "Flagger drives traffic at the mesh/ingress and leaves Deployments alone. Argo Rollouts owns the workload itself."

---

### Q11: How does a Flagger canary shift traffic in practice?

**How to Answer:**

"I write a Canary resource that names the Deployment, the provider — say Istio — and the analysis: interval, threshold, and max weight. Flagger creates a canary Deployment, shifts weight in steps like 10, 25, 50, and runs metric checks at each step. It can also run webhooks — a load test before promoting, or a Slack notification on failure. If checks fail past the threshold, it rolls traffic back and scales the canary to zero."

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata: { name: web, namespace: prod }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  provider: istio
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange: { min: 99 }
```

**Key Point:** "Flagger Canary = target Deployment + provider + stepped weights + Prometheus checks, with webhooks for load tests and alerts."

---

## Analysis, Rollback and Debugging

### Q12: A canary fails but the dashboard looks green. How do you debug it?

**How to Answer:**

"I check what the analysis actually measured first — a green dashboard often means the metric query is wrong, not the release is fine. I compare canary-only metrics against stable-only, not the blended traffic. I look for the failure users would feel: a broken checkout, a slow dependency, a 200 that returns an error page. And I check whether the canary even got traffic — a weight misconfiguration can leave it at zero requests, which passes every check by default."

**Key Point:** "Green dashboard + failing canary usually means wrong query, blended metrics, or a canary that got no traffic at all."

---
