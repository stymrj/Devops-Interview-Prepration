# AWS EKS Production Deep Dive Interview Preparation Guide

*How to Answer AWS EKS Production Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [EKS Fundamentals](#eks-fundamentals)
2. [Nodes, IAM & Networking](#nodes-iam--networking)
3. [Operating EKS in Production](#operating-eks-in-production)

---

## EKS Fundamentals

### Q1: What is EKS, and why would you pick a managed control plane over self-managed Kubernetes?

**How to Answer:**

"I'd describe EKS as AWS running the Kubernetes control plane for you — the API server, etcd, and scheduler — while you own the worker nodes and everything running on them. The win is that you stop babysitting etcd backups, control-plane upgrades, and API-server availability, and you get a multi-AZ control plane with an SLA. I'd still pick self-managed Kubernetes on EC2 if I needed a custom control-plane build or a Kubernetes version AWS doesn't offer yet, but that's rare. For most teams, the trade is simple: you pay per cluster per hour, and in return the hardest part of Kubernetes to operate well just disappears."

**Key Point:** "EKS = AWS runs the control plane, you run the workloads — you trade a per-hour cluster fee for never babysitting etcd or the API server again."

---

### Q2: How does EKS architecture actually fit together?

**How to Answer:**

"Think of it in two halves. The control plane lives in an AWS-managed VPC that you never see, and it talks to your worker nodes through elastic network interfaces placed in your VPC subnets. Your nodes, pods, and load balancers live in your VPC, so your traffic never leaves your network boundary. The API server endpoint can be public, private, or both, and I always recommend private-only for production with a bastion or VPN path for admins. If an interviewer pushes deeper, the key detail is that the control plane scales across AZs automatically, but your nodes still need at least two AZs to survive an AZ failure."

```bash
# See the cluster and its endpoint access mode
aws eks describe-cluster --name prod --query 'cluster.{endpoint:endpoint,public:resourcesVpcConfig.endpointPublicAccess,private:resourcesVpcConfig.endpointPrivateAccess}'
```

**Key Point:** "Control plane in AWS's VPC, nodes in yours, connected by ENIs — and your AZ spread on the node side is still your responsibility."

---

### Q3: How do you choose between managed node groups, self-managed nodes, and Fargate?

**How to Answer:**

"Managed node groups are my default — AWS handles node patching, draining, and rolling updates, and I just pick instance types and scaling config. Self-managed nodes, usually via Karpenter or an autoscaling group I control, win when I need custom AMIs, special bootstrapping, or very fast scale-up that managed groups handle more slowly. Fargate is for the opposite end: bursty or spiky workloads where I don't want to think about nodes at all, and I'm happy paying a premium per pod. The trap is mixing them without a reason — pick one model per workload type, and let cost and startup latency decide, not fashion."

**Key Point:** "Managed node groups by default, self-managed or Karpenter when you need control or speed, Fargate when you want zero node thinking and accept the premium."

---

### Q4: How do you design node groups for mixed workloads without blowing the budget?

**How to Answer:**

"I split node groups by workload shape, not by team. Stateless, fault-tolerant services go on Spot with a diverse set of instance types, because diversification is what actually protects you from Spot interruptions. Stateful or latency-sensitive workloads stay on On-Demand, and I use taints and tolerations so the wrong pods can't land on the wrong nodes. I also label nodes by workload class and use node selectors in the deployments, so scheduling intent is visible in Git, not tribal knowledge. The number that matters in the interview is your On-Demand base: enough capacity to survive losing all Spot at once, and no more."

```yaml
# A Spot node group that tolerates interruption-prone workloads only
taints:
  - key: workload
    value: spot-tolerant
    effect: NoSchedule
```

**Key Point:** "Split by workload shape, diversify Spot instance types, taint the cheap capacity — and size On-Demand to survive losing all Spot at once."

---

## Nodes, IAM & Networking

### Q5: How does IAM map to Kubernetes RBAC on EKS?

**How to Answer:**

"EKS has two mechanisms, and knowing which one you're on is half the answer. The classic way is the aws-auth ConfigMap, which maps IAM roles and users to Kubernetes groups and usernames. The modern way is EKS access entries, which attach IAM principals directly to the cluster with an access policy like cluster-admin or namespace-scoped edit, no ConfigMap editing required. I prefer access entries for anything new because they're auditable in the AWS API and you can't break the cluster by mangling a ConfigMap. Either way, the rule I never break is: humans get short-lived assumed roles, never long-lived access keys baked into kubeconfig."

**Key Point:** "aws-auth ConfigMap is the classic mapping; EKS access entries are the modern, auditable replacement — humans always use short-lived roles, never static keys."

---

### Q6: How does a pod get AWS permissions — IRSA or EKS Pod Identity?

**How to Answer:**

"Both solve the same problem: giving a pod its own IAM role instead of sharing the node's role. IRSA, the older mechanism, uses an OIDC provider on the cluster and a trust policy that says 'this service account in this namespace may assume this role,' with tokens projected into the pod. EKS Pod Identity is the newer, simpler version — an agent on the node hands out credentials for the role associated with the pod's service account, with no OIDC provider to manage. I'd pick Pod Identity for new clusters because setup is one association instead of OIDC plumbing, but I still explain IRSA in interviews because most existing clusters run it. The security point that matters is the same either way: least privilege per service account, never the node role."

```yaml
# Pod Identity association: this service account gets this IAM role
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-api
  namespace: prod
```

**Key Point:** "IRSA = OIDC trust per service account; Pod Identity = the same idea with less plumbing — either way, least privilege per service account, never the node role."

---

### Q7: How do you design EKS networking so you don't run out of pod IPs?

**How to Answer:**

"This is the question that catches people who've only run toy clusters, because the default VPC CNI assigns a real VPC IP to every pod. On a small subnet, you can exhaust IPs long before you exhaust CPU. My fixes, in order: size subnets generously from day one, use secondary CIDR blocks for pods, or switch to prefix delegation so each node gets a /28 block instead of individual IPs. I also put nodes in private subnets with a NAT gateway per AZ, so a single NAT failure doesn't take out a whole AZ's egress. And I keep the cluster's security groups tight — the control-plane security group should only accept traffic from node groups and admin ranges, never 0.0.0.0/0."

**Key Point:** "Default VPC CNI eats a VPC IP per pod — size subnets big, use secondary CIDRs or prefix delegation, and keep nodes private behind per-AZ NAT."

---

### Q8: How do you lock down EKS networking beyond the basics?

**How to Answer:**

"Once the VPC layout is right, I layer three controls. Kubernetes NetworkPolicies, enforced by the CNI or a policy engine, give me default-deny between namespaces so a compromised frontend can't wander into the database namespace. Security groups for pods let me attach a real AWS security group to a workload, which is how I control egress to RDS or ElastiCache at the AWS level. And for traffic entering the cluster, I terminate TLS at the ALB with AWS Certificate Manager and let the ingress controller route by host and path. The interview-ready summary is: NetworkPolicies for pod-to-pod, security groups for pod-to-AWS, ALB plus ACM for the front door."

**Key Point:** "Default-deny NetworkPolicies between namespaces, security groups for pods for AWS-side access, ALB plus ACM at the front door."

---

## Operating EKS in Production

### Q9: How do you run EKS upgrades without taking production down?

**How to Answer:**

"I upgrade in a strict order and never skip a step: control plane first, then add-ons, then nodes, then workloads that pin a Kubernetes version. Before touching anything I check the deprecation list for the target version — removed APIs are what actually break upgrades, not the upgrade itself. Nodes roll one at a time with PodDisruptionBudgets respected, and I drain with a generous timeout so stateful pods finish cleanly. I also keep one non-production cluster a version ahead, so upgrade surprises land in staging first. If someone asks about rollback, I'm honest: you can't roll the control plane back, which is exactly why the staging-first order matters."

```bash
# Check for deprecated APIs still in use before upgrading
kubectl get ingresses,deployments --all-namespaces -o json | grep -i "extensions/v1beta1" || echo "No deprecated APIs found"
```

**Key Point:** "Control plane, then add-ons, then nodes, in that order — check removed APIs first, because you can't roll a control-plane upgrade back."

---

### Q10: How do you manage EKS add-ons in production?

**How to Answer:**

"I treat add-ons as part of the platform, not afterthoughts — CoreDNS, kube-proxy, VPC CNI, EBS CSI driver, and the AWS Load Balancer Controller all get version-pinned and upgraded deliberately. The EKS add-on mechanism is my preference over raw Helm for the core ones, because AWS validates version compatibility with the cluster version. The classic production bug is the EBS CSI driver: if its IAM role is wrong, volumes fail to attach and pods sit pending with a confusing error. So my checklist for every add-on is the same: pinned version, correct IAM role via Pod Identity or IRSA, resource requests set, and an alert on its pods not being ready."

**Key Point:** "Pin every add-on version, validate it against the cluster version, and check its IAM role first when volumes or load balancers misbehave."

---

### Q11: How do you observe EKS and triage the classic production failures?

**How to Answer:**

"I start with control-plane logs in CloudWatch — api, audit, authenticator, and scheduler — because most 'the cluster is broken' stories are actually authentication or admission failures visible there. For workloads, Container Insights or Prometheus plus Grafana gives me node pressure, pod restarts, and pending pods in one view. Then I triage by symptom: pods pending usually means no node capacity or a failed volume attach; nodes not joining is almost always IAM or networking — the node role, the aws-auth mapping, or a security group blocking the API endpoint; API throttling means something is hammering the API server, usually a controller in a hot loop. The habit interviewers want to hear is: logs first, then the specific subsystem, never random kubectl commands."

**Key Point:** "Control-plane logs in CloudWatch first, then triage by symptom — pending pods, nodes not joining, and API throttling each point at a different subsystem."

---

### Q12: How do you control EKS cost without hurting reliability?

**How to Answer:**

"My biggest wins are boring, which is the point. Right-sizing requests matters more than any discount, because the scheduler and the autoscaler both trust your requests — over-request and you pay for idle capacity forever. Then I use Spot for stateless workloads, Karpenter or Cluster Autoscaler to remove idle nodes quickly, and I delete orphaned load balancers and EBS volumes left behind by deleted services. I also watch the per-cluster fee: one cluster per team feels tidy but multiplies the hourly fee, so I consolidate teams onto shared clusters with namespaces and RBAC unless there's a real isolation requirement. The rule I close with is: cut waste before cutting redundancy — never save money by dropping to one AZ."

**Key Point:** "Right-size requests first, Spot plus fast scale-down second, consolidate clusters third — and never save money by dropping to a single AZ."

---
