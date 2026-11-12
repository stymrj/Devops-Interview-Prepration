# Supply Chain Security — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Supply chain security | Prove nothing between your code and production was tampered with |
| Attack surfaces | Dependencies, the build, signing keys, the base image |
| Dependency pinning | Exact versions + committed lockfiles, no floating "latest" |
| SLSA | Grades how much you can trust your build (L1–L4) |
| SBOM | Ingredient list of your software — generate it in the build |
| SPDX | License-compliance format, ISO standard |
| CycloneDX | Security-workflow format, leaner in CI |
| Provenance | How the artifact was built — repo, commit, builder |
| Cosign | Signs images and artifacts |
| Fulcio | Short-lived certificates from your real identity |
| Rekor | Public log — nothing signed in secret |
| Admission policy | Signed images from your registry only, enforced at the gate |

## SLSA levels, at a glance

| Level | What it guarantees |
|---|---|
| L1 | Build is scripted, provenance recorded |
| L2 | Build runs on a hosted service, not a laptop |
| L3 | Hardened isolated build, provenance can't be faked |
| L4 | Two-person review + hermetic, reproducible build |

## Sigstore, at a glance

| Piece | Job |
|---|---|
| Cosign | Signs and verifies artifacts |
| Fulcio | Issues short-lived certs after OIDC login |
| Rekor | Transparency log of every signature |

## SBOM formats

| Format | Best at | Generate with |
|---|---|---|
| SPDX | Licenses, regulated industries | Syft, Trivy |
| CycloneDX | Security workflows, vulnerability data | Syft, Trivy |

## The pipeline gates

| Gate | Tool | Fail the build when |
|---|---|---|
| SBOM generation | Syft / Trivy | No SBOM produced |
| Vulnerability scan | Trivy / Grype | Critical with a fix available |
| Signing | Cosign | Unsigned artifact |
| Admission | Kyverno / Gatekeeper | Unsigned image or wrong registry |

## Zero-day playbook

| Step | Action |
|---|---|
| 1. Blast radius | Query SBOMs — which services carry it, is the path reachable? |
| 2. Patch or mitigate | Upgrade, or vendor workaround / WAF block |
| 3. Ship properly | Rebuild, re-sign, redeploy through the normal pipeline |
| 4. Record it | Update SBOM, close exception, note CVE-to-patched time |

## Common questions, one-line answers

| Question | The shape of a good answer |
|---|---|
| What is supply chain security? | Everything your code is built from, proven untampered |
| How do attackers get in? | Dependency, build, signing key, base image |
| What is SLSA? | L1 scripted → L2 hosted → L3 tamper-resistant → L4 reproducible |
| What is an SBOM? | Ingredient list, generated in CI, shipped with the artifact |
| SPDX vs CycloneDX? | SPDX for licenses, CycloneDX for security — either beats none |
| How does Sigstore work? | Cosign signs, Fulcio certifies, Rekor logs — keyless |
| How do you verify before deploy? | Check signature + signer identity at admission, fail stops the rollout |
| Vulnerable dependency? | Scan every build, fail on fixable criticals, time-box exceptions |
| Zero-day hits? | SBOM for blast radius, patch, rebuild through the pipeline, record the time |

## Red flags in your own answers

- [ ] "We trust our dependencies" with no scanning or pinning
- [ ] Floating on "latest" with no lockfile
- [ ] An SBOM generated once, by hand, and never updated
- [ ] Signing described but never actually done
- [ ] Verification that only warns — a failed check must stop the deploy
- [ ] Enforcement flipped on day one with no audit-mode run first

## Before the round, check

- [ ] Generated an SBOM with Syft or Trivy at least once
- [ ] Signed an image with Cosign and verified it
- [ ] Can explain SLSA L1–L4 without notes
- [ ] Know the SPDX vs CycloneDX trade-off from having generated both
- [ ] Have a zero-day story: blast radius, patch, time to ship
- [ ] Can walk through a Kyverno signed-image policy line by line

