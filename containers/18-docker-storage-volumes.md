# Docker Storage & Volumes Interview Preparation Guide

*How to Answer Docker Storage Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [How Docker Storage Actually Works](#how-docker-storage-actually-works)
2. [Named Volumes Deep Dive](#named-volumes-deep-dive)
3. [Bind Mounts and Tmpfs Mounts](#bind-mounts-and-tmpfs-mounts)
4. [Data Persistence Backup and Restore](#data-persistence-backup-and-restore)
5. [Storage Performance and Best Practices](#storage-performance-and-best-practices)
6. [Common Interview Traps](#common-interview-traps)

---

## How Docker Storage Actually Works

### Q1: Where does data go when a container writes to disk?

**How to Answer:**

"It goes into the container's writable layer, which sits on top of the read-only image layers. Docker stacks these with a union filesystem driver — overlay2 on most Linux setups — so the container sees one merged filesystem. The moment a container writes to a file that exists in the image, the driver copies it up into the writable layer — that's copy-on-write.

But that layer is tied to the container's life. Delete the container, and the writable layer is gone. That's the default ephemeral behavior — fine for logs and caches, fatal for data you care about. Anything that must survive needs a volume or a bind mount."

**Key Point:** "Container writes land in an ephemeral writable layer over the image's read-only layers — delete the container and that data dies with it."

---

### Q2: What is the copy-on-write mechanism? Why should I care in an interview?

**How to Answer:**

"Copy-on-write means multiple containers can share the same image layers on disk without duplicating them — only the layers a container actually modifies get copied into its own writable layer. That's why pulling ten nginx containers costs you one image's worth of storage, not ten.

I care about it because it explains performance behavior: the first write to a big file is expensive (a full copy-up happens), and lots of small writes across layers can fragment. It's also why you should never write hot data into the writable layer in production — volumes bypass the driver entirely and talk to the host filesystem directly."

**Key Point:** "Copy-on-write shares image layers between containers, but the first write to any file pays a full copy-up cost — hot data belongs on volumes, not the writable layer."

---

## Named Volumes Deep Dive

### Q3: What is a named volume, and why is it the default choice for persistent data?

**How to Answer:**

"A named volume is storage the Docker daemon manages for you — `docker volume create mydata` carves it out under /var/lib/docker/volumes on the host. You mount it into containers with `-v mydata:/data` or the newer `--mount type=volume,source=mydata,target=/data`. It survives container restarts, recreations, and image upgrades.

It's the default choice because the daemon handles the messy parts: permissions play nicely, it works identically on any host, and you never depend on host paths that may not exist somewhere else. Named volumes are for data; bind mounts are for host files."

**Key Point:** "Named volumes are daemon-managed persistent storage that survive container lifecycles — the default home for any data that must outlive the container."

---

### Q4: Can multiple containers share one named volume? What breaks?

**How to Answer:**

"Yes — mount the same named volume into as many containers as you want. That's how people run things like a data-only pattern or a backup sidecar that reads what the app container writes. Docker won't stop you.

What breaks is concurrency. There's no distributed lock — two containers writing the same file is a race, and even databases only tolerate shared volumes in clustered setups designed for it. I share volumes for read-write plus read-only consumers, or for sequential jobs like app-then-backup. Parallel writers without coordination will corrupt each other."

**Key Point:** "Docker lets many containers share a volume but gives you zero locking — share freely for sequential or read-mostly patterns, never for unsynchronized parallel writers."

---

## Bind Mounts and Tmpfs Mounts

### Q5: When do you use a bind mount instead of a named volume?

**How to Answer:**

"A bind mount maps a specific host path into the container — `-v /home/me/code:/app`. It's the right call when the data lives on the host and needs to stay there: mounting source code for live reload in development, reading host config files, or sharing a directory the host already owns.

But I never use them for production data. Bind mounts depend on the host's directory structure, so the compose file or run command isn't portable, and you fight host permission issues constantly. Volumes abstract all that away. Bind mounts are for development and host-owned files; volumes are for data the app owns."

**Key Point:** "Bind mounts point at explicit host paths — great for dev code and host files, wrong for production data where portability and daemon-managed permissions matter."

---

### Q6: What is a tmpfs mount and when does it matter?

**How to Answer:**

"A tmpfs mount keeps data in the host's memory instead of on disk — `--mount type=tmpfs,target=/app/cache`. Nothing is ever written to the container's writable layer or to disk, so reads and writes are fast and nothing persists past container death. When the container stops, the data evaporates completely.

It matters for sensitive or hot temp data: session caches, scratch space, anything you explicitly don't want landing on disk. Secrets mounted with `--mount type=tmpfs` are the cleanest way to keep credentials out of both the writable layer and any image layer. If persistence would be a bug, tmpfs is the answer."

**Key Point:** "Tmpfs mounts live only in host memory — fast, never persisted, invisible to disk — ideal for caches and secrets that must never touch storage."

---

## Data Persistence Backup and Restore

### Q7: How do you back up a named volume properly?

**How to Answer:**

"The standard pattern is a throwaway container that mounts the volume plus a host backup directory: `docker run --rm -v mydata:/data -v /backups:/backup alpine tar czf /backup/mydata.tar.gz -C /data .` That gives you a portable tarball without installing anything on the host. I script this into cron and push the tarballs to object storage.

The trap is trying to back up by copying /var/lib/docker/volumes directly — that's the daemon's internal layout and it can change between Docker versions, plus you might copy mid-write state. Always go through a container mount so Docker's own filesystem view is what you read."

**Key Point:** "Back up volumes through a throwaway container with tar, never by copying the daemon's internal /var/lib/docker layout — restore is the reverse command with xzf."

---

### Q8: Walk me through upgrading a stateful container without losing data.

**How to Answer:**

"Say Postgres 15 to 16: the data lives on a named volume, so I stop the old container and start the new image with the same volume — `docker run -d -v pgdata:/var/lib/postgresql/data postgres:16`. The container is gone but the volume is untouched, so the new one mounts the exact same data directory.

The risk is schema and format compatibility — a new major version may need the data migrated first, and Docker knows nothing about that. So I back up the volume with the tar pattern before touching anything, test the new image against a volume copy, and only then run it for real. The upgrade mechanic is trivial; the data-safety discipline around it is the actual interview answer."

**Key Point:** "Volumes decouple the container lifecycle from the data lifecycle — upgrade the image by re-attaching the same volume, but back up first because Docker can't help with format incompatibilities."

---

## Storage Performance and Best Practices

### Q9: What actually slows down Docker storage, and how do you fix it?

**How to Answer:**

"The classic one is lots of small writes to the writable layer — every first-write to a file triggers a copy-up from the image layer, which on overlay2 is slow and fragments over time. Databases on the writable layer are the poster child for this: you see degraded IOPS that disappears the moment you move the data directory onto a volume.

The fix is boring and effective: volumes for hot data, `.dockerignore` to keep the build context tight, and never generate huge numbers of tiny layers. On the host side, the daemon wants a filesystem that supports the driver well — ext4 or xfs on an SSD for overlay2 — and enough free space that garbage collection never kicks in under pressure. And run `docker system prune` on a schedule, because dangling images and dead volumes eat space silently."

**Key Point:** "Writes through the overlay driver pay copy-up costs — move hot data to volumes, prune dead storage on schedule, and keep the daemon on a fast filesystem with headroom."

---

### Q10: How do you handle storage limits and runaway disk usage on a Docker host?

**How to Answer:**

"The tooling is `docker system df` to see what's actually consuming space — images, containers, local volumes — broken out with reclaimable amounts. `docker system prune -a --volumes` is the big hammer: it removes stopped containers, unused networks, dangling images, and unreferenced volumes, which routinely recovers tens of gigs on dev hosts.

In production I don't rely on manual pruning. Daemon-level defaults like `--storage-opt` quotas, log rotation config in daemon.json so container logs don't fill the disk, and monitoring on the Docker data-root partition with alerts at 75 and 90 percent. The classic 2 AM outage is a full /var/lib/docker partition because nobody capped log growth — I configure max-size and max-file on every long-lived workload."

**Key Point:** "Cap log growth and monitor the data-root partition — disk-full on a Docker host almost always traces back to unbounded logs or never-pruned volumes."

---

## Common Interview Traps

### Q11: "I mounted a volume, but my container still shows old data." What do you check?

**How to Answer:**

"First I check whether the volume actually contains what I think it does — and specifically whether I hit the empty-volume initialization trap. If you mount an empty named volume over a directory that already had files in the image, Docker copies the image's content into the volume the first time. That's a one-time populate. If the volume was NOT empty, that copy never happens and the image's files stay hidden.

So stale data usually means the volume was initialized from an older image version and kept shadowing it. I verify with `docker volume inspect` and by mounting the volume into a temp container to look directly. The fix is recreating the volume or re-seeding it — and the design lesson is to never bake mutable config into images that volumes will shadow."

**Key Point:** "An existing named volume shadows the image directory forever — Docker only populates a volume from image content on its very first, empty use."

---

### Q12: Is `docker commit` a backup strategy? Defend your answer.

**How to Answer:**

"No — `docker commit` snapshots the writable layer into a new image, which is a state artifact, not a backup. It bakes whatever ad-hoc mutations someone made into an opaque layer, it duplicates all the data into the image store, and it tells you nothing about whether the captured state was consistent. For data backup, volumes with the tar pattern win every time: portable, versioned, restorable anywhere.

Where `docker commit` is defensible is debugging: freeze a broken container's exact state into an image so someone else can inspect the failure. But anything going near production through commit is a process failure — it should have been a Dockerfile or an image from CI. Commit is a debugger, not a deployment tool."

**Key Point:** "docker commit is a debugging snapshot of an ephemeral layer, not a backup — use it to freeze broken state for inspection, never as a data-protection or deployment mechanism."

---

*Day 18 of 58 — Containers.*
