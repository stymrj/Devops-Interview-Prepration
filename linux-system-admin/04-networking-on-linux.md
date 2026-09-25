# Networking on Linux Interview Preparation Guide

*How to Answer Networking on Linux Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Interfaces and the ip Command](#interfaces-and-the-ip-command)
2. [Listening Sockets and Active Connections](#listening-sockets-and-active-connections)
3. [DNS Deep Dive](#dns-deep-dive)
4. [Packet Filtering with iptables and nftables](#packet-filtering-with-iptables-and-nftables)
5. [Routing and NAT](#routing-and-nat)
6. [Network Namespaces and Container Networking](#network-namespaces-and-container-networking)
7. [Debugging Network Issues in Production](#debugging-network-issues-in-production)

---

## Interfaces and the ip Command

### Q1: Everyone says ifconfig is deprecated — what do you use instead, and why?

**How to Answer:**

"I use the `ip` command from iproute2. `ifconfig` and `netstat` are deprecated on Linux and they're blind to half the modern network state.

`ip addr` shows every interface with its addresses, state, and MTU — it's the first command I run on any box. `ip link set eth0 up` and `ip addr add` handle bring-up and addressing without restarting anything.

The real reason interviewers care: `ip` exposes things ifconfig never did — secondary addresses, policy routing with `ip rule`, and full tunnel state. If you only know ifconfig, you're working with a 20-year-old view of the box."

```bash
ip -brief addr     # compact view: interface, state, addresses
ip -s link         # per-interface packet and error counters
```

**Key Point:** "`ip addr` is your first command on any box. ifconfig is deprecated and blind to modern networking state."

### Q2: Explain the special IP ranges — 127.0.0.0/8, 10.0.0.0/8, 169.254.0.0/16. When do you bump into each?

**How to Answer:**

"Loopback `127.0.0.0/8` never leaves the box. I bind health checks and local dev servers to `127.0.0.1` so they're unreachable from the network by design.

Private ranges like `10.0.0.0/8` and `192.168.0.0/16` are RFC 1918 — routable inside your VPC, not on the public internet. Your cloud bill depends on knowing the difference between private and public traffic.

Link-local `169.254.0.0/16` is the interesting one. Cloud metadata endpoints like `169.254.169.254` live there, and it's also the fallback when DHCP fails and the box self-assigns an address.

The classic trap: binding to `0.0.0.0` means *all* interfaces including public. I've seen people accidentally expose admin panels to the internet that way."

**Key Point:** "Loopback stays local, RFC 1918 is your private VPC space, and 169.254 is cloud metadata plus DHCP-fallback. Know what `0.0.0.0` really binds to."

---

## Listening Sockets and Active Connections

### Q3: How do you figure out which process is listening on a port?

**How to Answer:**

"`ss -tlnp` — TCP, listening, numeric, with process info. That's the one command.

`netstat` is deprecated and slower; `ss` reads straight from the kernel. The `-p` flag is the whole point — it shows the PID and process name, which is what you're actually after.

If nothing's listening, that's your answer to 'connection refused' right there — the port has no owner. I check this before touching firewalls or DNS, because half of all 'network issues' are just a dead service."

```bash
ss -tlnp | grep ':80'     # who's listening on port 80?
ss -tup                   # UDP sockets too — DNS, DHCP, WireGuard
```

**Key Point:** "`ss -tlnp` tells you who's listening and which process owns the port. Half of all 'network issues' are just nothing listening."

### Q4: Connection refused vs connection timed out vs no route to host — what's each one telling you?

**How to Answer:**

"Refused means you reached the box but nothing's listening on that port — the service is down or bound to the wrong interface. That's an app problem.

Timed out means your packets vanished somewhere — a firewall dropping them silently, a wrong route, or the host genuinely unreachable. That's the network in between.

No route to host means your *own* box has no path — routing table or gateway problem, so I check `ip route`.

I debug in that order and I never skip it. Refused is the app, timeout is the middle, no-route is local routing. Knowing which error you got cuts the search space in half immediately."

**Key Point:** "Refused = reached the box, nothing listening. Timeout = packets vanished in transit. No route = your own box has no path. The error tells you where to look."

---

## DNS Deep Dive

### Q5: Walk me through what happens when you type curl example.com.

**How to Answer:**

"First the resolver checks its cache, then `/etc/nsswitch.conf` decides the lookup order — usually DNS via the nameservers in `/etc/resolv.conf`.

Your query hits a recursive resolver, which walks the chain: root servers, then the .com TLD servers, then example.com's authoritative servers. Each level returns a referral until you get the A or AAAA record, and the resolver caches it for the TTL.

Then curl opens a TCP connection to that IP and does the TLS handshake. The interview point: DNS is a distributed lookup with caching at every layer — and most 'slow first request' mysteries are TTL or resolver latency, not your app."

**Key Point:** "DNS walks root → TLD → authoritative, caching at every layer. Slow first requests are usually resolver latency or TTL, not the app."

### Q6: DNS is broken on a box. How do you debug it?

**How to Answer:**

"I start with `dig example.com` and read the status line — SERVFAIL, NXDOMAIN, and timeout each point somewhere different. NXDOMAIN means the name genuinely doesn't exist; timeout means the resolver isn't answering at all.

Then I check what's actually resolving: `cat /etc/resolv.conf` for the nameservers, and on systemd boxes `resolvectl status` because systemd-resolved can override that file entirely.

The classic gotcha: `/etc/resolv.conf` is a symlink managed by something else, and my edit gets wiped on reboot. And if dig works with `@8.8.8.8` but not the default resolver, the problem is the local resolver, not DNS itself — that distinction saves a lot of time."

```bash
dig example.com +short          # just the answer
dig @8.8.8.8 example.com        # bypass the local resolver
resolvectl status               # what systemd-resolved is actually doing
```

**Key Point:** "Read dig's status line first, then check which resolver you're actually using. If `@8.8.8.8` works but the default doesn't, it's your local resolver, not DNS."

---

## Packet Filtering with iptables and nftables

### Q7: Explain the iptables chains. Where does a packet actually go?

**How to Answer:**

"Three built-in chains in the filter table. INPUT is for packets destined to this box, OUTPUT is for packets this box generates, FORWARD is for packets passing through — like on a router or a Docker host.

A packet enters one chain, walks the rules top to bottom, and the first match wins — ACCEPT, DROP, or REJECT. The default policy at the end catches everything unmatched, and I always set INPUT's policy to DROP on servers.

The trap people fall into: appending an ACCEPT rule *after* a broad DROP. Order is everything — rules are read top-down, not by best match. A misplaced rule is silently dead."

```bash
iptables -L -n -v --line-numbers   # rules in order, with hit counters
```

**Key Point:** "INPUT, OUTPUT, FORWARD — first match wins, top to bottom. A rule placed after a broad DROP is silently dead."

### Q8: iptables vs nftables — what's the actual difference, and what should I know?

**How to Answer:**

"nftables is the modern kernel packet-filtering framework that replaces iptables, ip6tables, arptables, and ebtables with a single engine. On most distros today, the `iptables` command is a compatibility shim translating to nftables under the hood.

Rules-wise I still think in chains and matches, but nftables lets you combine IPv4 and IPv6 in one table with the `inet` family. That kills a whole class of 'forgot the v6 rule' bugs.

In interviews I say it straight: know iptables syntax cold because that's what you'll see in the wild, and know that nftables is what's actually running underneath."

```bash
nft list ruleset     # see what's REALLY configured, past the shim
```

**Key Point:** "Know iptables syntax — it's what you'll see in the wild. Know nftables is the engine underneath, unifying v4 and v6 in one ruleset."

---

## Routing and NAT

### Q9: How does NAT work? Why does MASQUERADE matter for containers?

**How to Answer:**

"NAT rewrites addresses as packets cross a boundary. SNAT rewrites the source — that's what MASQUERADE does, mapping many private IPs to the host's public IP on the way out.

Your containers get private IPs like 172.17.0.x. Without MASQUERADE on the Docker bridge, their reply packets would never route back — return traffic needs the host's address to find its way home.

DNAT rewrites the destination — that's port publishing. `-p 8080:80` turns traffic hitting the host into traffic for a specific container. The one-liner: SNAT is how many hosts share one public IP outbound, DNAT is how the outside reaches one host inbound."

**Key Point:** "MASQUERADE (SNAT) lets many private IPs share one public IP outbound — it's why containers can reach the internet. DNAT (port publishing) is the inbound path."

### Q10: How do you read the routing table, and when would you add a route?

**How to Answer:**

"`ip route` prints it — destination, gateway via, device, and source. The `default via 10.0.0.1 dev eth0` line is the default gateway, where everything unknown goes.

I add routes when a box needs to reach a network through a specific next hop — like a VPN subnet via the tunnel interface, `ip route add 10.8.0.0/24 via 10.0.0.5`.

The debugging trick: `ip route get 8.8.8.8` shows exactly which interface and gateway the kernel would pick. No guessing. And routes added with `ip route add` vanish on reboot — persistent routes go in netplan or NetworkManager config, never in a runbook command."

```bash
ip route get 8.8.8.8     # which interface + gateway the kernel picks
```

**Key Point:** "`ip route get` shows the kernel's exact choice — no guessing. `ip route add` is temporary; persistent routes live in netplan or NetworkManager."

---

## Network Namespaces and Container Networking

### Q11: What are network namespaces? How does Docker use them?

**How to Answer:**

"A network namespace is an isolated copy of the network stack — its own interfaces, IPs, routing table, and firewall rules. `ip netns add demo` creates one, and processes inside it can't see the host's interfaces at all.

Docker gives every container its own netns. That's why two containers can both bind to port 80 without conflicting — they're in different namespaces, so 'port 80' means something different in each.

When I debug container networking I use `nsenter` or `docker exec` to run commands inside that namespace. The key insight: a container's 'network' is just a namespace plus a virtual cable (a veth pair) plugged into the host."

**Key Point:** "A netns is a private network stack — interfaces, routes, firewall. Every container gets one, which is why port conflicts don't happen across containers."

### Q12: How do two containers on the same Docker network actually talk to each other?

**How to Answer:**

"Each container gets a veth pair — one end inside the container's namespace as eth0, the other end plugged into the docker0 bridge on the host. The bridge acts like a virtual switch, forwarding frames between the veth ends at layer 2.

Docker also runs an embedded DNS server at 127.0.0.11 inside each container, so containers resolve each other by name — that's why `ping web` works from the `api` container.

So the path is: eth0 → veth → bridge → veth → other container's eth0. All on the host, never touching the real network. When it breaks, I check the bridge with `ip addr show docker0` and the container's routes with `docker exec <name> ip route`."

**Key Point:** "veth pairs plug each container into the docker0 bridge — a virtual switch. Docker's embedded DNS at 127.0.0.11 gives you name resolution between containers."

---

## Debugging Network Issues in Production

### Q13: When do you reach for tcpdump, and how do you use it without drowning in packets?

**How to Answer:**

"tcpdump is the last resort that never lies — when logs and metrics disagree, I capture the actual wire. The trick is filtering hard so you see only the conversation you care about.

`-n` skips DNS resolution so it stays fast, and `-w capture.pcap` saves it for Wireshark instead of reading hex in the terminal. I always capture on both sides when I can — comparing what left the client versus what arrived at the server tells you exactly where packets die.

One warning: on a busy box, unfiltered tcpdump fills your disk in minutes. Always filter, always bound it with `-c`."

```bash
tcpdump -i eth0 -n -c 100 port 443 and host 10.0.0.5   # one conversation only
tcpdump -i any -w /tmp/cap.pcap port 5432             # save for Wireshark
```

**Key Point:** "tcpdump never lies — filter hard, capture on both ends, and compare. Unfiltered captures on a busy box will fill your disk in minutes."

### Q14: A service is unreachable. Walk me through your exact debugging order.

**How to Answer:**

"First I reproduce from the box itself — `curl localhost:port` tells me if the app is even alive. That splits the problem in half immediately.

Then `ss -tlnp` to confirm something's listening on the right interface. Bound to 127.0.0.1 instead of 0.0.0.0 is the classic — works locally, unreachable remotely.

Next DNS with `dig`, then routing with `ip route get`, then firewall with `iptables -L -n` — each layer in order, never skipping. If all of that looks right, tcpdump on both ends to see where packets actually die.

The principle: test locally first, then walk outward — app, socket, DNS, route, firewall, wire. You'll find it in minutes instead of hours."

**Key Point:** "Work inside-out: app, socket, DNS, route, firewall, wire. Each layer in order — skipping layers is how 10-minute problems become 3-hour ones."

---

*End of guide — practice saying these out loud until they sound like you.*
