# ECS & EKS — Cheat Sheet

## ECS vs EKS at a glance

| | ECS | EKS |
|---|---|---|
| Orchestrator | AWS-native, no control plane to manage | Managed K8s control plane (flat hourly fee) |
| Learning curve | Docker + AWS only | Must know Kubernetes |
| Portability | AWS-only | Same manifests anywhere |
| Best for | Small teams, ship fast | Platform teams, K8s ecosystem |

## ECS building blocks

- **Task definition** — immutable blueprint: image, CPU, memory, ports, env, logging. Every change = new revision.
- **Task** — a running instance of a task definition (like a pod).
- **Service** — keeps N tasks running, does rolling updates.
- **Cluster** — logical grouping; capacity via FARGATE, FARGATE_SPOT, or EC2.

## Launch types

| | Fargate | Fargate Spot | EC2 |
|---|---|---|---|
| Nodes | None (serverless) | None | You manage container instances |
| Pricing | Per vCPU-sec + GB-sec | Up to 70% off, can be reclaimed | Instance pricing, spot/GPU available |
| Use when | Variable load, small teams | Fault-tolerant batch | Steady high utilization |

## Two roles — don't mix them up

- **Execution role** — ECS pulls your image and ships logs (needs ECR + CloudWatch perms).
- **Task role** — your application code's AWS permissions. This is the one your app assumes.

## ECS service deployments

```bash
# Rolling update with rollback protection
aws ecs update-service --cluster prod --service web \
  --task-definition web:12 \
  --deployment-configuration "minimumHealthyPercent=50,maximumPercent=200" \
  --deployment-circuit-breaker enable=true,rollback=true

# First thing to check when a task dies
aws ecs describe-tasks --cluster prod --tasks <task-arn> \
  --query 'tasks[0].stoppedReason'

# Fargate: only valid CPU/memory pairs (256/.5GB, 512/1GB, 1024/2GB, 2048/4GB, 4096/8-30GB...)
```

## EKS essentials

- AWS manages the **control plane** (API server, etcd, scheduler) across 3 AZs — you own nodes, networking, IAM.
- **Upgrades are manual**: control plane first, then nodes (respect version-skew rules).
- **Nodes**: managed node groups (default) > self-managed (special AMIs/GPU) > Fargate (no nodes, no daemonsets, no privileged).

## IRSA (pods get IAM via OIDC)

```bash
eksctl utils associate-iam-oidc-provider --cluster prod --approve
eksctl create iamserviceaccount --cluster prod --name app --namespace default \
  --attach-policy-arn <policy-arn> --approve
# Annotates the SA: eks.amazonaws.com/role-arn — never reuse node instance roles for pods
```

## Networking quick facts

- ECS tasks on `awsvpc` mode each get their own ENI + security group.
- EKS: services resolve via `svc.cluster.local` DNS automatically.
- ECS has no built-in DNS — use **Cloud Map** or an **ALB/NLB** in front.
- ALB → ECS target groups supports IP target type (required for Fargate).

## Numbers interviewers like

- EKS control plane: flat hourly fee per cluster — fewer, bigger clusters.
- ECS circuit breaker rollback: ~2 min vs manual rollback during an outage.
- Fargate Spot: up to 70% cheaper, tasks can be interrupted with 2-min warning.
