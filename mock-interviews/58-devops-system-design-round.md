# DevOps System Design Round Interview Preparation Guide

*How to Answer System Design Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [The Design Framework](#the-design-framework)
2. [Scoping the Problem](#scoping-the-problem)
3. [Core Architecture Decisions](#core-architecture-decisions)
4. [Reliability and Scaling](#reliability-and-scaling)
5. [CI CD and Release Design](#ci-cd-and-release-design)
6. [Handling Follow Ups](#handling-follow-ups)

---

## The Design Framework

### Q1: How do you structure a DevOps system design answer when the interviewer says "design X"?

**How to Answer:**

"I follow the same skeleton every time: scope, requirements, rough sketch, deep dives. I start by asking questions and writing down functional and non-functional requirements. Then I draw the happy-path architecture in two or three boxes — I don't go deep yet. The rest of the round I spend on the parts the interviewer pokes at. And I close by naming what I'd do with ten times the scale."

**Key Point:** "Scope first, sketch the happy path, then let the interviewer's questions pick the deep dives."

---

### Q2: What should you clarify before you draw a single box?

**How to Answer:**

"I ask about scale first — users, requests per second, data volume — because everything downstream depends on it. Then I ask what matters most: is this about availability, cost, speed of delivery, or all three? I also ask what exists already — greenfield on AWS is a very different answer from brownfield with legacy VMs. And I confirm what 'done' looks like: are they judging the CI/CD story or the HA story more?"

**Key Point:** "Scale, priorities, and existing constraints decide the design — ask before you draw."

---

### Q3: Interviewers love "design a CI/CD pipeline for 50 engineers deploying 20 times a day." How do you answer?

**How to Answer:**

"I start with trunk-based development: short-lived branches, every merge to main builds and tests. Pull requests get ephemeral preview environments so reviewers click the real thing, not screenshots. Merging to main triggers the full suite and builds a release artifact with a versioned tag. Deployments go out progressively — canary, then rolling — with automatic rollback on error-rate or latency alarms. And everything is a pipeline-as-code file in the same repo, because the pipeline itself gets reviewed like code."

```yaml
# the shape every pipeline I draw has: build once, promote the same artifact
stages:
  - build    # compile + unit tests -> versioned image
  - test     # integration + contract tests on preview env
  - canary   # 5% traffic, auto-promote on clean metrics
  - rollout  # progressive to 100%, auto-rollback on alarms
```

**Key Point:** "Build once, promote the same artifact, and let metrics — not humans — decide promotions and rollbacks."

---

### Q4: How do you design deployments that never take the site down?

**How to Answer:**

"I reach for rolling or blue-green depending on the blast radius I'm comfortable with. Rolling updates keep capacity healthy because new pods only replace old ones once health checks pass. Blue-green gives me an instant cutover and an instant way back — I just flip the router. For risky releases I add a canary step: 5 percent of traffic to the new version with automatic promotion if SLOs hold. And database changes always ship separately from code, backward-compatible, so a rollback never fights a migrated schema."

**Key Point:** "Ship code and schema separately, and always keep a one-command path back to the old version."

---

## Scoping the Problem

### Q5: How do you talk about compute choices — VMs, containers, serverless — without sounding like you memorized a comparison table?

**How to Answer:**

"I tie the choice to what the workload does, not to what's trendy. Stateless web services with bursty traffic go on containers or serverless, because I want fast scale and I don't manage hosts. Stateful or latency-sensitive things — databases, stream processors — get VMs or dedicated nodes, because I want control over placement and storage. And I say the tradeoff out loud: serverless buys me speed but costs more at steady high load, containers need a platform team. Interviewers want the tradeoff, not the winner."

**Key Point:** "Justify compute with the workload's shape — stateless and bursty vs stateful and steady — then name the tradeoff."

---

### Q6: Walk me through designing infrastructure for a classic prompt: "a web app, three services, needs to survive a region failure."

**How to Answer:**

"I draw two regions, each with its own VPC, subnets across three AZs, and the full stack running in both. Traffic enters through a global DNS or anycast layer, and the database replicates across regions with a documented RPO. I make the failover decision explicit — active-active for the stateless tier, active-passive with tested promotion for the database. Then I say the honest part: multi-region doubles cost and complexity, so I'd ask whether the business actually needs it or whether multi-AZ plus fast restore is enough."

**Key Point:** "Multi-AZ is table stakes; multi-region is a business decision — state the RTO, the RPO, and the cost."

---

## Core Architecture Decisions

### Q7: How do you design observability into the system from the start, not as an afterthought?

**How to Answer:**

"I define the SLIs first — what the user actually feels — and the SLOs follow from those. Every service emits the same three signals: metrics, structured logs, and traces with propagated IDs. I put a central collection layer in the diagram so logs and metrics flow off the hosts, because debugging across SSH boxes is how you lose an incident. And I draw the alerting path explicitly: paging alerts only on SLO burn, everything else is a ticket. If the diagram has no alerting story, the design isn't finished."

**Key Point:** "Metrics, logs, traces from day one — and page only on user-impacting SLO burn."

---

### Q8: How do you design config and secrets management in a whiteboard answer?

**How to Answer:**

"I split them cleanly: config is non-sensitive and rides with the deploy — environment variables or ConfigMaps. Secrets never touch the repo or the image; they come from a vault or a managed secrets service and get injected at runtime. I rotate them on a schedule and scope each service to only the secrets it needs. And I say what happens when a secret leaks: rotate immediately, revoke the old one, and check the audit log. That last part is what makes it sound like I've actually done it."

```bash
# the pattern I name: fetch at runtime, never bake in
vault kv get -field=password secret/prod/db  # injected as env var at startup
```

**Key Point:** "Config ships with code, secrets arrive at runtime — and every design needs a leak-response story."

---

## Reliability and Scaling

### Q9: How do you answer "take this system from 100 users to 10 million"?

**How to Answer:**

"I scale in phases and I say so: the design that works at 100 users is not the one at 10 million. First I go stateless and horizontal — more app replicas behind a load balancer. Then I add caching in front of the database, because reads are the first thing that breaks. Then I split reads and writes, shard or partition the data, and move heavy work to queues. And at each step I name what breaks next — that's the whole point of the question, showing you know the failure mode before it finds you."

**Key Point:** "Scale in named phases — stateless replicas, cache, queues, sharding — and predict what breaks at each step."

---

### Q10: How do you design for failure without turning the diagram into spaghetti?

**How to Answer:**

"I keep it to a few patterns and apply them everywhere: timeouts and retries with backoff on every cross-service call, circuit breakers so one slow dependency can't cascade, bulkheads so one bad tenant or queue doesn't eat the whole system. Then I add graceful degradation — what does the user see when the recommendation service is down? Finally I mention testing it: game days and chaos drills, because an untested fallback is just a hope. Three patterns, applied consistently, beats ten drawn once."

**Key Point:** "Retries with backoff, circuit breakers, graceful degradation — consistent patterns beat clever one-offs."

---

## CI CD and Release Design

### Q11: How do you handle the cost question — "this design is nice but too expensive"?

**How to Answer:**

"I don't get defensive — cost is a real design input. I start by finding the expensive parts: usually data transfer, idle capacity, and over-provisioned databases. Then I right-size: autoscaling with real utilization targets, spot or preemptible instances for batch work, lifecycle policies moving old data to cheap storage. I also ask what 'too expensive' means — a number beats a vibe, and sometimes the honest answer is that the cheaper design drops an availability nine. I present the cheaper option with the reliability tradeoff attached."

**Key Point:** "Name the expensive parts, right-size with autoscaling and tiered storage, and attach the reliability tradeoff to every saving."

---

## Handling Follow Ups

### Q12: What do you do when the interviewer changes the constraints halfway through — "now make it multi-cloud" or "cut the budget in half"?

**How to Answer:**

"I treat it as the real test, because it is — they're checking whether I can reason, not whether my first drawing was perfect. I keep what survives the new constraint and rework what doesn't, and I say my reasoning out loud as I redraw. For multi-cloud I admit the cost honestly: abstractions like Kubernetes and Terraform buy portability, but managed services lock you in anyway, so I'd ask what the real driver is. And I never pretend the constraint is free — every new requirement trades against something, and I name what I'm giving up."

**Key Point:** "Mid-round constraint changes test reasoning, not the first sketch — rework it out loud and name the tradeoff."

---
