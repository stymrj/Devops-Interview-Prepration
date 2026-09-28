# Git Internals — Cheat Sheet

*One page. Revise in 5 minutes before the interview.*

## The four objects

| Object | Stores | Created by |
|--------|--------|-----------|
| `blob` | File content only (no name, no mode) | hashing file bytes |
| `tree` | Directory listing: names → blobs/sub-trees + modes | `git write-tree` / commit |
| `commit` | Tree hash + parent(s) + author/committer + message | `git commit` |
| `tag` | Pointer + tagger + date + message (+ optional GPG sig) | `git tag -a` |

## Content addressing

- Object ID = SHA-1 (or SHA-256) of `<type> <size>\0` + raw bytes.
- Same bytes → same hash → one stored object, whatever the filename.
- No revision numbers. History = parent links between commits.
- Corruption = hash mismatch → `git fsck` catches it.

## What a commit contains

```
tree <sha>            # the full snapshot
parent <sha>          # zero or more (merge = many)
author Name <e> <ts>  # who wrote it
committer Name <e> <> # who put it in (differs after rebase/cherry-pick)
<blank>
message
```

## Refs — it's pointers all the way down

- Branch = file `refs/heads/<name>` holding one SHA. Moves on commit.
- Tag (lightweight) = file `refs/tags/<name>` holding one SHA. Stays put.
- Annotated tag = real `tag` object + a ref pointing at it.
- `HEAD` = `ref: refs/heads/main` normally; raw SHA when detached.
- Deleting a branch deletes the pointer, not the commits.

## Plumbing commands (the interview flexes)

```bash
git cat-file -p <sha>      # pretty-print any object
git cat-file -t <sha>      # object type: blob|tree|commit|tag
git hash-object <file>     # hash without storing
git hash-object -w <file>  # hash AND store as blob
git ls-tree HEAD           # list tree entries
git rev-list --objects --all | head   # every reachable object
git count-objects -v       # loose vs packed stats
git verify-pack -v .git/objects/pack/*.idx | sort -k3 -n | tail  # biggest objects
```

## Storage: loose vs packed

- New objects → loose files: `.git/objects/ab/cdef...` (zlib-compressed).
- `git gc` → packfiles: `.git/objects/pack/pack-*.pack` (+ `.idx`).
- Packs use delta compression against similar objects — storage optimization only; the model is still snapshots, not diffs.
- Clone/fetch transfer packfiles directly (that's why they're fast).

## Recovery toolkit

```bash
git reflog                          # where HEAD has been (90-day safety net)
git branch recovery <sha>           # re-point a branch at a lost commit
git fsck --lost-found               # find dangling commits/blobs
git fsck --full                     # verify every object's hash
```

## One-line interview answers

- **"Does Git store diffs?"** → No. Snapshots in trees; diffs computed on demand. Deltas exist only inside packfiles as compression.
- **"Annotated vs lightweight tag?"** → `git cat-file -t v1.0`: prints `tag` = annotated (real object), `commit` = lightweight (bare ref).
- **"Author vs committer?"** → Same person normally; rebase/cherry-pick keeps the original author, sets you as committer.
- **"Detached HEAD?"** → HEAD holds a raw SHA instead of `ref: refs/heads/...`. Fine for reading/bisecting, don't commit there.
- **"Why is `git log` fast on huge repos?"** → Commits are linked by hash; walking parents needs no reconstruction, and packs keep it on disk-cheap.
