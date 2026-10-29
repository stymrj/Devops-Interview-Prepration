# Platform Engineering & IDPs — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Platform engineering | Paved roads that let product teams ship without becoming infra experts |
| IDP | Self-service layer: template in, repo + pipeline + infra out, no ticket |
| Golden path | The one supported way to do a thing; off-road is allowed, you own it |
| Thinnest viable platform | The smallest template that fixes the developers' biggest pain |
| Platform as product | Developers are customers: roadmap, feedback, changelog, satisfaction |

## Platform vs DevOps

| | DevOps | Platform engineering |
|---|---|---|
| What it is | Culture + shared responsibility | The product that makes that culture scale |
| Who builds it | Every team, together | A dedicated platform team |
| Output | Practices, pipelines, ownership | Self-service building blocks + golden paths |
| Success signal | Teams own their services | Teams ship faster without infra expertise |

## IDP building blocks

| Layer | Examples |
|---|---|
| Portal / catalogue | Backstage, Port, custom frontend |
| Templates | Backstage Software Templates, Cookiecutter, Copier |
| Provisioning | Terraform, Crossplane, Pulumi |
| Delivery | ArgoCD, GitHub Actions, Tekton |
| Observability | Pre-wired dashboards, alerts, SLOs per service |
| Governance | Policy as code (OPA/Kyverno), cost showback, SSO/RBAC |

## Backstage vs custom portal

| | Backstage | Custom portal |
|---|---|---|
| Time to value | Fast — catalogue, templates, TechDocs included | Slow — you build everything |
| Ecosystem | Large plugin library | Whatever you write |
| Pick when | Default choice for most teams | Genuinely unusual workflows (e.g. bespoke regulated approvals) |
| Trap | A portal with no golden paths is a catalogue | A frontend that doesn't fix the automation behind it |

## Abstraction levels

| Level | Example | Verdict |
|---|---|---|
| Too thin | "Write your own StatefulSet + storage class" | Just moved the YAML |
| Just right | "I need Postgres with daily backups" | Developer intent, platform decides how |
| Too thick | One button, zero knobs | Stuck the day someone needs a different instance type |
| Rule | Sensible defaults + documented escape hatches | Common case one click, uncommon case an override |

## Measuring the platform

| Measure | Why it matters |
|---|---|
| Lead time (commit → prod) | The outcome developers actually feel |
| Deployment frequency | Are teams shipping more often? |
| Golden-path adoption % | Are new services starting from templates? |
| Developer satisfaction | A hated platform gets abandoned, whatever its metrics |
| Time to first deploy (new hire) | The IDP day-one test |
| Don't measure | Templates shipped — output, not outcome |

## Day-2 checklist

- [ ] Templates and base images versioned
- [ ] Upgrades canaried to one team first
- [ ] Breaking changes ship with a migration guide + deadline
- [ ] Namespaces isolated (NetworkPolicy) per team
- [ ] Per-team cost visibility (labels → cost report)
- [ ] Deprecation policy published, not improvised

## Classic traps

| Trap | Symptom | Fix |
|---|---|---|
| Building for everyone day 1 | Abstraction so generic it helps nobody | Ship for one team, then generalize |
| Building in isolation | Nobody uses it at launch | Weekly developer interviews, dogfood it |
| Ticket-desk platform | A form that emails a human | Automate provisioning end to end |
| Over-automating early | Automating pain nobody proved real | Automate only proven, repeated pain |
| Mandated adoption | Shadow platforms appear | Win one friendly team; let success sell it |
