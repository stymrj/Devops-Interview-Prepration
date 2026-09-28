# Advanced Git — Cheat Sheet

*One page. Revise in 5 minutes before the interview.*

## Rebase

```bash
git rebase main                  # replay my commits onto main (rewrites SHAs)
git rebase -i HEAD~3             # interactive: pick/squash/fixup/reword/drop
git rebase --continue            # after resolving conflicts
git rebase --skip                # drop current commit, move on
git rebase --abort               # bail out, restore original branch
git pull --rebase                # fetch + rebase instead of merge
```

- Rebase = linear history, new hashes. Merge = preserves real history.
- Rule: rebase private branches, merge shared branches.
- Conflicts arrive **per commit** — enable `git config --global rerere.enabled true` to auto-reuse resolutions.

## Cherry-pick

```bash
git cherry-pick <sha>            # copy one commit here (new hash)
git cherry-pick A..B             # everything after A up to B
git cherry-pick -m 1 <merge-sha> # pick a merge commit (first parent)
git cherry-pick --continue / --abort / --skip
git log A..B --oneline           # VERIFY the range before picking
```

- Keeps original author, sets you as committer.
- Use for hotfixes across release branches — not as a daily workflow (duplicate history).

## Bisect

```bash
git bisect start
git bisect bad HEAD               # current commit is broken
git bisect good v2.4.0            # last known good
git bisect run ./reproduce.sh     # 0 = good, non-zero = bad
git bisect reset                  # ALWAYS run this after
```

- Binary search: ~8 runs for 200 commits.
- Script must be reliable — flaky test = wrong answer.
- Beats `blame` when there's no suspect: tests behavior, not authorship.

## Reflog — the undo button

```bash
git reflog                        # every HEAD move, 90 days, local only
git reset --hard HEAD@{1}         # undo last move (bad rebase/reset)
git checkout -b rescue HEAD@{3}   # branch from a past position
git branch recovery <sha>         # re-point at a deleted branch's tip
```

- Saves **committed** work only — `reset --hard` kills uncommitted changes forever.
- Local, expires, never pushed: a safety net, not a backup.

## Submodules

```bash
git submodule add <url> <path>        # nest a repo, pinned to a commit
git clone --recurse-submodules <url>  # clone WITH submodules
git submodule update --init          # fetch after a plain clone
git submodule update --remote --merge # pull latest on tracked branch
```

- Parent stores a **gitlink** (`160000 commit <sha>`), not the code.
- Update dance: commit in submodule → commit the new pointer in parent → push submodule FIRST.
- Detached HEAD inside submodules is normal (pinned to a commit, not a branch).

## One-line interview answers

- **"Rebase or merge?"** → Rebase private branches for clean history; merge shared branches to preserve what happened.
- **"When is cherry-pick right?"** → Porting one hotfix to a release branch — sparingly, to avoid duplicate commits.
- **"Bisect vs blame?"** → Blame finds who touched the line; bisect finds the commit where behavior changed.
- **"You deleted a branch — now what?"** → Reflog has the SHA for 90 days: `git checkout -b rescue HEAD@{n}`.
- **"What's in the parent repo for a submodule?"** → A gitlink: just the pinned commit hash, mode `160000`.
- **"Biggest submodule mistake?"** → Pushing the parent before pushing the submodule — CI then points at a commit that doesn't exist.
