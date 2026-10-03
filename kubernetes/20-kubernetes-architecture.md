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

PART2_PLACEHOLDER
