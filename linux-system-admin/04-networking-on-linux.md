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
