# Zero Trust & mTLS Basics Interview Preparation Guide

*How to Answer Zero Trust & mTLS Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Zero Trust](#why-zero-trust)
2. [What mTLS Actually Is](#what-mtls-actually-is)
3. [How the TLS Handshake Verifies Both Sides](#how-the-tls-handshake-verifies-both-sides)
4. [Certificates and PKI in Practice](#certificates-and-pki-in-practice)
5. [Service Mesh mTLS](#service-mesh-mtls)
6. [Zero Trust Beyond Certificates](#zero-trust-beyond-certificates)

---

## Why Zero Trust

### Q1: What is zero trust and why did it replace the castle-and-moat model?

**How to Answer:**

"Castle-and-moat said everything inside the network is safe. That broke the day laptops left the office and apps moved to shared cloud networks."

"Zero trust flips the assumption: never trust, always verify. Every request gets authenticated, authorized, and encrypted — even between two services in the same cluster."

"I've lived through the alternative: a dev pod got compromised and the attacker moved sideways freely because nothing inside the VPC questioned anything. Zero trust means that breach would have hit a wall at the first hop."

**Key Point:** "Trust no network location — verify identity on every request, every time."

---

### Q2: What's the difference between zero trust and just "encrypt everything"?

**How to Answer:**

"Encryption is one ingredient, not the recipe. Zero trust also needs identity, least-privilege access, and continuous verification."

"TLS between two services stops eavesdropping, but it says nothing about whether that caller should be allowed to hit your database endpoint. That's authorization, and encryption alone doesn't do it."

"The trap answer in interviews is treating HTTPS as the whole strategy. I'd say: encryption gives you confidentiality, identity plus policy gives you zero trust."

**Key Point:** "Encryption hides traffic; zero trust decides who is allowed to talk at all."

---

### Q3: What does "never trust, always verify" mean for how I design services?

**How to Answer:**

"It means every service carries its own identity, and every call checks that identity before doing anything. No more 'it came from inside the VPC, so it's fine'."

"In practice that looks like mTLS between pods, short-lived tokens for APIs, and network policies that default to deny. The identity travels with the workload, not the IP address."

"IPs lie — they get reused by DHCP and spoofed by attackers. A workload identity issued by a control plane can't be borrowed by the next tenant on the subnet."

**Key Point:** "Identity lives on the workload, not the IP — design around that and half the foot-guns disappear."

---

## What mTLS Actually Is

### Q4: What is mTLS and how does it differ from regular TLS?

**How to Answer:**

"Regular TLS only proves the server's identity to the client — your browser checks the site's certificate and that's it. mTLS adds the other direction: the client also presents a certificate, and the server verifies it."

"So with mTLS both sides know exactly who they're talking to before a single byte of application data flows. The handshake fails if either certificate is invalid or expired."

"The one-liner I use in interviews: TLS authenticates the server, mTLS authenticates both peers."

**Key Point:** "TLS proves the server to the client; mTLS makes them prove themselves to each other."

---

### Q5: When would I use mTLS instead of API keys or tokens?

**How to Answer:**

"mTLS shines for service-to-service calls where both sides are machines. It's handled at the transport layer, so the app code doesn't manage secrets — the certificates live in the sidecar or the runtime."

"API keys are bearer tokens: anyone who copies one becomes you. A client certificate with a private key never leaves the machine, so it's much harder to steal and replay."

"I reach for mTLS when I have a mesh of internal services and want auth that scales without a shared-secret management nightmare. For user-facing APIs, tokens still win."

**Key Point:** "mTLS for machine-to-machine where stealing a secret shouldn't be possible; tokens for user-facing APIs."
