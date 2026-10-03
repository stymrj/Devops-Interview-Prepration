# Docker Compose — Hands-On Lab

10 exercises. Run them in order on a machine with Docker + Compose v2 installed.

---

## Exercise 1: Your first multi-service stack

**Goal:** Define and run a web + API stack, and prove the services resolve each other by name.

**Commands:**
```bash
mkdir compose-lab && cd compose-lab
cat > compose.yaml <<'EOF'
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
  api:
    image: hashicorp/http-echo
    command: ["-text=hello from api"]
EOF
docker compose up -d
docker compose exec web wget -qO- http://api:5678
```

**Expected output:** `up -d` starts both containers; the `wget` inside `web` prints `hello from api` — DNS resolution by service name works.

**Why it matters:** This is the core Compose promise — one file, one command, services talking over the project's private network with zero manual wiring.

---

## Exercise 2: Named volume persistence for a database

**Goal:** Prove a named volume survives container recreation.

**Commands:**
```bash
cat >> compose.yaml <<'EOF'

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes: ["pg-data:/var/lib/postgresql/data"]
volumes:
  pg-data:
EOF
docker compose up -d db
docker compose exec db psql -U postgres -c "CREATE TABLE t(id int); INSERT INTO t VALUES (1);"
docker compose rm -f -s db   # destroy the container, not the volume
docker compose up -d db
docker compose exec db psql -U postgres -c "SELECT * FROM t;"
```

**Expected output:** After recreation, `SELECT * FROM t;` still returns the row `(1)`. `docker compose down -v` would have deleted it — notice the `-v`.

**Why it matters:** Interviews love "where does state live" — named volumes decouple data from container lifecycle.

---

## Exercise 3: depends_on with healthcheck (the readiness trap)

**Goal:** Show that plain depends_on doesn't wait, and fix it with a healthcheck.

**Commands:**
```bash
cat > compose-ready.yaml <<'EOF'
services:
  db:
    image: postgres:16
    environment: {POSTGRES_PASSWORD: secret}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 3s
      retries: 10
  app:
    image: alpine
    command: ["sh", "-c", "sleep 1 && echo trying-db"]
    depends_on:
      db:
        condition: service_healthy
EOF
docker compose -f compose-ready.yaml up -d
docker compose -f compose-ready.yaml logs app
```

**Expected output:** `app` logs appear only after `db` reports healthy. Remove `condition: service_healthy`, recreate, and watch `app` start while postgres is still initializing.

**Why it matters:** This exact question comes up in interviews — "why did my app crash even though I used depends_on?"

---

## Exercise 4: Dev/prod override files

**Goal:** Keep one base file and swap behavior per environment.

**Commands:**
```bash
cat > compose.override.yaml <<'EOF'
services:
  web:
    environment: {ENV: dev}
EOF
cat > compose.prod.yaml <<'EOF'
services:
  web:
    environment: {ENV: prod}
    deploy:
      resources:
        limits: {cpus: "0.5", memory: 256M}
EOF
docker compose config            # dev: override auto-merged
docker compose -f compose.yaml -f compose.prod.yaml config | grep -A2 ENV
```

**Expected output:** `config` shows the merged result — `ENV: dev` locally, `ENV: prod` plus resource limits with the prod file.

**Why it matters:** Same artifacts everywhere, only config changes — the standard answer for multi-environment Compose.

---

## Exercise 5: Secrets without leaks

**Goal:** Inject a secret via the secrets mechanism and verify it's invisible to `inspect`.

**Commands:**
```bash
echo "supersecret" > db-pass.txt
cat > compose-secrets.yaml <<'EOF'
services:
  db:
    image: postgres:16
    environment: {POSTGRES_PASSWORD_FILE: /run/secrets/db_password}
    secrets: [db_password]
secrets:
  db_password:
    file: ./db-pass.txt
EOF
docker compose -f compose-secrets.yaml up -d
docker compose -f compose-secrets.yaml exec db cat /run/secrets/db_password
docker inspect $(docker compose -f compose-secrets.yaml ps -q db) | grep -i secret
```

