# Deployments and Rollout Strategies Interview Preparation Guide

*How to Answer Deployments Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Deployment Basics](#deployment-basics)
2. [Rolling Updates](#rolling-updates)
3. [Rollout Strategies](#rollout-strategies)
4. [Troubleshooting Rollouts](#troubleshooting-rollouts)

---

## Deployment Basics

### Q1: What is a Kubernetes Deployment?

**How to Answer:**

"It's a controller that manages Pods through ReplicaSets. You declare the desired state — image, replica count, strategy — and the Deployment makes reality match it."

"It rolls out new versions, rolls back bad ones, and scales Pods up or down. Honestly, I almost never create Pods or ReplicaSets directly — Deployments are the way to run stateless workloads."

**Key Point:** "A Deployment is the standard wrapper around ReplicaSets that gives you declarative updates and rollbacks."

---

### Q2: How is a Deployment different from a ReplicaSet?

**How to Answer:**

"A ReplicaSet only guarantees a number of running Pods — that's it. A Deployment owns ReplicaSets and adds update logic on top."

"When you change the image in a Deployment, it creates a new ReplicaSet, scales it up gradually, and scales the old one down to zero. It also keeps revision history so you can roll back. A plain ReplicaSet can't do any of that."

**Key Point:** "ReplicaSet keeps Pod count steady; Deployment manages how that count transitions between versions."

---

### Q3: What are the key fields in a Deployment spec?

**How to Answer:**

"`replicas` is how many Pods you want. `selector` must match the Pod template labels — if those don't match, the Deployment can't find its own Pods."

"`template` is the Pod spec, and `strategy` controls how updates roll out. `revisionHistoryLimit` decides how many old ReplicaSets to keep around for rollbacks — I usually set it to 5 or 10."

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: myapp:v2
```

**Key Point:** "Selector, template, replicas, strategy — get the selector-template match right and most Deployment bugs disappear."

---

### Q4: How does rollback work in a Deployment?

**How to Answer:**

"Every time you change the Pod template, the Deployment creates a new revision and stores the old ReplicaSet. Rolling back just points the Deployment back at a previous revision."

"The command is simple — `kubectl rollout undo deployment/web`, or undo to a specific revision with `--to-revision=2`. The Deployment scales up the old ReplicaSet and scales down the broken one, using the same rolling strategy."

"One trap: rollback only restores the Pod template, not ConfigMaps or Secrets. If your bad release changed a config value, undo won't fix that."

**Key Point:** "Rollout undo rewinds the Pod template to a saved revision — but it doesn't restore external config."

---

## Rolling Updates

### Q5: How does a rolling update actually work under the hood?

**How to Answer:**

"When the Pod template changes, the Deployment creates a new ReplicaSet with the new image and starts scaling it up one step at a time. As new Pods become ready, it scales the old ReplicaSet down."

"It's governed by `maxSurge` and `maxUnavailable` — with the defaults, you get one extra Pod above your replica count and at most one down at a time. That keeps capacity stable through the whole update."

"New Pods go through their readiness probe before they count as 'available'. If the new Pods never become ready, the rollout stalls instead of taking everything down."

**Key Point:** "Rolling update = scale new ReplicaSet up and old one down in lockstep, gated by readiness probes."

---

### Q6: What do maxUnavailable and maxSurge control, and how do you tune them?

**How to Answer:**

"`maxUnavailable` is how many Pods can be down during the update — `maxSurge` is how many extra Pods can exist above the desired count. Both accept numbers or percentages."

"Defaults are 25% and 25%, which is fine for most services. If a service can't tolerate any downtime, I set `maxUnavailable: 0`. If the cluster is tight on resources and I can't afford extra Pods, I set `maxSurge: 0` so it kills before creating."

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

**Key Point:** "Zero downtime = maxUnavailable 0. Tight on resources = maxSurge 0. You tune the trade-off, not the mechanism."

---

### Q7: Why are readiness probes critical during a rollout?

**How to Answer:**

"A readiness probe tells Kubernetes when a Pod can actually serve traffic. Without one, a Pod is marked ready the moment the container starts — even if the app still needs 30 seconds to load its config."

"During a rolling update that's dangerous: the Deployment thinks new Pods are healthy, kills the old ones, and your service is down while the app is still booting. With a proper readiness probe, traffic only shifts to Pods that are genuinely up."

"I treat a missing readiness probe as a deployment bug. Liveness probes restart containers; readiness probes protect rollouts."

**Key Point:** "Readiness probe = the signal that gates traffic during a rollout. No probe, no safe update."

---

### Q8: How do you watch and control a rollout in progress?

**How to Answer:**

"`kubectl rollout status deployment/web` shows live progress. If something's wrong, `kubectl rollout pause deployment/web` freezes it mid-way so you can investigate without it completing."

"After the fix, `kubectl rollout resume` continues. And `kubectl rollout history deployment/web` shows all revisions with their change causes, which is the first place I look when debugging."

**Key Point:** "Rollout status to watch, pause to freeze, resume to continue, history to audit — the rollout subcommand is the whole toolkit."
