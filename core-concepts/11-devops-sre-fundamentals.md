# DevOps and SRE Fundamentals Interview Preparation Guide

*How to Answer DevOps and SRE Fundamentals Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [What DevOps Really Is](#what-devops-really-is)
2. [Culture and Collaboration](#culture-and-collaboration)
3. [What SRE Adds](#what-sre-adds)
4. [Toil and Automation](#toil-and-automation)
5. [Change Without Fear](#change-without-fear)
6. [Proving It Works](#proving-it-works)

---

## What DevOps Really Is

### Q1: What is DevOps, really? Is it a role, a team, or a tool?

**How to Answer:**

"DevOps isn't a job title and it isn't a tool — it's how dev and ops stop throwing work over the wall at each other. The same people who write the code own it in production: they build it, ship it, and get paged for it.

It exists because handoffs made releases slow and blameful. DevOps replaces the handoff with shared ownership, automation, and feedback loops so releases go from quarterly events to daily non-events.

The interview trap is defining it with tools — Jenkins, Docker, Kubernetes. Tools enable DevOps; the practice is cultural: small batches, automate everything repeatable, measure everything, and learn from failure without blame."

**Key Point:** "DevOps is shared ownership of software in production — one team that builds, ships, and runs it, using automation and fast feedback instead of handoffs."

---

### Q2: How is DevOps different from traditional IT operations?

**How to Answer:**

"Traditional ops was a ticket queue sitting between dev and prod. Devs finished code, tossed it over, and ops deployed it weeks later — then both sides blamed each other when it broke at 2am. Slow, siloed, and nobody owned the outcome.

DevOps collapses that wall. Developers carry a pager, infrastructure is code in the same repo, and the pipeline does the deploying. The feedback loop shrinks from weeks to minutes.

The shift I feel most day to day: in traditional ops I waited for permission to change things. In DevOps the pipeline is the permission — automated tests and small deploys make change safe by default instead of scary by default."

**Key Point:** "Traditional ops separated building from running with slow handoffs; DevOps merges them so the same team ships small, deploys often, and owns production."

---

## Culture and Collaboration

### Q3: What does CALMS mean? Which letter matters most?

**How to Answer:**

"CALMS is the DevOps checklist: Culture, Automation, Lean, Measurement, Sharing. Culture means no blame and shared ownership. Automation kills manual repetitive work. Lean means small batches and cutting waste. Measurement means you track what matters. Sharing means knowledge flows between teams.

If I have to pick the most important, it's Culture — because you can buy automation tools but you can't buy trust. A team with great tooling and a blame culture will still hide incidents and ship slowly.

In interviews I tie each letter to something I've done: blameless reviews for Culture, pipelines for Automation, small PRs for Lean, dashboards for Measurement, runbooks and demos for Sharing. That turns an acronym into evidence."

**Key Point:** "CALMS is Culture, Automation, Lean, Measurement, Sharing — and Culture leads, because tools can't fix a team that blames instead of learning."

---

### Q4: What does "you build it, you run it" change in practice?

**How to Answer:**

"It kills the 'works on my machine, not my problem' attitude. When the person who wrote the code gets paged for it at 3am, suddenly logging, metrics, and graceful degradation matter during development — not as an afterthought.

In practice it means devs write runbooks, add health checks, and care about deployability. It also means ops skills spread into dev teams instead of living in one silo that becomes a bottleneck.

The honest caveat: it only works if developers are actually given time and tooling for operations. 'You build it, you run it' without on-call compensation, good observability, or sane alerting is just dumping ops work on devs and calling it culture."

**Key Point:** "'You build it, you run it' makes developers own production, so operability gets built in — but it needs real on-call support and tooling, not just a slogan."

---

## What SRE Adds

### Q5: What is SRE, and where did it come from?

**How to Answer:**

"SRE — Site Reliability Engineering — is Google's answer to 'what if ops were done by software engineers?' It started at Google around 2003 when Ben Treynor's team decided to run massive systems with code instead of manual sysadmin work.

The core idea: operations is a software problem. Instead of hiring more people as systems grow, you automate, and you treat reliability as a feature with the same rigor as product code.

That's why SREs write code — at least half their time goes to engineering work like automation and tooling, not tickets. If a team's 'SRE' is just doing manual ops with a fancier title, it's not SRE, it's rebranded sysadmin."

**Key Point:** "SRE is Google's discipline of treating operations as a software problem — reliability engineered with code, not maintained with manual effort."

---

### Q6: What's the actual difference between DevOps and SRE?

**How to Answer:**

"I think of DevOps as the philosophy and SRE as one concrete implementation of it. DevOps says 'break the wall between dev and ops'; SRE says 'here's exactly how: error budgets, SLIs, toil caps, blameless postmortems.'

They share the same goals — ship fast, stay reliable, automate everything. The difference is specificity: DevOps is cultural and broad, SRE is prescriptive with practices you can copy from Google's books.

In an interview I'd add that they're not competing choices. Plenty of teams do DevOps culture with SRE practices mixed in — error budgets and blameless postmortems work fine even if nobody has 'SRE' in their title."

**Key Point:** "DevOps is the philosophy of shared ownership; SRE is a prescriptive implementation of it — error budgets, SLIs, and toil limits. They complement, not compete."
