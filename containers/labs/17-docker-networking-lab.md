# Docker Networking — Hands-On Lab

*9 exercises to make Docker networks, DNS, and port publishing click. Run on any Linux machine or VM with Docker installed.*

**Prereqs:** Docker Engine 24+ (`docker --version`), a shell, ~20 minutes.

---

## Exercise 1: Inspect the default bridge

**Goal:** See the docker0 network every container uses by default.

**Commands:**

```bash
docker network ls
docker network inspect bridge --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
ip addr show docker0 | head -5
```

**Expected output:** `bridge` listed in `docker network ls`; subnet like `172.17.0.0/16`; a `docker0` interface on the host with the gateway IP.

**Why it matters:** This is the invisible plumbing behind every `docker run`. Knowing it exists helps you understand everything else.

---

## Exercise 2: Prove the default bridge has no DNS

**Goal:** Confirm containers on the default bridge can only reach each other by IP.

**Commands:**

```bash
docker run -d --name srv1 nginx:alpine
docker run -d --name srv2 nginx:alpine
docker inspect -f '{{.NetworkSettings.IPAddress}}' srv1
docker exec srv2 getent hosts srv1 || echo "DNS FAILED"
docker exec srv2 ping -c2 <srv1-IP>   # replace with the IP above
```

**Expected output:** `getent hosts srv1` fails (no name resolution), but ping to the IP succeeds.

**Why it matters:** This is THE reason to use custom networks. Memorize this demo — it answers half of all Docker networking interview questions.

---

## Exercise 3: Create a network and get free DNS

**Goal:** See name resolution work on a user-defined network.

**Commands:**

```bash
docker network create demo-net
docker run -d --name web --network demo-net nginx:alpine
docker run -d --name api --network demo-net nginx:alpine
docker exec web getent hosts api
docker exec web wget -qO- http://api | head -3
```

**Expected output:** `getent hosts api` returns an IP (172.18.x.x or similar); wget returns nginx's welcome page HTML.

**Why it matters:** One command replaces all the old `--link` hacks. This is the default pattern for multi-container apps.

---

## Exercise 4: Port publishing, scoped and unscoped

**Goal:** See the difference between binding all interfaces vs localhost only.

**Commands:**

```bash
docker run -d -p 8080:80 --name pub nginx:alpine
docker run -d -p 127.0.0.1:8081:80 --name scoped nginx:alpine
ss -tlnp | grep -E '8080|8081'
```

**Expected output:** Port 8080 listens on `0.0.0.0` (all interfaces, LAN-reachable); port 8081 listens on `127.0.0.1` only.

**Why it matters:** The default `-p` exposes the port to your whole LAN. Scoping to 127.0.0.1 is how you keep admin dashboards private.

---

## Exercise 5: Traffic isolation between networks

**Goal:** Prove containers on different networks can't talk to each other.

**Commands:**

```bash
docker network create net-a && docker network create net-b
docker run -d --name a1 --network net-a nginx:alpine
docker run -d --name b1 --network net-b nginx:alpine
docker exec a1 wget -qO- --timeout=3 http://b1 || echo "UNREACHABLE (expected)"
docker network connect net-b a1
docker exec a1 wget -qO- --timeout=3 http://b1 | head -3
```

**Expected output:** First wget fails; after `docker network connect`, the same call returns nginx's page.

**Why it matters:** Networks are security boundaries. This is how you build front/back tier separation without any extra tooling.

---

## Exercise 6: Bind-address trap — the classic failure

**Goal:** Feel why binding to 127.0.0.1 inside a container breaks everything.

**Commands:**

```bash
docker run -d --name bindtrap -p 9090:9090 python:3.12-slim \
  sh -c "python3 -m http.server 9090 --bind 127.0.0.1"
curl --max-time 3 http://localhost:9090 || echo "UNREACHABLE despite -p"
docker rm -f bindtrap
docker run -d --name bindok -p 9090:9090 python:3.12-slim \
  sh -c "python3 -m http.server 9090 --bind 0.0.0.0"
curl -s --max-time 3 http://localhost:9090 | head -2
```

**Expected output:** First curl fails (app bound to the container's loopback, invisible even through the published port); second works.

**Why it matters:** Inside a container, 127.0.0.1 means only the container. Apps must bind 0.0.0.0 to be reachable — a favorite interview trap.

---

## Exercise 7: Host network mode — see the host's interfaces

**Goal:** Confirm a host-mode container shares the host's network namespace.

**Commands:**

```bash
docker run --rm --network host alpine ip -brief addr
ip -brief addr
```

**Expected output:** Identical interface lists — the container sees eth0, lo, everything, with no port mapping needed.

**Why it matters:** Shows exactly what `--network host` trades away (all isolation) for what it gains (zero NAT overhead).

---

## Exercise 8: Debug a broken connection, end to end

**Goal:** Practice the debug order: network → DNS → bind address.

**Commands:**

```bash
docker network create debug-net
docker run -d --name db --network debug-net postgres:16-alpine -c listen_addresses='127.0.0.1'
docker run -d --name client --network debug-net postgres:16-alpine sleep infinity
docker exec client getent hosts db          # step 1: DNS ok?
docker exec db ss -tlnp | grep 5432         # step 2: bound to what?
```

**Expected output:** DNS resolves fine, but `ss` shows Postgres listening on 127.0.0.1 only — the client can't connect despite perfect networking.

**Why it matters:** Most "networking" bugs are actually bind-address bugs. This exercise teaches you to check in the right order instead of guessing.

---

## Exercise 9: Clean up everything

**Goal:** Leave the machine tidy — and see how easy Docker makes it.

**Commands:**

```bash
docker rm -f srv1 srv2 web api pub scoped a1 b1 bindok db client
docker network rm demo-net net-a net-b debug-net
docker network ls
```

**Expected output:** Only the built-in `bridge`, `host`, and `none` networks remain.

**Why it matters:** Ephemeral networks are the whole point — create, experiment, delete. Never leave lab networks lying around on a real host.

---

*Day 17 of 58 — Containers.*
