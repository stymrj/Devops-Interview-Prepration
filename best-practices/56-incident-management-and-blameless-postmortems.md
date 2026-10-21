# Incident Management & Blameless Postmortems Interview Preparation Guide

*How to Answer Incident Management Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Incident Response Basics](#incident-response-basics)
2. [Incident Roles and Communication](#incident-roles-and-communication)
3. [Blameless Postmortems](#blameless-postmortems)
4. [Preventing Repeat Incidents](#preventing-repeat-incidents)

---

## Incident Response Basics

### Q1: What is an incident, and how do you decide severity levels?

**How to Answer:**

"An incident is any unplanned event that degrades or breaks service for users. I separate it from a bug — bugs live in the backlog, incidents get people woken up. Severity is about impact, not cause: how many users are affected and how badly. Sev-1 means significant user impact or revenue loss, like checkout being down. Sev-2 is partial degradation, like slow dashboards for a subset of users. I define these in a written severity matrix before anything breaks, so nobody debates it at 3 AM."

**Key Point:** "Severity measures user impact, and you define the levels before the incident, not during it."

---

### Q2: Walk me through how you'd respond to a Sev-1 outage.

**How to Answer:**

"First I declare the incident and page the on-call — minutes matter more than a perfect diagnosis. I spin up a dedicated incident channel and name an incident commander so there's one voice. Then I stabilize first and understand later: if I can roll back the last deploy, I do it before I know the root cause. I keep a running timeline in the channel as things happen. Once we're green, I schedule the postmortem and make sure the follow-ups get owners and deadlines."

**Key Point:** "Declare, stabilize, then investigate — in that order."

---

### Q3: What's the difference between MTTD, MTTR, and why do they matter?

**How to Answer:**

"MTTD is mean time to detect — how long between something breaking and us noticing. MTTR is mean time to resolve — how long until users are back to normal. They matter because they turn 'we're bad at incidents' into something you can actually improve. I'd rather cut detection from 30 minutes to 5 with better alerts than shave 2 minutes off a fix. Fast detection gives the team breathing room to make good decisions instead of panicked ones."

**Key Point:** "Measure detection and resolution separately — you can only improve what you measure."

---

## Incident Roles and Communication

### Q4: What is an incident commander, and why does the role matter?

**How to Answer:**

"The incident commander is the single decision-maker during an outage — they run the call, keep the timeline, and decide when to escalate or declare all-clear. Without one, you get five people debugging five theories and nobody talking to users. The commander usually doesn't touch the keyboard; their job is coordination, not fixing. At scale, companies rotate the role so everyone gets practice before a real disaster. It's the difference between an organized response and a group chat."

**Key Point:** "One commander, one voice — coordination is the real bottleneck in outages."

---

### Q5: How do you communicate during an incident?

**How to Answer:**

"I separate internal coordination from external status updates — they need different channels and different tones. Internally, the incident channel gets the timeline: what broke, what's being tried, what's working. Externally, I post status page updates every 30 minutes even if there's nothing new — silence is worse than bad news. I keep updates honest and specific: what we know, what we're doing, when the next update lands. And I avoid blame language entirely, even in the heat of the moment."

**Key Point:** "Internal channel for facts, status page for users — update regularly even with no news."

---

### Q6: How do you handle an incident when you don't know the cause yet?

**How to Answer:**

"I don't wait for a diagnosis to start helping users. If rolling back the last deploy or scaling up might help, I try the safest mitigation first and observe. I work through the change timeline — 'what changed in the last hour?' solves a surprising number of mysteries. I check metrics in layers: saturation, errors, latency, then narrow down the component. And I set a timebox — if my theory isn't paying off in 15 minutes, I switch hypotheses or pull in a second pair of eyes."

**Key Point:** "Mitigate first, diagnose second — and timebox your theories."

---

## Blameless Postmortems

### Q7: What is a blameless postmortem, and why does "blameless" matter?

**How to Answer:**

"A postmortem is a written review after an incident — timeline, root cause, what went well, and what we'll change. Blameless means we never ask 'who messed up'; we ask 'what allowed this to happen.' Blame makes people hide information, and hidden information is how incidents repeat. The engineer who made the mistake is usually the one who understood the system least ambiguously — they're a data source, not a target. Every good postmortem ends with action items that have owners and deadlines, or it's just a therapy session."

**Key Point:** "Blamelessness isn't about feelings — it's how you get honest information out of the people closest to the failure."

---

### Q8: What goes into a good postmortem document?

**How to Answer:**

"I start with a one-paragraph summary anyone can understand, then a minute-by-minute timeline of what actually happened. The root cause section explains the chain of events, not just the final trigger — the trigger is usually boring, the chain is where the lessons are. Then impact: how many users, for how long, what it cost. I list what went well alongside what went badly, because the response deserves credit too. And the most important part: concrete follow-up actions with names and dates attached."

**Key Point:** "Summary, timeline, root cause chain, impact, what went well, and owned action items."

---

### Q9: What's the most common mistake teams make with postmortems?

**How to Answer:**

"Writing the document and then never doing the follow-ups — the action items die in a wiki nobody reads. I fix that by turning every action item into a tracked ticket with an owner and a deadline. The second mistake is stopping at the human error — 'someone ran the wrong command' isn't a root cause, it's a symptom of missing guardrails. The real question is why the wrong command was even possible to run. And the third is doing postmortems only for Sev-1s — near-misses and Sev-2s teach you the same lessons cheaper."

**Key Point:** "A postmortem without tracked, owned follow-ups is just documentation theater."

---

## Preventing Repeat Incidents

### Q10: How do you make sure the same incident doesn't happen twice?

**How to Answer:**

"Every postmortem action item should remove a class of failure, not just patch the specific bug. If a wrong config took us down, I don't just fix the config — I add config validation to the pipeline. If alerts were late, I add the missing monitor, not just apologize for the delay. I also track repeat incidents as a metric — if the same failure mode shows up twice, the first postmortem's actions weren't sufficient. Prevention compounds: each incident should leave the system slightly harder to break."

**Key Point:** "Fix the class of failure, not the instance — and track repeats as a signal your fixes weren't enough."

---

### Q11: What's the relationship between error budgets and incidents?

**How to Answer:**

"An error budget is the amount of unreliability you've agreed to tolerate — say, 0.1% downtime for a 99.9% SLO. Incidents spend that budget, and when it's gone, you stop shipping features and focus on reliability. It turns the dev-vs-ops argument into math instead of politics. I've seen teams where every deploy after a budget burn needed extra review and canary time. The budget makes reliability a shared, measurable goal instead of one team's anxiety."

**Key Point:** "Error budgets turn incidents into a shared currency both teams manage."

---

### Q12: Tell me about an outage you handled. What did you learn?

**How to Answer:**

"I always pick a real example where I was close to the action, not one I heard about secondhand. I describe the symptom in one sentence, then what I actually did — the commands I ran, the decision I made under pressure. The key part is the lesson: what I changed afterward, like adding a missing alert or fixing a runbook. Interviewers are listening for honesty and ownership, not perfection — admitting you were slow or wrong about something makes the story credible. I end with the concrete follow-up I owned, because that shows the incident actually made me better."

**Key Point:** "Real story, your real actions, an honest lesson, and the follow-up you owned."
