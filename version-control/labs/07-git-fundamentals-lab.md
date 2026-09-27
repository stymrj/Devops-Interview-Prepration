# Git Fundamentals & Daily Workflow — Hands-On Lab

Ten exercises that drill the exact habits from the interview guide. Everything runs locally — no remote needed except exercise 10, and that one uses a repo you create.

```bash
mkdir -p ~/git-lab && cd ~/git-lab
git config --global init.defaultBranch main
```

---

## Exercise 1 — Initialize and inspect

**Goal:** Create a repo and understand what `.git` actually contains.

```bash
git init demo && cd demo
ls .git/                     # refs, objects, HEAD — the whole repo
cat .git/HEAD                # ref: refs/heads/main — HEAD points at your branch
git status
```

**Expected output:** `.git/` shows `objects/`, `refs/heads/`, `HEAD`, `config`. `git status` says "No commits yet" on branch `main`.

**Why it matters:** A repo is just a directory plus history. Nothing is remote until you say so.

---

## Exercise 2 — Configure identity

**Goal:** Set who you are so your commits aren't anonymous.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --list | grep -E "user|init"
```

**Expected output:** Your name, email, and `init.defaultBranch=main` listed.

**Why it matters:** Every commit carries an author. Missing identity is the most common "why does git complain" moment on a fresh machine.

---

## Exercise 3 — First commit

**Goal:** Stage and commit properly, then read the log.

```bash
echo "# demo" > README.md
echo "*.log" > .gitignore
git add README.md .gitignore
git commit -m "chore: initialize repo with readme and gitignore"
git log --oneline
```

**Expected output:** One commit line like `a1b2c3d chore: initialize repo with readme and gitignore`.

**Why it matters:** `.gitignore` goes in before the first real commit — secrets committed by accident are hard to truly erase.

---

## Exercise 4 — Stage hunks selectively

**Goal:** Split one messy edit into two clean commits with `git add -p`.

```bash
printf "line1\nline2\nline3\nline4\n" > notes.txt
git add notes.txt && git commit -m "chore: add notes file"
printf "line1\nLINE2-fixed\nline3\nLINE4-fixed\n" > notes.txt
git add -p notes.txt      # answer 'y' to the first hunk, 'n' to the second
git commit -m "fix: correct line 2"
git log --oneline -- notes.txt
```

**Expected output:** Two commits on notes.txt — the original plus "fix: correct line 2". Line 4's fix is still unstaged (`git status` shows it as modified).

**Why it matters:** `git add -p` is how you turn messy work into atomic, reviewable commits.

---

## Exercise 5 — Branch and merge

**Goal:** Feel how cheap branches are, then merge one.

```bash
git checkout -b feat/greeting
echo "hello" > hello.txt && git add hello.txt
git commit -m "feat: add greeting file"
git checkout main
git merge feat/greeting --no-edit
git log --oneline --graph | head -5
git branch -d feat/greeting
```

**Expected output:** The graph shows the branch merging into main, then the branch is deleted.

**Why it matters:** A branch is a pointer move. Create, merge, delete — that's the whole lifecycle.

---

## Exercise 6 — Undo the safe way vs the local way

**Goal:** Compare `revert` (safe, pushed) with `reset --soft` (local only).

```bash
echo "oops" > oops.txt && git add oops.txt && git commit -m "chore: accidental file"
git revert HEAD --no-edit          # creates an "undo" commit, history intact
git log --oneline | head -3
echo "scratch" > scratch.txt && git add scratch.txt && git commit -m "chore: scratch"
git reset --soft HEAD~1            # commit vanishes, changes stay staged
git status --short
```

**Expected output:** Log shows the revert commit on top. After `reset --soft`, `scratch.txt` is staged (shown as `A`) but there's no commit for it.

**Why it matters:** Revert undoes by adding — safe for shared history. Reset rewrites — local branches only.

---

## Exercise 7 — Stash a context switch

**Goal:** Park work mid-task, switch branches, come back.

```bash
echo "half done" >> notes.txt
git stash push -m "wip: notes update"
git status --short                 # clean — the change is parked
git stash list
git stash pop
git status --short                 # change is back
```

**Expected output:** Stash list shows your entry; status is clean after push, dirty again after pop.

**Why it matters:** Clean working tree between context switches — no more "I'll commit this broken thing just to switch."

---

## Exercise 8 — Make a merge conflict and resolve it

**Goal:** Cause a conflict on purpose, then fix it deliberately.

```bash
git checkout -b side-a && echo "version A" > conflict.txt && git add conflict.txt && git commit -m "feat: version A"
git checkout main && git checkout -b side-b && echo "version B" > conflict.txt && git add conflict.txt && git commit -m "feat: version B"
git merge side-a      # CONFLICT — same line, different content
cat conflict.txt      # read the <<<<<<< markers
```

**Expected output:** `git merge side-a` reports "CONFLICT (add/add)" and stops. The file shows `<<<<<<< HEAD` / `=======` / `>>>>>>> side-a` markers.

**Resolution:** Edit the file to keep the winning version, delete the markers, then `git add conflict.txt && git commit`. Check `git log --oneline --graph`.

**Why it matters:** Conflicts are questions, not errors. Read the markers, choose, stage, commit.

---

## Exercise 9 — Tidy history with interactive rebase

**Goal:** Squash messy commits into one clean commit (on your own branch only).

```bash
git checkout main && git checkout -b tidy-me
echo 1 >> log.txt && git add log.txt && git commit -m "wip"
echo 2 >> log.txt && git add log.txt && git commit -m "wip2"
echo 3 >> log.txt && git add log.txt && git commit -m "actually done"
GIT_SEQUENCE_EDITOR="sed -i -e '2,3s/^pick/fixup/'" git rebase -i HEAD~3
git log --oneline | head -3
```

**Expected output:** Three "wip" commits collapse into one. The log shows a single commit for the branch's work.

**Why it matters:** This is what you do before opening a PR — turn your draft commits into a story a reviewer can read.

---

## Exercise 10 — Push to a remote (GitHub)

**Goal:** Connect a local repo to GitHub and push a branch.

```bash
# create an empty repo named git-lab on github.com first, then:
git remote add origin git@github.com:<your-user>/git-lab.git
git push -u origin main
git checkout -b feat/first-push && echo x > x.txt && git add x.txt && git commit -m "feat: first push"
git push
```

**Expected output:** `git push -u origin main` uploads and sets the upstream. The second push needs no arguments thanks to `-u`.

**Why it matters:** `-u` sets the upstream once — after that, plain `git push` and `git pull` just work.

---

*Done? Run `git log --oneline --graph --all` and admire the history you built.*
