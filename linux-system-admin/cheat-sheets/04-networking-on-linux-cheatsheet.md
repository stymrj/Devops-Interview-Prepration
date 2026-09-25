# Networking on Linux — Cheat Sheet

*One-page command reference. Pair with the [full guide](../04-networking-on-linux.md).*

## Interfaces & Addresses

```bash
ip -brief addr                  # compact interface/address view
ip -s link                      # per-interface counters, errors, drops
ip addr add 10.0.0.5/24 dev eth0
ip link set eth0 up|down
ip addr show dev eth0
```

Special ranges: `127.0.0.0/8` loopback · `10/8`, `172.16/12`, `192.168/16` RFC 1918 private · `169.254/16` link-local (cloud metadata `169.254.169.254`) · `0.0.0.0` = all interfaces

## Sockets & Connections

```bash
ss -tlnp                        # TCP listening sockets + owning process
ss -tup                         # include UDP
ss -tn state established         # active TCP connections
ss -s                           # socket summary stats
lsof -i :8080                   # alt: what owns port 8080
```

Error decoder: **refused** = reached host, nothing listening · **timed out** = packets dropped in transit (firewall) · **no route** = local routing/gateway problem

## DNS

```bash
dig example.com +short          # just the answer
dig example.com +trace          # full root → TLD → authoritative walk
dig @8.8.8.8 example.com        # bypass local resolver
dig -x 1.2.3.4                  # reverse lookup
host example.com / nslookup example.com
resolvectl status               # systemd-resolved's real config
getent hosts example.com        # full nsswitch stack test
```

Files: `/etc/resolv.conf` (often a symlink — edits may not persist) · `/etc/nsswitch.conf` (lookup order) · `/etc/hosts` (static overrides)

## Routing

```bash
ip route                        # routing table
ip route get 8.8.8.8            # kernel's exact interface+gateway choice
ip route add 10.8.0.0/24 via 10.0.0.5        # temporary (lost on reboot)
ip route add 10.8.0.0/24 via 10.0.0.5 dev tun0
ip rule                         # policy routing rules
```

Persistent routes: netplan (`/etc/netplan/`) or NetworkManager — never `ip route add` in a runbook.

## iptables

```bash
iptables -L -n -v --line-numbers        # rules in order, with hit counters
iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT   # insert at top
iptables -A INPUT -p tcp --dport 80 -j ACCEPT      # append at bottom
iptables -D INPUT 3                     # delete rule #3
iptables -P INPUT DROP                  # default policy: drop unmatched
iptables -F                             # flush all rules (careful!)
iptables -t nat -L -n -v                # NAT table (MASQUERADE, DNAT live here)
```

Targets: `ACCEPT` · `DROP` (silent, client times out) · `REJECT` (ICMP back, client gets refused fast) — first match wins, top to bottom.

## nftables (the engine underneath)

```bash
nft list ruleset                # everything, past the iptables shim
nft add rule inet filter input tcp dport 22 accept
nft flush ruleset               # nuclear option
```

Chains: `INPUT` (to this box) · `OUTPUT` (from this box) · `FORWARD` (passing through — routers, Docker hosts)

## NAT

```bash
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -j MASQUERADE   # containers → internet
iptables -t nat -L -n -v        # verify
```

SNAT/MASQUERADE = many private IPs share one public IP outbound · DNAT = port publish `-p 8080:80`, outside reaches one host inbound

## Namespaces & veth (container networking by hand)

```bash
ip netns add demo
ip netns exec demo ip addr                          # commands inside the namespace
ip link add v-a type veth peer name v-b
ip link set v-a netns demo
ip netns del demo
nsenter -t <pid> -n ip addr                         # enter a container's netns
```

Docker: each container = own netns · veth pair → `docker0` bridge · embedded DNS at `127.0.0.11`

## Packet Capture

```bash
tcpdump -i eth0 -n port 443 and host 10.0.0.5
tcpdump -i any -n -c 100 -w /tmp/cap.pcap port 5432
tcpdump -r /tmp/cap.pcap -n                        # replay without root
```

Flags: `-n` no DNS (fast) · `-c N` stop after N packets · `-w` write pcap for Wireshark · always filter on busy boxes

## Inside-Out Debug Order

`curl localhost:port` → `ss -tlnp` (right interface?) → `dig` (DNS) → `ip route get` (routing) → `iptables -L -n` (firewall) → `tcpdump` both ends (wire)
