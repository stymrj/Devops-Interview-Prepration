# Advanced Git — Hands-On Lab

*Rebase, cherry-pick, bisect, reflog, and submodules in a scratch repo. Safe to break things — you'll work in `/tmp/advanced-git-lab`.*

**Setup:**

```bash
mkdir -p /tmp/advanced-git-lab && cd /tmp/advanced-git-lab
git init -q -b main && git config user.name "Lab" && git config user.email "lab@example.com"
echo "v1" > app.txt && git add app.txt && git commit -qm "initial commit"
git tag v1.0
```

---

## Exercise 1 — Rebase a feature branch onto main

**Goal:** See rebase replay commits with new hashes.

**Commands:**

```bash
git checkout -qb feature
echo "feature A" >> app.txt && git commit -qam "add feature A"
echo "feature B" >> app.txt && git commit -qam "add feature B"
git checkout -q main && echo "main fix" >> app.txt && git commit -qam "hotfix on main"
git checkout -q feature && git rebase main
git log --oneline --graph -5
```

**Expected output:** A straight line: `hotfix on main` at the bottom, your two feature commits re-applied on top with different SHAs than before the rebase.

**Why it matters:** This is the "rebase rewrites hashes" interview claim, witnessed live. The commits look the same but their SHAs changed because the parent changed.

---

## Exercise 2 — Clean up history with interactive rebase

**Goal:** Squash three messy commits into one.

**Commands:**

```bash
git checkout -qb cleanup
for i in 1 2 3; do echo "wip $i" >> app.txt && git commit -qam "wip $i"; done
GIT_SEQUENCE_EDITOR="sed -i -e '2s/^pick/fixup/' -e '3s/^pick/fixup/'" git rebase -i HEAD~3
git log --oneline -4
```

**Expected output:** One commit `wip 1` containing all three changes, instead of three separate `wip` commits.

**Why it matters:** Interactive rebase is the "clean up before review" skill. The sed trick automates the editor so you can see exactly what squash/fixup does.

---

## Exercise 3 — Abort a rebase gone wrong

**Goal:** Practice the escape hatch before you need it for real.

**Commands:**

```bash
git checkout -qb risky
echo "risky change" > app.txt && git commit -qam "risky change"
git checkout -q main && echo "conflicting change" > app.txt && git commit -qam "conflicting change"
git checkout -q risky && git rebase main || true
git rebase --abort
git log --oneline -2 && git status --short
```

**Expected output:** The rebase stops on a conflict; after `--abort`, the branch is exactly as it was — `risky change` still on top, working tree clean.

**Why it matters:** `--abort` is the reason rebasing is safe to practice: nothing is lost until the rebase completes. Interviewers like hearing that you know the escape hatch.

---

## Exercise 4 — Cherry-pick a hotfix onto another branch

**Goal:** Port one commit without the rest of the branch.

**Commands:**

```bash
git checkout -q main
echo "critical fix" >> app.txt && git commit -qam "fix: critical bug"
FIX=$(git rev-parse HEAD)
git checkout -qb release-1.0 v1.0
git cherry-pick $FIX
git log --oneline -3
```

**Expected output:** `release-1.0` has the `fix: critical bug` commit on top of `v1.0` — but with a different SHA than on main (new parent).

**Why it matters:** The classic hotfix-across-branches scenario. Same change, new hash — and you can now explain *why* the hash differs (content addressing + parent).

---

## Exercise 5 — Cherry-pick a range

**Goal:** Port several commits at once with `A..B` syntax.

**Commands:**

```bash
git checkout -q main
for i in 1 2 3; do echo "patch $i" >> app.txt && git commit -qam "patch $i"; done
START=$(git rev-parse HEAD~3)
git checkout -qb release-2.0 v1.0
git cherry-pick $START..main
git log --oneline -5
```

**Expected output:** `release-2.0` gains `patch 1`, `patch 2`, `patch 3` — everything *after* `$START` up to `main`, in order.

**Why it matters:** Range syntax is the off-by-one trap interviewers probe. Verifying with `git log A..B` first is the habit that prevents porting the wrong commits.

---

## Exercise 6 — Bisect to find a breaking commit

