# Kubernetes Autoscaling Deep Dive (HPA, VPA, KEDA & Cluster Autoscaler) Interview Preparation Guide

*How to Answer Kubernetes Autoscaling Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [The Four Autoscalers](#the-four-autoscalers)
2. [HPA in Depth](#hpa-in-depth)
3. [VPA & KEDA](#vpa--keda)
4. [Scaling the Nodes](#scaling-the-nodes)
5. [Autoscaling in Production](#autoscaling-in-production)

---

## The Four Autoscalers

### Q1: HPA, VPA, Cluster Autoscaler, KEDA — what does each one scale, and how are they different?

**How to Answer:**

"I'd start by saying they scale different layers, so they're complements, not alternatives. The Horizontal Pod Autoscaler scales the number of pods — it watches a metric like CPU and adds or removes replicas of a Deployment. The Vertical Pod Autoscaler scales the size of pods — it adjusts the CPU and memory requests of a container, and it restarts the pod to apply them. The Cluster Autoscaler scales the number of nodes — when pods can't be scheduled because there's no room, it asks the cloud provider for a bigger node group. KEDA is an event-driven scaler that sits on top of the HPA — it scales pods on things like queue length or messages per second, including scaling to zero. My line: HPA scales pod count, VPA scales pod size, Cluster Autoscaler scales node count, KEDA scales pod count on events."

**Key Point:** "HPA scales pod count, VPA scales pod size, Cluster Autoscaler scales node count, KEDA scales pod count on events — different layers, not alternatives."

---

### Q2: What is the autoscaling stack actually made of — who watches what, and who acts?

**How to Answer:**

"I'd describe it as a chain, because the interviewer wants to hear where it can break. The metrics-server collects CPU and memory from the kubelet and serves the metrics API — that's the HPA's data source for resource metrics. For anything fancier, like requests per second or queue depth, you need a custom metrics adapter — usually the Prometheus Adapter — which exposes Prometheus queries as metrics the HPA can read. The HPA controller runs its loop every 15 seconds by default, reads the metric, computes the replica count, and patches the Deployment. The Cluster Autoscaler runs its own loop, watches for unschedulable pods, and calls the cloud API to add nodes. My line: metrics flow from kubelet to metrics-server to the HPA, custom metrics come from Prometheus, and each scaler is its own controller with its own loop."

**Key Point:** "Metrics flow from kubelet to metrics-server to the HPA, custom metrics come from Prometheus via an adapter, and each scaler runs its own loop."

---

## HPA in Depth

### Q3: How does the HPA actually decide how many replicas to run?

**How to Answer:**

"I'd explain the formula, because it's simple and interviewers love it. The HPA computes desired replicas as the current replicas multiplied by the current metric value divided by the target — so if I have 4 pods at 80% CPU and my target is 50%, it wants 4 times 80 over 50, which is about 7 pods. It checks every 15 seconds by default, but it doesn't act on every reading — the stabilization window holds scale-downs back for 5 minutes by default, so one quiet minute doesn't kill half my pods. It also respects minReplicas and maxReplicas, and it never scales below the minimum even if the math says zero. My line: desired equals current times current over target, checked every 15 seconds, with a 5-minute cool-down on scaling down."

```yaml
# An HPA that scales on CPU with a slow scale-down so traffic dips don't kill pods.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-api
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 minutes before scaling down
```

**Key Point:** "Desired replicas equals current times current metric over target — checked every 15 seconds, scale-down held back 5 minutes by default."

---

### Q4: CPU vs memory vs custom metrics — what should you actually scale on?

**How to Answer:**

"I'd say CPU is the safe default, memory is usually a trap, and custom metrics are where the real wins are. CPU works well because it's compressible — a pod using more CPU just runs slower, it doesn't die, so scaling on it is forgiving. Memory is dangerous to scale on because memory doesn't shed load — if a pod is using a lot of memory it's often a leak or a big request, and adding pods can just spread the leak around. The honest answer is to scale on the thing your users feel: requests per second, queue depth, or latency — but that needs a custom metric through the Prometheus Adapter, which is one more thing to run. My line: CPU by default, never memory alone, and graduate to request rate or queue depth when the service is latency-sensitive."

**Key Point:** "CPU is the safe default, memory is a trap — it doesn't shed load — and the best signal is what users feel: request rate or queue depth."

---

### Q5: How do you stop the HPA from flapping — scaling up and down every few minutes?

**How to Answer:**

"I'd say flapping almost always means the target is too tight or the cool-downs are too short. The first fix is the stabilization window — the default already holds scale-downs for 5 minutes, and for a spiky service I'd raise the scale-up window too, so one spike doesn't add 20 pods that vanish a minute later. The second fix is the scale policies — I cap how fast it can grow, like at most double the replicas or add 4 pods per minute, so a single metric spike can't cause a stampede. The third fix is the target itself — if I target 90% CPU, every tiny wobble crosses the line, so I target 60 to 70% and leave headroom. My line: widen the stabilization windows, cap the scale-up rate with a policy, and target 60 to 70%, not 90%."

```yaml
# Scale up fast but in steps, scale down slowly — no flapping on a spiky service.
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60
    policies:
      - type: Percent
        value: 100          # at most double the replicas per minute
        periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
      - type: Pods
        value: 2            # remove at most 2 pods per minute
        periodSeconds: 60
```

**Key Point:** "Flapping means the target is too tight or the cool-downs too short — widen the stabilization windows, cap the scale-up rate, target 60–70% CPU."

---

### Q6: What happens when the HPA hits maxReplicas during a real spike — and how do you prepare for it?

**How to Answer:**

"I'd be honest: when the HPA hits maxReplicas it just stops, and the pods you have absorb the load until they fall over — so maxReplicas is a real design decision, not a random number. I set it from the load test: the highest replica count I proved the system can actually run, including the database connections and downstream rate limits at that count. I also alert on it — 'HPA at max replicas for 5 minutes' pages me, because it means the system is saturated and I need to either raise the ceiling or shed load. And I make sure the ceiling is reachable: if the Cluster Autoscaler can't add nodes fast enough, the HPA's extra pods just sit unschedulable. My line: maxReplicas comes from the load test, alert when it's hit, and make sure the nodes can actually appear to host the extra pods."

**Key Point:** "HPA at maxReplicas just stops — set the ceiling from the load test, alert when it's hit, and make sure nodes can appear to host the extra pods."

---

## VPA & KEDA

### Q7: What does the VPA do, and why can't you just run VPA and HPA together on CPU?

**How to Answer:**

"I'd say the VPA rightsizes pods — it watches actual usage over days and recommends or applies better CPU and memory requests, so pods stop being wildly over- or under-provisioned. It has three modes: Off just recommends, Initial sets requests only when a pod is created, and Auto evicts and recreates pods with the new requests. The clash is the classic trap: if the HPA scales on CPU percentage and the VPA lowers the CPU request, the same absolute CPU suddenly looks like a higher percentage, so the HPA adds pods, the VPA lowers requests again, and they fight each other in a loop. The fix is to run them on different signals — VPA for memory, HPA on CPU — or run VPA in Off mode and apply its recommendations by hand. My line: VPA rightsizes pod requests, it restarts pods to apply them, and it fights the HPA on the same metric — so split the signals."

**Key Point:** "VPA rightsizes pod requests and restarts pods to apply them — run it on a different signal than the HPA or they'll fight in a loop."

---

### Q8: How does KEDA work, and when is it the right scaler instead of the plain HPA?

**How to Answer:**

"I'd say KEDA is the HPA for events — it watches things the HPA can't see natively, like queue length, Kafka lag, or a cron schedule, and scales the workload from those. You write a ScaledObject that names a trigger — say, an SQS queue with a target of 10 messages per pod — and KEDA creates and manages the HPA for you, feeding it the metric. The killer feature is scale-to-zero: when the queue is empty, KEDA scales the deployment to zero pods, which the plain HPA can't do, so you stop paying for idle workers overnight. The catch is cold start — the first message after zero waits for a pod to boot, so it's wrong for user-facing APIs and right for workers and batch jobs. My line: KEDA scales on events and scales to zero — perfect for queue workers, wrong for anything a user is waiting on."

```yaml
# A KEDA ScaledObject: one pod per 10 queued messages, down to zero when idle.
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-worker
spec:
  scaleTargetRef:
    name: order-worker
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-south-1.amazonaws.com/123456789012/orders
        queueLength: "10"        # one pod per 10 messages waiting
        awsRegion: ap-south-1
```

**Key Point:** "KEDA scales on events like queue length and scales to zero — perfect for workers, wrong for user-facing APIs because of cold start."

---

### Q9: VPA recommendations — how do you trust them, and what guardrails do you put around them?

**How to Answer:**

"I'd say I trust VPA recommendations only after they've seen real traffic, and I fence them in with min and max. The VPA builds its recommendation from a histogram of actual usage over days, so a recommendation made after one quiet weekend is worthless — I want at least a full week including a peak. I set minAllowed and maxAllowed so a bug can't recommend 16 CPUs for a sidecar or 64Mi for a Java app that needs 2Gi to start. And I never let Auto mode touch stateful or latency-critical workloads — an eviction mid-request is a dropped request, so for those I run Off mode and apply recommendations in a pull request after a human looks at them. My line: a week of real traffic, min and max fences, Auto only for stateless workers, everything else by pull request."

**Key Point:** "VPA needs a week of real traffic, min/max fences, and Auto mode only for stateless workers — everything else by pull request."

---

## Scaling the Nodes

### Q10: How does the Cluster Autoscaler decide to add a node, and when does it remove one?

**How to Answer:**

"I'd explain it as two separate decisions with different triggers. It adds a node when a pod has been unschedulable for a while — usually it simulates the scheduling, and if the pod would fit on a bigger node group, it calls the cloud API to grow that group. It removes a node only when every pod on it can move somewhere else — it checks that the node is underutilised, that no pod has a local-storage or anti-affinity constraint pinning it there, and that the PodDisruptionBudget allows the eviction. The classic gotcha is a single fat pod that can never be evicted — like a pod with a local volume — which blocks the whole node from ever scaling down. My line: it adds nodes for unschedulable pods, removes nodes only when every pod can move, and one unmovable pod blocks the whole node."

**Key Point:** "Cluster Autoscaler adds nodes for unschedulable pods and removes them only when every pod can move — one unmovable pod blocks the whole node."

---

### Q11: Cluster Autoscaler vs Karpenter — how do you choose, and what changes in practice?

**How to Answer:**

"I'd say Karpenter is the newer answer to the same problem, and the difference is how they pick nodes. The Cluster Autoscaler works on fixed node groups — you predefine instance types and sizes, and it grows or shrinks those groups, which is simple but slow and rigid. Karpenter watches for unschedulable pods and provisions exactly the right instance for them, right then — it can pick a bigger instance for one fat pod instead of adding a whole group, and it consolidates underused nodes automatically. The trade-off is that Karpenter is another controller to run and it needs broad cloud permissions, while the Cluster Autoscaler is the managed, boring default on EKS. My line: Cluster Autoscaler for simple, predictable workloads on fixed groups — Karpenter when pods vary wildly in size and you want the cheapest node that fits, right now."

**Key Point:** "Cluster Autoscaler grows fixed node groups — simple and slow; Karpenter provisions the exact instance a pod needs — flexible, but another controller to run."

---

## Autoscaling in Production

### Q12: Design it end to end — a checkout API that goes from 200 to 20,000 requests per second on sale day. What scales, in what order?

**How to Answer:**

"I'd walk through the layers in the order they actually kick in. First the HPA on request rate, not CPU — a custom metric from Prometheus, target 60% CPU as a backstop, min 3 replicas for baseline, max set from the load test, with a 5-minute scale-down window so the post-sale dip doesn't flap. Second, the Cluster Autoscaler with a pre-warmed node pool — I scale the nodes up an hour before the sale, because nodes take minutes to appear and the HPA's extra pods can't run on nodes that don't exist yet. Third, the database and downstream — connection pools sized for the max replica count, and the payment gateway's rate limit confirmed in writing, because autoscaling the app into a database that can't take more connections just moves the failure. And the guardrails: PodDisruptionBudgets so a scale-down never takes out the last healthy pod, and an alert the moment the HPA sits at maxReplicas. My line: HPA on request rate, nodes pre-warmed before the sale, pools and rate limits sized for the max — and alert at the ceiling."

**Key Point:** "HPA on request rate, nodes pre-warmed before the sale, pools and rate limits sized for the max replica count — and alert the moment the HPA hits its ceiling."
