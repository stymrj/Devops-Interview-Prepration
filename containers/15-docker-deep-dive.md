# Docker Deep Dive Interview Preparation Guide

*How to Answer Docker Architecture, Daemon & containerd Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Containers Exist](#why-containers-exist)
2. [Docker Architecture and the Daemon](#docker-architecture-and-the-daemon)
3. [Namespaces and cgroups](#namespaces-and-cgroups)
4. [Images Layers and Storage](#images-layers-and-storage)
5. [Registries Pulls and OCI](#registries-pulls-and-oci)
6. [Interview Traps and Real World Calls](#interview-traps-and-real-world-calls)

---

## Why Containers Exist

### Q1: What is a container, really — not the marketing version?

**How to Answer:**

"A container is just an isolated process running on a shared Linux kernel. There is no guest OS inside it — the 'OS' you see is a filesystem bundle with libraries and binaries, but the kernel calls go straight to the host's kernel.

That single fact explains almost everything about containers: why they start in milliseconds, why they're tiny, and why a Linux container can't run on a Windows kernel without a VM in between.

When I explain this in interviews, I say: VMs virtualize hardware, containers virtualize the operating system. One sentence, and the whole comparison falls into place."

**Key Point:** "A container is an isolated process on a shared kernel — no guest OS — which is why containers start fast, stay small, and share the host kernel."

---

### Q2: Containers vs VMs — how do you answer the comparison question without rambling?

**How to Answer:**

"I keep it structural: a VM runs a full guest OS on a hypervisor, with its own kernel, its own memory reservation, and a boot cycle measured in minutes. A container shares the host kernel and gets isolation from namespaces and limits from cgroups — it starts as fast as the process inside it.

So the tradeoff is density versus isolation. Containers give you far higher density and startup speed, which is why CI pipelines and microservices love them. VMs give you stronger isolation — separate kernels — which is why multi-tenant or untrusted workloads still land on VMs or on containers inside VMs, like AWS Fargate does.

The trap is treating it as either-or. In production they're layered: containers inside VMs is completely normal, and Kubernetes clusters on cloud VMs prove it."

**Key Point:** "VMs virtualize hardware with separate kernels; containers share one kernel via namespaces and cgroups — so containers win on density and speed, VMs on isolation, and in practice they're layered together."

---

## Docker Architecture and the Daemon

### Q3: Walk me through Docker's architecture — client, daemon, containerd, runc.

**How to Answer:**

"There are four layers and they stack cleanly. The Docker CLI is just a client — it sends REST calls over a Unix socket to the Docker daemon, dockerd. The daemon handles the Docker API, builds, networking, volumes, and the user-facing logic.

Below the daemon sits containerd, which is the actual container runtime: it manages container lifecycle, image pulls, and storage. containerd then calls a low-level OCI runtime — usually runc — to do the final step of creating the Linux process with its namespaces and cgroups.

The key insight: Docker the product is the friendly top layer; containerd and runc are the real engine. Kubernetes talks to containerd directly and skips Docker entirely, which is exactly why this split exists."

**Key Point:** "CLI talks to dockerd, dockerd delegates to containerd, containerd invokes runc — Docker is the friendly top layer, containerd and runc are the engine Kubernetes uses directly."

---

### Q4: What actually happens when you run `docker run nginx`?

**How to Answer:**

"The CLI sends a POST to the daemon's API over the Unix socket. The daemon checks if the nginx image exists locally; if not, it pulls it via containerd from the registry. Then it creates the container: allocates a filesystem from the image layers, sets up namespaces, applies cgroup limits, attaches networking, and mounts any volumes.

Then containerd spawns the process through runc — runc creates the namespaces, applies the cgroups, and execs the nginx binary as PID 1 inside its own PID namespace. From that point the container IS that process; if nginx exits, the container stops.

I like mentioning PID 1 because it leads naturally to signal handling — if your entrypoint doesn't handle SIGTERM, `docker stop` waits ten seconds then SIGKILLs, and that's a classic graceful-shutdown interview follow-up."

**Key Point:** "docker run = API call to dockerd, image check and pull via containerd, filesystem and namespace setup, then runc launches your binary as PID 1 inside its own namespaces — the container is the process."

---

### Q5: Why did Docker split out containerd and runc? What does that buy anyone?

**How to Answer:**

"Before the split, the Docker daemon did everything — it was a monolith, and if dockerd crashed or restarted, your containers died with it. Splitting containerd out as a separate daemon meant containers survive a Docker daemon restart, which is a huge deal for production stability.

The second reason is standardization. containerd was donated to the CNCF and implements the OCI runtime spec, so it became the neutral runtime everyone could build on. Kubernetes, Docker, and cloud providers all converged on containerd — Docker even removed its own shim in Kubernetes 1.24, the famous 'dockershim deprecation.'

So the split bought two things: fault isolation between the management layer and running workloads, and one shared runtime standard for the whole ecosystem."

**Key Point:** "The split separated the management layer from running workloads — containers survive a dockerd restart — and gave the ecosystem one CNCF-standard runtime that Kubernetes uses directly."

---

## Namespaces and cgroups

### Q6: What are Linux namespaces, and which ones does Docker use?

**How to Answer:**

"Namespaces are the kernel feature that gives a process its own view of the system. Docker uses six of them: PID so the container sees only its own processes, NET for its own network stack and interfaces, MNT for its own filesystem mount table, UTS for its own hostname, IPC for isolated inter-process communication, and USER for UID mapping.

The classic demo I use: run a container and check `ps aux` inside versus outside. Inside you see PID 1 as your app; outside, that same process has some host PID like 42317. Same process, two views — that's namespaces doing their job.

Interviewers love asking 'can a container see host processes?' The answer is no — its PID namespace hides them — unless you run with `--privileged` or `--pid=host`, which punches that hole deliberately."

**Key Point:** "Namespaces give each container its own view of PIDs, network, mounts, hostname, IPC, and users — same process, two views depending on which namespace you look from."

---

### Q7: What are cgroups, and how are they different from namespaces?

**How to Answer:**

"If namespaces are about what a container can see, cgroups are about what it can use. Namespaces isolate the view; cgroups limit and account for resources — CPU shares, memory caps, block I/O weight, and process counts.

The flags map directly: `--memory=512m` sets a cgroup memory limit, and when the container hits it, the kernel's OOM killer terminates processes inside the container. `--cpus=1.5` caps CPU time. Without these limits, one runaway container can starve the whole host — I've seen a memory leak in one service take down its neighbors on a shared node, which is exactly why limits aren't optional in production.

The one-liner I use: namespaces lie to the container about what's there, cgroups ration what it gets."

**Key Point:** "Namespaces isolate the view, cgroups ration the resources — memory, CPU, I/O — and without cgroup limits one container can starve the whole host."
