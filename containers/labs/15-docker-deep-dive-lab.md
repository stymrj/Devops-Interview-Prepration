# Docker Deep Dive — Hands-On Lab

*8 exercises to make the architecture click. Run on any Linux machine or VM with Docker installed.*

**Prereqs:** Docker Engine 24+ (`docker --version`), a shell, ~10 minutes.

---

## Exercise 1: Prove a container shares the host kernel

**Goal:** See that the container has no kernel of its own.

**Commands:**

```bash
uname -r
docker run --rm alpine uname -r
```

**Expected output:** Both commands print the *same* kernel version (e.g. `6.8.0-...`) — the container reports the host's kernel because it is using the host's kernel.

**Why it matters:** This is the single fact behind every containers-vs-VMs interview answer. If you can demo it, you own the question.

---

## Exercise 2: Spot the client/server split

**Goal:** Confirm the CLI is just an API client talking to dockerd.

**Commands:**

```bash
docker version
sudo ss -x | grep docker.sock
```

**Expected output:** `docker version` shows separate `Client:` and `Server:` sections with their own versions. The socket listing shows `/var/run/docker.sock` listening — that's the REST endpoint the CLI calls.

**Why it matters:** "Docker" isn't one program — it's a client/server system. Knowing the socket exists explains remote daemons, `DOCKER_HOST`, and why root owns the socket.

---

## Exercise 3: Watch a container become a host process

**Goal:** See the same process from inside and outside its PID namespace.

**Commands:**

```bash
docker run -d --name pid-demo nginx
docker exec pid-demo ps aux | head -3
ps aux | grep -v grep | grep nginx | head -3
docker rm -f pid-demo
```

**Expected output:** Inside, nginx is PID 1. On the host, the same nginx worker shows a regular host PID (e.g. 42317). Same process, two views.

**Why it matters:** This is namespaces in action — the demo interviewers remember. It also shows why `docker stop` signals matter: PID 1 has special signal semantics.

---

## Exercise 4: List the namespaces of a running container

**Goal:** Map a container to its six namespaces directly.

**Commands:**

```bash
docker run -d --name ns-demo nginx
PID=$(docker inspect -f '{{.State.Pid}}' ns-demo)
sudo ls -l /proc/$PID/ns | awk '{print $9, $10, $11}'
docker rm -f ns-demo
```

**Expected output:** Six namespace entries — `ipc`, `mnt`, `net`, `pid`, `uts`, `user` — each with an inode number distinct from your shell's namespaces.

**Why it matters:** Turns "Docker uses namespaces" from a memorized line into something you've inspected. Compare with `ls -l /proc/$$/ns` to see your shell's different inodes.

---

## Exercise 5: Hit a cgroup memory limit on purpose

**Goal:** Feel what a memory limit does when a process exceeds it.

**Commands:**

```bash
docker run --rm --memory=50m alpine sh -c "apk add -q stress 2>/dev/null; stress --vm 1 --vm-bytes 200m --timeout 5s" ; echo "exit: $?"
```

**Expected output:** The container is OOM-killed (exit code 137) within seconds — the kernel enforced the 50 MB cgroup cap against a 200 MB allocation.

**Why it matters:** This is exactly what an OOMKilled pod looks like in Kubernetes. Limits aren't theoretical; this is the mechanism behind every memory-limit interview question.

---

## Exercise 6: Inspect image layers and sharing

**Goal:** See layers, digests, and sharing between images.

**Commands:**

```bash
docker pull nginx:alpine
docker pull httpd:alpine
docker images --digests
docker history --no-trunc nginx:alpine | head -8
```

**Expected output:** `docker history` lists each layer with its size and creating command. Both images share the `alpine` base layers — stored once on disk even though two images reference them.

**Why it matters:** Layers explain pull speed, disk usage, and build caching. "Content-addressed and shared" stops being abstract once you've seen the digests match.

---

## Exercise 7: Prove the writable layer is copy-on-write and ephemeral

**Goal:** Show container writes never touch image layers, and vanish with the container.

**Commands:**

```bash
docker run -d --name cow-demo nginx
docker exec cow-demo sh -c "echo hello > /usr/share/nginx/html/proof.txt"
docker diff cow-demo
docker rm cow-demo
docker run --rm nginx cat /usr/share/nginx/html/proof.txt || echo "file is gone"
```

**Expected output:** `docker diff` shows `A /usr/share/nginx/html/proof.txt` (added in the writable layer). After `docker rm`, a fresh container has no trace of the file.

**Why it matters:** This is why data must live in volumes. It also answers "what's the difference between an image and a container" with a live demo.

---

## Exercise 8: Trace a pull — manifest, missing layers only, digest pinning

**Goal:** See Docker download only what it lacks, and pin an image by digest.

**Commands:**

```bash
docker pull nginx:alpine 2>&1 | grep -E "Digest|Status"
DIGEST=$(docker inspect -f '{{.RepoDigests}}' nginx:alpine | tr -d '[]')
echo "Pinned ref: $DIGEST"
docker pull "$DIGEST" 2>&1 | tail -1
```

**Expected output:** First pull downloads layers; the digest-pinned pull reports everything already present ("Image is up to date") — the digest resolves to the exact same bits.

**Why it matters:** Tags move, digests don't. This exercise is the muscle memory behind "pin digests in production" — a line that lands harder when you've done the pull.

---

## Cleanup

```bash
docker system prune -f
```

*End of lab — Day 15 of 58. Pair with the [guide](../15-docker-deep-dive.md) and [cheat sheet](../cheat-sheets/15-docker-deep-dive-cheatsheet.md).*
