# Supply Chain Security — Hands-On Lab

Nine drills that turn supply chain answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Generate your first SBOM

**Goal:** An SBOM for a real image, generated in one command.

**Steps:**

```bash
# Pick an image you actually use, then generate its ingredient list:
syft packages nginx:1.27-alpine -o cyclonedx-json > nginx-sbom.json
# Count what you are shipping:
jq '.components | length' nginx-sbom.json
jq -r '.components[] | .name' nginx-sbom.json | head -20
```

**Expected output:** A CycloneDX SBOM file with a component count — and probably a surprise at how many packages hide inside one small image.

**Why it matters:** This is Q5 made concrete — an SBOM you generated yourself is the answer, not a definition of one.

---

## Exercise 2: Scan an image and triage the findings

**Goal:** A vulnerability list you can actually talk through in an interview.

**Steps:**

```bash
# Scan the same image and keep only what matters:
trivy image --severity HIGH,CRITICAL nginx:1.27-alpine
# Then check the SBOM route — scan from the file you generated in Exercise 1:
trivy image --sbom nginx-sbom.json nginx:1.27-alpine
```

**Expected output:** A short list of HIGH/CRITICAL findings with CVE IDs, plus a note on which ones have a fixed version available and which don't.

**Why it matters:** This is Q10 — interviewers want to hear how you decide what blocks a build and what gets an exception.

---

## Exercise 3: Sign an image with Cosign (keyless)

**Goal:** A signature you can verify, with no private key file anywhere.

**Steps:**

```bash
# Build and push a tiny test image, then sign it keyless:
cosign sign --yes ghcr.io/<your-user>/hello:supply-chain-lab
# Verify it the way a deploy job would:
cosign verify ghcr.io/<your-user>/hello:supply-chain-lab \
  --certificate-identity-regexp '.*@.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
# Check the signature is in the public transparency log:
rekor-cli search --artifact ghcr.io/<your-user>/hello:supply-chain-lab
```

**Expected output:** A verified signature, and a Rekor log entry proving the signing happened in public.

**Why it matters:** This is Q8 and Q9 — keyless signing is the part of Sigstore most candidates only describe, and doing it once makes it stick.

---

## Exercise 4: Generate SLSA provenance for a build

**Goal:** A provenance attestation that ties an artifact to its exact source commit.

**Steps:**

```bash
# In a GitHub Actions workflow, add the SLSA generator step:
#   - uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
#     with:
#       base64-subjects: "${{ needs.build.outputs.hashes }}"
# Then download the attestation and inspect it:
#   slsa-verifier verify-artifact myapp.tar.gz \
#     --provenance-path provenance.intoto.jsonl \
#     --source-uri github.com/<your-user>/<repo>
```

**Expected output:** A provenance file naming the source repo, commit SHA, builder, and build steps — proof the artifact came from that code.

**Why it matters:** This is Q7 — provenance is what turns "I built it from this commit" from a claim into something checkable.

---

## Exercise 5: Break the chain on purpose, then catch it

**Goal:** See a verification failure with your own eyes, so you can describe it honestly.

**Steps:**

```bash
# Build the same image twice from the same commit with different tags:
docker build -t myapp:good . && docker build -t myapp:tampered .
# Sign only the good one, then try to verify both:
cosign sign --yes myapp:good
cosign verify myapp:good --certificate-identity-regexp '.*@.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
cosign verify myapp:tampered --certificate-identity-regexp '.*@.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
# Note the exact error the unsigned one gives you.
```

**Expected output:** One passing verification and one clear failure — the exact output a deploy gate would see before it blocks the rollout.

**Why it matters:** This is Q9 — anyone can say "verify before deploy", but describing the failure you actually saw is what sounds real in an interview.

---

## Exercise 6: Write an admission policy that only allows signed images

**Goal:** A Kyverno or Gatekeeper policy you can explain line by line.

**Steps:**

```bash
# Start from the ClusterPolicy in Q11 of this guide.
# Install Kyverno in a local kind cluster and apply it in audit mode:
kubectl apply -f require-signed-images.yaml
# Deploy an unsigned image and check the policy report:
kubectl get policyreport -A
# Fix the image (sign it), redeploy, and confirm the report goes clean.
# Only then flip validationFailureAction to Enforce.
```

**Expected output:** A PolicyReport showing the unsigned pod flagged in audit mode, then clean after you sign the image.

**Why it matters:** This is Q11 — audit-first rollout is the detail that separates a textbook answer from someone who has done it.

---

## Exercise 7: Simulate a zero-day and measure your response time

**Goal:** A timed run of the Q12 playbook, so the "CVE to patched" number is real.

**Steps:**

```bash
# Pick a CVE with a fixed version (for example, a recent curl or openssl CVE).
# Start a timer, then:
# 1. Query your SBOMs: which images carry the vulnerable version?
#    (Use the syft/trivy output from Exercises 1-2, or: grype sbom:nginx-sbom.json)
# 2. Upgrade the base image, rebuild, rescan, re-sign.
# 3. Stop the timer and write down: time to find it, time to patch it.
```

**Expected output:** Two numbers — minutes to find the blast radius, minutes to a signed patched image — plus a note on which step was slowest.

**Why it matters:** This is Q12 — "I found it in four minutes and shipped a patched image in forty" is an interview answer nobody forgets.

---

## Exercise 8: Pin a project properly and set up update automation

**Goal:** A lockfile you trust and an updater that proposes changes you review.

**Steps:**

```bash
# Take a small Node or Python project and pin everything:
# Node: delete node_modules, run `npm install`, commit package-lock.json
# Python: run `pip freeze > requirements.txt` (or `poetry lock`) and commit it
# Then enable Dependabot or Renovate on the repo and watch the first PR arrive.
# Review one update PR properly: changelog, diff, and whether tests still pass.
```

**Expected output:** A committed lockfile, one automated update PR, and a short review note saying why you merged it or held it back.

**Why it matters:** This is Q3 — pinning without updates goes stale, floating without review gets you owned, and the updater PR is the middle ground.

---

## Exercise 9: Compare SPDX and CycloneDX on the same image

**Goal:** A first-hand answer to the format question, not a memorised one.

**Steps:**

```bash
# Generate both formats from the same image:
syft packages nginx:1.27-alpine -o spdx-json > nginx-spdx.json
syft packages nginx:1.27-alpine -o cyclonedx-json > nginx-cyclonedx.json
# Compare them:
jq '.packages | length' nginx-spdx.json
jq '.components | length' nginx-cyclonedx.json
# Open both files and note: which one lists licenses more clearly,
# which one carries vulnerability data more naturally?
```

**Expected output:** Two files with near-identical component counts, and a one-paragraph note on which format felt better for each job.

**Why it matters:** This is Q6 — "I generated both and SPDX was clearer on licenses" beats a definition every time.

---

