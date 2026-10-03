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
