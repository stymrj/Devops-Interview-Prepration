# Incident Response & On-Call Engineering — Hands-On Lab

Nine drills that turn incident-response answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Audit your alerts — page, ticket, or delete

**Goal:** An alert list where every page-worthy alert names a user impact and a first action.

**Steps:**

```bash
# Export your alert rules (Prometheus example) and sort them into three piles:
#   page — a human must act now or the customer loses something
#   ticket — matters, but can wait for working hours
#   delete — pages on a symptom with no user impact, or fires and nobody acts
# For every page alert, check: does it name the user impact and the first
# action in its description? Rewrite any that don't.
# Count your pages per week before and after — the target is a visible drop.
```

**Expected output:** A shorter alert list, every page alert carrying its user impact and first action, and a lower weekly page count.

**Why it matters:** This is Q2 made concrete — "page on user impact, not symptoms" is only real once you've deleted an alert that woke people for nothing.

---

## Exercise 2: Write a multi-window burn alert for one SLO

**Goal:** A burn-rate alert that pages on a real threat to the SLO, not on a blip.

**Steps:**

```bash
# Pick one service and SLO (e.g. checkout, 99.9% availability over 30 days).
# Compute the error budget in minutes: 0.1% of 30 days is about 43 minutes.
# Write a recording rule for the error ratio, then a two-window burn alert
# (1h and 6h, 14.4x burn rate) like the one in Q3, with severity: page.
# Test it: replay a past incident's metrics, or inject errors in staging,
# and confirm it fires during a sustained burn and stays quiet on a 2-minute blip.
```

**Expected output:** An alert that fires on a sustained burn and stays silent on a short spike, with the budget math written in a comment.

**Why it matters:** This is Q3 — burn-rate alerting stops being a formula once you've watched it ignore a blip and catch a real burn.

---

## Exercise 3: Build a four-level severity table for your service

**Goal:** A severity definition anyone on the team can apply in 30 seconds.

**Steps:**

```bash
# Write a one-page table: SEV1-SEV4, with user impact, who gets woken,
# first-update deadline, and who decides it's over for each level.
# Now take five past incidents and re-grade them with your table — at least
# one will land on a different level than the team gave it at the time.
# Share the table in your team's channel and ask for one disagreement each.
```

**Expected output:** A one-page severity table, five re-graded incidents, and at least one disagreement that sharpens a definition.

**Why it matters:** This is Q4 — "set by user impact" is the line interviewers remember, and re-grading old incidents is how you learn to apply it.

---

## Exercise 4: Run a 15-minute incident drill on paper

**Goal:** A first-15-minutes checklist you could follow at 3 AM without thinking.

**Steps:**

```bash
# Take a real past alert (or invent one: "checkout 5xx at 40%") and run it
# as a tabletop exercise with a timer:
#   0-2 min: acknowledge, open the alert link
#   2-5 min: check dashboards — what changed in the last hour?
#   5-10 min: declare severity, open the incident channel
#   10-15 min: decide rollback vs debug forward
# Write every timestamp down as you go, then read the timeline back —
# if it has gaps, your checklist has gaps.
```

**Expected output:** A written timeline with no gaps, and a one-page checklist you could hand to a new on-call.

**Why it matters:** This is Q6 — the checklist only exists once you've timed yourself through it and found where you'd freeze.

---

## Exercise 5: Be incident commander for a mock outage

**Goal:** Practice coordinating an incident without touching the keyboard.

**Steps:**

```bash
# With a teammate, simulate an outage (staging service down, or just a
# scenario on paper). You are incident commander: you may not run any
# command or look at any dashboard yourself.
# Your job: assign roles, keep the timeline, decide the next step when
# the two debuggers disagree, and post a status update every 10 minutes.
# After 20 minutes, debrief: did you stay out of the debugging?
```

**Expected output:** A timeline, three status updates, one decision where you broke a tie — and an honest note on whether you stayed out of the terminal.

**Why it matters:** This is Q5 — "the commander doesn't debug" is easy to say and hard to do, and this drill is where you learn it.

---

## Exercise 6: Write a rollback-first decision tree

**Goal:** A one-page rule for when to roll back, when to debug, and when to fail over.

**Steps:**

```bash
# Draw the tree: did it start within minutes of a deploy or config change?
# Is the change reversible? Is rollback itself risky (migration? schema?)?
# For each leaf, write the action: roll back / debug forward / fail over
# or shed load.
# Test it against three real past incidents — one should surprise you.
```

**Expected output:** A one-page decision tree and three past incidents mapped to it, with at least one that changes how you'd have acted.

**Why it matters:** This is Q8 — the tree stops "debug forward out of pride" before it costs you fifteen minutes of outage.

---

## Exercise 7: Write a real postmortem with owners and dates

**Goal:** A one-page postmortem where every action item has one owner and one date.

**Steps:**

```bash
# Take a real incident (or the one from Exercise 4) and write it up:
# timeline, impact in numbers, root cause as a chain of events,
# and action items — each with ONE owner and ONE due date.
# Now read it and delete any action item that says "be more careful" —
# replace it with the system change that makes the mistake impossible
# or harmless (guardrail, confirmation step, dry-run, circuit breaker).
# Put the action items in the next sprint planning doc.
```

**Expected output:** A one-page postmortem with dated, owned action items and zero "be more careful" lines.

**Why it matters:** This is Q9 — a postmortem that changes things ends in the sprint plan, not in a folder nobody opens.

---

## Exercise 8: Measure your rotation's health for one month

**Goal:** Numbers that prove whether your on-call rotation is sustainable.

**Steps:**

```bash
# For one rotation cycle, track: pages per person, night pages,
# acknowledged-but-no-action alerts, and a one-question survey after
# each turn: "was this week sustainable? (yes/no + one line)".
# At the end of the month, present the numbers: if pages per person
# are climbing or the survey says no, the fix is fewer alerts or
# more people — not more resilience.
```

**Expected output:** A one-month table of pages, night pages, no-action rates, and survey answers — plus the fix it points to.

**Why it matters:** This is Q10 — on-call health is a reliability metric, and this is how you collect it instead of guessing.

---

## Exercise 9: Run a game day in staging

**Goal:** A rehearsed failure that finds broken runbooks, not broken systems.

**Steps:**

```bash
# Pick one failure (kill a zone, stop the database primary, make a
# dependency time out) and run it in staging with the team watching:
# define the blast radius, the abort condition, and the rollback plan
# BEFORE you start. Run it, watch the alerts fire (or not), and follow
# the runbook step by step — note every step that's wrong or missing.
# End with the list of runbook fixes, not a celebration.
```

**Expected output:** A runbook with corrected steps, a list of alerts that fired late or not at all, and a game-day report under one page.

**Why it matters:** This is Q11 — the best game days find broken runbooks, and this is the one you'll describe in interviews.