**Goal:** Let binary search find the culprit across history.

**Commands:**

```bash
git checkout -qb buggy main
for i in $(seq 1 16); do echo "line $i" >> app.txt && git commit -qam "change $i"; done
echo "BROKEN" >> app.txt && git commit -qam "change 17 introduces bug"
cat > check.sh <<'EOF'
#!/bin/bash
! grep -q BROKEN app.txt
EOF
chmod +x check.sh
git bisect start && git bisect bad HEAD && git bisect good main
git bisect run ./check.sh
git bisect reset
```

**Expected output:** Bisect reports `change 17 introduces bug` as the first bad commit after ~5 test runs (log₂(17) ≈ 5), then `reset` returns you to the branch.

**Why it matters:** You now have a lived answer for "how do you find which commit broke it" — and you know why the script must be reliable (exit 0 = good).

---

## Exercise 7 — Recover a deleted branch from the reflog

**Goal:** Delete a branch, then resurrect it without knowing the SHA by heart.

**Commands:**

```bash
git checkout -qb doomed main && echo "precious" >> app.txt && git commit -qam "precious work"
git checkout -q main && git branch -D doomed
git reflog | head -6
git checkout -b rescued HEAD@{3}
git log --oneline -2
```

**Expected output:** The reflog shows `precious work` with its position; `rescued` points at it and the work is back. (Adjust `HEAD@{3}` to match your reflog.)

**Why it matters:** The "I deleted my branch" panic, solved in two commands. This is the single most practical recovery skill in the guide.

---

## Exercise 8 — Undo a bad rebase with the reflog

**Goal:** Rebase, regret it, rewind.

**Commands:**

```bash
git checkout -qb rewind-demo main
echo "demo" >> app.txt && git commit -qam "demo commit"
BEFORE=$(git rev-parse HEAD)
git rebase --onto v1.0 main rewind-demo 2>/dev/null || git rebase v1.0
git reflog | head -4
git reset --hard $BEFORE
git log --oneline -2
```

**Expected output:** After the rebase the history looks wrong; `git reset --hard $BEFORE` (or `HEAD@{1}`) restores the exact pre-rebase state.

**Why it matters:** Reflog entries are created for the rebase itself, so the "before" state is always one step back. Knowing this is what makes rebasing fearless.

---

## Exercise 9 — Add and use a submodule

**Goal:** Nest a repo inside a repo and see the gitlink.

**Commands:**

```bash
cd /tmp && git init -q --bare shared-lib.git
cd /tmp/advanced-git-lab
git submodule add /tmp/shared-lib.git libs/shared
git commit -qm "add shared-lib submodule"
git ls-tree HEAD libs/
cat .gitmodules
```

**Expected output:** `git ls-tree` shows `160000 commit <sha> libs/shared` — the gitlink (mode 160000 = submodule pointer), not file contents. `.gitmodules` records the URL and path.

**Why it matters:** The gitlink is the "what does the parent actually store" interview answer — a commit hash, not code. Mode `160000` is the detail that proves you looked.

---

## Exercise 10 — Update a submodule pointer

**Goal:** Practice the two-commit dance of submodule updates.

**Commands:**

```bash
cd /tmp && git clone -q shared-lib.git lib-work && cd lib-work
git config user.email "lab@example.com" && git config user.name "Lab"
echo "lib v2" > lib.txt && git add lib.txt && git commit -qm "lib v2" && git push -q origin HEAD:main
cd /tmp/advanced-git-lab
git submodule update --remote libs/shared
git status --short
git add libs/shared && git commit -qm "bump shared-lib to v2"
git log --oneline -2
```

**Expected output:** `git status` shows `modified: libs/shared (new commits)`; after committing, the parent records the new submodule SHA.

**Why it matters:** This is the fiddly two-step teams complain about — update inside the submodule, then commit the pointer in the parent. Forgetting the second step is the classic CI breakage.

---

**Cleanup:** `cd /tmp && rm -rf /tmp/advanced-git-lab /tmp/shared-lib.git /tmp/lib-work`

*Next: the [cheat sheet](../cheat-sheets/10-advanced-git-cheatsheet.md) for quick revision before the interview.*
