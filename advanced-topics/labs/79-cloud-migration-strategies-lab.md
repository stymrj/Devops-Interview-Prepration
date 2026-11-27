# Cloud Migration Strategies — Hands-On Lab

Nine drills that turn migration answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Inventory an estate and tag every app with its R

**Goal:** A written inventory of a small estate with every app tagged rehost, replatform, repurchase, refactor, retire or retain — and a one-line reason per tag.

**Steps:**

```bash
# Make an inventory file — one row per app, with owner, dependencies, and your R tag:
cat > estate-inventory.csv <<'EOF'
app,owner,dependencies,current_hosting,r_tag,reason
billing-api,payments-team,postgres-orders,vm-onprem,replatform,"RDS removes DB babysitting"
legacy-reports,ops-team,postgres-orders; s3,vm-onprem,retire,"Nobody opened it in 6 months — check logs first"
internal-wiki,platform-team,none,vm-onprem,repurchase,"Move to SaaS, stop running it"
monolith-shop,store-team,postgres-shop; redis,vm-onprem,refactor,"Scales badly at peak — strangler fig later"
EOF
cat estate-inventory.csv
```

**Expected output:** A complete inventory with an R tag and a reason on every row — and at least one app tagged retire or retain, proving you didn't default everything to rehost.

**Why it matters:** This is Q2 made concrete — migration is a per-application decision, and the inventory is where that decision gets made.

---

## Exercise 2: Map dependencies for one app — find the hidden callers

**Goal:** A dependency map for a single app that surfaces at least one caller or job nobody had documented.

**Steps:**

```bash
# 1. List what the app talks to from its config and connection strings:
grep -ri "postgres\|redis\|http" /etc/myapp/config/ 2>/dev/null | head -20
# 2. Find who talks to the app — check access logs for distinct callers:
awk '{print $1}' /var/log/nginx/access.log | sort -u | head -20
# 3. Find the batch jobs that touch its database — cron entries and their owners:
crontab -l; ls /etc/cron.d/; grep -r "myapp" /etc/cron* 2>/dev/null
```

**Expected output:** A short dependency map (app → database, app → downstream, callers → app, cron jobs → app) — with at least one hidden dependency written down, like a cron job or a reporting query nobody mentioned.

**Why it matters:** This is Q4 made concrete — the undocumented dependency is the thing that breaks a cutover, and discovery is how you find it before it does.

---

## Exercise 3: Size the target from real usage, not a guess

**Goal:** A right-sized cloud target for one workload, derived from a full week of real CPU and memory data.

**Steps:**

```bash
# Collect a week of usage for one VM (sar history, or cloud watch metrics if it exists):
sar -u -f /var/log/sa/sa$(date +%d -d "7 days ago") | tail -5
# Peak, not average — find the busiest hour:
sar -u -s 09:00:00 -e 18:00:00 -f /var/log/sa/sa$(date +%d) | sort -k3 -n | tail -3
# Compare against the instance type you would pick — and write down the headroom:
# e.g. peak 65% of 4 vCPU -> a 4 vCPU cloud instance with ~35% headroom, not an 8 vCPU one.
```

**Expected output:** Peak CPU and memory over a business cycle, the cloud instance type you picked from it, and a one-line note on the headroom — plus a note on what a single quiet weekend's numbers would have wrongly suggested.

**Why it matters:** This is Q4 made concrete — sizing on a full business cycle is the difference between a target that holds at peak and one that falls over on Monday morning.

---

## Exercise 4: Replicate a live database with DMS — snapshot plus CDC

**Goal:** A running DMS replication task in full-load-plus-CDC mode, with lag in single-digit seconds.

**Steps:**

```bash
# 1. Create the source and target endpoints (source: on-prem Postgres, target: RDS Postgres):
aws dms create-endpoint --endpoint-identifier source-pg --endpoint-type source \
  --engine-name postgres --server-name <on-prem-host> --port 5432 \
  --database-name shop --username dms_user --password <password>
# 2. Create the task in full-load-plus-CDC mode and start it:
aws dms create-replication-task --replication-task-identifier shop-migration \
  --source-endpoint-arn <source-arn> --target-endpoint-arn <target-arn> \
  --migration-type full-load-and-cdc --table-mappings file://table-mappings.json
aws dms start-replication-task --replication-task-arn <task-arn> \
  --start-replication-task-type start-replication
# 3. Watch the lag until it is stable in single-digit seconds:
aws dms describe-replication-tasks --filters Name=replication-task-arn,Values=<task-arn> \
  --query 'ReplicationTasks[0].ReplicationTaskStats'
```

**Expected output:** The task in `Running` state with CDC lag in single-digit seconds — and a written note of which tables, if any, had schema features (like a stored procedure writing to a file) that don't replicate cleanly.

**Why it matters:** This is Q6 made concrete — snapshot plus CDC is how a live database moves without a big-bang dump, and lag is the number that decides when cutover is safe.

---

## Exercise 5: Practise a blue-green cutover with a weighted traffic shift

**Goal:** A blue-green deploy where traffic shifts 10% → 50% → 100% to the new environment, with a rollback that takes under a minute.

**Steps:**

