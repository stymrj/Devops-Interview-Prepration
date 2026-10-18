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
