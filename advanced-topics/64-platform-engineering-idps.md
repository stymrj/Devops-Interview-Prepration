# Platform Engineering & IDPs Interview Preparation Guide

*How to Answer Platform Engineering Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [What Platform Engineering Actually Is](#what-platform-engineering-actually-is)
2. [Building the Internal Developer Platform](#building-the-internal-developer-platform)
3. [Adoption, Measurement & Traps](#adoption-measurement--traps)

---

## What Platform Engineering Actually Is

### Q1: What is platform engineering, in one simple answer?

**How to Answer:**

"I describe platform engineering as building the paved roads that let product teams ship without becoming infrastructure experts. DevOps asked every team to own their pipeline, their cluster, and their Terraform — and most teams drowned in it. A platform team takes that undifferentiated work, turns it into self-service building blocks, and hands it back as something a developer can use in minutes. The goal isn't to remove DevOps thinking. It's to stop making fifty teams solve the same Kubernetes problem fifty different ways."

**Key Point:** "Platform engineering = turning repeated infrastructure work into self-service paved roads, so product teams ship instead of wrestling with YAML."

---

### Q2: How is platform engineering different from DevOps?

**How to Answer:**

"DevOps is a culture — developers and operators sharing responsibility for shipping and running software. Platform engineering is how you make that culture scale. DevOps says 'you build it, you run it,' and the platform team answers 'here's a safe, boring way to do that.' I think of it as DevOps with a product mindset: the platform team has customers, a roadmap, and feedback loops, just like a product team. If DevOps is the philosophy, the internal developer platform is the thing you actually build to live it."

**Key Point:** "DevOps is the culture; platform engineering is the product that makes that culture scalable — the platform team treats developers as customers."

---

### Q3: What exactly is an Internal Developer Platform (IDP)?

**How to Answer:**

"An IDP is the self-service layer between developers and the infrastructure underneath. A developer picks a service template, fills in a few fields, and gets a repo, a pipeline, a namespace, and observability — without filing a ticket or reading a Terraform module. Under the hood it's usually a portal like Backstage, a set of templates, and golden paths wired to real automation. The test I use is simple: can a new developer ship something to production on day one without asking anyone for access or help? If yes, you have an IDP. If not, you have a wiki page."

**Key Point:** "An IDP = self-service templates + automation that take a developer from idea to production without a ticket — day-one deploy is the test."

---

### Q4: What is a golden path, and why does it matter more than a perfect platform?

**How to Answer:**

"A golden path is the one blessed, supported way to do something — deploy a service, add a database, set up an alert. It's opinionated on purpose: this language version, this base image, this pipeline shape. Developers can still go off-road if they really need to, but they own the consequences when they do. Golden paths matter because choice is expensive — every option you offer is something you have to document, secure, and upgrade forever. I'd rather support one great path that eighty percent of teams love than ten mediocre paths nobody maintains."

**Key Point:** "A golden path is the one supported way to do a thing — opinionated by design, because every extra option is a maintenance bill forever."

---

## Building the Internal Developer Platform

### Q5: If you had to build an IDP from scratch, where would you start?

**How to Answer:**

"I'd start by talking to developers about their single biggest pain, not by buying a portal. Usually it's something unglamorous — waiting three days for a namespace, or copying a pipeline from another repo and praying. I'd build the thinnest thing that fixes that pain: a template that scaffolds a repo, a pipeline, and a deployment, wired to the tools we already use. Then I'd watch how teams actually use it and extend from there. Starting with a shiny portal and no golden paths is the classic mistake — you end up with a beautiful catalogue of nothing."

**Key Point:** "Start from the developers' biggest pain, ship the thinnest template that fixes it, then grow — a portal with no golden paths is just a catalogue."

---

### Q6: Backstage vs building your own portal — how do you decide?

**How to Answer:**

"I pick Backstage when I want a proven, open-source developer portal fast — it gives me a software catalogue, templates, and TechDocs out of the box, and there's a huge plugin ecosystem. I build custom only when our workflows are genuinely unusual, like a heavily regulated environment with bespoke approval chains. Even then I'd start from Backstage and extend it, not from a blank page. The portal is the easy part, honestly. The hard part is the automation behind it — the templates, the pipelines, the provisioning — and a custom frontend doesn't fix any of that."

```yaml
# Backstage software template: scaffold a service in one click
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata: { name: nodejs-service, title: New Node.js Service }
spec:
  owner: platform-team
  type: service
  parameters:
    - title: Service details
      properties:
        name: { type: string, title: Service name }
        owner: { type: string, title: Owning team }
  steps:
    - id: fetch
      action: fetch:template
      input: { url: ./content, values: { name: ${{ parameters.name }} } }
    - id: publish
      action: publish:github
      input: { repoUrl: github.com?owner=myorg&repo=${{ parameters.name }} }
```

**Key Point:** "Backstage buys you a catalogue, templates, and docs fast — build custom only for genuinely unusual workflows, and even then extend, don't start blank."

---

### Q7: How do you actually implement self-service provisioning?

**How to Answer:**

"I wire a template to real automation, not to a ticket queue. A developer fills in a form, and behind it a workflow runs Terraform or Crossplane to create the namespace, the database, and the IAM roles, then opens a pull request for review. Everything is GitOps after that — the template output is code in a repo, so changes go through the same review and rollback as any other code. The trap is a form that emails a human. That's not self-service, that's a ticket with better UX. If a human has to approve every request, you've just automated the waiting room."

**Key Point:** "Self-service = a template that runs real provisioning and lands code in Git — a form that emails a human is a ticket with better UX."

---

### Q8: How do you pick the right level of abstraction — not too thin, not too thick?

**How to Answer:**

"I aim for the level where a developer describes what they need, not how the infrastructure works. 'I need a Postgres database with daily backups' is the right level. 'Give me a StatefulSet with these storage classes' is too thin — you've just moved the YAML. And hiding everything behind a single button with no knobs is too thick — the day someone needs a different instance type, they're stuck. My rule is: sensible defaults, visible escape hatches. The common case should be one click, and the uncommon case should be a documented override, not a fork of the platform."

**Key Point:** "Right abstraction = 'I need Postgres with backups,' not raw YAML and not a button with no knobs — sensible defaults plus documented escape hatches."

---

## Adoption, Measurement & Traps

### Q9: How do you get developers to actually adopt the platform instead of working around it?

**How to Answer:**

"I make the golden path genuinely easier than the workaround, and I treat developers like customers, not captives. That means dogfooding it myself, fixing the first ten complaints fast, and publishing a changelog so teams see it improving. I also never mandate adoption on day one — I find one friendly team, make them successful, and let their story do the selling. Mandates create shadow platforms. A platform people choose beats a platform people are forced onto, because forced users route around you the moment you're not looking."

**Key Point:** "Adoption comes from being easier than the workaround — win one friendly team, fix complaints fast, and never mandate on day one or you'll breed shadow platforms."

---

### Q10: How do you measure whether a platform is actually working?

**How to Answer:**

"I measure developer outcomes, not platform activity. Time from first commit to production, lead time for changes, and deployment frequency — the DORA metrics — tell me if teams are actually shipping faster. I also track golden-path adoption: what percentage of new services start from a template versus hand-rolled. And I run a short developer satisfaction survey, because a platform with great metrics that everyone hates is a platform about to be abandoned. What I don't measure is how many templates we shipped — that's output, not outcome, and output is easy to game."

**Key Point:** "Measure outcomes — lead time, deployment frequency, golden-path adoption, developer satisfaction — never count templates shipped; that's output, not outcome."

---

### Q11: How do you handle day-2 platform work — upgrades, breaking changes, multi-tenancy?

**How to Answer:**

"I treat the platform like a product with versions and a deprecation policy, not like a script I can change at will. Base images and templates get versioned, upgrades roll out to one team first, and breaking changes come with a migration guide and a deadline, not a surprise. For multi-tenancy I isolate by namespace and network policy, and I give each team cost visibility so they see their own bill. The unglamorous truth is that day-2 work is most of platform engineering — the launch is the easy ten percent, and the upgrades, support, and migrations are the other ninety."

**Key Point:** "Day-2 is ninety percent of platform work — versioned templates, canary upgrades, deprecation policies, namespace isolation, and per-team cost visibility."

---

### Q12: What are the classic mistakes that kill platform teams?

**How to Answer:**

"The biggest one is building for everyone on day one — you end up with an abstraction so generic it helps nobody. The second is building in isolation without talking to developers, then wondering why nobody uses it. The third is turning the platform team into a ticket desk that provisions things by hand — that's the old ops model with a new name. And the fourth is over-automating too early, before you've learned what developers actually need. I avoid all four the same way: ship small, talk to users weekly, and automate only the pain that's already proven real."

**Key Point:** "Platform teams die from building for everyone at once, building in isolation, becoming a ticket desk, or over-automating too early — ship small and talk to users weekly."

---
