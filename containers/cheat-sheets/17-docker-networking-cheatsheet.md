# Docker Networking — Cheat Sheet

*One-page networking reference. Day 17 of 58.*

## Network Drivers at a Glance

| Driver | Command | Key fact |
|---|---|---|
| bridge (default) | `docker run …` (no flag) | docker0, NAT outbound, **no DNS between containers** |
| user-defined bridge | `--network mynet` | Embedded DNS (127.0.0.11), name resolution, isolation |
| host | `--network host` | Shares host namespace, no port mapping, no isolation |
| none | `--network none` | Loopback only — airtight isolation |
| overlay | `-d overlay` (Swarm) | VXLAN tunnels over UDP 4789, multi-host |
| macvlan | `-d macvlan` | Containers get real LAN IPs (rare, legacy apps) |

## Essential Commands

```bash
docker network ls                                   # list networks
docker network create mynet                         # custom bridge (gets DNS)
docker network create -d overlay --attachable onet # Swarm-ready overlay
docker network inspect mynet                        # subnets, containers, DNS
docker network connect mynet container1             # attach to 2nd network
docker network disconnect mynet container1          # detach
docker network rm mynet                             # delete (must detach first)

docker run -d --name web --network mynet nginx      # attach at start
docker run -d -p 8080:80 nginx                      # host:container port map
docker run -d -p 127.0.0.1:8080:80 nginx            # localhost-only bind
docker run -d -P nginx                              # random host ports for all EXPOSEd
```

## DNS & Debugging

```bash
docker exec web getent hosts db       # test name resolution
docker exec web nslookup db           # fuller DNS debug
docker exec web cat /etc/resolv.conf  # shows nameserver 127.0.0.11
docker exec db ss -tlnp               # what is the app ACTUALLY bound to?
docker network inspect mynet --format '{{range .Containers}}{{.Name}} {{end}}'
```

## Rules That Answer Interviews

- Default bridge → IPs only, no names. Custom network → names work.
- Embedded DNS lives at **127.0.0.11** inside each container.
- `EXPOSE` = documentation only. `-p` = the actual NAT rule.
- Unscoped `-p 8080:80` binds **0.0.0.0** (all interfaces). Scope with `-p 127.0.0.1:8080:80`.
- App must bind **0.0.0.0** inside the container — 127.0.0.1 there is container-only.
- `localhost` in a container = the container itself. Cross-container = **names**.
- Debug order: same network → DNS resolves → app bound to 0.0.0.0.
- Overlay = VXLAN over UDP **4789**, needs Swarm mode.
- Swarm published port opens on **every node** (ingress routing mesh); `--publish mode=host` bypasses it.
- Networks are open by default — segmentation gives isolation; real policy needs k8s NetworkPolicies / service mesh.
