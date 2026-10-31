# AWS EKS Production Deep Dive — Hands-On Lab

Nine drills that turn EKS production answers into real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Create a production-shaped EKS cluster

**Goal:** A private-node, multi-AZ cluster you can safely experiment on.

**Commands:**

```bash
eksctl create cluster --name lab-eks --region ap-south-1 \
  --version 1.32 --nodegroup-name standard-workers \
  --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 4 \
  --managed --vpc-nat-gateway single --asg-access
aws eks update-kubeconfig --name lab-eks --region ap-south-1
kubectl get nodes
```

**Expected output:** Two Ready nodes spread across AZs, and `kubectl` talking to the cluster.

**Why it matters:** This is the cluster every later drill runs on — and the multi-AZ shape from Q2.

---

## Exercise 2: Inspect the control plane and endpoint access

**Goal:** See what AWS manages versus what you own.

**Commands:**

```bash
aws eks describe-cluster --name lab-eks --region ap-south-1 \
  --query 'cluster.{version:version,endpoint:endpoint,public:resourcesVpcConfig.endpointPublicAccess,private:resourcesVpcConfig.endpointPrivateAccess}'
kubectl cluster-info
```

**Expected output:** Cluster version, endpoint URL, and public/private access flags.

**Why it matters:** This is the control-plane versus data-plane split from Q2, in one command.

---

## Exercise 3: Give a pod its own IAM role with Pod Identity

**Goal:** A service account that can read S3, without touching the node role.

**Commands:**

```bash
# Create the IAM role and associate it with a service account
eksctl create podidentityassociation --cluster lab-eks --region ap-south-1 \
  --namespace default --service-account-name s3-reader \
  --role-name lab-s3-reader --permission-policy-arns arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
kubectl run s3-test --image=amazon/aws-cli --serviceaccount=s3-reader -- sleep 3600
kubectl exec s3-test -- aws s3 ls
```

**Expected output:** The pod lists S3 buckets; a pod without that service account cannot.

**Why it matters:** This is Q6 — least privilege per service account, not the node role.

---

## Exercise 4: Prove Spot interruption handling works

**Goal:** A deployment that survives Spot capacity being reclaimed.

**Commands:**

```bash
# Deploy with PodDisruptionBudget and topology spread
kubectl create deployment spot-web --image=nginx --replicas=3
kubectl create poddisruptionbudget spot-web-pdb --min-available=2 --selector app=spot-web
kubectl get pdb spot-web-pdb
# Simulate interruption: terminate one node and watch rescheduling
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -w
```

**Expected output:** At least two replicas stay available throughout; drained pods reschedule elsewhere.

**Why it matters:** This is the resilience argument behind the Spot design in Q4.

---

## Exercise 5: Enforce default-deny networking between namespaces

**Goal:** Two namespaces that cannot talk until you explicitly allow it.

**Commands:**

```bash
kubectl create namespace frontend && kubectl create namespace backend
cat <<'EOF' | kubectl apply -n backend -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF
kubectl run -n frontend probe --image=busybox --rm -i -- wget -qO- --timeout=3 backend-service.backend || echo "Blocked as expected"
```

**Expected output:** The probe fails while the policy is in place.

**Why it matters:** This is Q8 — default-deny is the control that stops lateral movement.

---

## Exercise 6: Upgrade the cluster safely, staging first

**Goal:** A rehearsed upgrade path with a rollback story for workloads.

**Commands:**

```bash
# Check for removed APIs before upgrading (repeat of the Q9 check)
kubectl get ingresses,deployments --all-namespaces -o json | grep -i "extensions/v1beta1" || echo "Clean"
# Upgrade control plane, then node groups, one at a time
eksctl upgrade cluster --name lab-eks --region ap-south-1 --approve
eksctl upgrade nodegroup --cluster lab-eks --region ap-south-1 --name standard-workers --approve
kubectl get nodes
```

**Expected output:** Control-plane version bumps first, nodes follow, all nodes Ready.

**Why it matters:** This is the Q9 order — control plane, add-ons, nodes — rehearsed, not improvised.

---

## Exercise 7: Install and verify the EBS CSI driver and a StorageClass

**Goal:** Dynamic volumes that actually attach, with the failure mode understood.

**Commands:**

```bash
aws eks describe-addon --cluster-name lab-eks --addon-name aws-ebs-csi-driver --region ap-south-1
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: lab-pvc }
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 1Gi } }
EOF
kubectl get pvc lab-pvc
```

**Expected output:** The PVC binds to a dynamically provisioned EBS volume.

**Why it matters:** This is the Q10 classic bug — wrong CSI IAM role, pods stuck pending — caught in a lab, not production.

---

## Exercise 8: Turn on control-plane logging and find a real event

**Goal:** CloudWatch logs you can actually triage from.

**Commands:**

```bash
aws eks update-cluster-config --name lab-eks --region ap-south-1 \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","scheduler"],"enabled":true}]}'
# Generate an auditable event, then find it
kubectl auth can-i create deployments --as system:anonymous || true
aws logs filter-log-events --log-group-name /aws/eks/lab-eks/cluster \
  --filter-pattern 'audit' --max-items 5
```

**Expected output:** Log streams appear and the denied request shows up in the audit log.

**Why it matters:** This is Q11 — logs first, then the subsystem, never random kubectl.

---

## Exercise 9: Find wasted capacity and cut it

**Goal:** A concrete cost saving you can defend in an interview.

**Commands:**

```bash
# Compare requests vs actual usage per pod
kubectl top pods --all-namespaces --sort-by=cpu
kubectl top nodes
# Find orphaned load balancers and volumes
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'
aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].VolumeId'
```

**Expected output:** At least one over-requested workload and any orphaned resources, listed by name.

**Why it matters:** This is Q12 — right-sizing and orphan cleanup are the boring wins that actually move the bill.

---
