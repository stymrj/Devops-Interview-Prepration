# Dockerfile Best Practices & Multi-Stage Builds Interview Preparation Guide

*How to Answer Dockerfile, Layering & Multi-Stage Build Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Layering and the Build Cache](#layering-and-the-build-cache)
2. [Instruction Order and Caching Strategy](#instruction-order-and-caching-strategy)
3. [Multi Stage Builds](#multi-stage-builds)
4. [Slimmer Images Distroless and Alpine](#slimmer-images-distroless-and-alpine)
5. [Dockerfile Security Best Practices](#dockerfile-security-best-practices)
6. [Debugging Builds and Interview Traps](#debugging-builds-and-interview-traps)

---

## Layering and the Build Cache

### Q1: What is an image layer, and why does the layer model matter?

**How to Answer:**

"Every Dockerfile instruction — FROM, RUN, COPY, ADD — creates one read-only layer, and the final image is just a stack of those layers with a thin writable layer on top at runtime. Layers are immutable and content-addressed, so if two images share a base layer, it's stored once and reused.

That sharing is the whole point. It makes pulls faster, builds faster, and registries smaller — ten Node services on the same `node:22-slim` base share almost everything except their app code.

The flip side is that layers can never be deleted within one build. So a secret you COPY in layer 3 and delete in layer 5 is still sitting in layer 3's tarball, retrievable with `docker save`. Anyone can pull that layer out."

**Key Point:** "Layers are immutable, shared, content-addressed filesystem diffs — great for caching and dedup, but anything ever written into a layer is in the image forever."

---

### Q2: How does the Docker build cache actually work?

**How to Answer:**

"Docker walks your Dockerfile top to bottom and reuses a cached layer only if the instruction, its context inputs, and the parent layer all match exactly. The moment one instruction misses the cache — a COPY whose files changed, a RUN with different args — every instruction below it rebuilds too, because their parent layer changed.

That's why builds feel fast on an unchanged project and slow the moment you touch a dependency file. Only that one layer invalidates, but everything beneath it cascades.

The classic mistake is COPYing your whole repo before installing dependencies. Any code change then busts the cache above the install step, so dependencies reinstall on every build. I copy package files first, install, then copy the rest — that keeps the expensive step cached."

```dockerfile
COPY package*.json ./
RUN npm ci --only=production
COPY . .
```

**Key Point:** "Cache matching is exact and strictly top-down — one miss rebuilds everything below it, so put slow, stable steps early and volatile steps late."

---

## Instruction Order and Caching Strategy

### Q3: Why do we combine RUN commands with && instead of writing many RUNs?

**How to Answer:**

"Every RUN is its own layer, so ten RUNs means ten layers that each add overhead and bloat the final image. I chain related commands with `&&` in a single RUN, and I always clean package-manager caches in that same RUN.

The classic example is apt: if you `RUN apt-get update` in one layer and `RUN apt-get install` in another, the update layer is cached separately from the install — a stale package index gets reused and the build breaks or installs outdated packages. Combining them in one layer makes update and install atomic.

And the cleanup has to be in the same RUN. `rm -rf /var/lib/apt/lists/*` in a later layer doesn't shrink anything, because the deleted files still exist in the earlier layer."

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

**Key Point:** "Combine related RUNs into one layer so package index, install, and cache cleanup are atomic — and cleaning in a later layer never shrinks the image."

---

### Q4: What's the difference between COPY and ADD? Which should I use?

**How to Answer:**

"COPY does one thing: copy files from the build context into the image. ADD does that plus two magic behaviors — it auto-extracts tar archives and can fetch remote URLs. Almost every interview expects the same verdict: use COPY unless you specifically need ADD's behavior.

The auto-extraction is unpredictable and the URL fetching skips TLS verification guarantees and caching control that a RUN with curl gives you. I've seen builds break because someone ADDed a tarball expecting it to copy as a file, and it exploded into a directory.

So the rule I state in interviews: COPY is explicit and boring, which is exactly what you want in a build file. ADD's extra behaviors are the kind of implicit magic that causes 2 AM surprises."

**Key Point:** "COPY is the explicit, predictable choice — ADD's tar-extraction and URL fetching are surprises waiting to happen, so use it only when you need those behaviors."

---

### Q5: What is .dockerignore, and what happens if you skip it?

**How to Answer:**

"A .dockerignore lists files and directories excluded from the build context — the set of files Docker ships to the daemon before the build even starts. If you skip it, the entire repo — node_modules, .git, build artifacts, local .env files — gets sent every build, which slows everything down.

But the real damage is cache invalidation. COPY checks the checksums of its source files; with no .dockerignore, a changed file in node_modules or .git invalidates your COPY layer even though the app code didn't change, and your whole build cache cascades from there.

It's the first thing I add to any project. One line for node_modules, one for .git, one for .env, and the build is deterministic and fast. Same purpose as .gitignore, just protecting the build instead of the repo."

**Key Point:** "Without .dockerignore, stray files bloat the build context and poison the cache — one junk file change can invalidate every layer below it."

---

## Multi Stage Builds

### Q6: What problem do multi-stage builds solve?

**How to Answer:**

"They separate the build environment from the runtime environment. Without them, you either ship your compiler, build tools, and source code inside the production image — bloating it and expanding the attack surface — or you run awkward build scripts outside Docker and COPY the artifact in.

With multi-stage builds, one Dockerfile has named stages: a builder stage with the full toolchain compiles the app, and a final runtime stage copies only the artifact — a binary, a dist folder — into a slim base. The builder stage never ships.

The size difference is real. A Go binary built with the Go toolchain and then copied into `scratch` or `distroless` goes from hundreds of megabytes to a few megabytes. And the attack surface shrinks with it: no shell, no package manager, no compiler in production."

```dockerfile
FROM golang:1.23 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM gcr.io/distroless/static
COPY --from=builder /app/server /server
CMD ["/server"]
```

**Key Point:** "Multi-stage builds keep compilers and source out of production images — one stage builds, the final stage ships only the artifact."

---

## Slimmer Images Distroless and Alpine

### Q7: How do you shrink an image — and what are the real tradeoffs between Alpine, slim, and distroless?

**How to Answer:**

"Three knobs: pick a smaller base, copy fewer things, and drop the build toolchain with multi-stage. The base-image decision is where interviews focus. `alpine` is a full OS in ~5 MB with a shell and apk — great for debugging but musl libc can subtly break compiled dependencies, so Node and Python apps sometimes behave differently. `slim` variants keep glibc compatibility with a minimal package set, which is the safer middle ground. `distroless` ships only your app and its runtime deps — no shell, no package manager — so it's the smallest and most secure, but you can't exec into it to debug.

So I frame it as a tradeoff, not a ranking: distroless for minimal attack surface in prod, slim when you need glibc compatibility, alpine when you want tiny plus a shell for debugging. I've seen teams standardize on distroless and then lose an hour in an outage because nobody could get a shell in the container — the answer is knowing which tradeoff you chose."

**Key Point:** "Alpine gives you tiny plus a shell, slim gives you glibc compatibility, distroless gives you minimal attack surface but no shell — pick based on debugging needs versus security."

---

### Q8: Why shouldn't containers run as root?

**How to Answer:**

"Because a container process escaping to the host — through a kernel bug or a misconfigured volume — lands with whatever UID the container process had, and root in the container is UID 0, which is root on the host for filesystem access unless user namespaces remap it. Running as a non-root user turns a breakout into a low-privilege annoyance instead of a host compromise.

The Dockerfile fix is simple: create a user, set ownership, and USER into it. But the interview trap is that many official images already ship with a non-root user — `nginx`, `postgres`, `node` — and people override it with `USER root` to dodge a permission error, which is exactly backwards.

The right move is fixing the permissions, not the user. If your app can't write to a directory, chown it in the image or mount a writable volume — don't hand the container root because file permissions were annoying."

```dockerfile
RUN useradd -m appuser && chown -R appuser /app
USER appuser
```

**Key Point:** "A breakout from a root container is a host compromise; running as non-root and fixing permissions instead of overriding to root keeps the blast radius small."

---

## Dockerfile Security Best Practices

### Q9: What Dockerfile practices keep secrets out of images?

**How to Answer:**

"Three rules. First, never COPY or ENV a secret into the image — layers are immutable, so even if you delete the secret in a later layer, it's still extractable from the earlier one with `docker save`. ENV is equally bad because anyone can read it with `docker inspect`.

Second, for build-time secrets like private repo tokens or npm credentials, use BuildKit's `--mount=type=secret` in a RUN command. The secret is mounted into the layer during the build and never committed to any layer — it literally cannot leak into the image.

Third, runtime secrets come from outside the image entirely: environment variables injected by the orchestrator, mounted secret files, or a secrets manager like Vault or AWS Secrets Manager. The image itself should contain zero credentials."

**Key Point:** "Secrets never go in layers or ENV — build-time secrets use BuildKit secret mounts, runtime secrets are injected from outside, and anything once written to a layer is recoverable forever."

---

### Q10: What's the difference between CMD and ENTRYPOINT, and why do people combine them?

**How to Answer:**

"ENTRYPOINT defines the fixed executable for the container; CMD defines the default arguments. When you combine them, ENTRYPOINT is the binary and CMD is its defaults, so `docker run myimage --debug` appends to CMD and the entrypoint still controls what actually runs.

The interview trap is exec form versus shell form. Shell form — `CMD npm start` without brackets — wraps the command in `/bin/sh -c`, which means your app runs as PID 1's child, not PID 1, and signals like SIGTERM go to the shell, not your app. That's the classic 'container takes 10 seconds to stop in Kubernetes' bug.

So the rule: always use the JSON exec form for both. `ENTRYPOINT ["node", "server.js"]` with `CMD ["--port", "3000"]` gives you clean signal handling, easy argument overrides, and the container dies when the app dies."

**Key Point:** "ENTRYPOINT is the fixed executable, CMD the default args — and always use JSON exec form, or signal handling breaks and your containers won't stop cleanly."

---

## Debugging Builds and Interview Traps

### Q11: How do you debug a failing Docker build?

**How to Answer:**

"I start with the build output, which shows exactly which step failed and the exit code, then I rerun with `--progress=plain` to get the full logs instead of the collapsed BuildKit view. If the error is in a RUN step, I comment out everything after it and add a layer that drops me into a shell — or better, use `docker build --target <stage>` on a multi-stage Dockerfile to build just up to the failing stage.

The modern trick is `docker build --progress=plain --no-cache` plus inspecting intermediate containers: BuildKit prints the container ID for each failed step, and you can `docker run -it <id> sh` on it to reproduce the failure interactively.

The traps I call out: stale cache masking a real fix — always test suspicious fixes with `--no-cache` — and RUN steps that download from the internet without pinning versions, which make builds non-reproducible. `docker history` and `dive` show me what's actually in each layer when size or content surprises me."

**Key Point:** "Debug with --progress=plain and --target to isolate the failing stage, inspect failed intermediate containers interactively, and always verify fixes with --no-cache."

---

### Q12: Walk me through a real Dockerfile you wrote for production — what did you optimize?

**How to Answer:**

"Sure — a Node API I shipped recently. Multi-stage: builder on `node:22-slim` installs deps with `npm ci` after copying only package files, so the install layer stays cached. Final stage is `gcr.io/distroless/nodejs22` with just the built dist copied over — no source, no devDependencies, no shell.

The ordering matters: dependency files first, then install, then app code, so code changes never trigger a reinstall. Non-root user throughout the final stage. Secrets — npm token during build — come via BuildKit secret mounts, never ENV.

Result: image went from about 1.1 GB to under 150 MB, builds take seconds when only code changes, and the runtime has nothing an attacker can use — no shell, no package manager, no build tools. The interviewer usually follows up on distroless debugging, and I say honestly: we kept one slim-tagged image for debugging sessions, because knowing your tradeoff is better than pretending it doesn't exist."

**Key Point:** "Production Dockerfiles are multi-stage, dependency-first ordered, non-root, secret-free, and minimal-base — and I always name the tradeoff I accepted, like distroless debugging."

---

*Built for interview prep, one deep-dive at a time.*
