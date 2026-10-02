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
