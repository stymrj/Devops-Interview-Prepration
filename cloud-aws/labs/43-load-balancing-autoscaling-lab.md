# Hands-On Lab: Load Balancing & Autoscaling on AWS

10 exercises. Goal: build a real ALB + ASG stack, watch it scale, break it, and fix it.

**Prerequisites:** AWS CLI configured, a VPC with 2 public subnets in different AZs, one key pair.

---

## Exercise 1: Launch template with user data

**Goal:** Create a versioned launch template that boots a tiny web server.

```bash
cat > user-data.sh <<'EOF'
#!/bin/bash
yum install -y httpd
systemctl enable --now httpd
INSTANCE=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
echo "Hello from $INSTANCE" > /var/www/html/index.html
echo OK > /var/www/html/healthz
EOF

aws ec2 create-launch-template --launch-template-name web-lt \
  --launch-template-data '{"ImageId":"ami-0abcdef1234567890","InstanceType":"t3.micro","UserData":"'"$(base64 -w0 user-data.sh)"'"}'
```

**Expected output:** A launch template named `web-lt`, version 1.
**Why it matters:** The template is the single source of truth for every instance the ASG ever creates.

---

## Exercise 2: Target group with a real health check

**Goal:** Create a target group that probes `/healthz`.

```bash
aws elbv2 create-target-group --name web-tg --protocol HTTP --port 80 \
  --vpc-id vpc-123 --health-check-path /healthz \
  --health-check-interval-seconds 30 --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2
```

**Expected output:** Target group ARN; initially 0 registered targets.
**Why it matters:** Health checks are what keep broken instances out of rotation.

---

## Exercise 3: ALB with a listener and rule

**Goal:** Create an internet-facing ALB and forward port 80 to the target group.

```bash
aws elbv2 create-load-balancer --name web-alb --subnets subnet-a subnet-b \
  --security-groups sg-alb
aws elbv2 create-listener --load-balancer-arn <alb-arn> \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=<tg-arn>
```

**Expected output:** ALB DNS name like `web-alb-1234567890.ap-south-1.elb.amazonaws.com`.
**Why it matters:** This is the request path from the internet to your fleet.

---

## Exercise 4: Attach the ASG to the target group

**Goal:** Create an ASG (min 2, max 4, desired 2) registered with the target group.

```bash
aws autoscaling create-auto-scaling-group --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-lt,Version='$Default' \
  --min-size 2 --max-size 4 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-a,subnet-b" \
  --target-group-arns <tg-arn> --health-check-type ELB \
  --health-check-grace-period 120
```

**Expected output:** 2 instances in `InService`, both healthy in the target group within ~3 minutes. Hit the ALB DNS — you see "Hello from i-..." from different instances on refresh.
**Why it matters:** `--health-check-type ELB` lets the ASG trust the ALB's verdict and replace sick instances.

---

## Exercise 5: Target tracking scaling policy

**Goal:** Keep average CPU at 50% and watch the ASG react to load.

```bash
aws autoscaling put-scaling-policy --policy-name cpu-50 \
  --auto-scaling-group-name web-asg --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":50.0}'
```

Then generate load (e.g. `stress` on the instances or ApacheBench against the ALB).

**Expected output:** `DesiredCapacity` climbs above 2 as CPU stays over 50%; new instances register and appear in the target group.
**Why it matters:** This is the thermostat pattern — the most common scaling answer in interviews.

---

## Exercise 6: Terminate an instance and watch self-healing

**Goal:** Kill one instance manually and observe the ASG replace it.

```bash
INSTANCE=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names web-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id $INSTANCE --should-decrement-desired-capacity false
```

**Expected output:** ASG launches a replacement; target group shows it `initial` then `healthy`; ALB keeps serving throughout.
**Why it matters:** Self-healing is the whole point of pairing an ASG with an ALB.

---

## Exercise 7: Lifecycle hook for graceful drain

**Goal:** Add a termination hook so in-flight requests drain before the instance dies.

```bash
aws autoscaling put-lifecycle-hook --lifecycle-hook-name drain-before-terminate \
  --auto-scaling-group-name web-asg --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING \
  --heartbeat-timeout 120
```

**Expected output:** On the next scale-in, instances enter `Terminating:Wait` for up to 120s before dying.
**Why it matters:** Hooks are how real teams avoid dropping requests during scale-in.

---

## Exercise 8: Instance refresh for a safe rolling update

**Goal:** Ship a new AMI version across the fleet without downtime.

```bash
# Create launch template version 2 with your new AMI, then:
aws autoscaling start-instance-refresh --auto-scaling-group-name web-asg \
  --preferences '{"MinHealthyPercentage": 50, "InstanceWarmup": 120}'
aws autoscaling describe-instance-refreshes --auto-scaling-group-name web-asg
```

**Expected output:** Refresh status moves `InProgress` → `Successful`; at least 50% of instances stay in service the whole time.
**Why it matters:** This is the safe-deploy story for ASG fleets.

---

## Exercise 9: Break it — unhealthy health check path

**Goal:** See what happens when the health check is misconfigured.

Change the target group's health check path to `/does-not-exist` and watch:

```bash
aws elbv2 modify-target-group --target-group-arn <tg-arn> \
  --health-check-path /does-not-exist
```

**Expected output:** Targets go `unhealthy`; ALB returns 503; the ASG starts terminating and replacing instances (which also fail) — a replacement loop.
**Why it matters:** This is the #1 real-world ALB footgun. Revert the path to fix it.

---

## Exercise 10: Clean up (and price the lesson)

**Goal:** Tear everything down and check what the lab cost.

```bash
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name web-asg --force-delete
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>
aws elbv2 delete-target-group --target-group-arn <tg-arn>
aws ec2 delete-launch-template --launch-template-name web-lt
```

Then open Cost Explorer and filter by the last hour.

**Expected output:** Everything gone; total cost for the lab typically under $1 with t3.micro in ap-south-1.
**Why it matters:** Forgetting to delete an ALB is how side projects rack up surprise bills.
