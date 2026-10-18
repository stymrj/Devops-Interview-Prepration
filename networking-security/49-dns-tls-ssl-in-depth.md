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
