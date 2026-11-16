# Multi-Cloud & Hybrid Cloud Operations — Hands-On Lab

Nine drills that turn multi-cloud answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Write the one-paragraph multi-cloud justification

**Goal:** A business case for multi-cloud that names the specific risk it removes — or the honest conclusion that one cloud is enough.

**Steps:**

```bash
# Pick a real product (yours, or a familiar one like a food-delivery app).
# Write one paragraph answering: which of the three real reasons applies —
# lock-in risk, a best-of-breed service only one cloud does well, or a
# regulatory requirement? Name the specific service or regulation.
# Then list the complexity tax honestly: two IAM models, two networking
# models, two bills, two on-call runbooks. If no reason survives the
# paragraph, write "single cloud" and why — that's a valid answer.
```

**Expected output:** One paragraph with a named reason (or an honest single-cloud call), plus a short list of the complexity costs.

**Why it matters:** This is Q1 made concrete — "multi-cloud is a business decision" only counts when you can name the risk it actually removes.

---

## Exercise 2: Draw the three-way comparison — hybrid vs multi-cloud vs cloud-agnostic

**Goal:** A one-page diagram you could explain to an interviewer without notes.

**Steps:**

```bash
# Draw three boxes: hybrid (on-prem + one cloud), multi-cloud (two public
# clouds in active use), cloud-agnostic (portable design, any cloud).
# For each, write where the workloads live, what connects them, and one
# real example (e.g. bank core on-prem + AWS bursting = hybrid).
# Now grade a company you know: which box are they really in? Most
# "multi-cloud" teams are running everything in one cloud with a second
# account — write down the evidence.
```

**Expected output:** A one-page diagram with the three definitions, real examples, and an honest grading of one real company.

**Why it matters:** This is Q2 — interviewers ask this to catch people who repeat marketing definitions, and the grading exercise is what makes yours real.

---

## Exercise 3: Measure a VPN tunnel and decide VPN vs private link

**Goal:** A data-backed call on VPN vs Direct Connect for one real traffic pattern.

**Steps:**

```bash
# Take a real traffic pattern (e.g. nightly 500 GB backup sync, or
# steady 200 Mbps of app traffic between clouds).
# Check the numbers: a single IPsec tunnel caps around 1.25 Gbps, and
# shared internet latency jitters — look at CloudWatch TunnelDataIn /
# TunnelDataOut or the GCP VPN tunnel metrics for a real tunnel.
# Write the decision: VPN (cheap, fast to set up) or Direct Connect /
# ExpressRoute / Interconnect (stable, costly, weeks to provision) —
# with the throughput math that justifies it.
```

**Expected output:** The throughput and latency numbers for one real tunnel, and a written VPN-vs-private-link decision with the math behind it.

**Why it matters:** This is Q3 — "check the numbers first" is only an interview line once you've actually looked at a tunnel's metrics.

---

## Exercise 4: Federate one identity provider to two clouds

**Goal:** Human sign-in through one IdP and a workload that gets cloud access with no stored key.

**Steps:**

```bash
# Humans: set up OIDC/SAML federation from one IdP (Okta, Entra ID, or
# even GitHub) to two clouds, so one login reaches both consoles.
# Machines: give a pod in EKS an IAM role through IRSA, and the same
# workload in GKE access through Workload Identity Federation — no
# access keys anywhere in the repo or secret store.
# Now grep your repos for long-lived access keys (AKIA...) and list
# every place one is stored — each one is a leak waiting to happen.
```

**Expected output:** One IdP login reaching two clouds, two keyless workload identities, and a list of any long-lived keys still hiding in repos.

**Why it matters:** This is Q4 — federation stops being theory once you've revoked access in one place and watched it die in both clouds.

---

## Exercise 5: One pipeline, two clouds — values file per cloud

**Goal:** A single Helm chart and Terraform module that deploys the same app to two clouds, with the differences in data files only.

**Steps:**

