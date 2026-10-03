# Docker Compose Interview Preparation Guide

*How to Answer Docker Compose Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Compose Basics and Core Concepts](#compose-basics-and-core-concepts)
2. [Networking and Service Discovery](#networking-and-service-discovery)
3. [Volumes Dependencies and Startup Order](#volumes-dependencies-and-startup-order)
4. [Configuration and Environment Management](#configuration-and-environment-management)
5. [Production and Deployment Patterns](#production-and-deployment-patterns)
6. [Common Interview Traps](#common-interview-traps)

---

## Compose Basics and Core Concepts

### Q1: What is Docker Compose, and when would you reach for it?

**How to Answer:**

"Compose lets me define a whole multi-container app — API, database, cache, whatever — in one YAML file, then bring it up with a single command. Without it I'd be juggling a bunch of docker run commands with ports, networks, and volumes wired by hand, which breaks the moment anyone adds a service. With Compose, `docker compose up` reproduces the entire stack identically on any machine. I reach for it in local dev, integration tests in CI, and small production deployments. It's the fastest way to make 'it works on my machine' actually true for the whole team."

```yaml
services:
  api:
    build: ./api
    ports: ["8080:8080"]
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16
    volumes: ["db-data:/var/lib/postgresql/data"]
volumes:
  db-data:
```

**Key Point:** "Compose defines your whole multi-container stack in one YAML file and brings it up with one command — dev-parity for free."

---

### Q2: How does Compose differ from a bunch of docker run commands or a Dockerfile?

**How to Answer:**

"A Dockerfile builds one image; Compose orchestrates many containers running together. Compared to raw docker run commands, Compose is declarative — I describe the desired state of all services and Compose figures out the networks, volumes, and startup order. Those imperative commands drift the moment someone forgets a flag, while a compose file is version-controlled documentation of the stack. The trade is that Compose adds an abstraction layer, so when things break I still need to know the underlying docker primitives to debug. But for any stack with more than two services, that abstraction pays for itself."

**Key Point:** "Dockerfiles build images, docker run is imperative one-liners, Compose is a declarative description of the whole stack checked into git."

---

## Networking and Service Discovery

### Q3: How do services in Compose talk to each other?

**How to Answer:**

"Every compose project gets its own default bridge network, and services reach each other by service name — my API just hits http://db:5432 and Compose's embedded DNS resolves db to the container's IP. I never hardcode IPs because containers get new ones on restart. That's also why every service needs a healthcheck on its dependencies rather than assuming the database is ready. If I need to isolate services, I add custom networks and attach only the services that should talk. The rule is simple: services are DNS names, and networks are the firewall."

```yaml
networks:
  frontend: {}
  backend: {}
services:
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]  # unreachable from frontend
```

**Key Point:** "Compose gives each project its own network and DNS, so services reach each other by service name — never hardcode container IPs."

---

### Q4: How do you expose ports without creating conflicts?

**How to Answer:**

"The ports directive maps host to container — `8080:80` means port 8080 on my machine reaches port 80 in the container. For services that only need to talk to each other, I skip ports entirely — they're already reachable on the project network by service name. On shared dev boxes I use variable ports like `${APP_PORT:-8080}:80` so two developers don't fight over the same port. And if a service really doesn't need outside access — a private worker, a database behind the API — I publish nothing and keep it off the host entirely. Fewer open ports means fewer surprises."

**Key Point:** "Publish ports only for what the host needs to reach; internal services communicate on the project network without any port mapping."

---

## Volumes Dependencies and Startup Order

### Q5: What's the difference between depends_on and actually waiting for a dependency to be ready?

**How to Answer:**

"This is the classic Compose trap: depends_on only controls startup order, not readiness. It makes sure the database container starts before my app container — but Postgres still needs a few seconds to accept connections, so my app can crash on its first connection attempt. The real fix is depends_on with condition: service_healthy plus a healthcheck on the database service. In production-grade setups I also make the app itself retry the connection a few times on boot, because even healthchecks have a gap. Order of starting is not the same as being ready to serve."

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
  api:
    depends_on:
      db:
        condition: service_healthy
```

**Key Point:** "depends_on orders container starts; only a healthcheck plus condition: service_healthy makes Compose actually wait for readiness."

---

### Q6: Where should state live in a Compose stack — named volumes or bind mounts?

**How to Answer:**

"Databases and anything that must survive a restart go on named volumes — `db-data:/var/lib/postgresql/data` — because the daemon manages them and they survive container recreation. I use bind mounts for things the host owns: source code I'm hot-reloading in dev, config files I'm iterating on. The trap is mixing them up — bind-mounting a database directory into a path that doesn't exist on a teammate's machine, or losing dev work by putting source in a named volume. My rule in dev: bind-mount code, named volumes for state. In production: everything persistent is a named volume, ideally declared external so Compose never deletes it by accident."

**Key Point:** "Named volumes for persistent state that must survive restarts; bind mounts for host-owned code and config you're actively editing."

---

## Configuration and Environment Management

### Q7: How do you manage different environments — dev, staging, prod — in Compose?

**How to Answer:**

"I keep a base compose.yaml and override files per environment: compose.override.yaml auto-loads in dev with hot-reload mounts and debug settings, while prod runs with `-f compose.yaml -f compose.prod.yaml` for replicas and resource limits. Environment-specific values come from .env files or the shell environment, so secrets never land in the compose file itself. The key discipline is that the base file stays runnable on its own — overrides only adjust, never define. That way the same artifacts deploy everywhere, and the only thing changing between environments is configuration."

```bash
# dev (override auto-loads)
docker compose up -d
# prod (explicit files, no override)
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

**Key Point:** "One base compose file plus per-environment override files and .env variables — same artifacts everywhere, only config changes."

---

### Q8: How do you handle secrets in Compose without leaking them?

**How to Answer:**

"The naive move is environment variables in the compose file, which leaks secrets into docker inspect output and git history. I use the secrets top-level key with a file source in dev — Compose mounts the file at /run/secrets/<name> inside the container, never in the environment. For production I point secrets at external sources or inject them through the shell with ${DB_PASSWORD} from a vault-populated, gitignored .env. The rule I tell interviewers: if a secret is visible in your compose file or in docker inspect, you've already leaked it."

**Key Point:** "Use Compose secrets (file-mounted at /run/secrets) or vault-backed env vars — never hardcode secrets where docker inspect or git can see them."

---

## Production and Deployment Patterns

### Q9: Is Docker Compose suitable for production?

**How to Answer:**

"Honestly — yes for small to medium workloads, with caveats. Compose is brilliant on a single host: a side project, a small SaaS, internal tools. But it has no built-in high availability, no rolling updates across nodes, and no auto-scaling — when the host dies, everything dies. If I outgrow one host, that's the signal to move to ECS, Swarm, or Kubernetes. In interviews I say Compose is a deployment tool for single-host simplicity, not an orchestrator. Using it in production is fine as long as you can answer 'what happens when this box dies' without flinching."

**Key Point:** "Compose is a fine production tool for single-host workloads; it's not an orchestrator — know its limits and the migration path off it."

---

### Q10: How do you do zero-downtime deploys with Compose?

**How to Answer:**

"Compose's built-in answer is docker compose up -d again with the new image — it recreates only changed services, but there's still a restart gap. For real zero downtime I run the new version alongside the old one: scale up a second container on a different port, healthcheck it, then flip a reverse proxy like nginx or Traefik to the new one and tear the old down. That's a manual blue-green deploy. Some teams use --scale with a load balancer in front for rolling updates. It's doable, but it takes plumbing that Kubernetes gives you for free."

**Key Point:** "Recreating services causes a restart gap — zero downtime needs blue-green behind a proxy or rolling updates via --scale, which you build yourself."

---

## Common Interview Traps

### Q11: Why did my Compose setup work locally but fail in CI?

**How to Answer:**

"Nine times out of ten it's an implicit assumption: a hardcoded port that CI already uses, a bind mount pointing at a path that doesn't exist on the runner, or depends_on without a healthcheck so the app started before the database was ready. My debugging order is docker compose logs, then docker compose ps to check container states, then verifying every bind mount and env var actually resolves on the runner. I also pin image tags — :latest on my machine might be a different digest than CI pulled yesterday. Reproducible beats convenient every time."

**Key Point:** "Local-vs-CI failures are almost always implicit assumptions: ports, host paths, missing healthchecks, or unpinned image tags."

---

### Q12: Compose v1 vs v2 — what changed, and why does it matter?

**How to Answer:**

"Compose v1 was a Python package invoked as docker-compose with a hyphen; v2 is a Go plugin built into the Docker CLI, invoked as docker compose with a space. V2 is dramatically faster at starting large stacks because it creates containers in parallel and talks to the daemon natively instead of shelling out. It also handles healthcheck-based depends_on conditions more reliably. In an interview the honest answer is: if you still type docker-compose with a hyphen, you're on the legacy version — v2 has been the default since Docker Desktop 4.x. Migration is mostly painless since the compose file format barely changed."

**Key Point:** "V1 was the Python docker-compose; V2 is the built-in docker compose plugin — faster, parallel starts, and the default for years now."

---

*Day 19 of 58 — next up: Kubernetes architecture (control plane & nodes).*
