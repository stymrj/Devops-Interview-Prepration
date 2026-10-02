# Docker Storage & Volumes — Cheat Sheet

*One-page storage reference. Day 18 of 58.*

## Storage Types at a Glance

| Type | Flag | Survives container? | Lives where |
|---|---|---|---|
| Writable layer | (default) | No — dies with container | Union FS over image layers |
| Named volume | `-v myvol:/data` | Yes | `/var/lib/docker/volumes/` (daemon-managed) |
| Bind mount | `-v /host/path:/data` | Yes (on host path) | Exact host path you named |
| tmpfs mount | `--mount type=tmpfs,…` | No — memory only | Host RAM, never hits disk |

## Essential Commands

```bash
docker volume create myvol                          # create named volume
docker volume ls                                    # list volumes
docker volume inspect myvol                         # mountpoint, driver, labels
docker volume rm myvol                              # delete (must detach first)
docker volume prune                                 # delete ALL unused volumes

docker run -v myvol:/data nginx                     # mount named volume
docker run --mount type=volume,source=myvol,target=/data nginx   # long syntax
docker run -v /host/path:/data:ro nginx             # bind mount, read-only
docker run --mount type=tmpfs,target=/cache,tmpfs-size=100m app  # tmpfs

docker run --rm -v myvol:/data -v /backups:/b alpine tar czf /b/myvol.tar.gz -C /data .
docker run --rm -v myvol:/data -v /backups:/b alpine tar xzf /b/myvol.tar.gz -C /data
docker system df                                    # see space per object type
docker system df -v                                 # per-container breakdown
docker system prune -a --volumes                    # full cleanup hammer
```

## Rules That Answer Interviews

- Container writes → writable layer → **gone with the container**. Data that matters → volume.
- First write to an image file triggers **copy-on-write** (copy-up) — expensive; hot data belongs on volumes.
- **Empty** named volume mounted over an image dir gets **populated from the image once**; a non-empty one shadows the image forever.
- Volumes decouple data lifecycle from container lifecycle — that's the whole upgrade mechanic.
- Back up via a **throwaway container + tar**, never by copying `/var/lib/docker/volumes` by hand.
- Bind mounts = **dev code and host-owned files** (non-portable, permission headaches). Volumes = app data.
- tmpfs = memory-only; ideal for **secrets and scratch** that must never touch disk.
- Readers that only consume shared data mount with **:ro**.
- `docker commit` is a **debugging snapshot**, not a backup and not a deployment path.
- Disk-full on a Docker host? Check **log growth** (set `max-size`/`max-file` in daemon.json) and **prune dead volumes**.
