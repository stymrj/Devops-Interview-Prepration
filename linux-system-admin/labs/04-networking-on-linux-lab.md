# Networking on Linux — Hands-On Lab

*Pair this with the [Networking on Linux interview guide](../04-networking-on-linux.md). Do every exercise on a VM you can break — a disposable Ubuntu/Debian box is ideal. Exercises 5–8 need sudo. Nothing here should touch a production box.*

---

## Exercise 1: Read Your Interfaces Cold

**Goal:** Decode `ip addr` output without thinking.

```bash
ip -brief addr
ip -s link
ip addr show lo
```

**Observe:** Each interface shows state (`UP`/`DOWN`), MTU, and addresses with prefix lengths (`/24`). `lo` always has `127.0.0.1/8`. Note the `qlen`, RX/TX counters, and any `errors` or `dropped` in `ip -s link` — non-zero drops are a real-world smoking gun.

**Why this matters in interviews/ops:** "The box can't reach anything" starts with `ip -brief addr`. If the interface is DOWN or has no address, nothing downstream matters.

---

## Exercise 2: Find Who's Listening

**Goal:** Map ports to processes with `ss`.

```bash
sudo python3 -m http.server 8888 &
ss -tlnp | grep ':8888'
ss -tup
kill %1
```

**Observe:** `-p` shows the PID and process name (`users:(("python3",pid=...))`). After `kill`, the socket disappears — if it doesn't, something else owns the port.

**Why this matters in interviews/ops:** "Connection refused" is answered by `ss -tlnp` in five seconds. This is the single most-used network debugging command in production.

---

## Exercise 3: Trace a Real DNS Lookup

**Goal:** See every step of resolution with `dig`.

```bash
dig example.com +trace
dig example.com +short
dig @8.8.8.8 example.com +noall +stats
```

**Observe:** `+trace` walks root → TLD → authoritative, showing each referral. `+stats` shows query time and which server answered — compare your default resolver's time against 8.8.8.8.

**Why this matters in interviews/ops:** "The first request is slow" is often resolver latency. `dig +stats` proves it in one command instead of guessing.

---

## Exercise 4: Break DNS and Fix It

**Goal:** Feel what a bad resolver looks like, then recover.

```bash
cp /etc/resolv.conf /tmp/resolv.conf.bak
echo "nameserver 192.0.2.1" | sudo tee /etc/resolv.conf   # TEST-NET-1, unroutable
time getent hosts example.com        # watch it hang, then fail
sudo cp /tmp/resolv.conf.bak /etc/resolv.conf
getent hosts example.com             # works again
```

**Observe:** With a dead nameserver, lookups hang until timeout — every tool that resolves names stalls. Note how `getent hosts` respects nsswitch, testing the full stack, not just DNS.

**Why this matters in interviews/ops:** "Everything is slow" is sometimes just a dead resolver making every lookup wait out a timeout. This failure mode is invisible until you've seen it once.

---

## Exercise 5: Write Your First iptables Rule

**Goal:** Block and unblock a port, and watch the rule order.

```bash
sudo python3 -m http.server 9999 &
# From another terminal/SSH session:
curl -m 3 http://localhost:9999/ && echo "reachable"

sudo iptables -I INPUT 1 -p tcp --dport 9999 -j DROP
curl -m 5 http://localhost:9999/ ; echo "exit: $?"   # hangs, then fails

sudo iptables -L INPUT -n --line-numbers
sudo iptables -D INPUT 1
curl -m 3 http://localhost:9999/ && echo "reachable again"
kill %1
```

**Observe:** The DROP rule makes curl hang until timeout (packets vanish silently — this is why "connection timed out" means firewall). `-I INPUT 1` inserts at the top; rules below a matching DROP never run. `-D INPUT 1` removes it by line number.

**Why this matters in interviews/ops:** Rule order is the #1 iptables footgun. Inserting vs appending is the difference between a rule that works and a rule that's silently dead.

---

## Exercise 6: DROP vs REJECT — Feel the Difference

**Goal:** See why the two targets produce different client behavior.

