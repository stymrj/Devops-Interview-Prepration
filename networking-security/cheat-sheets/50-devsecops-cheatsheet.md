# DevSecOps Cheat Sheet

*Shift-left security commands and gates — one page.*

## Core idea
Security automated into the pipeline, not bolted on at the end. Fast checks on PRs, slow checks nightly.

## Secrets: prevention
```bash
gitleaks detect --source . --no-git -v          # scan a tree
gitleaks protect --staged -v                    # pre-commit hook
trufflehog git https://github.com/org/repo --only-verified
```
Rules of thumb: scan every PR, inject secrets at deploy time from a secrets manager, rotate aggressively.

## SAST (reads code, no running)
```bash
semgrep --config auto src/                       # fast, multi-lang
semgrep --config rules/my-rules.yaml src/        # custom rules
```
Tools: Semgrep (custom rules easy), CodeQL (deep, GitHub-native), SonarQube (bigger codebases).
Gate: PR-level. Block on high/critical; advisory for the rest. Track per-rule false-positive rates.

## DAST (attacks the running app)
```bash
docker run -t zaproxy/zap-stable zap-baseline.py -t https://staging.example.com
```
Schedule: nightly against staging/preview, full active scan on release candidates. Not on every PR.

## SAST vs DAST
SAST = code bugs, early, cheap (SQLi in a function). DAST = runtime bugs, later (auth bypass, missing headers). Neither replaces a pen test.

## Image scanning
```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed myapp:latest
trivy fs --severity HIGH,CRITICAL .              # scan the repo filesystem
grype myapp:latest                              # alternative scanner
```
Gate on HIGH/CRITICAL **with available fixes**. Scan on build AND on a schedule (CVEs publish after your build date).

## CVE triage
CVE = known vuln with ID + CVSS score (1-10). Score = severity; exploitability = whether it matters to you. Zero-CVE is not the goal — deliberate decisions on every CVE is.

## SBOM (ingredients label)
```bash
syft myapp:latest -o cyclonedx-json > sbom.json
syft myapp:latest -o spdx-json > sbom.spdx.json
```
Formats: SPDX, CycloneDX. Generate per release, store with the artifact. Log4Shell answer: query the SBOM, don't grep repos.

## Supply chain defense
- Lockfiles: pin exact dependency versions (no floating ranges).
- Signatures: `cosign sign --yes img:tag` / `cosign verify` — proves the image you run is the one you scanned.
- Trusted registries only; scan OS packages + language deps on every build.
- Minimal base images (distroless): fewer packages, fewer CVEs, no shell.

## IaC scanning
```bash
checkov -f main.tf        # Terraform, CloudFormation, Dockerfiles
tfsec .                   # Terraform-focused, fast
```
Catches public buckets, open security groups, missing encryption — before they deploy.

## Dockerfile hardening quick list
`FROM` pinned digest + minimal base → `USER nonroot` → no secrets in `ENV`/`COPY` → `.dockerignore` the repo → healthcheck defined.

## Pipeline gate layout
| Stage | Checks | Speed |
|---|---|---|
| PR | gitleaks, semgrep, trivy on image, IaC scan | seconds |
| Nightly | ZAP full scan, scheduled trivy rescan | minutes |
| Release | SBOM generation + storage, cosign sign | minutes |

## Interview one-liners
- "Shift left: a leaked secret caught in a PR takes 30 seconds; in prod it's an incident."
- "Gate builds on high/critical with fixes; make everything else advisory."
- "A scanner that cries wolf teaches developers to dismiss real findings too."

*Day 50 of 58 — networking-security.*
