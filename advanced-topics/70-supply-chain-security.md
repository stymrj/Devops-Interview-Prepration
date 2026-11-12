# Supply Chain Security (SLSA, Sigstore & SBOM) Interview Preparation Guide

*How to Answer Supply Chain Security Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [The Supply Chain Problem](#the-supply-chain-problem)
2. [SLSA, SBOM & Provenance](#slsa-sbom--provenance)
3. [Signing, Verification & Enforcement](#signing-verification--enforcement)

---

## The Supply Chain Problem

### Q1: What is software supply chain security, and why should a DevOps engineer care?

**How to Answer:**

"I'd explain it simply: your software isn't just your code, it's everything that code is built from — the dependencies, the base images, the build tools, the pipeline itself. If any of those is tampered with, the bad code ships under your name and your signing key. I care because I'm the one who owns the pipeline, so if a compromised package or a poisoned build step slips through, that's my deploy that pushed it to production. The SolarWinds attack is the example everyone knows — attackers didn't hack the product, they hacked the build, and the update channel delivered the malware for them."

**Key Point:** "You don't just ship your code — you ship every dependency, image, and build step behind it, and the pipeline owner is the one who has to prove none of it was tampered with."

---

### Q2: What are the common ways attackers compromise a software supply chain?

**How to Answer:**

"I'd walk through the four doors I check first. One, a compromised dependency — a popular package gets hijacked or a typosquatted name gets installed by mistake. Two, a poisoned build — someone tampers with the CI runner or the build script so the artifact differs from the source. Three, a stolen signing key or leaked credential that lets an attacker publish a fake release. And four, an unpatched base image that carries a known vulnerability into every container built on it. I always add that most of these don't need a genius attacker — they need one missed check in a pipeline that trusts everything by default."

**Key Point:** "Dependencies, the build, the signing keys, and the base image — four doors, and most breaches walk through the one the pipeline never bothered to check."

---

### Q3: What is dependency pinning, and why isn't "latest" good enough?

**How to Answer:**

"I'd say pinning means locking every dependency to an exact version — and ideally an exact hash — so a rebuild today produces the same artifact as a rebuild last month. If you float on 'latest', a new release of a library can land in your build without anyone reviewing it, and that's exactly how compromised or broken packages sneak in. I use lockfiles — package-lock.json, poetry.lock, go.sum — and I commit them, because a lockfile nobody commits is just a suggestion. The trade-off is real: pinned versions go stale, so I pair pinning with a scheduled job like Dependabot or Renovate that proposes updates I actually review."

**Key Point:** "Pin exact versions with committed lockfiles so rebuilds are reproducible — then let an automated updater propose upgrades you review, instead of silently floating on 'latest'."

---

## SLSA, SBOM & Provenance

### Q4: What is SLSA, and what do its levels actually mean?

**How to Answer:**

"SLSA is a framework for proving your artifact was built the way you claim it was — basically a checklist for build integrity. Level 1 just means the build is scripted and the provenance is recorded, so you can see what went in. Level 2 adds a hosted build service, so the build doesn't run on someone's laptop. Level 3 is the one most teams aim for: the build runs on a hardened, isolated platform and the provenance can't be faked by the build itself. Level 4 adds a two-person review on changes and a fully hermetic, reproducible build. I'd be honest that most teams live between 1 and 3 — and that the point isn't the badge, it's knowing which guarantees your build actually gives you."

**Key Point:** "SLSA grades how much you can trust your build — scripted at L1, hosted at L2, tamper-resistant at L3, reproducible and reviewed at L4 — and most real teams aim for L3."

---

### Q5: What is an SBOM, and how do you generate and use one?

**How to Answer:**

"I'd describe an SBOM as an ingredient list for your software — every library, version, and license in the thing you ship. I generate it during the build, not after, using a tool like Syft or Trivy, and I attach it to the release artifact so it travels with the software. The real value shows up when a CVE drops: instead of guessing which services use the vulnerable library, I query the SBOMs and get a list in minutes. I also use it in the pipeline to fail builds that introduce a dependency with a license my company can't ship, which saves a legal argument after the release."

```bash
# Generate an SBOM for an image and scan it in the same step:
syft packages myapp:1.4.2 -o cyclonedx-json > sbom.json
trivy image --sbom sbom.json myapp:1.4.2
```

**Key Point:** "Generate the SBOM in the build with Syft or Trivy, ship it with the artifact, and when a CVE drops you query it instead of guessing which services are affected."

---

### Q6: What's the difference between SPDX and CycloneDX?

**How to Answer:**

"I'd keep it practical: both are just formats for writing down the same ingredient list, but they come from different worlds. SPDX started in open-source license compliance, so it's strong on licensing and it's the ISO standard, which matters in regulated industries. CycloneDX was built for security workflows, so it's leaner, easier to generate in CI, and it carries vulnerability and dependency-graph data more naturally. Honestly, for most teams the right answer is whichever one your tools already produce — Syft and Trivy can emit both — and the mistake to avoid is picking a format and then never actually generating the SBOM at all."

**Key Point:** "SPDX is the license-compliance ISO standard, CycloneDX is the security-workflow favourite — either works, what matters is generating the SBOM in CI and actually using it."

---

### Q7: What is build provenance, and how is it different from an SBOM?

**How to Answer:**

"I'd put it this way: the SBOM tells you what went into the artifact, provenance tells you how it was built. Provenance records the source repo and commit, the build command, the builder identity, and the parameters — so you can prove this binary came from that code and nobody swapped it mid-build. SLSA provenance is the standard format, and tools like GitHub's artifact attestations generate it automatically. The SBOM answers 'what's inside it', provenance answers 'can I trust how it was made', and in an interview I'd say you need both — knowing the ingredients doesn't help if the kitchen was compromised."

**Key Point:** "The SBOM lists what's inside the artifact, provenance proves how it was built — you need both, because a clean ingredient list means nothing if the build itself was tampered with."

---

## Signing, Verification & Enforcement

### Q8: How does Sigstore work — Cosign, Fulcio, and Rekor?

**How to Answer:**

"I'd explain the three pieces like a team. Cosign is the tool that actually signs your container image or artifact. Fulcio is the certificate authority — it gives you a short-lived signing certificate after you prove who you are with your normal identity, like your GitHub login, so you never manage a long-lived private key. And Rekor is the public transparency log that records every signature, so anyone can check that a signature exists and nobody can quietly sign something in secret. The beauty of it is keyless signing: there's no private key file to leak, because the certificate expires in minutes and the log makes everything auditable."

**Key Point:** "Cosign signs, Fulcio issues a short-lived certificate from your real identity, Rekor logs it publicly — keyless signing means no private key to leak and nothing signed in secret."

---

### Q9: How do you sign a container image and verify it before deploying?

**How to Answer:**

"I'd describe the flow end to end. In CI, after the image is built and scanned, I sign it with Cosign using my workload identity — no key file, just the OIDC token from the pipeline. Then the signature and the SLSA provenance go alongside the image, usually in the same registry. On the deploy side, I verify before anything runs: a policy in the admission controller or in the deploy job checks the signature, the signer identity, and the provenance, and refuses unsigned or unknown images. If verification fails, the deploy stops — and I'd add that the check has to be in the pipeline, not a wiki page, or nobody will actually do it under deadline pressure."

```bash
# Sign in CI (keyless, via the pipeline's OIDC identity):
cosign sign --yes ghcr.io/myorg/myapp:1.4.2
# Verify before deploy — refuse anything unsigned or wrongly signed:
cosign verify ghcr.io/myapp:1.4.2 \
  --certificate-identity-regexp '.*@myorg\.com' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

**Key Point:** "Sign in CI with Cosign's keyless flow, ship the signature with the image, and verify the signer identity before deploy — a failed check stops the rollout, it doesn't warn about it."

---

### Q10: How do you handle vulnerable dependencies in a pipeline?

**How to Answer:**

"I'd say it's a pipeline stage, not a one-off chore. I scan on every build with a tool like Trivy or Grype, and I fail the build on critical vulnerabilities with a known fix, while mediums and lows get logged and triaged. The important part is the policy is written down and agreed with the team — otherwise the first release deadline kills it. For vulnerabilities with no fix yet, I track them in a short exceptions list with an owner and an expiry date, so they don't become permanent. And I scan the running images too, not just the build, because a CVE published last week affects the container you deployed last month."

**Key Point:** "Scan every build, fail on fixable criticals, time-box exceptions with an owner — and keep scanning deployed images, because new CVEs hit old containers too."

---

### Q11: How do you enforce supply chain policy in Kubernetes?

**How to Answer:**

"I'd enforce it at admission, because that's the last gate before a pod actually runs. I use a policy engine like Kyverno or OPA Gatekeeper with a rule that only allows images from my registry, with a valid Cosign signature and a matching SBOM. The policy also blocks the usual escapes — images referenced by mutable tags like 'latest', images from public Docker Hub, and pods that skip the check by pulling from a sidecar. I'd roll it out in audit mode first so I can see what would break, fix the real violations, and only then flip it to enforce. The classic mistake is turning enforcement on day one and taking down a deployment at 2 a.m. because someone's image wasn't signed yet."

```yaml
# Kyverno: only signed images from our registry may run
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-image-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences: ["ghcr.io/myorg/*"]
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

**Key Point:** "Enforce at admission with Kyverno or Gatekeeper — signed images from your registry only — and always start in audit mode so the first enforcement doesn't break a live deploy."

---

### Q12: What do you do when a zero-day hits a dependency you ship?

**How to Answer:**

"I'd walk through the order I'd actually do it in. First, find the blast radius — query the SBOMs to list every service and image carrying the vulnerable library, and check whether the vulnerable code path is even reachable. Second, patch or mitigate — upgrade the dependency, and if no fix exists yet, apply the vendor workaround or block the endpoint at the WAF. Third, rebuild, re-sign, and redeploy through the normal pipeline, not a hotfix shortcut, because a rushed manual deploy is how you introduce the next incident. And last, write it down — update the SBOM, close the exception, and note the time from CVE to patched deploy, because that's the number that tells you whether your process actually works."

**Key Point:** "Blast radius from the SBOM first, then patch or mitigate, rebuild and redeploy through the normal pipeline — and record the CVE-to-patched time, because that's the metric that proves the process works."

---

