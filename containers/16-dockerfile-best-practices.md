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
