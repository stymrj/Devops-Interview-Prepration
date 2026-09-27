# Branching Strategies Interview Preparation Guide

*How to Answer Branching Strategy Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Branching Models Overview](#branching-models-overview)
2. [Trunk-Based Development](#trunk-based-development)
3. [GitFlow](#gitflow)
4. [GitHub Flow and Release Branches](#github-flow-and-release-branches)
5. [Pull Requests and Merging](#pull-requests-and-merging)

---

## Branching Models Overview

### Q1: Why does a team need a branching strategy at all? Isn't a branch just a branch?

**How to Answer:**

"A branch is cheap — a strategy is about what merges when and into what. Without one, people open branches off random points, PRs sit for weeks, and nobody knows which commit actually went to production.

The strategy answers three questions for me: where does new work start, how does it get reviewed, and what represents 'released.' Once everyone agrees on that, merging stops being a debate.

Honestly, the worst branching messes I've seen weren't a tooling problem. They were five developers with five different ideas of what `develop` meant."

**Key Point:** "A branching strategy answers three questions: where work starts, how it gets reviewed, and what represents released."

---

### Q2: How do you pick between GitFlow, trunk-based, and GitHub Flow?

**How to Answer:**

"I look at two things: release cadence and team size. If you ship multiple times a day with good CI, trunk-based wins — branches are short-lived and merges are boring. If you ship versioned software on a schedule, GitFlow's release branches make sense.

The trap is copying a famous company's model without their constraints. A five-person startup running full GitFlow is carrying process for a problem they don't have.

My default for most product teams is something close to GitHub Flow — main plus short-lived feature branches, PR review, deploy from main. It's the least machinery that still gives you review and rollback."

**Key Point:** "Release cadence plus team size decides it — ship daily, use trunk; ship versions on a schedule, use GitFlow; default is GitHub Flow."

---

## Trunk-Based Development

### Q3: What is trunk-based development?

**How to Answer:**

"Everyone commits to one shared branch — trunk, usually `main` — and they do it at least daily. Feature branches exist but they're tiny: a few hours to a day of work, then merged back.

Long-lived branches basically don't exist. Instead of branching for a week, you branch in the morning and merge by evening.

The whole model only works with strong CI on every commit and a team culture that reviews fast. Without that, it's just chaos with a fancy name."

**Key Point:** "One shared branch, commits at least daily, no long-lived branches — CI and fast review make it possible."

---

### Q4: How do you merge unfinished work to main without breaking production?

**How to Answer:**

"Feature flags. That's the whole trick. The code ships to main — and even to production — but it's dark, toggled off, until it's ready.

I'll also break big features into small shippable chunks. First merge the migration, then the backend with no UI, then the UI behind a flag. Each merge is independently safe.

The anti-pattern is a 'do not deploy' commit sitting in main for a week. If it's in main, it should be deployable. The flag is what makes that true."

```bash
# example: simple env-driven flag in an app
if [ "$FEATURE_NEW_CACHE" = "true" ]; then
  use_new_cache
fi
```

**Key Point:** "Feature flags — unfinished code merges to main but stays dark until it's ready."

---

### Q5: When is trunk-based development a bad idea?

**How to Answer:**

"When you can't afford main to always be releasable. If your test suite takes three hours or your deploys are manual and scary, committing to main daily is just frequent breakage.

It also struggles with teams that need review gates for compliance — some regulated environments want explicit approval chains per change, and a single fast-moving trunk fights that.

And honestly, it needs developers who are comfortable integrating constantly. A team used to month-long branches will find the daily rhythm more painful than the merge conflicts it replaces."

**Key Point:** "Trunk-based fails when CI is weak, deploys are scary, or the team needs heavyweight review gates."

---

## GitFlow

### Q6: What is GitFlow, and what problem was it designed for?

**How to Answer:**

"GitFlow is a branching model for versioned software — think libraries, mobile apps, on-prem releases. It uses long-lived `main` and `develop` branches, plus supporting branches for features, releases, and hotfixes.

The idea was to separate 'what we're building' from 'what we're stabilizing.' Develop is where daily work lands; main only gets release commits. So main is always the last shipped version.

It fits software with release cycles and version numbers. For a SaaS team deploying daily, it's usually too much ceremony."

**Key Point:** "GitFlow separates building (develop) from shipping (main) — built for versioned releases, heavy for continuous deployment."

---

### Q7: Walk me through the branch roles in GitFlow.

**How to Answer:**

"Two long-lived branches: `main` holds the last released version — nothing unreleased ever touches it. `develop` is the integration branch where finished features land for the next release.

Feature branches come off develop and merge back into develop. When you're ready to cut a release, you branch `release/x.y` off develop — that's where you stabilize: bug fixes only, no new features. Then it merges into both main and develop.

Hotfix branches come off main for production emergencies, then merge back into main and develop so the fix isn't lost. That's the whole dance."

```bash
git checkout -b feature/login-oauth develop
# ... work, merge to develop ...
git checkout -b release/2.4.0 develop
git checkout -b hotfix/payment-timeout main
```

**Key Point:** "main = last release, develop = next release, features off develop, releases and hotfixes are supporting branches."

---

### Q8: What's the real-world downside of GitFlow people discover too late?

**How to Answer:**

"Merge hell on long release branches. The release branch lives for weeks, develop keeps moving, and the final merge back is a conflict resolution marathon. I've seen release days turn into release weeks because of it.

The second cost is mental overhead — new hires take a month to stop merging into the wrong branch. 'Which branch does this go to?' becomes a daily question.

And the subtlest one: it delays integration feedback. A feature merged to develop but not released for three weeks means three weeks of not knowing if it actually works in production."

**Key Point:** "Long release branches create merge hell, confuse everyone about merge targets, and delay real production feedback."

---
