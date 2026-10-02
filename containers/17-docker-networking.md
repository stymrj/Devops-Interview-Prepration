# Docker Networking Interview Preparation Guide

*How to Answer Container Networking Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Network Drivers and Defaults](#network-drivers-and-defaults)
2. [User Defined Networks and DNS](#user-defined-networks-and-dns)
3. [Port Publishing and Connectivity](#port-publishing-and-connectivity)
4. [Overlay Networks and Multi Host Communication](#overlay-networks-and-multi-host-communication)
5. [Container Network Security](#container-network-security)
6. [Common Interview Traps](#common-interview-traps)

---

## Network Drivers and Defaults

### Q1: What happens to a container's networking when you run `docker run` with no `--network` flag?

**How to Answer:**

"It lands on the default bridge network, docker0, which the daemon created for you. The container gets a veth pair linking it to that bridge and an IP from Docker's private range — usually 172.17.0.0/16. Outbound traffic works fine through the host's NAT rules.

But there's no name resolution between containers on this network. The default bridge has no embedded DNS, so containers can only reach each other by IP. It exists for backwards compatibility, not for real workloads — I never run production on it."

**Key Point:** "The default bridge gives you an IP and outbound NAT but no name resolution — always create your own networks for anything real."

---

### Q2: Explain host network mode. When would you actually use it?

**How to Answer:**

"With `--network host`, the container skips Docker's virtual networking entirely and shares the host's network namespace — it sees eth0, localhost, everything. That removes the NAT overhead, so latency-sensitive things like high-throughput proxies get measurably better performance.

But you lose all isolation. There's no port mapping because the container's port IS the host's port, and a compromised container can see host traffic. I use it rarely — mostly for quick debugging or daemon-style containers like monitoring agents that need host visibility."

**Key Point:** "Host mode trades the whole network namespace for raw performance — use it deliberately, not by default."

---

### Q3: What is `--network none` used for? It sounds useless.

**How to Answer:**

"It creates a container with only the loopback interface — completely cut off from the network. That sounds pointless until you need airtight isolation: batch jobs that process sensitive data locally, or build steps that must not phone home.

I've seen it in secure CI pipelines — a build container that can't exfiltrate secrets or download rogue dependencies even if the job gets compromised. You can attach networking later, but the starting posture is zero connectivity, and that's the point."

**Key Point:** "None mode starts a container with zero network access — a deliberate isolation posture for security-sensitive batch and build workloads."

---

## User Defined Networks and DNS

### Q4: What changes when you create a network with `docker network create`?

**How to Answer:**

"You get your own bridge with Docker's embedded DNS server at 127.0.0.11 inside each container, so containers reach each other by name automatically. Spin up `db` and `api` on the same network and the api container can just hit `http://db:5432` — no links, no /etc/hosts hacks, no hardcoded IPs.

It also isolates traffic: containers on different user-defined networks can't talk to each other unless you explicitly connect them. And containers can attach to multiple networks at once — that's how you build front-end/back-end segmentation."

**Key Point:** "User-defined networks give you automatic name resolution, traffic isolation, and multi-network attachment — the default setup for anything non-trivial."

---

### Q5: How does Docker's embedded DNS actually work?

**How to Answer:**

"Every container on a user-defined network resolves against 127.0.0.11 — the daemon's embedded DNS server listening inside the container's own namespace. When you query a container name, the daemon answers from its live registry of that network's containers, so it's always consistent with actual state.

If the name isn't a container on the network, it forwards to the host's configured DNS servers. That's why you can resolve both fellow containers and google.com from inside. And a container on multiple networks gets answers scoped to each one."

**Key Point:** "The embedded DNS at 127.0.0.11 resolves container names from the daemon's live state and forwards everything else to the host's resolvers."

---

### Q6: What's the difference between EXPOSE in a Dockerfile and `-p` at runtime?

**How to Answer:**

"EXPOSE is documentation plus metadata — it declares which ports the image intends to use, and it shows up in `docker inspect`. But it publishes nothing. An EXPOSEd port is still unreachable from the host.

`-p` is the actual action. It creates the NAT rule mapping a host port to a container port, like `-p 8080:80`. Without it, your app listens fine inside its own namespace and nobody outside can reach it. EXPOSE tells people how to run your image; `-p` actually opens the door."

```dockerfile
EXPOSE 8080   # documents intent — does not publish anything
```

**Key Point:** "EXPOSE documents intent; `-p` creates the NAT rule — only published ports are reachable from outside."

---

## Port Publishing and Connectivity

### Q7: Walk me through what happens when you run `docker run -p 8080:80 nginx`.

**How to Answer:**

"The daemon creates an iptables DNAT rule on the host forwarding traffic on host port 8080 into the container's namespace on port 80, plus a small userland proxy as fallback. The container itself never knows its external port — it just listens on 80 like always.

That's why you can run ten copies of the same image on one host, each with a different host port. One more thing: `-p 8080:80` binds all interfaces, but `-p 127.0.0.1:8080:80` only listens on localhost. That's my trick for admin dashboards that should never be public."

**Key Point:** "Port publishing is a host-side NAT rule mapping a host port into the container's namespace — the container only ever sees its own port."

---

### Q8: A container can't reach the database container. How do you debug it?

**How to Answer:**

"First I check they're actually on the same network — `docker network inspect` shows every attached container. If one was started on the default bridge and the other on a custom network, name resolution silently fails. That's the most common cause.

Then I test DNS directly: `docker exec` into the client and run `getent hosts db`. If the name doesn't resolve, it's a network or DNS problem. If it resolves but connections time out, I check whether the database is bound to all interfaces — listening on 127.0.0.1 inside a container means nothing outside it, not even the same network, can connect."

```bash
docker exec app getent hosts db        # DNS check
docker exec db ss -tlnp | grep 5432    # is it bound to 0.0.0.0 or 127.0.0.1?
```

**Key Point:** "Debug in layers: same network, then name resolution, then the server's bind address — that order catches 90% of cases."

---

## Overlay Networks and Multi Host Communication

### Q9: How do containers on different hosts talk to each other in Swarm mode?

**How to Answer:**

"Through an overlay network — a virtual network built on VXLAN tunnels between the hosts. Each host's Docker daemon wraps container-to-container packets in UDP on port 4789 and ships them to the right host, where they're unwrapped and delivered. Your containers just see one flat network and talk by name.

You create it with `docker network create -d overlay`, and Swarm's routing mesh makes any node's published port reach any replica. The catch: Swarm mode must be initialized first — overlays don't work on standalone Docker. And encrypted overlays cost CPU, so benchmark if you're pushing serious throughput."

**Key Point:** "Overlay networks stitch hosts together with VXLAN tunnels so containers communicate by name as if on one flat network."

---

### Q10: What is the ingress routing mesh in Swarm?

**How to Answer:**

"When you publish a port on a Swarm service, that port opens on EVERY node in the cluster — not just nodes running the task. The ingress overlay network routes incoming traffic to a node that actually runs a replica, load-balancing across them.

That's powerful — clients can hit any node's IP — but it surprises people when their service responds from a node with zero replicas. That's the mesh doing its job. If you want traffic pinned to nodes actually running the service, use `--publish mode=host`, which bypasses the mesh entirely."

**Key Point:** "The routing mesh forwards every node's published port to some live replica — convenient, but know the difference from host-mode publishing."

---

## Container Network Security

### Q11: Containers on the same network can talk freely — how do you restrict that?

**How to Answer:**

"By default, yes — any container on a user-defined network can reach any other container's open ports, and plain Docker gives you no per-container firewall rules. The standard answer is segmentation: separate networks per tier, containers attached only where they need access.

For real policy — api may reach db on 5432, nothing else may — you step up to an orchestrator: Kubernetes NetworkPolicies or a service mesh. On plain Docker the workarounds are host-level iptables rules or a proxy sidecar. The interview point is: Docker networking is connectivity-first, and restriction is something you layer on."

**Key Point:** "Docker networks are open by default — you get isolation through segmentation, and real per-container policy needs Kubernetes NetworkPolicies or a service mesh."

---

## Common Interview Traps

### Q12: A container can curl external sites but can't be reached from the browser. What's wrong?

**How to Answer:**

"Outbound works because the default bridge NATs egress automatically, but inbound needs an explicit published port. If you ran without `-p`, nothing on the host forwards into the container — the app listens happily in a namespace nobody routes to.

The fix is `docker run -p <host>:<container>`. The trap version: they DID publish, but the app bound to 127.0.0.1 instead of 0.0.0.0. Inside a container, localhost is the container's own namespace — binding there means even published ports forward into silence. The app must listen on 0.0.0.0 to be reachable at all."

**Key Point:** "Outbound is NATed by default but inbound needs `-p` — and the app inside must bind 0.0.0.0, because localhost in a container is only the container."

---

### Q13: From inside a container, what does `localhost` point to?

**How to Answer:**

"The container itself. Each container has its own network namespace, so localhost inside one is completely separate from the host's localhost or another container's. This trips people with service dependencies: if your app config points at `localhost:5432` expecting Postgres but Postgres is in another container, it fails.

The correct address is the container's name on their shared network — `db:5432` — resolved by the embedded DNS. The only time localhost-to-container works is `--network host` mode, where namespaces are shared. For everything else: names, not localhost."

**Key Point:** "localhost inside a container is only that container — cross-container calls go by name on a shared user-defined network."

---

*Day 17 of 58 — Containers. Next: Docker storage & volumes.*
