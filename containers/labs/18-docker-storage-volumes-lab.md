# Docker Storage & Volumes — Hands-On Lab

*9 exercises to make Docker volumes, bind mounts, and the copy-on-write layer click. Run on any Linux machine or VM with Docker installed.*

**Prereqs:** Docker Engine 24+ (`docker --version`), a shell, ~20 minutes.

---

## Exercise 1: Prove the writable layer is ephemeral

**Goal:** Watch container data die with the container.

**Commands:**

```bash
docker run -d --name ephemeral nginx:alpine
docker exec ephemeral sh -c "echo 'doomed data' > /usr/share/nginx/html/important.txt"
docker exec ephemeral cat /usr/share/nginx/html/important.txt
docker rm -f ephemeral
docker run -d --name ephemeral2 nginx:alpine
docker exec ephemeral2 ls /usr/share/nginx/html/ | grep important || echo "DATA GONE"
docker rm -f ephemeral2
```

**Expected output:** The file exists in the first container, and is absent after recreate.

**Why it matters:** This is the foundational fact behind every volume decision. Data written to the writable layer is temporary by design.

---

## Exercise 2: Create a named volume and survive a recreate

**Goal:** See data persist across container recreation.

**Commands:**

```bash
docker volume create appdata
docker run -d --name db1 -v appdata:/data alpine sh -c "echo hello-persist > /data/marker.txt && sleep 300"
docker rm -f db1
docker run -d --name db2 -v appdata:/data alpine sleep 300
docker exec db2 cat /data/marker.txt
```

**Expected output:** `hello-persist` — the file survived the container swap.

**Why it matters:** Named volumes decouple data lifecycle from container lifecycle. This is the upgrade-without-loss mechanic from the guide.

---

## Exercise 3: See where volumes actually live on the host

**Goal:** Connect the daemon's view to the host filesystem.

**Commands:**

```bash
docker volume inspect appdata --format '{{.Mountpoint}}'
sudo ls "$(docker volume inspect appdata --format '{{.Mountpoint}}')" 
docker system df -v | head -20
```

**Expected output:** A Mountpoint like `/var/lib/docker/volumes/appdata/_data`; `ls` shows `marker.txt` there.

**Why it matters:** Makes volumes concrete instead of magic — and shows why you should never hand-edit that path (backups go through a container mount).

---

## Exercise 4: Empty volume initialization — the populate trap

**Goal:** Watch Docker copy image content into an empty volume once.

**Commands:**

```bash
docker volume create populate-me
docker run --rm -v populate-me:/usr/share/nginx/html nginx:alpine ls /usr/share/nginx/html
docker run --rm -v populate-me:/usr/share/nginx/html alpine sh -c "echo override > /usr/share/nginx/html/index.html"
docker run --rm -v populate-me:/usr/share/nginx/html nginx:alpine ls /usr/share/nginx/html
```

**Expected output:** First run lists nginx's default files (populated from the image); third run still lists only them — the old `index.html` from the image is hidden by the volume.

**Why it matters:** A non-empty volume shadows the image directory forever. This is the "container shows old data" trap from Q11, felt firsthand.

---

## Exercise 5: Bind mount for live code reload

**Goal:** Feel the dev-loop advantage of bind mounts.

**Commands:**

```bash
mkdir -p /tmp/bindlab && echo "version one" > /tmp/bindlab/index.html
docker run -d -p 8080:80 --name binddev -v /tmp/bindlab:/usr/share/nginx/html:ro nginx:alpine
curl -s http://localhost:8080
echo "version two" > /tmp/bindlab/index.html
curl -s http://localhost:8080
docker rm -f binddev
```

**Expected output:** First curl prints "version one", second prints "version two" — no rebuild needed.

**Why it matters:** Bind mounts are the right tool for dev iteration. The `:ro` flag is the production habit to note — read-only whenever the container shouldn't write back.

---

## Exercise 6: Tmpfs mount — data that evaporates

**Goal:** Confirm tmpfs contents never touch disk.

**Commands:**

```bash
docker run -d --name tmpfstest --mount type=tmpfs,target=/cache,tmpfs-size=100m alpine sleep 300
docker exec tmpfstest sh -c "echo secret > /cache/token && df -h /cache && ls /cache"
docker rm -f tmpfstest
docker run --rm --mount type=tmpfs,target=/cache alpine ls /cache || echo "EMPTY — evaporated"
```

**Expected output:** `df` shows a tmpfs filesystem sized 100m; after container removal the cache is gone — nothing to recover.

**Why it matters:** This is how you keep secrets and scratch data off disk. Know the flags; the question comes up in security-tinged storage interviews.

---

## Exercise 7: Share a volume between two containers

**Goal:** See the app-sidecar pattern in action.

**Commands:**

```bash
docker run -d --name writer -v appdata:/data alpine sh -c "while true; do date > /data/tick.txt; sleep 2; done"
docker run --rm -v appdata:/data:ro alpine cat /data/tick.txt
docker rm -f writer
```

**Expected output:** The reader container sees the timestamp written by the writer.

**Why it matters:** Volumes are how sidecars consume app data. The `:ro` on the consumer is the pattern interviewers want to hear.

---

## Exercise 8: Back up and restore a volume with tar

**Goal:** Practice the canonical backup pattern.

**Commands:**

```bash
docker run --rm -v appdata:/data -v /tmp:/backup alpine tar czf /backup/appdata.tar.gz -C /data .
ls -lh /tmp/appdata.tar.gz
docker volume rm appdata   # simulate disaster (after the backup!)
docker volume create appdata
docker run --rm -v appdata:/data -v /tmp:/backup alpine tar xzf /backup/appdata.tar.gz -C /data
docker run --rm -v appdata:/data alpine ls /data
```

**Expected output:** tarball created in /tmp; after wipe-and-restore, the files are back.

**Why it matters:** This two-command pair is the answer to half of all "how do you protect Docker data" interview questions. Muscle-memory it.

---

## Exercise 9: Prune dead storage and see the savings

**Goal:** Learn what `docker system df` and prune reclaim.

**Commands:**

```bash
docker volume create junk1 && docker volume create junk2
docker system df
docker volume prune -f
docker system df
docker volume rm appdata
rm -f /tmp/appdata.tar.gz
```

**Expected output:** `docker system df` shows reclaimed space after prune; `junk1`/`junk2` gone, `appdata` removed.

**Why it matters:** Unused volumes are silent disk killers on Docker hosts. Prune-on-schedule is a real production hygiene answer, not just a lab trick.

---

*Day 18 of 58 — Containers.*
