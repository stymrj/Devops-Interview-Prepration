# Load Balancing & Autoscaling on AWS Interview Preparation Guide

*How to Answer Load Balancing & Autoscaling Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [ELB Types and When to Use Each](#elb-types-and-when-to-use-each)
2. [ALB Deep Dive](#alb-deep-dive)
3. [Auto Scaling Groups Deep Dive](#auto-scaling-groups-deep-dive)
4. [Scaling Policies and Dynamic Scaling](#scaling-policies-and-dynamic-scaling)
5. [Interview Traps and Real Scenarios](#interview-traps-and-real-scenarios)

---

## ELB Types and When to Use Each

### Q1: ALB vs NLB vs GWLB vs CLB — how do I choose in an interview?

**How to Answer:**

"CLB is the legacy one — I mention it only to say I wouldn't pick it for anything new. ALB is my default for HTTP and HTTPS apps because it does path and host-based routing at layer 7. NLB is for extreme performance or non-HTTP TCP traffic — millions of requests per second, static IPs, ultra-low latency. GWLB is niche: it chains third-party virtual appliances like firewalls in front of my traffic. In an interview I say ALB unless the question screams performance or static IP, then it's NLB."

**Key Point:** "ALB for HTTP apps, NLB for raw TCP performance and static IPs — CLB is legacy, GWLB is for chaining security appliances."

---

### Q2: Why does an NLB preserve the client IP but an ALB hides it?

**How to Answer:**

"NLB operates at layer 4 and passes the packet through without terminating the connection, so my backend sees the real client IP directly. ALB terminates the connection at layer 7 and opens a new one to the target, so the backend sees the ALB's IP — I read the client IP from the X-Forwarded-For header instead. Interviewers use this to check whether I understand what layer 7 actually costs me. If I need the real IP in my app logs, I configure the app to trust X-Forwarded-For."

**Key Point:** "NLB passes the original packet through; ALB terminates the connection, so the client IP comes via the X-Forwarded-For header."

---

## ALB Deep Dive

### Q3: Walk me through how an ALB routes a request.

**How to Answer:**

"A request hits a listener, which is just a port plus protocol — say HTTPS on 443. The listener evaluates rules in priority order: host-based like api.example.com, or path-based like /checkout/*. The first matching rule forwards to a target group, which is the list of my backends — EC2 instances, IPs, or Lambda functions. So routing is listener → rule → target group → target. I explain it this way in interviews because it shows the request path, not just vocabulary."

**Key Point:** "Listener catches the request, rules match on host or path, and the winning rule forwards to a target group."

---

### Q4: What actually happens during a target health check?

**How to Answer:**

"The ALB periodically hits a health check path I configure on each target — the default is slash, but I always set a real endpoint like /healthz. If the target returns 2xx within the timeout for the healthy threshold count, it's marked healthy and gets traffic. Fail the unhealthy threshold and the ALB stops sending it traffic, and the attached ASG can terminate and replace it. The classic trap: the health check passes on the app port but the security group blocks the ALB's probes, so targets flap between healthy and unhealthy."

```bash
aws elbv2 create-target-group --name api-tg \
  --protocol HTTP --port 8080 --vpc-id vpc-123 \
  --health-check-path /healthz --health-check-interval-seconds 30
```

**Key Point:** "ALB probes a real health endpoint; fail the threshold and the target is pulled out of rotation automatically."

---

### Q5: Sticky sessions — when are they okay and when are they a smell?

**How to Answer:**

"Sticky sessions pin a user to one target using a cookie the ALB injects, so stateful sessions don't break when I scale. They're acceptable as a short-term bridge while I'm migrating a legacy stateful app. But in an interview I call them a smell, because they fight the whole point of stateless scaling — one hot target gets all the traffic and rolling deploys get messy. The right fix is externalizing session state to Redis or a database, then turning stickiness off."

**Key Point:** "Stickiness is a migration crutch; the real answer is stateless apps with sessions in Redis or the database."

---

## Auto Scaling Groups Deep Dive

### Q6: What are the moving parts of an Auto Scaling Group?

**How to Answer:**

"A launch template defines what each instance looks like — AMI, instance type, user data, IAM role. The ASG itself sets min, max, and desired capacity and spreads instances across AZs for resilience. I point the ASG at a target group so the ALB discovers new instances automatically, and health checks decide when a sick instance gets terminated and replaced. In an interview I lead with the template because interviewers want to hear that instances are cattle, created from a versioned template, not hand-built."

**Key Point:** "Launch template defines the instance; the ASG holds desired capacity across AZs and replaces unhealthy ones."

---

### Q7: When my ASG scales in, which instance does it kill?

**How to Answer:**

"By default the ASG uses the Default termination policy, which keeps AZs balanced first — it picks the AZ with the most instances, then the oldest launch template, then the instance closest to its billing hour. But I can set custom termination policies, like terminating the newest instances first to protect long-running ones, or the oldest to force freshness. Interviewers love this because most people assume it's random. If my app runs long-lived jobs, I care deeply about this answer."

**Key Point:** "Default policy balances AZs then kills the oldest template; custom policies let me protect or rotate instances deliberately."

---

### Q8: What are lifecycle hooks, and when would I actually use one?

**How to Answer:**

"A lifecycle hook pauses the ASG before it finishes launching or terminating an instance, so I can run custom work first. The classic use: on scale-in, wait for in-flight requests to drain and push final logs before the instance dies. On launch, I use one to run a warmup script or register the instance with an external system before it takes traffic. The hook publishes to SNS or EventBridge, my automation does its thing, then it signals complete — or the timeout fires and the ASG moves on."

**Key Point:** "Lifecycle hooks pause launch or termination so I can drain, warm up, or register instances before the ASG proceeds."

---

## Scaling Policies and Dynamic Scaling

### Q9: Simple, step, and target tracking scaling — what's the difference?

**How to Answer:**

"Simple scaling adds or removes a fixed number of instances when an alarm fires — it's the oldest and crudest, and it has a cooldown so it doesn't flap. Step scaling is smarter: the adjustment depends on how far the metric breached the threshold, so a bigger spike adds more instances. Target tracking is what I reach for most — I tell it to keep average CPU at 50 percent, and it continuously adjusts capacity like a thermostat. In an interview I say: target tracking for the steady state, step scaling when I need aggressive responses to big spikes."

```bash
aws autoscaling put-scaling-policy --policy-name cpu-target \
  --auto-scaling-group-name web-asg \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration file://config.json
```

**Key Point:** "Simple is fixed-step and crude, step scales by breach size, target tracking holds a metric at a set value like a thermostat."

---

### Q10: Scheduled scaling vs predictive scaling — when does each make sense?

**How to Answer:**

"Scheduled scaling changes capacity on a timetable — I know traffic triples at 9 AM on weekdays, so I pre-warm instances at 8:30. It's perfect for predictable patterns and avoids the lag of reactive scaling entirely. Predictive scaling uses machine learning on historical CloudWatch data to forecast demand and pre-provision capacity, which handles patterns I can't easily write a cron for. The interview answer: scheduled for known spikes like sales events, predictive when the pattern is real but irregular."

**Key Point:** "Scheduled scaling pre-warms on a timetable for known spikes; predictive scaling forecasts demand from history."

---

### Q11: What's instance refresh, and why do I care about warm pools?

**How to Answer:**

"Instance refresh rolls my ASG to a new launch template version — new AMI, new user data — replacing instances gradually with a configurable minimum healthy percentage, so it's a safe rolling deploy for my fleet. A warm pool keeps a set of pre-initialized instances stopped and ready, so scale-out is fast because the slow boot and warmup already happened. Together they solve the two pain points of ASGs: safe updates and slow scale-out. I mention both when interviewers ask how I handle deploys on ASGs."

**Key Point:** "Instance refresh is a rolling update for the fleet; warm pools pre-boot instances so scale-out skips the slow startup."

---

## Interview Traps and Real Scenarios

### Q12: My ALB returns 502s but the instances look healthy. What do I check?

**How to Answer:**

"A 502 from the ALB usually means the target closed the connection or responded badly — the ALB is fine, the backend isn't. First I check the target's health status and the ALB's TargetResponseTime and HTTPCode_Target_5XX metrics in CloudWatch. Common causes I've seen: the app crashed but the health check endpoint still returns 200, the app's keep-alive timeout is shorter than the ALB's idle timeout, or a security group changed and the ALB can't reach the app port. I debug target-side, not ALB-side."

**Key Point:** "ALB 502s mean the backend broke the connection — check target metrics, keep-alive timeouts, and security groups."

---

### Q13: Traffic spiked and my ASG scaled too slowly. Users saw errors for ten minutes. What went wrong?

**How to Answer:**

"Reactive scaling has a lag: the CloudWatch alarm needs several breached periods, then instances take minutes to boot, pass health checks, and register with the target group. Ten minutes of pain usually means my alarm thresholds were too loose or the AMI took forever to initialize. The fixes I talk about: tighter alarms with fewer evaluation periods, target tracking instead of simple scaling, warm pools to skip boot time, and scheduled or predictive scaling if the spike was foreseeable. And a little over-provisioned headroom is cheaper than an outage."

**Key Point:** "Slow scale-out is alarm lag plus boot time — fix with tighter alarms, warm pools, and pre-scaling for known spikes."

---

### Q14: How do I do zero-downtime deploys with an ALB and ASG?

**How to Answer:**

"I keep the ALB in front so traffic only flows to healthy targets, then roll the fleet with instance refresh at a safe minimum healthy percentage. The deregistration delay on the target group gives in-flight requests time to finish before the old instance is removed. For extra safety I use CodeDeploy blue-green, which spins up a whole new ASG, shifts the ALB listener to it, and keeps the old one around for instant rollback. The interview one-liner: the ALB's health checks plus deregistration delay are what make the deploy safe."

**Key Point:** "Instance refresh with minimum healthy percentage, deregistration delay for draining, or CodeDeploy blue-green for instant rollback."
