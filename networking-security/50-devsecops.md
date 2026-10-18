# DevSecOps Interview Preparation Guide

*How to Answer DevSecOps Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [DevSecOps and Shift Left Security](#devsecops-and-shift-left-security)
2. [SAST Static Analysis](#sast-static-analysis)
3. [DAST Dynamic Testing](#dast-dynamic-testing)
4. [Container Image Scanning](#container-image-scanning)
5. [SBOM and Supply Chain Security](#sbom-and-supply-chain-security)
6. [Secrets and Pipeline Security Gates](#secrets-and-pipeline-security-gates)

---

## DevSecOps and Shift Left Security

### Q1: What is DevSecOps and how is it different from DevOps?

**How to Answer:**

"DevOps without security is just shipping fast. DevSecOps is the same delivery culture, but security is everyone's job — not a gate at the end where someone says no."

"I treat security like tests: automated, early, and fast enough that developers don't hate it. If your security process slows releases, developers will route around it."

"In my last project I moved image scans into CI and added secret scanning on every PR. We caught a leaked AWS key in a commit within minutes instead of learning about it from a bill spike."

**Key Point:** "DevSecOps = DevOps with security automated into the pipeline, not bolted on at the end."

---

### Q2: What does shift left security actually mean in practice?

**How to Answer:**

"Shift left means catching problems earlier, when they're cheap. A leaked secret caught in a PR takes 30 seconds to fix; the same secret in prod means key rotation, incident review, and possibly a breach."

"In practice I lint Dockerfiles for bad patterns, scan images in CI, and run SAST on every pull request. By the time code hits staging, most easy wins are already handled."

"The tricky part is noise. If every build breaks on low-severity findings, developers stop paying attention. I gate on high and critical, and make the rest advisory."

**Key Point:** "Earlier detection = cheaper fixes. Gate the build only on high and critical findings."

---

## SAST Static Analysis

### Q3: What is SAST and where does it run in a pipeline?

**How to Answer:**

"SAST is static application security testing — it reads your source code without running it and flags risky patterns like SQL injection, hardcoded secrets, or unsafe deserialization."

"I run it on every pull request so feedback lands where developers are already working. A finding posted as a PR comment with the exact line number gets fixed; the same finding emailed a week later gets ignored."

"Tools I use: Semgrep and CodeQL on GitHub repos, SonarQube for bigger codebases. Semgrep is my favorite for custom rules because you can write a check in ten lines of YAML."

```yaml
rules:
  - id: no-hardcoded-aws-key
    pattern: AKIA[0-9A-Z]{16}
    message: "Possible hardcoded AWS access key"
    severity: ERROR
```

**Key Point:** "SAST reads code without running it; run it on PRs so findings show up as comments, not emails."

---

### Q4: How do you deal with SAST false positives?

**How to Answer:**

"False positives are the number one reason SAST programs fail. A scanner that cries wolf on every PR teaches developers to click 'dismiss' on the real ones too."

"My approach: tune rules per repo, suppress known-safe patterns with justification, and separate gates from noise. High and critical fail the build; everything else is advisory and visible but non-blocking."

"I also track the false-positive rate per rule. If a rule has never found a real bug in three months, I disable it or rewrite it. Signal quality is the metric that matters."

**Key Point:** "Track signal quality per rule — disable rules that never find real bugs."

---

## DAST Dynamic Testing

### Q5: What is DAST and when do you run it?

**How to Answer:**

"DAST is dynamic testing — it attacks a running application from the outside, like a real attacker would. It finds things SAST can't: auth bypasses, config mistakes, and issues only visible at runtime."

"I run it against a staging or preview environment, not PRs, because it needs a deployed app. Typically it runs nightly or on releases — it's slower than SAST, so it doesn't belong in the inner loop."

"OWASP ZAP is the standard free tool. I run the baseline scan on every staging deploy and the full active scan on release candidates."

```bash
docker run -t zaproxy/zap-stable zap-baseline.py -t https://staging.example.com
```

**Key Point:** "DAST attacks the running app; schedule it nightly against staging, not on every PR."

---

### Q6: SAST vs DAST — when do you use each?

**How to Answer:**

"I use both because they see different things. SAST finds code-level bugs early and cheap — SQL injection in a specific function. DAST finds runtime reality — your login actually bypasses MFA when the session cookie is reused."

"The interview trap is picking one. Security-conscious teams run SAST on every PR for fast feedback and DAST nightly on staging for runtime coverage. Together they cover code bugs and config bugs."

"Neither replaces a threat model or a pen test. Scanners are your smoke alarms — they don't make the house fireproof, they just wake you up early."

**Key Point:** "SAST catches code bugs early, DAST catches runtime bugs late — you need both, neither replaces a pen test."

---
## Container Image Scanning

### Q7: How do you scan container images in a CI pipeline?

**How to Answer:**

"Every image gets scanned before it can reach production. Trivy is my go-to — it's fast, free, and covers OS packages plus language dependencies in one pass."

"My pipeline builds the image, scans it, and fails the build on high or critical CVEs that have fixes available. The 'fix available' filter is key, otherwise you block on vulnerabilities nobody can patch yet."

"For ongoing risk, I also scan images on a schedule — base images get CVEs published after you build, so a scan that was clean on Monday can be dirty by Friday."

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed myapp:latest
```

**Key Point:** "Scan on build AND on a schedule — CVEs get published after your build date."

---

### Q8: What is a CVE, and how do you decide what to block the build on?

**How to Answer:**

"A CVE is a known vulnerability with an ID — like CVE-2024-1234 — and a severity score from 1 to 10. The score tells you how bad it is; exploitability tells you whether it actually matters."

"I gate builds on high and critical CVEs with an available fix. Low and medium go on a dashboard for the next sprint. And I always check exploitability — a critical CVE in a library function my app never calls is a ticket, not an emergency."

"New engineers often think zero CVEs is the goal. It's not — the goal is knowing about every CVE and deciding deliberately. An ignored scan is just a false sense of security."

**Key Point:** "Block on high/critical with fixes; track exploitability, not just scores."

---

## SBOM and Supply Chain Security

### Q9: What is an SBOM and why do teams generate one?

**How to Answer:**

"An SBOM is a software bill of materials — a machine-readable list of every component inside your software, like an ingredients label. It's usually in SPDX or CycloneDX format."

"Its real value showed up during Log4Shell. Teams with SBOMs knew in minutes whether they were exposed; teams without them spent days grepping repos. It's an inventory problem, and attackers don't wait for manual audits."

"I generate one for every release with syft and store it next to the artifact. Some customers and regulations now require it, but I'd do it anyway for my own incident response."

```bash
syft myapp:latest -o cyclonedx-json > sbom.json
```

**Key Point:** "SBOMs turn 'are we affected?' from a week of grepping into a one-minute query."

---

### Q10: How do you protect against software supply chain attacks?

**How to Answer:**

"Supply chain attacks come through your dependencies — a compromised npm package or a poisoned base image. Your code is clean; everything you downloaded isn't."

"My defenses: pin exact dependency versions with lockfiles, verify image signatures with cosign, pull base images only from trusted registries, and scan everything — OS packages and language deps — on every build."

"I also reduce the attack surface with minimal base images like distroless. A smaller image has fewer packages to have CVEs, and no shell for an attacker to play with if they do get in."

**Key Point:** "Lockfiles, signature verification, trusted registries, minimal base images — layered defense for things you download."

---

## Secrets and Pipeline Security Gates

### Q11: How do you stop secrets from leaking into git and images?

**How to Answer:**

"Prevention first: gitleaks or trufflehog runs on every PR and blocks commits containing tokens, keys, or passwords. A secret that never gets committed is a secret you never have to rotate."

"For runtime secrets, nothing goes in environment variables on disk or in images — I pull from a secrets manager at deploy time. Vault or AWS Secrets Manager, injected as mounted files or env vars at container start."

"And assume leakage happens anyway: secrets are short-lived and rotatable. If every credential expires in 90 days, a leaked one has a limited blast radius by design."

```bash
gitleaks detect --source . --no-git -v
```

**Key Point:** "Block secrets at commit time with scanners; inject at deploy time from a secrets manager; rotate aggressively."

---

### Q12: How do you enforce security gates without slowing developers down?

**How to Answer:**

"This is the core DevSecOps tradeoff. My rule is simple: fast checks run early and everywhere, slow checks run later and asynchronously. SAST and secret scans run on the PR in seconds; DAST and deep scans run nightly on staging."

"I make failures actionable — every blocked build links to the finding, the fix, and the person to ask. A gate that says 'vulnerability found' with no guidance just creates a Slack thread asking what to do."

"And I measure. I track scan duration per PR, false-positive rates, and how often security gates actually change code. If a gate adds two minutes and catches real bugs, developers defend it themselves."

**Key Point:** "Fast checks early, slow checks async, every failure links to its fix."

---

*Day 50 of 58 — networking-security subject.*
