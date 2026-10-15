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
