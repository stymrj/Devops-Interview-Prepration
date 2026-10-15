# ECS & EKS on AWS Interview Preparation Guide

*How to Answer ECS & EKS Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [ECS vs EKS: Picking the Right One](#ecs-vs-eks-picking-the-right-one)
2. [ECS Deep Dive](#ecs-deep-dive)
3. [EKS Deep Dive](#eks-deep-dive)
4. [Networking and Service Discovery](#networking-and-service-discovery)
5. [Interview Traps and Real Scenarios](#interview-traps-and-real-scenarios)

---

## ECS vs EKS: Picking the Right One

### Q1: ECS vs EKS — how do I pick one in an interview?

**How to Answer:**

"I ask what the team already knows. If the team knows Docker but not Kubernetes, ECS gets them shipping fast because AWS hides the orchestration complexity. If the team knows Kubernetes or wants portability across clouds, EKS is the pick — same manifests everywhere. For a simple web app with a small team, I recommend ECS on Fargate. For platform teams running dozens of services with complex networking needs, EKS pays off."

**Key Point:** "Small team, ship fast: ECS. Kubernetes expertise or multi-cloud future: EKS."

---

### Q2: Why would anyone still choose ECS over EKS?

**How to Answer:**

"Because it's simpler and cheaper to run. There's no control plane to think about, the pricing is straightforward — I pay for the tasks I run — and the AWS integration feels native: CloudWatch, ALB, IAM all click together. I don't need a Kubernetes specialist on the team. The trap answer is 'ECS is legacy' — it's not. AWS keeps investing in it and plenty of production workloads run on ECS."

**Key Point:** "ECS wins on simplicity, native AWS integration, and not needing Kubernetes expertise."

---

### Q3: When is EKS the obvious choice?

**How to Answer:**

"When the org already runs Kubernetes somewhere — on-prem or another cloud — because then skills and manifests transfer directly. Also when I need the Kubernetes ecosystem: Helm, service mesh, operators, custom controllers. And when hiring matters: finding Kubernetes engineers is easier than finding deep ECS specialists. If the interviewer asks about vendor lock-in, I say EKS reduces it because my workloads are plain Kubernetes."

**Key Point:** "Pick EKS when you need the Kubernetes ecosystem, existing K8s skills, or portability."

---

## ECS Deep Dive

### Q4: What is a task definition, and what trips people up?

**How to Answer:**

"A task definition is the blueprint for my containers: image, CPU, memory, ports, environment variables, and logging config. The gotcha is that it's immutable — every change creates a new revision, and my service points at a specific revision. The interview trap: CPU and memory are set at the task level AND per container, and the task-level values cap the total. I always mention Fargate requires exact CPU/memory combos from its supported list."

```json
{
  "family": "web-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024"
}
```

**Key Point:** "Task definitions are immutable blueprints — every change is a new revision, and Fargate only accepts specific CPU/memory pairs."

---

### Q5: Fargate vs EC2 launch type — how do I compare them?

**How to Answer:**

"Fargate is serverless containers: no instances to manage, I pay per vCPU-second and GB-second, and it scales fast. EC2 launch type means I run container instances myself — more control, cheaper at steady high utilization, and I can use spot or GPU instances. My rule of thumb: Fargate for variable workloads and small teams, EC2 launch type for steady-state workloads where bin-packing saves real money."

**Key Point:** "Fargate trades cost for zero ops; EC2 launch type saves money at steady scale and unlocks spot and GPU."

---

### Q6: How do ECS services handle deployments and rollbacks?

**How to Answer:**

"The service runs a rolling update by default: it starts new tasks with the new revision, waits for them to pass health checks, then drains the old ones. The key knobs are minimumHealthyPercent and maximumPercent — they control how aggressive the rollout is. If the new tasks keep failing health checks, the deployment fails and ECS rolls back automatically when I enable circuit breaker. I always mention circuit breaker in interviews — it turns a broken deploy into a two-minute rollback instead of an outage."

**Key Point:** "Rolling updates are governed by min/max healthy percents; circuit breaker gives automatic rollback on failed deploys."

---

### Q7: A task keeps stopping right after launch. How do I debug it?

**How to Answer:**

"First I run `aws ecs describe-tasks` and read the stoppedReason — it usually says exactly what happened, like 'CannotPullContainerError' or a health check failure. If it's an image pull issue, I check ECR permissions and the execution role. If it starts then dies, I check the CloudWatch logs — and the classic miss is that awslogs logging was never configured, so there's nothing to read. I also verify the container's health check isn't failing against the wrong port or path."

```bash
aws ecs describe-tasks --cluster prod --tasks <task-arn> \
  --query 'tasks[0].{reason:stoppedReason, code:containers[0].exitCode}'
```

**Key Point:** "stoppedReason usually names the failure; missing logs almost always means awslogs was never configured."

---

## EKS Deep Dive

### Q8: What does AWS actually manage for me on EKS?

**How to Answer:**

"AWS runs the control plane: the API server, etcd, and the scheduler, spread across three AZs. I still own the worker nodes — unless I use Fargate — plus networking, IAM, and everything running inside the cluster. The part people forget: the control plane costs a flat hourly fee per cluster, so running many tiny clusters gets expensive. I keep clusters few and use namespaces for separation."

**Key Point:** "AWS manages the multi-AZ control plane; I own nodes, networking, and everything inside — and each cluster has a flat hourly cost."

---

### Q9: Managed node groups vs Fargate vs self-managed nodes on EKS?

**How to Answer:**

"Managed node groups are the sweet spot for most teams: AWS handles patching and upgrades, I pick the instance types. Self-managed nodes give me full control — custom AMIs, special networking, GPU setups — but I own the operational burden. Fargate on EKS means no nodes at all, which is great for bursty or batch workloads but pricier at scale and slower to cold-start. In interviews I say managed node groups first, then justify the other two by exception."

**Key Point:** "Managed node groups are the default; self-managed for special hardware, Fargate for bursty workloads without nodes."

---

### Q10: How do pods get IAM permissions on EKS?

**How to Answer:**

"Through IRSA — IAM Roles for Service Accounts. I create an IAM role with a trust policy that references the cluster's OIDC provider and the Kubernetes service account name, then annotate the service account with the role ARN. Pods using that service account get temporary credentials injected. The mistake I see in interviews is reaching for node instance roles — that gives every pod on the node the same permissions, which breaks least privilege."

```yaml
metadata:
  name: payment-service
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/payment-role
```

**Key Point:** "IRSA maps a service account to an IAM role via OIDC — never share node roles across pods."

---

## Networking and Service Discovery

### Q11: How does service discovery differ between ECS and EKS?

**How to Answer:**

"In EKS it's native Kubernetes: a Service gets a stable DNS name like `api.default.svc.cluster.local` and kube-dns resolves it. In ECS I have to opt in — either Cloud Map service discovery, which registers tasks in a private DNS namespace, or put an ALB/NLB in front and let the load balancer route. The common interview question is how containers find each other, and the honest ECS answer is 'Cloud Map or a load balancer, there's no built-in DNS like Kubernetes has.'"

**Key Point:** "EKS gets Kubernetes DNS for free; ECS needs Cloud Map or a load balancer for service discovery."

---

## Interview Traps and Real Scenarios

### Q12: What are the classic ECS/EKS interview traps?

**How to Answer:**

"First: task role vs execution role on ECS. Execution role pulls the image and ships logs; task role is what my application code uses to call AWS APIs — mixing them up is the number one ECS trap. Second: EKS upgrades — they're manual, version-skew rules apply, and I upgrade the control plane before the nodes. Third: Fargate has no daemonsets and no privileged containers, so monitoring agents that need host access don't work there. I drop these three in interviews because they prove I've actually run the stuff, not just read the docs."

**Key Point:** "Execution role is for ECS itself, task role is for my app; EKS upgrades are manual and ordered; Fargate can't run privileged or daemonset workloads."

---

*Built for interview prep, one deep-dive at a time.*
