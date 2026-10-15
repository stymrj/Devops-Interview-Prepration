# Hands-On Lab: ECS & EKS on AWS

8 exercises. Goal: run a real service on ECS Fargate, break and fix it, then stand up an EKS cluster and wire IAM the right way.

**Prerequisites:** AWS CLI configured, a VPC with 2 public subnets in different AZs, `eksctl` and `kubectl` installed for the EKS exercises.

---

## Exercise 1: Create an ECS cluster

**Goal:** Stand up a Fargate-ready cluster.

```bash
aws ecs create-cluster --cluster-name lab-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1
```

**Expected output:** Cluster `lab-cluster` in ACTIVE state.
**Why it matters:** Everything in ECS lives inside a cluster — this is the starting point of every ECS setup.

---

## Exercise 2: Register a task definition

**Goal:** Register an immutable task definition for nginx.

```bash
cat > task-def.json <<'EOF'
{
  "family": "lab-nginx",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "nginx",
    "image": "nginx:alpine",
    "portMappings": [{"containerPort": 80}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/lab-nginx",
        "awslogs-region": "ap-south-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file://task-def.json
```

**Expected output:** Task definition `lab-nginx:1`.
**Why it matters:** Every deployment is a new revision — rollbacks are just pointing back at an older one.

---

## Exercise 3: Run it as a service behind an ALB

**Goal:** Create a service with 2 tasks on Fargate.

```bash
aws ecs create-service --cluster lab-cluster --service-name lab-web \
  --task-definition lab-nginx:1 --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-a,subnet-b],assignPublicIp=ENABLED}"
```

**Expected output:** Service `lab-web` with 2 RUNNING tasks.
**Why it matters:** This is the smallest real ECS deployment — the pattern every interview question builds on.

---

## Exercise 4: Break it, then read the stoppedReason

**Goal:** Register a bad revision (wrong image tag) and diagnose the failure.

```bash
# Register a broken revision
sed 's/nginx:alpine/nginx:does-not-exist/' task-def.json > bad-task-def.json
aws ecs register-task-definition --cli-input-json file://bad-task-def.json

# Deploy it, then inspect the failure
aws ecs update-service --cluster lab-cluster --service lab-web \
  --task-definition lab-nginx:2

aws ecs describe-tasks --cluster lab-cluster \
  --tasks $(aws ecs list-tasks --cluster lab-cluster --desired-status STOPPED \
    --query 'taskArns[0]' --output text) \
  --query 'tasks[0].stoppedReason'
```

**Expected output:** `stoppedReason` like `CannotPullContainerError: ... not found`.
**Why it matters:** In real incidents, `stoppedReason` tells you the answer in seconds — this is the first command I run.

---

## Exercise 5: Rolling update with circuit breaker

**Goal:** Deploy a good revision with automatic rollback protection.

```bash
aws ecs update-service --cluster lab-cluster --service lab-web \
  --task-definition lab-nginx:1 --desired-count 2 \
  --deployment-configuration "minimumHealthyPercent=50,maximumPercent=200" \
  --deployment-circuit-breaker enable=true,rollback=true
```

**Expected output:** A new deployment that ECS can auto-roll-back if tasks fail health checks.
**Why it matters:** Circuit breaker turns a bad deploy into a 2-minute automatic rollback instead of an outage.

---

## Exercise 6: Stand up an EKS cluster

**Goal:** Create a 2-node EKS cluster with eksctl.

```bash
eksctl create cluster --name lab-eks --region ap-south-1 \
  --node-type t3.medium --nodes 2 --nodes-min 1 --nodes-max 3 \
  --managed
```

**Expected output:** `kubectl get nodes` shows 2 Ready managed nodes.
**Why it matters:** Managed node groups are the default production choice — this is the command behind that answer.

---

## Exercise 7: IRSA the right way

**Goal:** Give a pod S3 read access without touching node roles.

```bash
# Associate the OIDC provider
eksctl utils associate-iam-oidc-provider --cluster lab-eks --approve

# Create a service account bound to an IAM role
eksctl create iamserviceaccount --cluster lab-eks \
  --name s3-reader --namespace default \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

**Expected output:** A `s3-reader` service account annotated with `eks.amazonaws.com/role-arn`.
**Why it matters:** This is least-privilege IAM on EKS — the thing interviewers check when you say "I secure my pods."

---

## Exercise 8: Service discovery showdown

**Goal:** Compare discovery in both worlds.

```bash
# EKS: a ClusterIP Service is discoverable by DNS immediately
kubectl create deployment demo --image=nginx:alpine
kubectl expose deployment demo --port=80
kubectl run -it --rm dns-test --image=busybox -- nslookup demo

# ECS: list services — there is no built-in DNS; you'd add Cloud Map or an ALB
aws ecs list-services --cluster lab-cluster --query 'serviceArns'
```

**Expected output:** `nslookup demo` resolves inside the cluster; ECS returns just a service ARN with no DNS.
**Why it matters:** This contrast is a favorite interview follow-up — "how do containers find each other?"

---

*Built for interview prep, one deep-dive at a time.*