```bash
sudo iptables -A INPUT -p tcp --dport 9998 -j REJECT
time curl -m 5 http://localhost:9998/ ; echo "exit: $?"

sudo iptables -F INPUT
sudo iptables -A INPUT -p tcp --dport 9998 -j DROP
time curl -m 8 http://localhost:9998/ ; echo "exit: $?"
sudo iptables -F INPUT
```

**Observe:** REJECT fails fast — the kernel sends back an ICMP "port unreachable" and curl exits immediately with "connection refused". DROP makes curl wait out the full timeout. Same firewall intent, completely different symptom on the client.

**Why this matters in interviews/ops:** This is the refused-vs-timed-out distinction from the guide, made physical. Clients, load balancers, and health checks behave very differently under each.

---

## Exercise 7: Build an Isolated Network Sandbox

**Goal:** Create two network namespaces joined by a veth pair — a mini container network by hand.

```bash
sudo ip netns add red
sudo ip netns add blue
sudo ip link add v-red type veth peer name v-blue
sudo ip link set v-red netns red
sudo ip link set v-blue netns blue

sudo ip netns exec red ip addr add 10.99.0.1/24 dev v-red
sudo ip netns exec blue ip addr add 10.99.0.2/24 dev v-blue
sudo ip netns exec red ip link set v-red up
sudo ip netns exec blue ip link set v-blue up
sudo ip netns exec red ip link set lo up
sudo ip netns exec blue ip link set lo up

sudo ip netns exec red ping -c 3 10.99.0.2
```

**Observe:** Two fully isolated stacks, pinging each other through the veth pair. `ip netns exec red ip addr` shows only `lo` and `v-red` — the host's interfaces are invisible. This is exactly what Docker does per container, minus the bridge.

**Why this matters in interviews/ops:** Once you've built container networking by hand, "how do containers talk?" stops being magic. You'll also reach for `ip netns exec` naturally when debugging real container issues.

---

## Exercise 8: Watch Packets Die with tcpdump

**Goal:** Capture one conversation and read it.

```bash
sudo python3 -m http.server 7777 &
sudo tcpdump -i lo -n -c 6 -w /tmp/lo.pcap port 7777 &
sleep 1
curl -s http://127.0.0.1:7777/ > /dev/null
sleep 2
tcpdump -r /tmp/lo.pcap -n
kill %1; kill %2 2>/dev/null
```

**Observe:** The SYN → SYN-ACK → ACK handshake, then the HTTP request and response — the full TCP conversation in six packets. `-r` replays the capture without needing root again.

**Why this matters in interviews/ops:** When logs say "sent" and the other side says "never arrived", tcpdump on both ends is the only thing that settles it. Reading a handshake is a rite of passage.

---

## Exercise 9: Diagnose the "Broken" Service (Putting It Together)

**Goal:** Run the full inside-out debugging order on a deliberately misconfigured service.

```bash
# Terminal A: bind to loopback only (the classic mistake)
sudo python3 -m http.server 6666 --bind 127.0.0.1 &

# Terminal B: walk the layers
curl -m 3 http://127.0.0.1:6666/ && echo "STEP 1 OK: app alive locally"
ss -tlnp | grep ':6666'                          # STEP 2: what interface is it on?
ip route get 192.168.1.1                         # STEP 3: routing sane?
sudo iptables -L INPUT -n | head -5              # STEP 4: firewall blocking?
# Fix: kill it, rebind to all interfaces
kill %1
sudo python3 -m http.server 6666 --bind 0.0.0.0 &
ss -tlnp | grep ':6666'                          # now 0.0.0.0 — reachable remotely
kill %1
```

**Observe:** `ss` shows `127.0.0.1:6666` vs `0.0.0.0:6666` — the entire bug, visible in one column. Everything else (DNS, routes, firewall) was fine; the app was never listening where the network could reach it.

**Why this matters in interviews/ops:** This exact scenario — works locally, unreachable remotely — is the most common "network issue" in production. The fix is one flag, but only if you check the socket before blaming the network.

---

*Clean up when done: `sudo ip netns del red; sudo ip netns del blue; sudo iptables -F`*
