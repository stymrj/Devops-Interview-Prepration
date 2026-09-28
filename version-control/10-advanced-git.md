# Advanced Git Interview Preparation Guide

*How to Answer Advanced Git Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.


## Table of Contents

1. [Rebase Fundamentals](#rebase-fundamentals)
2. [Rebase Versus Merge in Practice](#rebase-versus-merge-in-practice)
3. [Cherry Picking](#cherry-picking)
4. [Bisecting Bugs](#bisecting-bugs)
5. [Reflog Recovery](#reflog-recovery)
6. [Submodules](#submodules)

---

## Rebase Fundamentals

### Q1: What does `git rebase` actually do? How is it different from merge?

**How to Answer:**

"Rebase picks up my commits and replays them on top of a new base — it rewrites them with fresh hashes. Merge keeps both histories and ties them together with a merge commit. So rebase gives a linear story, merge preserves what actually happened.

Under the hood, rebase finds the new commits on my branch, stashes them aside, fast-forwards to the target, then re-applies each commit one by one. Every commit gets a new SHA because the parent changed — content addressing again.

I rebase my own feature branch onto main before opening a PR, because the diff is cleaner to review. I never rebase main itself, or anything my teammates have already pulled."

**Key Point:** "Rebase replays commits onto a new base with new hashes for a linear history; merge keeps both timelines and joins them with a merge commit."

---

### Q2: What is interactive rebase, and when would you use it?

**How to Answer:**

"Interactive rebase opens an editor listing my commits, and I can pick, squash, fixup, reword, or drop each one. It's how I clean up a messy branch before anyone reviews it — seven 'wip' commits become two logical ones with good messages.

I run `git rebase -i HEAD~3` all the time before pushing. Squash combines commits, fixup does it without keeping the message, reword fixes a bad message, and drop deletes the commit entirely.

The rule I never break: interactive rebase only on branches nobody else has pulled. Rewriting shared history is what causes those 'my commits vanished' disasters."

```bash
$ git rebase -i HEAD~3
# pick  a1b2c3d  Add login endpoint
# squash e4f5g6h  wip
# reword h7i8j9k  fix typo in auth
```

**Key Point:** "Interactive rebase lets you squash, reword, and reorder your own commits before review — only on branches nobody else has pulled."

---

## Rebase Versus Merge in Practice

### Q3: Rebase or merge — which one, and when?

**How to Answer:**

"I use both, for different jobs. Rebase for my private feature branches to keep the PR history linear and reviewable. Merge for integrating into shared branches like main, because I want the record of what actually landed and when.

The deciding question is: has anyone else seen this history? If no — rebase freely. If yes — merge, because rewriting commits other people already pulled forces them to untangle conflicts.

My team's actual workflow: rebase the feature branch onto main, get review, then merge with a merge commit or squash-merge into main. You get clean feature history and an honest main timeline."

**Key Point:** "Rebase private branches for a clean story; merge into shared branches to preserve real history — the deciding factor is whether others have pulled the commits."

---

### Q4: What goes wrong during a rebase, and how do you fix it?

**How to Answer:**

"Conflicts, same as a merge — except they hit per commit, not once at the end. If my branch has five commits touching the same file as main, I can resolve the same conflict five times. That's the honest cost of a linear history.

The escape hatches are simple: `git rebase --abort` bails out completely and restores the branch to where it was. `git rebase --skip` drops the current commit and moves on. And `git rebase --continue` resumes after I've resolved conflicts.

One tip that saves real pain: `git rerere` — reuse recorded resolution. Once enabled, Git remembers how I resolved a conflict and replays it automatically if the same conflict appears again in later commits of the same rebase."

```bash
$ git rebase main              # conflict on commit 2 of 5
# resolve the files, then:
$ git add . && git rebase --continue
# or bail out entirely:
$ git rebase --abort
```

**Key Point:** "Rebase conflicts arrive per commit — resolve with `--continue`, drop with `--skip`, bail out with `--abort`, and enable `rerere` to stop resolving the same conflict twice."

---

## Cherry Picking

### Q5: What is cherry-picking, and when is it actually the right tool?

**How to Answer:**

"Cherry-pick copies one specific commit onto my current branch — same change, new hash, new parent. It's for when I need exactly one fix somewhere else without dragging the whole branch along.

The classic case: a hotfix landed on main, but the release branch still needs it. I cherry-pick the hotfix commit onto the release branch. Another one: I committed something to the wrong branch — cherry-pick it where it belongs, then clean up.

It keeps the original author but sets me as committer, just like rebase. And the duplicate-commit smell is real — if that commit later merges back normally, you get the change twice in the log. I use it for exceptions, not as a workflow."

**Key Point:** "Cherry-pick copies a single commit to another branch — perfect for hotfixes across release branches, but use it sparingly to avoid duplicate history."

---

### Q6: How do you cherry-pick a range of commits, or handle a cherry-pick conflict?

**How to Answer:**

"For a range, it's `git cherry-pick A..B` — that takes everything after A up to and including B. I double-check the range with `git log A..B --oneline` first, because off-by-one here applies the wrong commits.

Conflicts work like any other: Git stops, I resolve the files, `git add` them, and `git cherry-pick --continue`. `--abort` and `--skip` exist here too, same as rebase.

One gotcha with ranges: merge commits. Cherry-pick doesn't know which parent to follow for a merge, so you have to pass `-m 1` to say 'take the first parent's side.' I hit this the first time I tried to port a merged PR to a release branch."

```bash
$ git log v1.0..main --oneline   # verify the range first
$ git checkout release-1.0
$ git cherry-pick v1.0..main     # everything after v1.0
```

**Key Point:** "`git cherry-pick A..B` ports a range — verify it with `git log A..B` first, and pass `-m 1` when the range includes a merge commit."
