# Load Balancing & Autoscaling — Cheat Sheet

## ELB types at a glance

| Type | Layer | Use when |
|---|---|---|
| ALB | 7 (HTTP/HTTPS) | Web apps, host/path routing |
| NLB | 4 (TCP/UDP) | Extreme perf, static IPs, non-HTTP |
| GWLB | 3 | Chaining firewall/IDS appliances |
| CLB | 4/7 | Legacy only — don't pick for new builds |

## ALB essentials

```bash
# Create ALB
aws elbv2 create-load-balancer --name web-alb --subnets subnet-a subnet-b \
  --security-groups sg-alb

# Target group with health check
aws elbv2 create-target-group --name web-tg --protocol HTTP --port 80 \
  --vpc-id vpc-123 --health-check-path /healthz \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 2

# Listener forwarding to target group
aws elbv2 create-listener --load-balancer-arn <alb-arn> \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=<tg-arn>

# Path-based rule (priority matters, lower = checked first)
aws elbv2 create-rule --listener-arn <listener-arn> --priority 10 \
  --conditions Field=path-pattern,Values='/api/*' \
  --actions Type=forward,TargetGroupArn=<api-tg-arn>

# Register / deregister targets
aws elbv2 register-targets --target-group-arn <tg-arn> \
  --targets Id=i-1234567890abcdef0
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

**Key ALB facts:** request path = listener → rules (priority order) → target group → target. Client IP via `X-Forwarded-For` header (ALB terminates the connection). Deregistration delay drains in-flight requests (default 300s — lower it for fast deploys). Sticky sessions = ALB-injected cookie; fine as a migration crutch, externalize state to Redis for the real fix. 502 = backend broke the connection (check target metrics, keep-alive vs idle timeout, SGs). 503 = no healthy targets.

## ASG essentials

```bash
# Create ASG from launch template
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-lt,Version='$Default' \
  --min-size 2 --max-size 10 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-a,subnet-b" \
  --target-group-arns <tg-arn> \
  --health-check-type ELB --health-check-grace-period 120

# Target tracking (thermostat) — the default right answer
aws autoscaling put-scaling-policy --policy-name cpu-50 \
  --auto-scaling-group-name web-asg --policy-type TargetTrackingScaling \
  --target-tracking-configuration \
  '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":50.0}'

# Scheduled scaling (known spikes)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name web-asg --scheduled-action-name morning-rush \
  --recurrence "30 8 * * MON-FRI" --min-size 4 --desired-capacity 6

# Rolling update of the fleet
aws autoscaling start-instance-refresh --auto-scaling-group-name web-asg \
  --preferences '{"MinHealthyPercentage":50,"InstanceWarmup":120}'

# Lifecycle hook (drain on terminate)
aws autoscaling put-lifecycle-hook --lifecycle-hook-name drain \
  --auto-scaling-group-name web-asg \
  --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING \
  --heartbeat-timeout 120

# Inspect
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names web-asg
aws autoscaling describe-scaling-activities --auto-scaling-group-name web-asg
```

**Scaling policy cheat:** Simple = fixed step, cooldown, crude. Step = adjustment scales with breach size. Target tracking = hold metric at value (use for steady state). Predictive = ML forecast from history. Scheduled = cron for known spikes.

**Termination order (default policy):** balance AZs → oldest launch template → closest to billing hour. Custom policies: `NewestInstance`, `OldestInstance`, `OldestLaunchTemplate`, etc.

## Metrics to watch

- ALB: `RequestCount`, `TargetResponseTime`, `HTTPCode_Target_5XX`, `HealthyHostCount`, `UnHealthyHostCount`, `RejectedConnectionCount`
- ASG: `GroupInServiceInstances`, `GroupDesiredCapacity`, `GroupPendingInstances` (via CloudWatch ASG metrics)

## Interview one-liners

- "ALB routes at layer 7; NLB passes packets at layer 4 and keeps the client IP."
- "Health check grace period stops the ASG from killing instances that are still booting."
- "Slow scale-out is alarm lag plus boot time — warm pools and pre-scaling fix it."
- "Zero-downtime deploy: instance refresh + deregistration delay, or CodeDeploy blue-green."
