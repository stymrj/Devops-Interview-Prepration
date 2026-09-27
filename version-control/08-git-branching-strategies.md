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

## GitHub Flow and Release Branches

### Q9: What is GitHub Flow, and how does it differ from GitFlow?

**How to Answer:**

"GitHub Flow is GitFlow's simpler sibling. One long-lived branch — main — and everything else is a short-lived feature branch off main, merged back via pull request.

There's no develop branch, no release branch ceremony. Main is always deployable, and you deploy straight from it after the PR merges. Tags mark releases.

The difference from GitFlow is basically philosophy: GitHub Flow assumes you deploy continuously, so there's no 'release' to prepare. GitFlow assumes releases are events, so it builds branches around them."

**Key Point:** "GitHub Flow: main plus short feature branches, PR review, deploy from main — built for continuous deployment, not release events."

---

### Q10: How do release branches work for teams that do need versions?

**How to Answer:**

"When you ship versions — SDKs, on-prem software, mobile apps — you branch off main at the release point, like `release/2.4`. That branch freezes the version while main keeps moving.

Bug fixes for that version go onto the release branch and get cherry-picked or merged where needed. You tag the actual release from it, like `v2.4.0`.

The rule I enforce: release branches are for stabilization and fixes only. The moment someone sneaks a feature into a release branch, you've lost the point of having it."

```bash
git checkout -b release/2.4 main
git tag -a v2.4.0 -m "Release 2.4.0"
git push origin release/2.4 v2.4.0
```

**Key Point:** "Release branches freeze a version for stabilization while main moves on — fixes only, never new features."

---

### Q11: How do hotfixes differ between trunk-based and GitFlow teams?

**How to Answer:**

"In trunk-based, a hotfix is just another tiny branch off main — you fix, review fast, merge, deploy. The whole cycle is minutes because main is already deployable.

In GitFlow, you branch off main, fix, then merge into both main and develop. That double merge exists so the fix lands in the current release and doesn't get lost in the next one.

The GitFlow mistake I watch for is forgetting the develop merge. Skip it and the same bug ships again in the next release, which is embarrassing in a way I know firsthand."

**Key Point:** "Trunk-based hotfix: branch off main, merge, deploy — done. GitFlow hotfix: merge into main AND develop so the fix survives."

---

## Pull Requests and Merging

### Q12: What makes a pull request actually reviewable?

**How to Answer:**

"Small. That's ninety percent of it. A PR under 200 lines gets a real review; a 2,000-line PR gets a rubber stamp. I break work into stacked or sequential small PRs instead of one mega-PR.

The description should say what changed and why — not what the code says, but the reasoning a reviewer can't see in the diff. Link the ticket, mention the risky parts.

And I keep the branch short-lived. A PR open for three weeks has merge conflicts, stale reviews, and a reviewer who has to re-learn the context. Old branches are where quality goes to die."

**Key Point:** "Small diffs, explain the why in the description, and merge fast — old branches get rubber-stamp reviews."

---

### Q13: Merge commit, squash merge, or rebase merge — when do you use each?

**How to Answer:**

"For feature branches into main, I default to squash merge. It turns a messy 15-commit branch into one clean commit on main with a proper message. Main's history reads like a changelog.

I use regular merge commits when the branch history itself is valuable — release branches, long-lived integration work — because squash would destroy information.

Rebase-and-merge gives a linear history but rewrites commits, so I only use it on branches nobody else has pulled. Rewriting shared history is how you ruin someone's morning."

```bash
git checkout main && git merge feature/x     # merge commit: preserves history
git merge --squash feature/x && git commit   # squash: one clean commit
```

**Key Point:** "Squash for features (clean changelog), merge commits when history matters, never rewrite history others have pulled."

---

*Day 8 of 58 — next: Git internals (blobs, trees, commits, refs).*
