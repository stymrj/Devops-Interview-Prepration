# DNS & TLS/SSL Interview Preparation Guide

*How to Answer DNS and TLS Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [DNS Basics](#dns-basics)
2. [DNS in Production](#dns-in-production)
3. [TLS Deep Dive](#tls-deep-dive)

---

## DNS Basics

### Q1: Walk me through what happens when you type a domain into the browser and hit enter?

**How to Answer:**

"First my machine checks its local cache, then asks the configured resolver — usually my ISP's or 8.8.8.8. That resolver walks the chain: root servers, then the TLD servers for .com, then the authoritative server for the domain, and returns the IP. Then the browser opens a TCP connection and does the TLS handshake."

"The important bit for interviews: DNS is a distributed lookup, not a single database. Each level only knows where to send you next. And almost everything is cached somewhere — browser, OS, resolver — with a TTL deciding how long that answer is trusted."

**Key Point:** "DNS resolution is a referral chain — cache, resolver, root, TLD, authoritative — and caching with TTLs is what makes it fast."

---

### Q2: What's the difference between a recursive and an authoritative nameserver?

**How to Answer:**

"A recursive resolver does the work for clients — it chases the full chain on your behalf and hands you the final answer. 8.8.8.8 and your ISP's resolver are recursive. An authoritative server holds the actual zone data and only answers for domains it's responsible for."

"In production I run authoritative servers for my own domains and point everything at public resolvers for lookups. Mixing the two jobs on one box is a classic anti-pattern — an open recursive resolver will happily join a DNS amplification DDoS."

**Key Point:** "Recursive resolvers chase answers for clients; authoritative servers own the truth for their zones — never run an open recursive resolver."

---

### Q3: A, AAAA, CNAME, ALIAS, TXT, MX, SRV — when do you use each one?

**How to Answer:**

"A points a name at an IPv4 address, AAAA at IPv6. CNAME aliases one name to another — but it can't sit at the zone apex, so for the root of your domain you use an ALIAS or ANAME record if your provider supports it. TXT is freeform text — SPF, DKIM, domain verification all live there."

"MX is mail routing, SRV is service discovery with port and priority — Kubernetes-adjacent tools and SIP use it. The interview trap is the CNAME-at-apex rule: if someone says 'just CNAME the root domain,' that's wrong — CNAME can't coexist with the SOA and NS records the apex requires."

**Key Point:** "A/AAAA for IPs, CNAME for aliases (never at apex), TXT for verification records, MX/SRV for mail and service discovery."

---

### Q4: What is TTL, and how do you plan a DNS cutover with it?

**How to Answer:**

"TTL is how long resolvers cache your record before asking again. A 24-hour TTL means a change can take a full day to be visible everywhere. So before a migration I lower the TTL to something like 300 seconds a day or two ahead, make the change, verify it propagated, then raise the TTL back up."

"The trap is lowering the TTL at the last minute — old cached answers with the long TTL are still out there, so you're not actually safe. And some resolvers and ISPs ignore very low TTLs or clamp them, so '300 seconds' doesn't guarantee 300 seconds everywhere."

**Key Point:** "Lower the TTL well before a DNS change, migrate, then raise it back — and remember some resolvers clamp low TTLs."

---

## DNS in Production

### Q5: How do you design DNS for high availability?

**How to Answer:**

"You never rely on one nameserver or one provider. I run at least two authoritative nameservers in different networks, and for anything serious I'd use a managed anycast provider — Route 53, Cloudflare — so the same IP is announced from dozens of PoPs and a datacenter failure doesn't take out lookups."

"On the client side, resolv.conf lists multiple nameservers and glibc falls through them. The real design point is health: Route 53 health checks can flip a record to a failover target automatically, so DNS becomes a cheap global load balancer. DNS failing is one of those outages that breaks literally everything else, so I treat it as the most critical dependency."

**Key Point:** "Anycast authoritative DNS across providers, multiple resolvers client-side, and health-check failover — DNS outages break everything, so it's designed for survival first."

---

### Q6: What is DNS propagation, and why does everyone misunderstand it?

**How to Answer:**

"There's no actual 'propagation' — your change is live on the authoritative server instantly. What people see is cached old answers expiring across thousands of resolvers on different schedules. The worst-case wait is the old TTL, but most users see the change much faster."

"The confusion matters in incidents. When someone says 'DNS hasn't propagated,' what they usually mean is a stale cache somewhere in the chain. I verify with `dig @1.1.1.1` and `dig +trace` to see what the authoritative servers actually serve versus what a resolver hands back."

```bash
dig api.example.com @ns-1.awsdns-00.com +short   # authoritative answer
dig api.example.com +trace                        # full referral chain
```

**Key Point:** "Propagation is really cache expiry — the change is instant on authoritative servers; staleness lives in resolver caches."

---

### Q7: What is split-horizon DNS, and where do you actually use it?

**How to Answer:**

"Split-horizon serves different answers depending on who's asking. Inside the VPC, `db.internal.example.com` resolves to a private 10.x address; from the internet, the same name either doesn't resolve or returns something else. Route 53 private hosted zones and CoreDNS in Kubernetes both do this."

"I use it so internal service names never leak onto the public internet and internal traffic never hairpins through a public IP. The interview trap: debugging a 'works on my laptop, fails in the pod' DNS issue is almost always split-horizon — the laptop resolves the public answer, the pod resolves the private one."

**Key Point:** "Split-horizon gives internal clients private answers and external clients public ones — mismatch between the two is a classic debug rabbit hole."

---

### Q8: Walk me through debugging a DNS problem.

**How to Answer:**

"I start with `dig` against a known-good resolver to see if it's my machine or the record: `dig name @1.1.1.1`. Then `dig +trace` to walk the referral chain and find which level returns the wrong or missing answer. Then I check which resolver the box is actually using — `/etc/resolv.conf` or systemd-resolved — because a stale corporate resolver has burned me before."

"For Kubernetes it's `nslookup svc.namespace.svc.cluster.local` from a debug pod and checking CoreDNS logs for NXDOMAIN or SERVFAIL. Nine times out of ten it's a stale cache, a missing trailing dot in a zone file, or the search-domain list in resolv.conf silently appending junk to short names."

**Key Point:** "dig against a public resolver, then +trace the chain, then check the local resolver config — it's stale cache, zone file syntax, or search domains 90% of the time."

---

## TLS Deep Dive

### Q9: Walk me through the TLS handshake.

**How to Answer:**

"The client sends a ClientHello with its TLS version and cipher list. The server replies with its certificate and picks a cipher. Then they run a key exchange — in TLS 1.3 this is folded into the first two flights — derive session keys, and everything after that is encrypted. By the third round trip, application data is flowing."

"The interview-worthy detail: the client validates the certificate against its trust store — expiry, hostname match, chain to a trusted root — before trusting anything. If any of those checks fail, the connection dies. The handshake is the slow part, which is why TLS 1.3 cutting it to one round trip matters at scale."

**Key Point:** "ClientHello, server certificate, key exchange, encrypted session — and the client validates expiry, hostname, and chain before trusting anything."

---

### Q10: What's the difference between TLS 1.2 and 1.3, and why should I care?

**How to Answer:**

"TLS 1.3 removed all the legacy junk — RSA key exchange, CBC ciphers, static DH — so there's a smaller attack surface and fewer ways to misconfigure. Handshake went from two round trips to one, and session resumption is now 0-RTT. It also encrypts the handshake itself much earlier, so less metadata leaks."

"In practice: if your infra still negotiates 1.2, it works but you're leaving latency and security on the table, and some of those old cipher suites are the ones PCI auditors flag. My rule is 1.2 minimum, 1.3 preferred, and I kill anything older at the load balancer."

**Key Point:** "TLS 1.3: faster 1-RTT handshake, legacy ciphers removed, encrypted handshake metadata — enforce 1.2 minimum and prefer 1.3."

---

### Q11: What's in a certificate chain, and what breaks in production?

**How to Answer:**

"Leaf certificate for your domain, one or more intermediates, then the root that browsers trust. The server has to send the leaf plus the intermediates — if it only sends the leaf, clients that don't have the intermediate cached fail the chain. I've seen this as a 'works in Chrome, fails in curl' bug because Chrome had the intermediate cached."

"The classic production breakage is expiry — Let's Encrypt's 90-day certs renew automatically until the day the cron breaks, and suddenly the API is down at 3am. I monitor cert expiry with alerts at 30 and 7 days out, and I always test the full chain with `openssl s_client`, not just that the page loads."

```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | openssl x509 -noout -dates
```

**Key Point:** "Serve the full chain including intermediates, and alert on expiry well before 90-day certs die — 'works in Chrome, fails in curl' means a missing intermediate."

---

### Q12: What is SNI, and why does it exist?

**How to Answer:**

"Before TLS existed for shared hosting, the server couldn't know which certificate to present because the Host header is inside the encrypted stream. SNI fixes that: the client puts the hostname in the ClientHello, in plaintext, so the server picks the right cert for that domain."

"This is how one IP serves a hundred HTTPS sites — ALBs, ingress controllers, and CDNs all route on SNI. The tradeoff is privacy: that hostname travels unencrypted, so anyone watching the wire sees which domain you're hitting. That's the problem Encrypted Client Hello is trying to solve."

**Key Point:** "SNI puts the hostname in the plaintext ClientHello so one IP can serve many certs — at the cost of leaking the domain to network observers."

---

### Q13: Where do you terminate TLS in production — load balancer, ingress, or app?

**How to Answer:**

"I terminate at the edge — ALB, Cloudflare, or the ingress controller — because TLS is CPU-heavy and managing certs in one place beats doing it in every pod. Behind that, traffic to pods is either plain HTTP inside a private VPC or re-encrypted if the compliance team demands end-to-end encryption."

"Cert provisioning is automated: cert-manager with Let's Encrypt in Kubernetes, ACM on ALB with auto-renewal. I never hand-roll cert distribution. And when someone says 'just terminate in the app,' I ask how they're rotating keys across fifty replicas without a restart — that's the argument that ends the debate."

**Key Point:** "Terminate TLS at the edge with automated cert management — managing certificates inside every app instance doesn't scale."

---

*Related: [hands-on lab](labs/49-dns-tls-ssl-in-depth-lab.md) · [cheat sheet](cheat-sheets/49-dns-tls-ssl-in-depth-cheatsheet.md)*
