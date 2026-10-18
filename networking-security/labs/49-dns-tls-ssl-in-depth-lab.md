# DNS & TLS/SSL Hands-On Lab

*Break DNS and TLS on purpose, then fix them — 60-90 minutes.*

**Prereqs:** A Linux box or VM with `dig`, `openssl`, and Docker. No cloud account needed.

---

## Exercise 1: Trace a real DNS lookup end to end

**Goal:** See the referral chain instead of just trusting the answer.

**Commands:**
```bash
dig example.com +trace | grep -E "^(example|\.)" | head -30
```

**Expected output:** Root servers → `.com` TLD servers → `example.com`'s authoritative servers, then the final A record. You'll see the `NS` and `A` referrals at each level.

**Why it matters:** Interviewers love "walk me through DNS resolution" — after watching `+trace`, you answer from memory of the actual packets, not a textbook.

---

## Exercise 2: Compare recursive vs authoritative answers

**Goal:** Feel the difference between asking a resolver and asking the source of truth.

**Commands:**
```bash
dig github.com @8.8.8.8 +short
dig github.com @ns1.dnsimple.com +short
```

**Expected output:** The recursive answer (8.8.8.8) comes back fast from cache; the authoritative answer goes straight to GitHub's nameservers. Try `dig github.com @8.8.8.8` twice and watch the query time drop on the second run.

**Why it matters:** The first call in any DNS debug is "what does a known-good resolver say vs what does the authoritative server say." This is that muscle memory.

---

## Exercise 3: Discover record types on a real domain

**Goal:** Read A, AAAA, MX, TXT, and CNAME records like a native.

**Commands:**
```bash
dig google.com A +short; dig google.com AAAA +short
dig google.com MX +short; dig google.com TXT +short | head -3
dig www.google.com CNAME +short
```

**Expected output:** Multiple A and AAAA records (anycast), MX records pointing at Google's mail servers, TXT records including SPF, and a CNAME you may or may not see. No output for a type means the record simply doesn't exist.

**Why it matters:** Reading record types fluently is the difference between "I know what MX is" and actually debugging mail delivery or domain verification in an interview scenario.

---

## Exercise 4: Watch TTL in action

**Goal:** See caching happen with your own eyes.

**Commands:**
```bash
dig example.com @1.1.1.1 | grep -E "^(example.com.*IN A|;; Query)"
dig example.com @1.1.1.1 | grep -A1 "ANSWER SECTION"
```

**Expected output:** The first query's TTL shows the full value (e.g. 86400); run the second command a minute later and the TTL has ticked down. That's the cache counting down to expiry.

**Why it matters:** This is the "DNS propagation" myth made visible — nothing propagates, caches just count down. You'll explain it with total confidence now.

---

## Exercise 5: Simulate the CNAME-at-apex trap

**Goal:** Understand why you can't CNAME a zone apex.

**Commands:**
```bash
dig example.com CNAME +short
dig example.com SOA +short
dig example.com NS +short
```

**Expected output:** No CNAME record at the apex (empty first command), but SOA and NS records are there. A CNAME would conflict with those mandatory apex records — that's the rule, demonstrated live.

**Why it matters:** "Just CNAME the root domain" is a classic wrong answer. Now you can say exactly why it's wrong, with the SOA/NS evidence.

---

## Exercise 6: Inspect a real certificate chain

**Goal:** Read a live TLS certificate like a pro.

**Commands:**
```bash
echo | openssl s_client -connect github.com:443 -servername github.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

**Expected output:** The leaf's subject (`CN=github.com`), its issuer (an intermediate CA), and the validity dates. Note the expiry date — this is the field that causes 3am outages.

**Why it matters:** Cert expiry is one of the most common real-world outages. Knowing how to read dates and issuer from the CLI is a concrete skill interviewers respect.

---

## Exercise 7: Verify the full chain (and spot a broken one)

**Goal:** Check whether a server sends its intermediates.

**Commands:**
```bash
echo | openssl s_client -connect github.com:443 -servername github.com 2>/dev/null | grep "Verify return code"
```

**Expected output:** `Verify return code: 0 (ok)`. If you ever see `21 (unable to verify the first certificate)`, the server forgot its intermediates — the "works in Chrome, fails in curl" bug.

**Why it matters:** Chain validation is the second half of every TLS interview answer. This command is the one-liner you run in production.

---

## Exercise 8: See which TLS version and cipher you negotiate

**Goal:** Check whether a site still allows legacy TLS.

**Commands:**
```bash
echo | openssl s_client -connect github.com:443 -tls1_2 2>/dev/null | grep "Protocol"
echo | openssl s_client -connect github.com:443 -tls1_3 2>/dev/null | grep "Protocol"
```

**Expected output:** One or both of `Protocol : TLSv1.2` / `Protocol : TLSv1.3`. If a server only answers TLS 1.0/1.1, that's a finding, not a config.

**Why it matters:** "Enforce 1.2 minimum, prefer 1.3" is easy to say in an interview — this shows you know how to verify it for real.

---

## Exercise 9: Break and fix DNS locally with /etc/hosts

**Goal:** Simulate a DNS hijack and fix it the way you'd debug in production.

**Commands:**
```bash
echo "93.184.215.14 mytestapp.local" | sudo tee -a /etc/hosts
getent hosts mytestapp.local
sudo sed -i '/mytestapp.local/d' /etc/hosts
getent hosts mytestapp.local
```

**Expected output:** The name resolves while the hosts entry exists, then stops resolving after removal (NSS checks files before DNS — that's why it worked without any nameserver).

**Why it matters:** `/etc/hosts` overriding DNS is the cause of so many "works on my machine" bugs. Experiencing it once means you check it first forever.

---

## Exercise 10: Generate a self-signed cert and serve HTTPS locally

**Goal:** Touch the whole cert lifecycle in five minutes.

**Commands:**
```bash
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 7 -nodes -subj "/CN=localhost"
docker run -d --name tls-test -p 8443:443 -v $PWD/cert.pem:/etc/ssl/certs/cert.pem -v $PWD/key.pem:/etc/ssl/private/key.pem nginx:alpine
```

**Expected output:** OpenSSL prints the key generation; nginx starts on 8443. `curl -k https://localhost:8443` works, `curl https://localhost:8443` fails with a self-signed error — exactly the trust-store check from the guide.

**Why it matters:** Self-signed certs are how every local dev environment works. Seeing the trust failure proves you understand that clients validate against a trust store, not just "does a cert exist."
