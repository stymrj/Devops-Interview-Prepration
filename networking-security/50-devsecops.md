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
