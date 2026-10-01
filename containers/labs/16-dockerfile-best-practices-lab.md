# Dockerfile Best Practices — Hands-On Lab

*8 exercises to make layering, caching, and multi-stage builds click. Run on any Linux machine or VM with Docker installed.*

**Prereqs:** Docker Engine 24+ (`docker --version`), a shell, ~15 minutes.

---

## Exercise 1: See the layers in an image

**Goal:** Understand that an image is a stack of layers, one per instruction.

**Commands:**

```bash
docker pull nginx:alpine
docker history nginx:alpine --no-trunc | head -12
```

**Expected output:** One row per Dockerfile instruction — each `RUN`/`COPY` shows its size. Big layers (package installs) stand out; `COPY` layers show how much your files add.

**Why it matters:** Every Dockerfile line is a layer. This is the mental model behind every Dockerfile best practice.

---

## Exercise 2: Feel the build cache working

**Goal:** See which layer invalidates the cache and what cascades below it.

**Commands:**

```bash
mkdir cache-test && cd cache-test
printf 'FROM alpine\nRUN sleep 2 && echo first\nCOPY . /app\n' > Dockerfile
echo hello > file.txt
time docker build -t cache1 .
echo changed > file.txt
time docker build -t cache2 .
```

**Expected output:** First build takes ~2s+ (the `sleep` runs). Second build reruns everything after `COPY` — but the `RUN sleep` stays cached because nothing above it changed. Now swap the order (COPY before RUN) and watch the sleep rerun every time.

**Why it matters:** Cache is strictly top-down. This one exercise explains why dependency installs must come before code copies.

---

## Exercise 3: Build a multi-stage Dockerfile

**Goal:** Ship only the binary, not the toolchain.

**Commands:**

```bash
mkdir ms && cd ms
printf 'package main\nimport "fmt"\nfunc main(){ fmt.Println("hi from go") }\n' > main.go
cat > Dockerfile <<'DF'
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY main.go .
RUN CGO_ENABLED=0 go build -o server main.go
FROM scratch
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
DF
docker build -t ms-demo .
docker images ms-demo --format '{{.Size}}'
```

**Expected output:** Image size is ~2 MB — the 300+ MB Go toolchain never shipped. Run it: `docker run --rm ms-demo` prints `hi from go`.

**Why it matters:** Multi-stage is the single biggest lever for image size and attack surface.

---

## Exercise 4: COPY vs ADD — see the trap

**Goal:** Watch ADD auto-extract a tarball when you expected a plain copy.

**Commands:**

```bash
mkdir addtrap && cd addtrap
echo data > file.txt && tar -czf archive.tar.gz file.txt
printf 'FROM alpine\nADD archive.tar.gz /tmp/\nRUN ls -la /tmp/\n' > Dockerfile
docker build --progress=plain -t addtrap . 2>&1 | grep -A3 'RUN ls'
```

**Expected output:** `/tmp/` contains the *extracted* `file.txt`, not `archive.tar.gz` — ADD silently unpacked it. Replace ADD with COPY and the tarball arrives as a file.

**Why it matters:** ADD's implicit behaviors break expectations. Use COPY unless you explicitly want extraction.

---

## Exercise 5: Measure what .dockerignore saves

**Goal:** See the build context shrink and the cache stabilize.

**Commands:**

```bash
mkdir ignoretest && cd ignoretest
dd if=/dev/zero of=junk.bin bs=1M count=50 2>/dev/null
printf 'FROM alpine\nCOPY . /app\n' > Dockerfile
docker build -t ignore1 . 2>&1 | grep 'transferring context'
printf 'junk.bin\n' > .dockerignore
docker build -t ignore2 . 2>&1 | grep 'transferring context'
```

**Expected output:** First build transfers ~50 MB of context; with `.dockerignore` it's a few KB. Bonus: touch junk.bin and rebuild — the cache now survives because junk isn't in the context.

**Why it matters:** .dockerignore cuts transfer time and stops junk files from invalidating your COPY cache.

---

## Exercise 6: Confirm root is the default — then fix it

**Goal:** See that containers run as root unless told otherwise.

**Commands:**

```bash
docker run --rm alpine whoami
printf 'FROM alpine\nRUN adduser -D appuser\nUSER appuser\n' > Dockerfile
docker build -t nonroot .
docker run --rm nonroot whoami
```

**Expected output:** First command prints `root`. Second prints `appuser`. One `USER` instruction removes the default root.

**Why it matters:** This is the smallest Dockerfile security fix there is. Interviewers love the follow-up: "what breaks?" — answer: file permissions you haven't fixed.

---

## Exercise 7: Watch exec form vs shell form signals

**Goal:** See why shell-form CMD makes containers stop slowly.

**Commands:**

```bash
printf 'FROM alpine\nCMD sleep 1000\n' > Dockerfile
docker build -t shellform .
printf 'FROM alpine\nCMD ["sleep", "1000"]\n' > Dockerfile
docker build -t execform .
docker run -d --name s1 shellform && docker run -d --name e1 execform
time docker stop s1; time docker stop e1
```

**Expected output:** `s1` (shell form) takes ~10s to stop — SIGTERM hits `/bin/sh`, not sleep, so Docker waits for the timeout. `e1` (exec form) stops instantly.

**Why it matters:** This is the famous Kubernetes "pod takes forever to terminate" bug, reproducible in 30 seconds.

---

## Exercise 8: Pull a secret out of a deleted layer

**Goal:** Prove that deleting a secret in a later layer doesn't remove it.

**Commands:**

```bash
mkdir leak && cd leak
printf 'FROM alpine\nRUN echo "supersecret123" > /tmp/key\nRUN rm /tmp/key\n' > Dockerfile
docker build -t leaky .
docker save leaky -o leaky.tar && mkdir layers && tar -xf leaky.tar -C layers
grep -r "supersecret123" layers/ | head -2
```

**Expected output:** The secret is found in one of the extracted layer tarballs, even though the final filesystem has no `/tmp/key`.

**Why it matters:** This is the single most convincing demo in Dockerfile security interviews. Secrets in layers are recoverable forever.

---

*Built for interview prep, one deep-dive at a time.*
