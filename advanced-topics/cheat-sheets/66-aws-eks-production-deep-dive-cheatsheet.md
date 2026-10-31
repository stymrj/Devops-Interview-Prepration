# AWS EKS Production Deep Dive — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| EKS | AWS runs the control plane; you run nodes and workloads |
| Control plane | API server + etcd + scheduler, multi-AZ, in AWS's VPC |
| Managed node group | AWS patches, drains, and rolls nodes for you — the default |
| Karpenter | Fast, just-in-time node provisioning for spiky workloads |
| Fargate | Serverless pods, no nodes to manage, premium price |
| aws-auth | Classic ConfigMap mapping IAM principals to k8s groups |
| EKS access entries | Modern, API-auditable IAM-to-cluster mapping |
| IRSA | OIDC trust: service account assumes its own IAM role |
| EKS Pod Identity | Same idea as IRSA, simpler — no OIDC provider to manage |
| VPC CNI | Every pod gets a real VPC IP — watch IP exhaustion |
| Prefix delegation | /28 blocks per node instead of one IP per pod |
| NetworkPolicy | Pod-to-pod firewall, default-deny between namespaces |
| Security groups for pods | Real AWS security group attached to a workload |
| EKS add-ons | Version-pinned platform components (CoreDNS, CNI, CSI) |
| Control-plane logs | api, audit, authenticator, scheduler — in CloudWatch |

## Node strategy at a glance

| Model | Best for | Trap |
|---|---|---|
| Managed node groups | Default, steady workloads | Slower scale-up than Karpenter |
| Self-managed / Karpenter | Custom AMIs, fast scale-up | You own node lifecycle again |
| Fargate | Bursty, spiky, zero node ops | Premium per-pod price |
| Spot | Stateless, fault-tolerant | No diversification = mass interruption |
| On-Demand base | Stateful, latency-sensitive | Paying for it when Spot would do |

## IAM mapping: which mechanism, when

| Mechanism | Setup | Prefer when |
|---|---|---|
| aws-auth ConfigMap | Edit a ConfigMap | Legacy clusters already on it |
| EKS access entries | API call per principal | New clusters, auditability matters |
| IRSA | OIDC provider + trust policy | Existing clusters, fine-grained roles |
| Pod Identity | One association per service account | New clusters, least plumbing |

## Networking decisions

| Decision | Production choice | Why |
|---|---|---|
| Node subnets | Private, ≥ 2 AZs | No direct internet exposure; AZ survival |
| NAT | One per AZ | Single NAT failure must not kill an AZ |
| Pod IPs | Secondary CIDR or prefix delegation | Default CNI exhausts small subnets |
| Ingress | ALB + ACM | TLS termination and host/path routing |
| Pod-to-pod | Default-deny NetworkPolicy | Stops lateral movement |
| Pod-to-AWS | Security groups for pods | AWS-level egress control |

## Upgrade order (never reorder)

1. Check removed APIs in the target version
2. Control plane
3. Add-ons (CoreDNS, kube-proxy, VPC CNI, EBS CSI)
4. Node groups, one at a time, respecting PodDisruptionBudgets
5. Workloads pinned to a Kubernetes version

There is no control-plane rollback — staging one version ahead is the safety net.

## Triage by symptom

| Symptom | Check first |
|---|---|
| Pods pending | Node capacity, volume attach, CSI driver IAM role |
| Nodes not joining | Node IAM role, aws-auth / access entries, security groups to API endpoint |
| API throttling | Controller in a hot loop hammering the API server |
| Volume fails to attach | EBS CSI driver role and add-on health |
| Pods can't reach AWS services | Pod Identity / IRSA association, security group egress |

## Cost levers, in order

1. Right-size CPU/memory requests — the scheduler trusts them
2. Spot for stateless workloads, diversified instance types
3. Fast scale-down (Karpenter / Cluster Autoscaler)
4. Delete orphaned load balancers and EBS volumes
5. Consolidate teams onto shared clusters (namespaces + RBAC) instead of one cluster per team
6. Never: dropping to a single AZ to save money
