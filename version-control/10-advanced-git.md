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


---

## Bisecting Bugs

### Q7: How does `git bisect` find the commit that introduced a bug?

**How to Answer:**

"Bisect is binary search over your history. I mark one commit as good and one as bad, then Git checks out the middle commit and asks me: good or bad? Each answer halves the search space until it lands on the exact commit that broke things.

The power move is automating it. If I have a script that exits 0 when the bug is absent and non-zero when it's present, `git bisect run ./test.sh` walks the whole thing hands-free. I've had it find a regression across 200 commits in about eight test runs.

When it's done, `git bisect reset` takes me back to where I started. I always run reset — people forget and end up committing from a weird detached state."

```bash
$ git bisect start
$ git bisect bad HEAD
$ git bisect good v2.4.0
$ git bisect run ./reproduce.sh   # 0 = good, non-zero = bad
$ git bisect reset
```

**Key Point:** "Bisect binary-searches history between a good and bad commit — pair it with `git bisect run` and a test script, and always finish with `git bisect reset`."

---

### Q8: When is bisect better than just reading the diff or checking blame?

**How to Answer:**

"Blame and diffs work when I have a suspect. Bisect is for when I don't — the bug appeared somewhere in a month of commits and nobody knows which one. Binary search beats eyeballing a thousand commits every time.

It's also honest in a way blame isn't. Blame points at the last person who touched the line, which is often an innocent refactor. Bisect points at the commit where behavior actually changed, because it's testing behavior, not authorship.

My real-world flow: reproduce the bug with a script first, make sure the script passes on the good commit and fails on the bad one, then hand it to `git bisect run`. If the script is flaky, bisect is useless — garbage in, garbage out."

**Key Point:** "Blame finds who touched a line; bisect finds the commit where behavior changed — use bisect when there's no suspect, with a reliable repro script."

---

## Reflog Recovery

### Q9: What is the reflog, and what can you recover with it?

**How to Answer:**

"The reflog is Git's private diary of where HEAD and branch tips have been — every checkout, commit, rebase, reset, and amend, with timestamps. It's local only, never pushed, and entries expire after 90 days by default.

It's the undo button for Git. Bad rebase? `git reset --hard HEAD@{1}` takes me back to before it. Deleted a branch without the SHA? The reflog has the commit. Force-pushed over someone's work? Their reflog still has it — for 90 days.

The interview trap is thinking reflog is a backup. It's not — it's local, it expires, and `git gc` can prune old entries. It's a safety net, not an archive. For real safety, the work needs to be pushed."

```bash
$ git reflog                     # every move HEAD has made
$ git reset --hard HEAD@{2}      # undo the last two moves
$ git checkout -b rescue HEAD@{1} # branch from a past position
```

**Key Point:** "The reflog logs every local HEAD move for 90 days — it's Git's undo button for bad rebases and deleted branches, but it's local and temporary, not a backup."

---

### Q10: Walk me through recovering from a bad `git reset --hard`.

**How to Answer:**

"Say I ran `git reset --hard` and wiped uncommitted work — no wait, that's unrecoverable, and interviewers test whether I say that. `reset --hard` destroys uncommitted changes permanently. The reflog only saves *committed* work.

But if I reset --hard to the wrong commit and lost commits from the branch tip — that's fully recoverable. `git reflog` shows where HEAD was before the reset, I copy that SHA or use `HEAD@{1}`, and `git reset --hard` back to it. The commits were never deleted, just unreferenced.

So the interview answer has two halves: uncommitted work after --hard is gone forever, which is why I `git stash` before risky resets. Committed work is safe in the reflog for 90 days. Knowing which half applies is the whole point."

**Key Point:** "`reset --hard` kills uncommitted work forever, but committed work survives in the reflog — recover with `git reset --hard HEAD@{1}`, and stash before risky resets."

---

## Submodules

### Q11: What are Git submodules, and why do teams argue about them?

**How to Answer:**

"A submodule is a repo nested inside another repo, pinned to a specific commit. The parent repo doesn't contain the code — it stores a gitlink, basically a pointer saying 'this path is commit X of that repo.' Cloning the parent gives you an empty directory until you run `git submodule update --init`.

Teams use them to share a library or config repo across projects while keeping separate histories. The arguments start because the workflow is fiddly: updating a submodule is a two-step dance — commit in the submodule, then commit the new pointer in the parent. Forget the second step and CI builds the old code.

The other pain is that `git clone` doesn't fetch submodules by default, so new joiners get empty directories and broken builds. `--recurse-submodules` on clone and pull fixes most of it."

**Key Point:** "A submodule pins an external repo at a specific commit via a gitlink — great for shared code, fiddly because updates need two commits and clones need `--recurse-submodules`."

---

### Q12: What are the common submodule pitfalls, and what are the alternatives?

**How to Answer:**

"The big three: detached HEAD inside the submodule — you're always on a raw commit, not a branch, so committing there feels weird. Forgetting to push the submodule before pushing the parent, which leaves CI pointing at a commit that doesn't exist on the server. And merge conflicts on the gitlink, where two branches pinned different submodule commits.

When teams outgrow submodules, the common moves are: a package manager — publish the shared code as an npm/PyPI/Maven artifact instead of a repo reference. Or a monorepo, where everything lives in one repo and the problem disappears entirely.

My take in interviews: submodules are the right call when you need strict version pinning of shared code with independent release cycles — like a shared Terraform modules repo. If the coupling is looser, a package registry is simpler. I name the tradeoff instead of picking a side blindly."

```bash
$ git submodule add https://github.com/org/shared-lib.git libs/shared
$ git clone --recurse-submodules <repo>   # clone WITH submodules
$ git submodule update --remote --merge   # pull latest on tracked branch
```

**Key Point:** "Submodule pitfalls: detached HEAD inside, pushing the parent before the submodule, and gitlink conflicts — prefer package registries for loose coupling, submodules for strict pinning."