```bash
# Take one small app and one Helm chart. Deploy it to two clusters
# (two kinds, minikube + kind, or two cloud free-tier clusters) using
# only a values file per cloud: region, node size, load balancer type.
# Do the same in Terraform: one module for the app, a thin per-cloud
# wrapper that fills in the provider-specific bits (ALB vs GCLB, IAM role).
# Now try the forbidden version: copy the whole pipeline per cloud, and
# change one thing in only one copy — that's the drift you're avoiding.
```

**Expected output:** One chart + one module deployed to two clouds from per-cloud values files, and a note on what drifted in the copy-paste version.

**Why it matters:** This is Q5 — "the differences live in a values file" is the line interviewers remember, and this drill is the proof behind it.

---

## Exercise 6: Price the egress of one cross-cloud data flow

**Goal:** A real egress number for one replication flow, and the decision it forces.

**Steps:**

```bash
# Pick one data flow (e.g. Postgres writes replicated AWS -> GCP).
# Measure the write volume for a day, then price it: AWS egress is
# roughly $0.09/GB, so 100 GB/day of writes is about $270/month just
# to copy data — before storage, before the replica.
# Write the decision: replicate everything, replicate reads only, or
# keep the data in one cloud and accept cross-cloud read latency.
# Include the consistency answer: what happens to writes during a
# partition between the clouds?
```

**Expected output:** A priced egress number for one real flow, the replicate-or-keep decision it forces, and the partition answer in writing.

**Why it matters:** This is Q6 — "data is sticky" becomes real the moment you see what copying every write across clouds actually costs.

---

## Exercise 7: Build the IP allocation sheet and DNS map

**Goal:** One sheet that prevents the overlapping-CIDR disaster before it happens.

**Steps:**

```bash
# On one page, allocate CIDR blocks: AWS VPC 10.10.0.0/16, GCP VPC
# 10.20.0.0/16, on-prem 10.30.0.0/16, with subnets per region — all
# non-overlapping, all from one sheet, with an owner per block.
# Add the DNS story: internal.aws.example.com and internal.gcp.example.com,
# or one shared private zone — and write which service names resolve
# where. Now find a real overlap (two 10.0.0.0/16 VPCs somewhere you
# work) and write what renumbering it would cost.
```

**Expected output:** A one-page IP allocation sheet, a DNS map, and a written cost of one real overlap you found.

**Why it matters:** This is Q7 — the sheet is the artifact that stops a three-year renumbering project, and finding a real overlap makes the lesson stick.

---

## Exercise 8: Run a DNS failover flip and measure the real RTO

**Goal:** A tested failover with a measured recovery time, not a hope.

**Steps:**

```bash
# Set up a Route 53 (or Cloud DNS) health check on /healthz of a real
# service, with a failover routing policy: PRIMARY in one cloud,
# SECONDARY in the other. Make the health check test the app endpoint,
# not just the load balancer.
# Now break the primary (stop the app, not the whole cloud) and time
# the flip: when did DNS start answering from the secondary, and when
# was the service actually usable? Write the real RTO down.
# Repeat with the check pointed at the load balancer only — watch the
# flip fail to fire while the app is down.
```

**Expected output:** A working DNS failover, a measured RTO in minutes, and the comparison showing why the check must test the app endpoint.

**Why it matters:** This is Q8 — "failover you've never tested is a hope" is the line, and this drill is the test that turns it into a number.

---

## Exercise 9: Build one pane of glass and one tagging standard

**Goal:** One dashboard and one cost view that work across clouds.

**Steps:**

```bash
# Telemetry: ship metrics from two sources (two clusters, or one cluster
# plus one cloud) into one Prometheus/Loki with the same labels:
# service, environment, cloud, region. Build one dashboard that shows
# both, and one alert rule that pages the same way for either.
# Cost: write the tagging standard (owner, service, environment,
# cost-center) and apply it to every resource in a test account; list
# the top ten spenders and find one untagged resource.
# End with the weekly review habit: top ten spenders, one untagged
# resource fixed, one alert retuned.
```

**Expected output:** One dashboard spanning two sources, one alert rule that works for both, a tagging standard, and a top-ten spenders list.

**Why it matters:** This is Q9 and Q10 together — one pane of glass and one tagging standard are habits, and this drill starts both.
