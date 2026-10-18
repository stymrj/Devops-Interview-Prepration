# DevSecOps Hands-On Lab

*9 exercises that teach you to secure a pipeline the way interviews expect.*

**Prerequisites:** Docker installed, a GitHub account, and a test repo you don't mind making dirty. Use a disposable AWS key — never your real one.

---

## Exercise 1: Block Secrets at Commit Time

**Goal:** Install gitleaks and catch a leaked secret before it commits.

**Commands:**
```bash
# Install gitleaks
curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.24.2/gitleaks_8.24.2_linux_x64.tar.gz | tar -xzf - gitleaks
# Fake a leak in your test repo
echo "AWS_KEY=AKIAIOSFODNN7EXAMPLE" >> app/config.env
./gitleaks detect --source . --no-git -v
```

**Expected output:** A finding naming `app/config.env`, rule `aws-access-key`, and the offending line redacted.

**Why it matters:** Interviewers want to hear "prevention beats detection." A pre-commit scan means leaked secrets never reach the remote — no rotation, no incident.

---

## Exercise 2: Write a Custom Semgrep Rule

**Goal:** Catch an insecure pattern in your own codebase with a rule you wrote.

**Commands:**
```bash
pip install semgrep
# Write a rule: flag subprocess with shell=True in Python
cat > rules/insecure-shell.yaml << 'EOF'
rules:
  - id: python-subprocess-shell-true
    patterns:
      - pattern: subprocess.$FUNC(..., shell=True, ...)
    message: "shell=True invites command injection; use argument lists instead"
    languages: [python]
    severity: WARNING
EOF
semgrep --config rules/insecure-shell.yaml src/
```

**Expected output:** Findings pointing at the exact lines using `shell=True`, with your custom message.

**Why it matters:** SAST isn't just "run the tool." Writing your own rules for your org's patterns is what senior engineers do.

---

## Exercise 3: Scan an Image with Trivy and Gate on Severity

**Goal:** Scan an image and fail a CI step on high/critical CVEs.

**Commands:**
```bash
# Build a deliberately old base for findings
docker build -t vuln-demo - << 'EOF'
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y curl
EOF
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed vuln-demo
echo "exit code: $?"
```

**Expected output:** A table of HIGH/CRITICAL CVEs and a non-zero exit code — that's your CI gate.

**Why it matters:** `--exit-code 1` is the one-liner that turns a scan into an enforcement point. Know the `--ignore-unfixed` flag — blocking on unfixable CVEs is how you get the gate disabled.

---

## Exercise 4: Generate an SBOM for an Image

**Goal:** Produce a CycloneDX SBOM for a running image.

**Commands:**
```bash
# Install syft
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
syft vuln-demo -o cyclonedx-json > sbom.json
jq '.bomFormat, .components | length' sbom.json
```

**Expected output:** `CycloneDX` and a component count (dozens of OS packages even for a slim image).

**Why it matters:** Log4Shell is the interview anchor — teams with SBOMs answered "are we affected?" in minutes. Generating one per release is the concrete habit.

---

## Exercise 5: Sign an Image with Cosign

**Goal:** Sign an image so consumers can verify it wasn't tampered with.

**Commands:**
```bash
# Install cosign
go install github.com/sigstore/cosign/v2/cmd/cosign@latest
# Keyless signing with OIDC (works from GitHub Actions too)
cosign sign --yes ghcr.io/YOURUSER/vuln-demo:latest
cosign verify ghcr.io/YOURUSER/vuln-demo:latest \
  --certificate-identity-regexp ".*" --certificate-oidc-issuer-regexp ".*"
```

**Expected output:** A `Verified OK` message with the signing identity.

**Why it matters:** Signing closes the supply-chain loop: scan tells you it's clean *now*, the signature proves the image you're running is the one you scanned.

---

## Exercise 6: Harden a Dockerfile (Distroless + Non-Root)

**Goal:** Rebuild the demo image with a minimal base and no root user.

**Commands:**
```bash
docker build -t hardened-demo - << 'EOF'
FROM golang:1.22 AS build
WORKDIR /src
COPY main.go .
RUN CGO_ENABLED=0 go build -o app .

FROM gcr.io/distroless/static:nonroot
COPY --from=build /src/app /app
USER nonroot
ENTRYPOINT ["/app"]
EOF
trivy image --severity HIGH,CRITICAL hardened-demo | tail -5
```

**Expected output:** Drastically fewer findings than Exercise 3, and `docker run hardened-demo whoami` fails — there's no shell at all.

**Why it matters:** Smaller attack surface is defense, not hygiene. Fewer packages = fewer CVEs; no shell = nowhere for an attacker to land.

---

## Exercise 7: Scan Infrastructure Code (tfsec/checkov)

**Goal:** Catch an insecure Terraform pattern before it deploys.

**Commands:**
```bash
pip install checkov
# A deliberately bad bucket config
cat > s3.tf << 'EOF'
resource "aws_s3_bucket" "data" { bucket = "my-data" }
EOF
checkov -f s3.tf
```

**Expected output:** Failures like "S3 bucket has an ACL defined which allows public READ access" and a versioning/encryption check.

**Why it matters:** Security of the app means nothing if the bucket is public. IaC scanning extends the same shift-left idea to infrastructure.

---

## Exercise 8: Run a DAST Baseline Scan on a Test Target

**Goal:** Run OWASP ZAP's baseline scan against a known-vulnerable app.

**Commands:**
```bash
docker run -d -p 3000:3000 bkimminich/juice-shop
docker run -t zaproxy/zap-stable zap-baseline.py -t http://host.docker.internal:3000
```

**Expected output:** A report listing real issues — missing security headers, cookie flags, etc. — with risk ratings.

**Why it matters:** Running the scan yourself turns "DAST attacks the running app" from a memorized line into something you've actually seen.

---

## Exercise 9: Build a Security Gate Pipeline

**Goal:** Wire exercises 1–4 into a single GitHub Actions workflow.

**Commands:** Create `.github/workflows/security.yml` in your test repo:
```yaml
name: security
on: [pull_request]
jobs:
  gates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gitleaks/gitleaks-action@v2
      - uses: aquasecurity/trivy-action@master
        with: { image-ref: 'vuln-demo', exit-code: '1', severity: 'HIGH,CRITICAL' }
      - uses: anchore/sbom-action@v0
        with: { image: 'vuln-demo', format: 'cyclonedx-json' }
```

**Expected output:** A PR check named `security` that goes red on leaks or CVEs.

**Why it matters:** This is the interview closer: "I don't just know the tools, I wired them into a pipeline." Fast gates on PRs, slow scans scheduled nightly — that's the architecture answer in code.

---

*Day 50 of 58 — networking-security lab.*
