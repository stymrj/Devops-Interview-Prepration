# Git Fundamentals & Daily Workflow — Cheat Sheet

One-page dense reference. Tape it to the wall until it's muscle memory.

## setup

```bash
git config --global user.name "Name"      # identity for commits
git config --global user.email "a@b.com"
git config --global init.defaultBranch main
git init                                   # new local repo
git clone <url>                            # copy a remote repo
```

## daily status

```bash
git status -sb          # short status: branch + changed files
git diff                # unstaged changes (worktree vs index)
git diff --cached       # staged changes (index vs HEAD)
git log --oneline -10   # last 10 commits, compact
git log --oneline --graph --all   # everything, with branch graph
```

## staging & committing

```bash
git add <file>          # stage a file
git add -p              # stage interactively, hunk by hunk
git add .               # stage everything (careful)
git commit -m "feat: add retry logic"   # commit staged changes
git commit --amend      # fold into last commit (unpushed only)
git commit --amend --no-edit            # same, keep message
```

## branches

```bash
git branch                        # list local branches
git branch -a                     # include remote-tracking
git checkout -b feat/x            # create and switch
git switch -c feat/x              # modern equivalent
git merge feat/x                  # merge into current branch
git branch -d feat/x              # delete merged branch
git branch -D feat/x              # force delete unmerged
git push origin --delete feat/x   # delete on remote
```

## undo (pick by safety)

```bash
git restore <file>        # discard worktree edits to a file
git restore --staged <f>  # unstage a file, keep edits
git reset --soft HEAD~1   # undo commit, keep staged
git reset --mixed HEAD~1  # undo commit + unstage (default)
git reset --hard HEAD~1   # throw everything away (danger)
git revert <sha>          # undo via new commit (safe for pushed)
```

## stash

```bash
git stash push -m "wip: x"   # park changes with a label
git stash list               # what's parked
git stash pop                # restore + drop latest stash
git stash apply              # restore, keep stash
git stash drop               # delete latest stash
```

## remotes & sync

```bash
git remote -v                      # show remotes
git remote add origin <url>        # connect a remote
git fetch --prune                  # download, clean stale refs
git pull --ff-only                 # sync only if clean fast-forward
git push -u origin feat/x          # push + set upstream
git push                           # after upstream is set
git push --force-with-lease        # safe force-push
```

## conflicts

```bash
git status                         # lists conflicted files
# edit file: pick winning lines, delete <<<<<<< ======= >>>>>>> markers
git add <file>                     # mark resolved
git commit                         # finish the merge
git merge --abort                  # bail out of the merge entirely
```

## history cleanup (own branch only)

```bash
git rebase -i HEAD~3     # squash/reword/reorder last 3
# in editor: pick | reword | squash | fixup | drop
git rebase --continue    # after resolving rebase conflicts
git rebase --abort       # bail out of the rebase
```

## inspect

```bash
git show <sha>            # one commit: message + diff
git blame <file>          # who changed each line (and when)
git log -- <file>         # history of one file
git log -S "retry"        # commits that added/removed "retry"
git reflog                # every HEAD move — your safety net
git tag v1.0.0 && git push --tags   # mark a release
```
