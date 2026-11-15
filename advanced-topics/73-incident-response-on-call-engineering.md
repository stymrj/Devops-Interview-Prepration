# Incident Response & On-Call Engineering: Paging, Incident Command & Sustainable Rotations Interview Preparation Guide

*How to Answer Incident Response & On-Call Engineering Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [On-Call Design & Alert Quality](#on-call-design--alert-quality)
2. [Running the Incident](#running-the-incident)
3. [After the Incident & On-Call Health](#after-the-incident--on-call-health)

---

## On-Call Design & Alert Quality

### Q1: What makes a good on-call rotation — and what makes a bad one?

**How to Answer:**

"I'd say a good rotation is one where the person on call can actually live their life. That means a follow-the-sun or at least a weekly rotation with enough people that nobody is on call two weeks in a row, a secondary on-call for backup, and a handover note so the next person isn't starting blind. A bad rotation is two people trading a phone forever, no backup, and pages that go to whoever is newest. The test I use in interviews: if the on-call engineer dreads their week, the rotation is broken — not the engineer. Good rotations also pay for the burden, either in money or in time off, because being woken at 3 AM is real work."

**Key Point:** "A good rotation has enough people, a secondary backup, and a handover note — if the on-call engineer dreads their week, the rotation is broken, not the engineer."

---

### Q2: How do you tell a page-worthy alert from one that should just be a ticket?

**How to Answer:**

"I'd use one question: does a human need to act on this right now, or the customer loses something? A page means 'wake someone up, revenue or data is at risk' — error rate over the burn threshold, checkout down, database primary unreachable. A ticket means 'this matters but it can wait for working hours' — disk at 70%, a single pod restarting once, a deploy that finished with warnings. The mistake I call out is paging on symptoms instead of user impact — a CPU spike is not a page, but the latency it causes for real users is. Every alert should name the user impact and the first action, or it shouldn't page anyone."

**Key Point:** "Page only when a human must act right now or the customer loses something — page on user impact, not on symptoms like a CPU spike."

---

### Q3: What is error-budget-based alerting, and why is it better than fixed thresholds?

**How to Answer:**

"I'd explain it as alerting on how fast you're spending your allowed mistakes, not on a magic number. If my SLO is 99.9% availability, I get about 43 minutes of downtime a month — that's my error budget. A multi-window burn alert watches the burn rate: if I'm burning budget 14 times faster than I can afford, it pages me now, because at that rate the budget is gone in hours. A fixed threshold like 'error rate over 5%' can't tell the difference between a five-minute blip and a real outage, so it either pages too much or too late. Burn-rate alerting pages only when the trend actually threatens the SLO, which is why it cuts noise so hard."

```yaml
# Multi-window burn alert: page only when the budget is burning fast
# AND the burn is sustained — a 1h window plus a 6h window.
- alert: ErrorBudgetBurningFast
  expr: |
    job:slo_errors_per_request:ratio_rate1h{job="checkout"} > (14.4 * 0.001)
    and
    job:slo_errors_per_request:ratio_rate6h{job="checkout"} > (14.4 * 0.001)
  for: 2m
  labels:
    severity: page
```

**Key Point:** "Error-budget alerting pages on how fast you're spending your allowed mistakes — a fast burn sustained over two windows means the budget is gone in hours, so it's worth waking someone."

---

### Q4: How do you design severity levels, and what changes at each level?

**How to Answer:**

"I'd keep it to four levels, because more than that and nobody remembers which is which in a panic. SEV1 is a full outage or data loss — all hands, incident commander assigned, updates every 30 minutes, and the response starts immediately. SEV2 is a major feature down for many users — on-call plus the owning team, updates hourly. SEV3 is a degraded experience for some users — handled in working hours with a ticket and a workaround. SEV4 is a minor bug or cosmetic issue — backlog. What changes at each level is three things: who gets woken, how fast the first update goes out, and who decides it's over. And I always add that severity is set by user impact, not by which system broke — a tiny service failing can be a SEV1 if it takes checkout down with it."

**Key Point:** "Four levels, set by user impact not by which system broke — what changes is who gets woken, how fast the first update goes out, and who decides it's over."

---

## Running the Incident

### Q5: What does the incident commander actually do during an outage?

**How to Answer:**

"I'd say the incident commander runs the incident so the engineers can fix it. They own the timeline, they decide what we try next when two people disagree, they keep the communication channel clean, and they make the call to roll back, fail over, or page in more help. The key rule is that the commander doesn't debug — the moment they start typing commands, nobody is coordinating anymore. They also protect the responders from the noise: status updates, stakeholder questions, and the 'any news?' messages all go through the commander. I tell interviewers the best commanders sound boring — calm, short sentences, next step, owner, time — because boring is what a panicking room needs."

**Key Point:** "The incident commander runs the incident so engineers can fix it — they own the timeline, the decisions, and the comms, and they never debug, because the moment they do, nobody is coordinating."

---

### Q6: Walk me through your first 15 minutes after getting paged at 3 AM.

**How to Answer:**

"I'd give it as a checklist, because that's what the interviewer wants to hear. Minute zero to two: acknowledge the page so it stops escalating, and open the alert link — don't guess from the title. Minutes two to five: check the dashboard for what changed — error rate, latency, deploys in the last hour, and whether it's one service or everything. Minutes five to ten: decide if this is a SEV1 and declare it if it is, because declaring early costs nothing and declaring late costs trust. Minutes ten to fifteen: if there's an obvious safe rollback or a recent deploy that matches the start time, do it — don't debug a burning system when you can un-burn it. And the whole time I'm writing timestamps in the incident channel, because the timeline I write at 3 AM is the postmortem I read at 10 AM."

**Key Point:** "Acknowledge, check what changed, declare the severity early, roll back the obvious suspect before debugging — and write timestamps as you go, because that's your postmortem."

---

### Q7: How do you communicate with customers and stakeholders during an incident?

**How to Answer:**

"I'd say the rule is: say what you know, say what you don't, and say when you'll say more. The first update goes out within 30 minutes of declaring a SEV1 — even if it only says 'we're investigating checkout errors since 03:12, next update in 30 minutes.' Never guess a cause in public, never give a recovery time you can't defend, and never go silent because there's nothing new — silence reads as 'nobody is working on it.' I separate the channels too: one channel for responders, one for stakeholders, one for customers, so the debugging doesn't drown in 'any update?' messages. And the all-clear is just as important — say exactly what broke, what fixed it, and what's being watched, so customers don't have to ask."

**Key Point:** "Say what you know, what you don't, and when you'll say more — updates every 30 minutes even with no news, because silence reads as nobody working on it."

---

### Q8: When do you roll back, and when do you keep debugging forward?

**How to Answer:**

"I'd give the decision rule I actually use. If the incident started within minutes of a deploy or a config change, and the change is reversible, I roll back first and debug later — restoring service beats understanding the bug while users are down. If the change was hours ago or the blast radius doesn't match, rolling back is a gamble, so I keep debugging and look for the real cause. The third case is the dangerous one: rolling back itself is risky, like a database migration — then I fail over or shed load instead of rolling back. The mistake I call out is debugging forward out of pride — 'I can fix this in five minutes' said three times is fifteen minutes of outage. Rollback is not admitting defeat, it's choosing the fastest safe path back to green."

**Key Point:** "Roll back first when the incident lines up with a recent reversible change — debugging forward out of pride is how a five-minute outage becomes a fifteen-minute one."

---

## After the Incident & On-Call Health

### Q9: What makes a blameless postmortem actually change things, instead of being a document nobody reads?

**How to Answer:**

"I'd say the difference is action items with owners and dates, not blame. A good postmortem has five parts: a timeline, the impact in numbers, the root cause as a chain of events — not a single person's mistake — the action items, and what we learned. Every action item gets one owner and one due date, and someone reviews them in the next sprint planning, or they rot. The blameless part is structural, not just polite: if a human error caused it, the question is why the system let a tired person at 3 AM make that mistake — was there no guardrail, no confirmation step, no dry-run? The interview line I use: a postmortem that ends with 'engineer will be more careful' has learned nothing, because carefulness is not a system fix."

**Key Point:** "A postmortem that ends with 'be more careful' learned nothing — action items need one owner and one date, and the question is always why the system let a human make that mistake."

---

### Q10: How do you keep on-call sustainable so people don't burn out?

**How to Answer:**

"I'd start with the numbers: if someone is getting paged more than a couple of times a week, or losing sleep most rotations, the system is unhealthy and no amount of resilience training fixes it. The fixes are structural — reduce the alerts first, because every false page is stolen sleep; rotate weekly with at least a month between turns for small teams; give real recovery time after a bad night, not a pat on the back. I also track it openly: pages per rotation, acknowledged-but-no-action rates, and a short survey after each rotation asking 'was this week sustainable?' The interview point is that on-call health is a reliability metric, not a personal toughness test — a burned-out responder makes worse decisions at 3 AM, and that's an incident risk."

**Key Point:** "On-call health is a reliability metric, not a toughness test — track pages per rotation and ask after every turn if the week was sustainable, because a burned-out responder is an incident risk."

---

### Q11: What are game days, and how do you run one without causing a real incident?

**How to Answer:**

"I'd describe a game day as a rehearsed fire drill for production. You pick one failure — a zone goes down, the database primary dies, a dependency starts timing out — and you run it in staging first, then in production with guardrails, during working hours, with the whole team watching. The rules that keep it safe: a big red abort button, a defined blast radius, a rollback plan rehearsed before you start, and a real customer-impact check every few minutes. The point isn't to prove the system survives — it's to prove the team knows what to do, the runbooks are accurate, and the alerts fire when they should. The best game days find broken runbooks, not broken systems, and that's the win."

**Key Point:** "A game day is a fire drill with an abort button and a rehearsed rollback — its real win is finding broken runbooks and silent alerts, not proving the system survives."

---

### Q12: Walk me through an incident you would describe in an interview.

**How to Answer:**

"I'd tell it as a short story with the system fix at the end, not just the drama. Our checkout error rate jumped at 3 AM, and I got paged — the first thing I checked was recent deploys, and there was one ten minutes before the spike, a change to the payment retry logic. I declared a SEV1, rolled the deploy back, and errors dropped within four minutes. The postmortem showed the real cause: the retry change had no rate limit, so one failing payment retried forever and took down the pool. We fixed the category, not just the bug — a retry budget and a circuit breaker on every payment call, plus a deploy-time check that pages us if error rate moves within five minutes. The interview point is the ending: the rollback fixed the night, but the circuit breaker is what stops the next one."

**Key Point:** "Tell it as page → check deploys → declare → roll back → postmortem → fix the category, not the bug — the circuit breaker that stops the next incident is the real answer."
