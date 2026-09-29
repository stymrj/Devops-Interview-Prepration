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

---

## Toil and Automation

### Q7: What is toil? Give me a real example.

**How to Answer:**

"Toil is the repetitive, manual, automatable operational work that scales with the system but creates no lasting value — and Google's SRE book gives it a strict test. It's toil only if it's manual, repetitive, automatable, tactical rather than strategic, and grows as the service grows.

My real example: every Monday someone on my team manually rotated log files on eight app servers — SSH in, check disk, gzip old logs, delete ancient ones. Pure toil: manual, weekly, scriptable, and it got worse every time we added a server.

The fix took one afternoon: a logrotate config plus a disk-usage alert. That's the SRE move — toil eliminated permanently beats toil done faster. If you just get quicker at manual work, you've optimized the wrong thing."

**Key Point:** "Toil is manual, repetitive, automatable ops work that scales with the system — the SRE answer is to eliminate it with automation, not get faster at it."

---

### Q8: Why does SRE cap operational work at 50%? What happens past that?

**How to Answer:**

"Google's rule is that an SRE should spend at most half their time on ops work — tickets, pages, manual fixes — and at least half on engineering: automation, tooling, reliability improvements. It's a forcing function.

Past 50%, the team is drowning in toil and has no time left to automate their way out, so the toil keeps growing. It's a death spiral: more manual work, less automation, even more manual work next quarter.

The enforcement mechanism is real too: if a service needs more than 50% ops time, SRE hands operational load back to the product team until reliability improves. That creates the right incentive — product teams feel the pain of unreliability directly, so they invest in fixing it."

**Key Point:** "The 50% cap forces engineering time for automation — past it, toil compounds and the team can't automate its way out, so excess load goes back to the product team."

---

## Change Without Fear

### Q9: Why do DevOps teams deploy in small batches?

**How to Answer:**

"Small batches fail small. A deploy with three changes that breaks prod is a ten-minute rollback and an obvious culprit. A deploy with three hundred changes that breaks prod is a war room and a guessing game.

Small batches also ship faster overall, which sounds backwards but isn't. Big releases sit in branches for weeks, merge into conflicts, and need risky big-bang deploys. Small changes flow through the pipeline daily with almost no ceremony.

And there's a human factor: when deploys are boring and frequent, nobody fears them. When deploys are rare and huge, everyone fears them — which makes them rarer and huger. Small batches break that cycle."

**Key Point:** "Small batches fail small, roll back fast, and make deploys boring — big-bang releases do the opposite on all three."

---

### Q10: How does SRE think about risk? Is the goal zero incidents?

**How to Answer:**

"No — the goal is explicitly not zero incidents, because 100% reliability is the wrong target. It's infinitely expensive and it freezes all change. SRE asks instead: how much unreliability can we afford, and spends that budget on shipping features.

That's the error budget idea in one line: if your SLO is 99.9%, you get about 43 minutes of downtime a month to 'spend' on deploys, experiments, and the occasional incident. Budget remaining means ship fast; budget burned means freeze changes and invest in reliability.

This reframes the dev-versus-ops fight completely. It's no longer 'move fast' versus 'don't break things' — it's a shared, numeric agreement both sides can see. Risk becomes a budget you manage, not a fear you obey."

**Key Point:** "SRE doesn't chase zero incidents — it budgets acceptable unreliability with error budgets, so teams ship fast while the budget lasts and harden when it's spent."

---

## Proving It Works

### Q11: How do you measure whether DevOps is actually working?

**How to Answer:**

"I use the four DORA metrics because they're research-backed, not vanity. Deployment frequency and lead time for changes measure speed; change failure rate and mean time to recovery measure stability. Elite teams ship multiple times a day and recover in under an hour.

The key insight from the DORA research is that speed and stability move together — they don't trade off. Teams that deploy often have lower failure rates, because small changes are safer. If someone claims 'we go slow to be safe,' the data says they're wrong on both counts.

Practically, I pull deployment frequency straight from git history, and MTTR from incident timestamps. You don't need fancy tooling to start — a spreadsheet and honest timestamps beat a dashboard nobody looks at."

```bash
$ git log --since="30 days ago" --oneline --grep="deploy" | wc -l
27
```

**Key Point:** "Measure with DORA's four — deployment frequency, lead time, change failure rate, MTTR — and remember the research: speed and stability improve together."

---

### Q12: You join a team with zero DevOps practices. Where do you start?

**How to Answer:**

"I start with visibility before automation — you can't improve what you can't see. First week: get the app building and deploying from a script anyone can run, add basic health checks, and set up one dashboard showing the four DORA-ish numbers, even roughly.

Then I pick the single most painful manual thing and automate it. Not a grand pipeline redesign — one toil task, killed permanently. That builds trust, and trust buys permission for bigger changes.

What I deliberately don't do: arrive with a Kubernetes migration plan on day one. Tool-first transformations fail because the team hasn't felt the pain the tool solves yet. Small wins, measured impact, then bigger bets — in that order."

**Key Point:** "Start with visibility — reproducible deploys, health checks, basic metrics — then kill the most painful toil first. Small measured wins earn permission for bigger changes."
