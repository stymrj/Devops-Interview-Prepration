# Git Fundamentals & Daily Workflow Interview Preparation Guide

*How to Answer Git Fundamentals & Daily Workflow Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Git Basics](#git-basics)
2. [Staging and Committing](#staging-and-committing)
3. [Branching Essentials](#branching-essentials)
4. [Undoing Mistakes](#undoing-mistakes)
5. [Working With Remotes](#working-with-remotes)
6. [Daily Workflow and Hygiene](#daily-workflow-and-hygiene)

---

## Git Basics

### Q1: What is Git, and why does every DevOps workflow depend on it?

**How to Answer:**

"Git is a distributed version control system — every clone carries the full history, not just the latest files. That's why I can commit, branch, and review on a plane and sync later.

For DevOps it's the source of truth everything else hangs off. Terraform plans from a commit hash, CI pipelines trigger on pushes, GitOps deploys from a branch — if git is messy, all of that is messy.

Distributed also means there's no single point of failure for code. Lose the server, and anyone's clone can rebuild it. That's the property I care about most."

**Key Point:** "Git is distributed — full history in every clone — and it's the source of truth CI, IaC, and GitOps all hang off."

---

### Q2: How do you set up a new repo and configure Git on a fresh machine?

**How to Answer:**

"I run `git config --global user.name` and `user.email` first, because every commit you make without those is an identity headache later. I also set `init.defaultBranch main` so new repos don't surprise me with master.

`git init` creates the repo locally — it's just a `.git` directory, nothing leaves my machine until I add a remote. Then I add a `.gitignore` before the first commit so I'm not committing `.env` files or `node_modules` by accident.

My global `.gitignore` covers OS junk like `.DS_Store` everywhere. And I alias `st` to `status -sb` — I check status dozens of times a day, so the short flag earns its keep."

```bash
git config --global user.name "Satyam Raj"
git config --global user.email "satyamg567@gmail.com"
git config --global init.defaultBranch main
git init my-project && cd my-project
```

**Key Point:** "Configure identity and default branch first, `git init` is local-only, and `.gitignore` goes in before the first commit."

---

## Staging and Committing

### Q3: What is the staging area? Why not just commit files directly?

**How to Answer:**

"The staging area — the index — is the draft of my next commit. `git add` moves changes there, `git commit` snapshots what's staged. Two steps, and that separation is the whole point.

It lets me craft precise commits. If I fixed a bug and also reformatted a file, I stage only the bug-fix hunks with `git add -p` and keep the commits clean. Reviewers read commits, not diffs.

Working tree, staging area, repository — I think of them as 'messy desk, outgoing tray, archive.' Knowing which state a file is in tells me which command to reach for."

```bash
git add -p                  # stage interactively, hunk by hunk
git diff                    # unstaged changes (working tree vs index)
git diff --cached           # staged changes (index vs HEAD)
```

**Key Point:** "The index is a draft of the next commit — `git add -p` stages hunks selectively so each commit tells one story."

---

### Q4: What does a good commit message look like? Give an example.

**How to Answer:**

"I write in the imperative — 'Fix retry backoff in deploy script,' not 'Fixed' or 'fixes.' It reads like a command the commit performs, which matches how git itself generates messages like 'Merge branch...'

First line under 50 characters, then a blank line, then the why — not the what, the diff already shows that. If it closes a ticket I reference it: 'Fixes #214.' Future me will thank present me.

Atomic commits are the bigger rule. One commit, one logical change. A commit that fixes the bug and reformats and adds a feature is three commits wearing a trench coat."

**Key Point:** "Imperative first line under 50 chars, explain the why in the body, one logical change per commit."

---

## Branching Essentials

### Q5: How do branches actually work under the hood?

**How to Answer:**

"A branch is just a pointer to a commit — a 40-byte file in `.git/refs/heads/`. Creating a branch costs nothing because git isn't copying files, it's writing a label.

`HEAD` is the pointer to your current branch, so checking out just moves HEAD and rewrites the working tree. That's why branch switches are instant even in huge repos.

This is the thing that makes git workflows cheap. In old centralized systems branching was a ceremony; in git it's a pointer move, so I branch for everything — experiments, fixes, config changes."

**Key Point:** "A branch is a lightweight pointer to a commit — branching costs nothing, so branch for everything."

---

### Q6: Walk me through your typical branch workflow for a feature.

**How to Answer:**

"I start from an updated main — `git checkout main && git pull --ff-only`, then `git checkout -b feat/retry-backoff`. Branch names get a prefix: feat, fix, chore, hotfix. It makes the branch list self-documenting.

I commit as I go on the feature branch, push it with `-u` to set the upstream, and open a PR. Small branches, merged daily — a branch that lives two weeks is a merge conflict waiting to happen.

After merge I delete the branch locally and remotely. `git branch -d` for merged branches, and I prune stale remote-tracking refs with `git fetch --prune`. Dead branches are noise."

```bash
git checkout -b fix/payment-timeout
git push -u origin fix/payment-timeout
git branch -d fix/payment-timeout && git push origin --delete fix/payment-timeout
```

**Key Point:** "Branch from fresh main, name with a prefix, merge small and often, delete and prune when done."

---

## Undoing Mistakes

### Q7: You committed to the wrong branch. How do you fix it?

**How to Answer:**

"Depends on whether I pushed. If it's still local, `git reset --soft HEAD~1` unstages the commit but keeps my changes, then I switch branches and commit there. The commit never happened, as far as history is concerned.

If I already pushed to the wrong branch, I don't rewrite history — I revert or just merge it forward and move on. Force-pushing a shared branch to 'fix' it is how you ruin someone's morning.

The interview trap here is reaching for `--hard` reflexively. `--soft` keeps changes in the index, `--mixed` keeps them in the working tree, `--hard` throws them away. I default to the least destructive option."

```bash
git reset --soft HEAD~1        # undo last commit, keep changes staged
git checkout correct-branch
git commit -m "feat: add retry logic"
```

**Key Point:** "Unpushed mistake: `reset --soft`, move branches, recommit. Pushed mistake: don't rewrite shared history."

---

### Q8: What's the difference between reset, revert, and restore?

**How to Answer:**

"`git reset` moves the branch pointer — it rewrites history. `--soft`, `--mixed`, `--hard` decide what happens to the changes. I only use it on commits nobody else has seen.

`git revert` creates a new commit that undoes an old one. History stays intact, so it's the safe choice for anything already pushed. 'Undo by adding, not by erasing.'

`git restore` is the newer, safer command for working-tree operations — restoring a file from the index or HEAD without touching history. I use it to discard unwanted edits in a file. Three tools, three different safety profiles."

**Key Point:** "Reset rewrites history (local only), revert undoes via a new commit (safe for pushed), restore fixes files without touching history."

---

