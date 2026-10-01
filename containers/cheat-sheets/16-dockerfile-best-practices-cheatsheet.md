# Dockerfile Best Practices — Cheat Sheet

*One-page Dockerfile reference. Day 16 of 58.*

## The Golden Order (top to bottom)

```dockerfile
FROM node:22-slim          # 1. Pin a specific tag, never :latest
WORKDIR /app
COPY package*.json ./       # 2. Dependency files FIRST (stable)
RUN npm ci --only=production # 3. Expensive install stays cached
COPY . .                    # 4. App code LAST (volatile)
RUN adduser -D appuser && chown -R appuser /app
USER appuser                # 5. Non-root
EXPOSE 3000
ENTRYPOINT ["node"]         # 6. Exec form (JSON array)
CMD ["server.js"]           # 7. Default args, overridable
```

## Instruction Rules

| Instruction | Rule |
|---|---|
| `FROM` | Pin exact version (`node:22.11-slim`), never `latest` — reproducibility |
| `RUN` | Chain with `&&`; clean caches in the **same** layer (`rm -rf /var/lib/apt/lists/*`) |
| `COPY` | Prefer over `ADD`; only ADD when you want tar-extraction or remote fetch |
| `WORKDIR` | Always set it — avoids `RUN cd` hacks that don't persist across layers |
| `USER` | Non-root before runtime; fix perms with `chown`, don't override to root |
| `CMD`/`ENTRYPOINT` | Always JSON exec form — shell form breaks signal handling |
| `EXPOSE` | Documentation only — doesn't publish ports |
| `ENV` | Defaults only — never secrets (`docker inspect` exposes them) |
| `ARG` | Build-time only, also visible in `docker history` — not for secrets either |

## Multi-Stage Pattern

```dockerfile
FROM golang:1.23 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM gcr.io/distroless/static
COPY --from=builder /app/server /server
CMD ["/server"]
```

`--target builder` builds only up to a stage (debugging). Unnamed final stage is the default build target.

## Base Image Cheat

| Base | Size | Shell? | libc | Use when |
|---|---|---|---|---|
| `alpine` | ~5 MB | yes (ash) | musl | tiny + debuggable, watch compiled deps |
| `-slim` | ~50-80 MB | yes | glibc | safe default for Node/Python |
| `distroless` | ~20 MB | **no** | glibc | minimal attack surface, no shell debugging |
| `scratch` | 0 | no | — | static binaries only |

## Cache Rules

- Cache matches only if instruction + inputs + parent layer are identical
- One miss → **everything below rebuilds**
- Volatile things (code, timestamps) go **last**; stable things (deps) go **first**
- `.dockerignore` protects COPY cache from junk files (`node_modules`, `.git`, `.env`, `*.log`)

## Secrets (the right way)

```dockerfile
# Build-time secret — never lands in a layer:
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t app .
```

- Secrets in layers are **recoverable forever** (`docker save` → extract layer tarballs)
- Runtime secrets come from outside: env injection, mounted files, Vault / Secrets Manager

## Debug Commands

```bash
docker history <img>              # layer sizes, one per instruction
docker build --progress=plain .  # full build logs (BuildKit collapses them)
docker build --no-cache .        # rule out stale cache
docker build --target builder .  # build only up to a stage
docker run -it <failed-id> sh    # inspect a failed intermediate container
dive <img>                       # explore layers interactively
```

## Interview One-Liners

- "Layers are immutable — cleaning in a later layer never shrinks the image."
- "Combine apt-get update and install in one RUN or the index goes stale."
- "COPY is explicit; ADD's magic is a surprise at 2 AM."
- "Exec form JSON arrays — or your app won't receive SIGTERM."
- "Distroless is the answer until you need a shell at 3 AM; know your tradeoff."
