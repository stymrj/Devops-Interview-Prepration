# Docker Deep Dive — Cheat Sheet

*One-page architecture reference. Day 15 of 58.*

## The Stack (top to bottom)

| Layer | Role | Talks to |
|---|---|---|
| Docker CLI | API client, UX | dockerd via `/var/run/docker.sock` |
| dockerd (daemon) | API, builds, networks, volumes | containerd |
| containerd | Container lifecycle, image store, pulls | runc |
| runc | OCI runtime: creates the Linux process | kernel (namespaces + cgroups) |

Kubernetes skips Docker entirely and talks to containerd directly (dockershim removed in K8s 1.24).

## `docker run nginx` — the lifecycle

1. CLI → `POST /containers/create` on the Unix socket
2. dockerd checks local image; pulls via containerd if missing
3. Filesystem assembled from image layers + thin writable layer (overlay2)
4. Namespaces created (pid, net, mnt, uts, ipc, user), cgroups applied, network attached
5. containerd → runc execs the binary as **PID 1** in its own namespaces
6. Container lives exactly as long as PID 1

## Namespaces (what the container *sees*)

| Namespace | Isolates | Punch-through flag |
|---|---|---|
| pid | Process IDs | `--pid=host` |
| net | Network stack, interfaces | `--network=host` |
| mnt | Mount table | `-v` / `--mount` |
| uts | Hostname, domain | `--hostname` |
| ipc | Shared memory, semaphores | `--ipc=host` |
| user | UID/GID mapping | `--userns` |

Same process, two views: PID 1 inside, some host PID outside.

## cgroups (what the container *gets*)

| Flag | Enforces |
|---|---|
| `--memory=512m` | RAM cap → OOM-kill (exit 137) on breach |
| `--memory-swap=1g` | RAM + swap combined cap |
| `--cpus=1.5` | CPU time quota |
| `--pids-limit=100` | Max processes (fork-bomb guard) |

Rule of thumb: namespaces lie about what's there, cgroups ration what's available.

## Images & layers

- Image = read-only layers (SHA256 content-addressed) + config metadata
- Each Dockerfile instruction → one layer; BuildKit caches by content hash
- Layers shared across images; pulls download only missing layers
- Copy-on-write: container writes go to a thin top layer, image layers untouched
- `docker history <img>` — list layers; `docker diff <c>` — writable-layer changes
- Tag = mutable pointer; **digest = immutable identity** → pin digests in prod

## Registries

- `docker pull` = fetch manifest → download missing layers → verify SHA256
- Multi-arch: registry serves the manifest matching your platform
- `docker images --digests` — see the digest each tag points to

## Storage paths (default install)

| What | Where |
|---|---|
| Everything | `/var/lib/docker` |
| Volumes | `/var/lib/docker/volumes` |
| Storage driver | overlay2 |

`docker rm` deletes the writable layer — data outside volumes is gone. Stopped containers still occupy disk.

## Security one-liners

- Containers share the host kernel — kernel exploit = potential escape
- `USER` directive in Dockerfile; never run as root if avoidable
- `--cap-drop ALL` then `--cap-add` only what's needed
- Read-only root FS: `--read-only` + tmpfs for scratch space

## Interview soundbites

- "VMs virtualize hardware; containers virtualize the OS."
- "dockerd is management, containerd is the runtime, runc creates the process."
- "The container IS the process — kill PID 1, the container exits."
- "Namespaces isolate the view, cgroups ration the resources."
- "Tags move, digests don't."

*Pair with the [guide](../15-docker-deep-dive.md) and [lab](../labs/15-docker-deep-dive-lab.md).*
