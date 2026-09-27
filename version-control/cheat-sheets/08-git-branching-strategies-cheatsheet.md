# Branching Strategies — Cheat Sheet

One-page dense reference. Tape it to the wall until it's muscle memory.

## core commands

```bash
git branch                       # list local branches
git branch -a                    # include remote-tracking branches
git checkout -b <name>           # create + switch to branch
git switch -c <name>             # modern equivalent
git branch -d <name>             # delete merged branch
git branch -D <name>             # force delete unmerged branch
git push -u origin <name>        # push and set upstream
```

## merging

```bash
git merge <branch>               # merge into current branch
git merge --no-ff <branch>       # force a merge commit (no fast-forward)
git merge --squash <branch>      # squash branch into working tree, then commit
git merge --abort                # bail out of a conflicted merge
git log --oneline --graph --all  # read the branch graph
```

## rebasing

```bash
git rebase main                  # replay my commits on top of main
git rebase --continue            # after fixing a rebase conflict
git rebase --abort               # bail out
git pull --rebase                # fetch + rebase instead of merge
# NEVER rebase branches others have pulled
```

## trunk-based

```bash
git checkout -b task-123 main    # short branch off main
# ... small change ...
git checkout main && git pull && git merge task-123
git branch -d task-123           # merge fast, delete immediately
# unfinished work -> feature flag, not a long branch
```

## gitflow

```bash
git checkout -b develop main            # long-lived integration branch
git checkout -b feature/x develop       # features off develop
git checkout -b release/2.4 develop     # stabilize: bug fixes only
git checkout main && git merge release/2.4 && git tag v2.4.0
git checkout develop && git merge release/2.4   # don't forget the merge-back
git checkout -b hotfix/x main           # production emergency
git checkout main && git merge hotfix/x && git tag v2.4.1
git checkout develop && git merge hotfix/x      # propagate the fix
```

## github flow

```bash
git checkout -b feature/x main   # everything off main
# ... open PR, review, CI passes ...
# merge to main (squash), deploy from main
git tag v3.1.0                   # tag releases from main
```

## release branches

```bash
git checkout -b release/2.4 main  # freeze a version
# fixes only on this branch; cherry-pick where needed
git tag -a v2.4.0 -m "Release 2.4.0"
git cherry-pick <sha>            # pull a single commit across branches
```

## conflicts

```bash
git status                       # shows conflicted files
# edit file, remove <<<<<<< markers, keep the right code
git add <file> && git commit     # resolves the merge
git mergetool                    # visual resolution
```

## picking a strategy

| Signal | Strategy |
|---|---|
| Deploy multiple times a day, strong CI | Trunk-based |
| Versioned releases (SDK, mobile, on-prem) | GitFlow / release branches |
| SaaS product, PR review, deploy from main | GitHub Flow |
| Regulated, heavyweight review gates | GitFlow or strict GitHub Flow |
| 5-person startup | Not GitFlow. Keep it simple. |
