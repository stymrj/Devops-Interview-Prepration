# Branching Strategies — Hands-On Lab

Ten exercises that drill branching models for real. Everything runs locally — the exercises simulate team workflows on a single repo.

```bash
mkdir -p ~/branch-lab && cd ~/branch-lab
git config --global init.defaultBranch main
```

---

## Exercise 1 — Feature branch workflow

**Goal:** Practice the basic branch → commit → merge cycle.

```bash
git init flow && cd flow && git commit --allow-empty -m "init"
git checkout -b feature/add-healthcheck
echo "ok" > health.txt && git add . && git commit -m "add healthcheck"
git checkout main && git merge feature/add-healthcheck
git log --oneline --graph
```

**Expected output:** Graph shows main moving forward with the feature commit merged in.

**Why it matters:** This tiny cycle is the atom of every branching strategy. If it's not boring and smooth here, it won't be at team scale.

---

## Exercise 2 — See what a merge commit does

**Goal:** Compare merge commits vs fast-forward merges.

```bash
git checkout -b feature/banner
echo "v1" > banner.txt && git add . && git commit -m "banner v1"
git checkout main
echo "other" > other.txt && git add . && git commit -m "unrelated work"
git merge feature/banner        # creates a merge commit (histories diverged)
git log --oneline --graph -5
```

**Expected output:** A merge commit with two parents joining the two lines of history.

**Why it matters:** Diverged branches produce merge commits; interviews love asking why a fast-forward didn't happen. The answer is always "main moved while you were away."

---

## Exercise 3 — Simulate trunk-based development

**Goal:** Feel the rhythm of small, frequent merges to main.

```bash
for i in 1 2 3; do
  git checkout -b task-$i main
  echo "change $i" >> changes.txt && git add . && git commit -m "task $i"
  git checkout main && git merge task-$i && git branch -d task-$i
done
git log --oneline
```

**Expected output:** Three clean commits on main, each from a branch that lived for seconds. Branches deleted right after merging.

**Why it matters:** Trunk-based is this loop, repeated daily. Short branches, immediate merge, no lingering.

---

## Exercise 4 — Feature flag in action

**Goal:** Merge unfinished work to main safely using a flag.

```bash
git checkout -b feature/dark-mode
cat > app.sh << 'SCRIPT'
if [ "$FEATURE_DARK_MODE" = "true" ]; then echo "dark UI"; else echo "light UI"; fi
SCRIPT
git add . && git commit -m "dark mode behind flag"
git checkout main && git merge feature/dark-mode
bash app.sh                        # prints "light UI" — safe in prod
FEATURE_DARK_MODE=true bash app.sh # prints "dark UI"
```

**Expected output:** Default run shows old behavior; flag flips it. Main stayed deployable.

**Why it matters:** This is how trunk-based teams merge half-done features. The flag, not the branch, controls what's live.

---

## Exercise 5 — GitFlow setup: main + develop

**Goal:** Build the two long-lived branches GitFlow starts with.

```bash
git checkout -b develop
echo "dev" > dev.txt && git add . && git commit -m "start develop"
git branch                          # shows main, develop, plus feature branches
git checkout -b feature/api develop # features branch off DEVELOP, not main
```

**Expected output:** `develop` exists alongside `main`; the feature branch is based on develop.

**Why it matters:** The single most common GitFlow mistake is branching features off main. Drill the habit: develop is where daily work lands.

---

## Exercise 6 — Cut a release branch

**Goal:** Freeze a version on a release branch while main keeps moving.

```bash
git checkout develop && git checkout -b release/2.4
echo "fix typo" > fix.txt && git add . && git commit -m "fix typo on release branch"
git checkout main && git merge release/2.4     # release ships
git tag -a v2.4.0 -m "Release 2.4.0"
git checkout develop && git merge release/2.4 # don't forget this one!
```

**Expected output:** Tag `v2.4.0` on main; develop also received the stabilization fixes.

**Why it matters:** The develop merge-back is the step people forget — skip it and your release fixes never reach the next version.

---

## Exercise 7 — Hotfix off main

**Goal:** Patch production and propagate the fix everywhere.

```bash
git checkout main
git checkout -b hotfix/timeout-bug
echo "timeout=30" > config.txt && git add . && git commit -m "fix timeout"
git checkout main && git merge hotfix/timeout-bug && git tag v2.4.1
git checkout develop && git merge hotfix/timeout-bug
```

**Expected output:** Fix on main tagged as v2.4.1, and develop carries it too.

**Why it matters:** Hotfixes must land in both release and future work. Practice the double merge until it's automatic.

---

## Exercise 8 — Squash merge

**Goal:** Collapse a messy branch into one clean commit.

```bash
git checkout -b feature/messy main
for m in "wip" "fix typo" "oops" "actually done"; do
  echo "$m" >> messy.txt && git add . && git commit -m "$m"
done
git checkout main && git merge --squash feature/messy
git commit -m "feat: add messy feature (cleaned up)"
git log --oneline -2
```

**Expected output:** Main shows one clean commit instead of four junk ones. The branch's history is gone from main.

**Why it matters:** Squash keeps main's history readable. Know the tradeoff: you lose the branch's intermediate commits forever.

---

## Exercise 9 — Rebase a feature branch

**Goal:** Move your branch onto the latest main without a merge commit.

```bash
git checkout main && echo "base" > base.txt && git add . && git commit -m "base"
git checkout -b feature/rebase-me main~0
git checkout main && echo "new" > new.txt && git add . && git commit -m "main moved"
git checkout feature/rebase-me
git rebase main
git log --oneline --graph -4
```

**Expected output:** Linear history — your branch commits sit on top of the new main, no merge commit.

**Why it matters:** Rebasing keeps feature branches fresh and history linear. Never rebase a branch someone else has pulled — rewriting shared history breaks their clone.

---

## Exercise 10 — Resolve a merge conflict

**Goal:** Handle the conflict every interviewer asks about.

```bash
git checkout -b feature/conflict main
echo "theirs version" > shared.txt && git add . && git commit -m "feature edit"
git checkout main && echo "main version" > shared.txt && git add . && git commit -m "main edit"
git merge feature/conflict   # CONFLICT
cat shared.txt               # shows <<<<<<< markers
echo "resolved version" > shared.txt
git add shared.txt && git commit -m "resolve conflict: keep resolved version"
git log --oneline --graph -4
```

**Expected output:** Merge completes after you edit the file, stage it, and commit.

**Why it matters:** Conflicts aren't errors — they're git asking you to decide. Edit, stage, commit. Say that sentence in the interview.

---

*Done? Run `git log --oneline --graph --all` and read your whole history. If the graph makes sense, you understand branching.*
