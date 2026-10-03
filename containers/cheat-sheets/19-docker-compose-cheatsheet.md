# Docker Compose Cheat Sheet

One page. Pin it before the interview.

## Lifecycle

| Command | Does |
|---|---|
| `docker compose up -d` | Build + create + start everything (auto-loads `compose.override.yaml`) |
| `docker compose up -d --build` | Force image rebuilds before starting |
| `docker compose up -d --force-recreate` | Recreate containers even if config unchanged |
| `docker compose stop` / `start` | Stop/start without removing anything |
| `docker compose down` | Remove containers + networks (volumes kept) |
| `docker compose down -v` | Also delete named volumes — **data gone** |
| `docker compose down --rmi all` | Also remove images built/used by the stack |
| `docker compose restart <svc>` | Restart one service |

## Inspect & debug

| Command | Does |
|---|---|
| `docker compose ps` | Container states per service |
| `docker compose logs -f <svc>` | Follow one service's logs |
| `docker compose logs --tail=50 <svc>` | Last 50 lines |
| `docker compose exec <svc> sh` | Shell into a running container |
| `docker compose config` | Show merged YAML after overrides + interpolation |
| `docker compose top` | Processes running in each container |

## Build & images

| Command | Does |
|---|---|
| `docker compose build` / `build <svc>` | Build images (respects `build:` context/dockerfile) |
| `docker compose pull` | Pull images without starting |
| `docker compose images` | Images used by the project |

## Multi-file & env

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d  # env-specific stack
docker compose --env-file .env.prod up -d                  # alternate env file
docker compose -p myproject up -d                          # custom project name
COMPOSE_PROJECT_NAME=myproject docker compose up -d        # same via env
```

## The YAML that matters

```yaml
services:
  api:
    build: ./api                       # or: image: myorg/api:1.2.3
    ports: ["${APP_PORT:-8080}:8080"]  # host:container
    expose: ["8080"]                   # inter-service only, no host port
    environment:
      ENV: prod
    env_file: [.env]                   # gitignore this
    volumes:
      - "data:/var/lib/app"            # named volume (state)
      - "./src:/app/src"               # bind mount (dev code)
    networks: [frontend, backend]
    depends_on:
      db:
        condition: service_healthy     # waits for readiness, not just start
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    restart: unless-stopped
    deploy:
      resources:
        limits: {cpus: "1.0", memory: 512M}
  db:
    image: postgres:16
    secrets: [db_password]             # mounted at /run/secrets/db_password
secrets:
  db_password:
    file: ./db-pass.txt                # or: external: true
networks:
  frontend: {}
  backend: {}
volumes:
  data:
```

## Healthcheck & readiness

- `depends_on` alone = **startup order only**. Add `condition: service_healthy` + a `healthcheck` on the dependency.
- `start_period` gives slow services a grace window before failures count.
- Apps should still retry connections on boot — healthchecks have a gap.

## Secrets rules

- `secrets:` mounts files at `/run/secrets/<name>` — never in env, never in `docker inspect`.
- `*_FILE` convention (e.g. `POSTGRES_PASSWORD_FILE`) tells official images to read the secret file.
- `.env` for non-secret config; vault-populated `.env` (gitignored) for real secrets.

## Scaling

```bash
docker compose up -d --scale api=3     # 3 replicas (no published host ports!)
docker compose up -d --no-deps api     # recreate api without touching deps
```

- Scaled services can't publish the same host port — put a proxy (nginx/Traefik) in front.

## Common flags

`-d` detach · `--build` rebuild · `--force-recreate` · `--no-deps` · `-f` file · `-p` project · `--profile` enable profiles · `--wait` block until healthy (with healthchecks) · `--dry-run` preview changes (v2.20+)

## Interview one-liners

- "Compose is declarative; `docker run` is imperative."
- "Services are DNS names on the project network — never hardcode IPs."
- "`depends_on` orders starts; healthchecks wait for readiness."
- "Named volumes for state, bind mounts for code."
- "If a secret is in `docker inspect`, you've leaked it."
- "Compose deploys single hosts; it is not an orchestrator."