**Expected output:** The file inside the container contains the secret; `inspect` shows no secret value — only the mount metadata.

**Why it matters:** "Secrets in env vars leak via docker inspect" is a favorite interview gotcha.

---

## Exercise 6: Scale a service and load-balance

**Goal:** Run 3 copies of an app behind nginx.

**Commands:**
```bash
docker compose -f compose.yaml up -d --scale api=3
docker compose ps api
```

**Expected output:** `docker compose ps api` lists three api containers (`api-1`, `api-2`, `api-3`). Note: scaling a service with a published host port fails — remove the `ports` mapping first.

**Why it matters:** Shows you know Compose's single-host scaling limits and the port-conflict constraint interviewers ask about.

---

## Exercise 7: Debugging a broken stack

**Goal:** Build a repeatable debug workflow.

**Commands:**
```bash
docker compose ps            # which containers are unhealthy/exited?
docker compose logs --tail=50 db   # read one service's logs
docker compose exec db pg_isready  # run a probe inside a container
docker compose config        # verify the merged YAML is what you think
```

**Expected output:** You'll practice the exact sequence — state, logs, probe, config — that resolves most Compose failures in minutes.

**Why it matters:** In interviews, "how do you debug it" matters more than "does it work" — this is your answer.

---

## Exercise 8: Network isolation between tiers

**Goal:** Put the database on a backend-only network unreachable from the web tier.

**Commands:**
```bash
cat > compose-nets.yaml <<'EOF'
networks:
  frontend: {}
  backend: {}
services:
  web:
    image: nginx:alpine
    networks: [frontend]
  api:
    image: hashicorp/http-echo
    command: ["-text=ok"]
    networks: [frontend, backend]
  db:
    image: postgres:16
    environment: {POSTGRES_PASSWORD: secret}
    networks: [backend]
EOF
docker compose -f compose-nets.yaml up -d
docker compose -f compose-nets.yaml exec web ping -c1 db || echo "UNREACHABLE (expected)"
docker compose -f compose-nets.yaml exec api ping -c1 db
```

**Expected output:** `web` cannot reach `db` (DNS/ping fails), while `api` reaches it — tier isolation via networks.

**Why it matters:** Defense in depth for Compose stacks; interviewers ask "how do you stop the web tier touching the DB directly?"

---

## Exercise 9: .env interpolation and variable defaults

**Goal:** Make ports configurable without editing the compose file.

**Commands:**
```bash
echo "WEB_PORT=8081" > .env
sed -i 's/"8080:80"/"${WEB_PORT:-8080}:80"/' compose.yaml
docker compose up -d web
docker compose port web 80
```

**Expected output:** `docker compose port web 80` reports `0.0.0.0:8081`. Delete `.env` and repeat — it falls back to `8080` via the `:-` default.

**Why it matters:** Keeps compose files portable across dev machines — no hardcoded ports in git.

---

## Exercise 10: Clean teardown — and what each flag destroys

**Goal:** Learn the difference between `stop`, `down`, and `down -v`.

**Commands:**
```bash
docker compose stop          # containers stopped, everything kept
docker compose start         # back up instantly
docker compose down          # containers + networks removed, volumes kept
docker compose down -v       # ALSO removes named volumes (data gone)
docker volume ls | grep pg-data || echo "volume destroyed"
```

**Expected output:** After `down -v`, the `pg-data` volume no longer exists — the database from Exercise 2 is gone for good.

**Why it matters:** The most destructive Compose footgun. Know it before it happens in production.

---

*Cleanup: `docker compose -f compose.yaml -f compose-ready.yaml -f compose-secrets.yaml -f compose-nets.yaml down -v` removes every lab artifact.*
