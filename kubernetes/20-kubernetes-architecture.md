# Kubernetes Architecture Interview Preparation Guide

*How to Answer Kubernetes Architecture Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Kubernetes at a Glance](#kubernetes-at-a-glance)
2. [Control Plane Components](#control-plane-components)
3. [Worker Node Components](#worker-node-components)
4. [etcd and the API Server](#etcd-and-the-api-server)
5. [Scheduling and Resource Management](#scheduling-and-resource-management)
6. [Networking and Service Communication](#networking-and-service-communication)
7. [Common Interview Traps](#common-interview-traps)

---

## Kubernetes at a Glance

### Q1: What problem does Kubernetes solve that Docker alone can't?

**How to Answer:**

"Docker runs containers on one machine, but the moment you have more than one host you're manually deciding where things run, what happens when a box dies, and how services find each other. Kubernetes is a scheduler and a control loop on top of containers: I declare the desired state — 'three replicas of this app, expose it on port 80' — and it continuously drives the cluster toward that state. If a node dies, it reschedules the pods somewhere else without me paginating at 3 AM. So Docker is the runtime, Kubernetes is the operating system for a fleet of machines."

**Key Point:** "Docker runs a container; Kubernetes runs your whole fleet — it declares desired state and self-heals when reality drifts."

---

### Q2: Give me the 30-second tour of a Kubernetes cluster's architecture.

**How to Answer:**

"Two halves: the control plane and the worker nodes. The control plane is the brain — the API server is the single front door, etcd is the single source of truth, the scheduler picks nodes for pods, and controller managers run the reconciliation loops that fix drift. The worker nodes do the actual work — kubelet talks to the API server, the container runtime runs the containers, and kube-proxy wires up the networking. Nothing talks to the nodes directly; everything goes through the API server, and etcd backs up what the API server promised. That's why losing etcd or the API server is a cluster-down event."

**Key Point:** "Control plane is the brain — API server, etcd, scheduler, controllers; worker nodes are the muscle — kubelet, runtime, kube-proxy."

---

## Control Plane Components

### Q3: What are the control plane components, and what does each one actually do?

**How to Answer:**

"Four pieces. The kube-apiserver is the only component users and everything else talks to — it validates requests, handles auth, and is the only thing allowed to write to etcd. etcd is a distributed key-value store holding the entire cluster state; lose quorum and your control plane is brain-dead. The kube-scheduler is pure logic — it watches for unscheduled pods and binds them to nodes based on resources, affinity, and taints. And the kube-controller-manager runs dozens of small control loops — the node controller, the replication controller, the endpoints controller — each watching state and fixing drift. One design rule to remember: every other component only talks to the API server, never directly to each other."

**Key Point:** "API server is the front door and the only writer to etcd; scheduler binds pods; controllers fix drift; everything talks through the API server."

---

### Q4: How is the cloud-controller-manager different from the kube-controller-manager?

**How to Answer:**

"In a cloud cluster like EKS or GKE, the cloud-controller-manager runs the loops that touch provider APIs — the node controller that calls EC2 when a node disappears, the service controller that provisions an AWS load balancer when you create a LoadBalancer service, the route controller that wires pod CIDRs. Keeping it separate means the core Kubernetes controllers don't need cloud credentials, and you can swap the cloud without touching core logic. In interviews I just say: kube-controller-manager fixes Kubernetes state, cloud-controller-manager talks to the cloud provider. If someone asks why it's a separate binary — so upstream Kubernetes stays cloud-agnostic."

**Key Point:** "Kube-controller-manager reconciles Kubernetes state; cloud-controller-manager is the adapter that calls AWS/GCP APIs for nodes, load balancers, and routes."

---

## Worker Node Components

### Q5: What's on a worker node, and what does the kubelet actually do?

**How to Answer:**

"Three pieces: kubelet, the container runtime, and kube-proxy. The kubelet is the node's agent — it registers the node with the API server, watches for pods assigned to it, and makes sure the described containers are running. If a pod dies, the kubelet restarts it per the restart policy; if the node dies, it's the control plane — not the kubelet — that notices the heartbeats stopped. The container runtime — containerd these days — actually pulls images and runs containers. And kube-proxy programs the networking rules so Services work. The trap in interviews: people say the kubelet schedules pods. It doesn't — the scheduler assigns, the kubelet executes."

```bash
# kubelet status is the first thing I check on a sick node
systemctl status kubelet
journalctl -u kubelet --since "30 min ago" | tail -50
```

**Key Point:** "Kubelet is the node's agent — scheduler assigns pods, kubelet executes; it never schedules anything itself."

---

### Q6: What's the container runtime, and why did Kubernetes drop dockershim?

**How to Answer:**

"The runtime is what actually creates containers from images — today that's almost always containerd, sometimes CRI-O. Kubernetes talks to it through the CRI, a standard interface, so any runtime can plug in. The old dockershim was a special-case bridge just for Docker as a runtime, and it became a maintenance burden while adding nothing. So Kubernetes removed it in 1.24 and standardized on CRI. Interviewers love asking if you can still use Docker — yes, as a build tool and it can still pull images, but the kubelet talks to containerd through CRI, not to the Docker daemon. Docker images work fine because both speak the OCI format."

**Key Point:** "Kubernetes removed dockershim to standardize on CRI; containerd runs the containers, Docker images still work because everything speaks OCI."

---

## etcd and the API Server

### Q7: Why is etcd the most critical part of the cluster?

**How to Answer:**

"Because etcd holds every object in the cluster — pods, deployments, secrets, the whole desired state. Every other component is stateless: kill the scheduler and unscheduled pods just wait; kill etcd quorum and the control plane can't read or write anything. That's why etcd runs as an odd-numbered quorum — 3 or 5 members — so it survives losing a member, and why I always back it up separately from the cloud provider's snapshots. The interview trap: people say 'just scale etcd for performance.' No — adding members past 5 slows consensus down, not up. Three members is the sweet spot for almost everyone."

```bash
# verify etcd health and take a backup
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

**Key Point:** "etcd is the single source of truth — every other control plane component is stateless, so protect quorum and back it up."

---

### Q8: How do components and users authenticate with the API server?

**How to Answer:**

"Everything presents credentials to the API server — the only component that verifies identity. Human users usually get a client certificate or a token, often issued through an OIDC provider like Google or Okta in managed clusters. Every in-cluster component — kubelet, scheduler, controller managers — uses TLS client certs issued by the cluster CA. Then authorization happens: RBAC roles and bindings decide what that identity can do. The neat part I always mention: each pod gets a ServiceAccount token automatically mounted, so pods can talk to the API server as themselves, and I restrict those with RBAC so a compromised pod can't read the whole cluster."

**Key Point:** "API server is the only identity checker — TLS certs for components, tokens or OIDC for users, and RBAC decides what each identity may touch."

---

## Scheduling and Resource Management

### Q9: Walk me through what happens when I run kubectl apply on a Deployment.

**How to Answer:**

"kubectl sends the YAML to the API server, which validates it, stores it in etcd, and returns. The deployment controller sees the new Deployment and creates a ReplicaSet; the ReplicaSet controller sees it wants three replicas and creates three pod objects — all still unscheduled. The scheduler watches for pods with no node, picks the best node based on resources and constraints, and writes the binding. The kubelet on that node sees its assigned pod, pulls the images, and starts the containers, reporting status back to the API server. Meanwhile the endpoints controller wires up the Service. Nothing in that chain is synchronous — it's all small loops watching etcd and nudging reality toward the declared state."

**Key Point:** "kubectl writes desired state to etcd; controllers fan it out into ReplicaSets and pods; scheduler binds; kubelets execute — all async control loops."

---

### Q10: How does the scheduler pick a node for a pod?

**How to Answer:**

"It filters, then scores. First it throws out nodes that can't run the pod — not enough free CPU or memory after requests are accounted for, wrong OS or architecture, taints the pod doesn't tolerate, a node selector or affinity that doesn't match. Then it scores the survivors: spreading pods across zones, bin-packing onto already-warm nodes, preferring nodes with the image already cached. Highest score wins, and the scheduler writes the binding to the API server. Two things trip people up: the scheduler only looks at requests, not limits, when checking capacity — and it never evicts or moves existing pods to make room, it just picks among nodes that fit."

**Key Point:** "Scheduler filters out unfit nodes, scores the rest, binds the winner — it uses requests, not limits, and never moves running pods to make room."

---

## Networking and Service Communication

### Q11: How does a Service actually route traffic to pods?

**How to Answer:**

"A Service is just a stable virtual IP plus a set of endpoints — the current matching pods — maintained by the endpoints controller. Kube-proxy watches Services and endpoints on every node and programs the routing rules: in iptables mode it writes NAT rules that load-balance to pod IPs, in IPVS mode it programs the kernel's load balancer. So when my app calls http://backend:80, the packet hits the Service's cluster IP and gets DNAT-ed to a real pod. CoreDNS resolves the service name to that cluster IP in the first place. That's also why a Service survives pods dying and getting new IPs — the endpoints list updates, the rules get reprogrammed, and clients never notice."

```bash
# see the actual routing kube-proxy programmed for a service
kubectl get endpoints my-app
iptables -t nat -L KUBE-SERVICES -n | head -20
```

**Key Point:** "Service is a stable virtual IP over a dynamic pod list; kube-proxy programs NAT or IPVS rules on every node so traffic follows the pods."

---

## Common Interview Traps

### Q12: What's the difference between requests and limits, and why does getting them wrong cause outages?

**How to Answer:**

"Requests are what the scheduler uses to place the pod — it's a reservation of node capacity. Limits are the hard ceiling the kubelet enforces at runtime. Set requests too low and pods pile onto a node until it runs out of memory and the OOM killer starts murdering containers — that's the classic noisy-neighbor outage. Set limits too low and your own pod gets throttled on CPU or OOM-killed on memory. My rule of thumb: requests equal what the app actually uses at steady state, limits have headroom above the p99. And never set CPU limits equal to requests for bursty apps — throttling is worse than sharing."

**Key Point:** "Requests drive scheduling, limits cap runtime — under-set requests and nodes overcommit until the OOM killer fires."

---

### Q13: How do you debug when kubectl suddenly can't reach the cluster?

**How to Answer:**

"I start top-down at the front door. First kubectl with -v to see if it's a cert or network issue — expired kubeconfig certs are embarrassingly common. Then I check whether the API server process is alive on the control plane nodes, then etcd health — a lost quorum means the API server can't serve anything. If those are fine, I look at the network path: firewall rules, load balancer health checks on the control plane. The insight interviewers want: kubectl talks only to the API server, so 'connection refused' is almost never a worker node problem — it's the control plane or the network in front of it."

**Key Point:** "kubectl only talks to the API server — connection failures mean API server, etcd, certs, or the network in front of the control plane, not worker nodes."

---

*Built for interview prep, one deep-dive at a time.*