```bash
# 1. Deploy green next to blue behind the same load balancer (two target groups):
aws elbv2 create-target-group --name shop-green --protocol HTTP --port 80 --vpc-id <vpc-id>
# 2. Shift 10% of traffic to green, watch errors for 10 minutes:
aws elbv2 modify-listener --listener-arn <listener-arn> --default-actions \
  '[{"Type":"forward","ForwardConfig":{"TargetGroups":[{"TargetGroupArn":"<blue-arn>","Weight":90},{"TargetGroupArn":"<green-arn>","Weight":10}]}}]'
# 3. Step to 50%, then 100% — and practise the rollback: one call flips all traffic back to blue.
aws elbv2 modify-listener --listener-arn <listener-arn> --default-actions \
  '[{"Type":"forward","TargetGroupArn":"<blue-arn>"}]'
```

**Expected output:** Traffic split 90/10 visible in the target-group request counts, no error-rate rise on green at each step, and a timed rollback to blue that completes in under a minute.

**Why it matters:** This is Q7 made concrete — a gradual shift plus a one-command rollback is what turns "zero downtime" from a hope into a plan.

---

## Exercise 6: Strangle one endpoint — route a single slice through a gateway

**Goal:** One endpoint of a monolith served by a new service behind a gateway, while everything else still hits the monolith.

**Steps:**

```bash
# 1. Put a gateway (or a simple nginx reverse proxy) in front of the monolith:
cat > /etc/nginx/conf.d/strangler.conf <<'EOF'
upstream monolith { server 10.0.1.10:8080; }
upstream search-svc { server 10.0.2.20:8080; }
server {
  listen 80;
  location /api/search { proxy_pass http://search-svc; }   # strangled slice
  location / { proxy_pass http://monolith; }               # everything else stays
}
EOF
nginx -t && nginx -s reload
# 2. Prove it: /api/search answers from the new service, /api/orders still from the monolith.
curl -s http://localhost/api/search | head -c 200; echo
curl -s http://localhost/api/orders | head -c 200; echo
```

**Expected output:** `/api/search` answered by the new service and `/api/orders` by the monolith — and a note on what the new service reads from: its own store, or the old database through an anti-corruption layer.

**Why it matters:** This is Q8 made concrete — the strangler fig is one route at a time, and the routing rule is the whole pattern in miniature.

---

## Exercise 7: Prove hybrid DNS and non-overlapping CIDRs before the first wave

**Goal:** A written check that on-prem and cloud resolve each other's hostnames and that no CIDR overlaps — done before any workload moves.

**Steps:**

```bash
# 1. Confirm the ranges don't overlap (on-prem 10.0.0.0/16, cloud VPC 10.1.0.0/16):
ip route | grep "10\.0\|10\.1"
# 2. Confirm DNS resolves both ways through the conditional forwarder:
dig +short db.internal.example.com @10.0.0.2
dig +short app.internal.example.com @10.1.0.2
# 3. Confirm the VPN tunnel is up and a packet actually crosses it:
ping -c 3 <cloud-private-ip>
```

**Expected output:** Both `dig` commands returning private IPs from the other side, a successful ping across the tunnel, and a one-line note of the two CIDR ranges proving they don't overlap.

**Why it matters:** This is Q9 made concrete — DNS and CIDRs are decided on day one because fixing them mid-migration is painful, and the check takes five minutes.

---

## Exercise 8: Audit identity and secrets for one app before its wave

**Goal:** A written audit for one app listing every identity it uses and every secret it needs — with nothing left as "it just works".

**Steps:**

```bash
# 1. Find every identity: service accounts, AD groups, IAM roles the app relies on:
grep -ri "service-account\|rolearn\|assume" /etc/myapp/config/ 2>/dev/null | head -20
# 2. Find every secret: connection strings, API keys, certificates baked into the VM:
grep -ri "password\|api_key\|BEGIN CERTIFICATE" /etc/myapp/ /opt/myapp/ 2>/dev/null | cut -c1-80 | head -20
# 3. For each one, write its cloud home: IAM role / Secrets Manager / ACM — and who rotates it.
```

**Expected output:** A table with one row per identity and secret, each with its cloud equivalent and a rotation owner — and at least one secret you found that nobody on the team could have named from memory.

**Why it matters:** This is Q10 made concrete — identity and secrets are what actually break in a migration, and the audit is how you catch them before the wave, not during it.

---

## Exercise 9: The payments cutover drill — full runbook, one rehearsal

**Goal:** A written cutover runbook for a payments monolith, rehearsed once end to end, with rollback triggers agreed in advance.

**Steps:**

```bash
# 1. Lower the DNS TTL days ahead of the drill (so the flip propagates fast):
#    Set the app record TTL to 60s in Route 53, and wait out the old TTL first.
# 2. Rehearse the whole cutover in staging: replicate, flip traffic, run smoke tests:
#    - smoke: login, checkout with a test card, nightly report job
#    - watch: latency, error rate, payment success rate vs the baseline
# 3. Write the runbook: exact commands in order, rollback triggers (e.g. error rate > 1% for 5 min),
#    the rollback command, and who makes the call. Time the rehearsal.
```

**Expected output:** A one-page runbook with the ordered commands, the smoke-test list, the rollback triggers, the rollback command, and the rehearsal timing — plus a note on what the rehearsal exposed that the plan had missed.

**Why it matters:** This is Q12 made concrete — the runbook plus a rehearsal is what makes a 5-minute cutover real, and the rehearsal always finds the thing the plan missed.
