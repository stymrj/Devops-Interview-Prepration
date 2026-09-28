# Git Internals Interview Preparation Guide

*How to Answer Git Internals Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.


## Table of Contents

1. [Git Objects Model](#git-objects-model)
2. [Content Addressable Storage](#content-addressable-storage)
3. [Blobs and Trees](#blobs-and-trees)
4. [Commits and Tags](#commits-and-tags)
5. [Refs Branches and HEAD](#refs-branches-and-head)
6. [The Object Database in Practice](#the-object-database-in-practice)

---

## Git Objects Model

### Q1: What does it mean that Git is "content-addressable"?

**How to Answer:**

"Everything Git stores is keyed by the hash of its own content, not by a name or a version number. Give Git the same content twice and you get the same SHA — it literally can't store the same bytes twice.

That's the opposite of something like SVN, where history is a sequence of revisions: revision 1, 2, 3. In Git there's no 'revision number' at all. Objects just exist, addressed by hash, and history is the links between them.

The practical payoff is integrity. If one byte in a file changes, its hash changes, so Git notices corruption instantly. That's why `git fsck` can verify a whole repo."

**Key Point:** "Git doesn't have revision numbers — objects are addressed by their content's hash, and history is just links between hashes."

---

### Q2: What are the four Git object types, and what does each one hold?

**How to Answer:**

"There are only four: blob, tree, commit, and tag. A blob holds file content — just the bytes, no name, no metadata. A tree is a directory listing: it maps names to blobs and sub-trees, plus file modes.

A commit points at one tree — the snapshot — and to its parent commits, plus the author, committer, and message. An annotated tag is a small object pointing at another object with a tagger, date, and message.

Notice what's missing: there's no 'diff' object anywhere. Git stores snapshots and derives diffs on demand. That surprises most people."

**Key Point:** "Four objects — blobs hold content, trees hold directory listings, commits hold snapshots plus parents, tags annotate objects."

---

## Content Addressable Storage

### Q3: How is a Git object's SHA actually computed? Why do identical files share one blob?

**How to Answer:**

"Git prepends a header — `blob <size>\0` — to the raw bytes, then SHA-1s (or SHA-256 in newer repos) that whole thing. The header means a blob and a commit with the same bytes still hash differently.

So when two files have identical content, the header plus bytes are identical, so the hash is identical, and Git stores a single blob. A thousand copies of the same dependency cost you one object's worth of storage.

I've seen interviewers ask 'does the filename affect the blob hash?' — no. The blob doesn't know the filename. Names live in trees, content lives in blobs. That's the separation that makes deduplication work."

```bash
# prove it: same content -> same blob hash, regardless of name
$ printf 'hello' | git hash-object --stdin
b6fc4c620b67d95f953a5c1cbfe6dab6eb31dea7
$ echo 'hello' > other-name.txt && git hash-object other-name.txt
b6fc4c620b67d95f953a5c1cbfe6dab6eb31dea7
```

**Key Point:** "SHA is computed over header plus content — identical bytes always produce one shared blob, no matter the filename."

---

### Q4: Why did Git pick content addressing? What breaks if you don't have it?

**How to Answer:**

"Two big wins: deduplication and integrity. Without content addressing you'd store every version of every file separately, and you'd need a separate mechanism to detect corruption. Git gets both from one design choice.

It also makes distributed workflows natural. If you and I both have object `abc123`, we know without asking that we have the exact same bytes. Syncing repos becomes 'send me the objects I don't have' — a pure set operation.

The tradeoff is that history is immutable. You can't edit a commit, only create a new one with a new hash. Rewriting history in Git is really writing *new* history, which is exactly why force-pushing is dangerous on shared branches."

**Key Point:** "Content addressing gives Git free deduplication, corruption detection, and cheap distributed sync — at the cost of immutable history."

---

## Blobs and Trees

### Q5: Walk me through what Git stores when I run `git commit` on one changed file.

**How to Answer:**

"Say you edit `app.js` and commit. Git hashes the new content into a blob. Then it builds a new tree for the directory: mostly pointers to unchanged blobs from the previous tree, plus your new blob for `app.js`. Unchanged files cost zero new storage — their blobs are reused.

Then it creates a commit object pointing at that new tree, with your parent commit's hash, your name, timestamps, and message. Finally it moves the branch ref — `refs/heads/main` — to the new commit's hash.

So a commit is cheap because it's mostly pointers. The tree diff between two commits is just 'which blobs got swapped.' Once you see it this way, `git log`, `git diff`, and `git checkout` all become obvious."

**Key Point:** "A commit is cheap: new blobs only for changed files, a tree of mostly reused pointers, and the branch ref moves to the new hash."

---

### Q6: How does Git store history if there are no diff objects?

**How to Answer:**

"It doesn't store diffs at all — it stores snapshots. Each commit points to a complete tree of the whole repo at that moment. When you ask for `git log -p` or `git diff`, Git computes the diff on the fly by comparing the two trees.

People coming from SVN expect delta storage, and Git does compress similar objects together in packfiles — but that's a storage optimization, not the model. The model is snapshots.

This is why checking out any commit is instant and why `git blame` walks parent pointers. There's no 'reconstruct version 47 from 46 deltas' step. The snapshot is just there."

**Key Point:** "Git stores snapshots, not diffs — diffs are computed on demand, and packfile compression is just a storage optimization."

---
## Commits and Tags

### Q7: What's actually inside a commit object? Which parts can you see with plumbing commands?

**How to Answer:**

"A commit holds the tree hash, parent commit hashes, author name/email/date, committer name/email/date, and the message. That's it — no branch name, no filename list. The branch is just a ref pointing at the commit.

I like to demo this with `git cat-file -p HEAD` — it prints the raw object and the mystery disappears. Interviewers love when you can show, not just tell.

The author vs committer split trips people up. They're usually the same person, but when you rebase or cherry-pick, Git keeps the original author and sets you as the committer. That distinction is how `git log` knows who originally wrote the change."

```bash
$ git cat-file -p HEAD
tree 9f86d081884c7d659a2feaa0c55ad015a3bf4f1b
parent a615b0bd956ffa1715f523a49751b9cdc8c42f89
author Satyam Raj <satyam@example.com> 1787954400 +0530
committer Satyam Raj <satyam@example.com> 1787954400 +0530

Add Git Internals interview guide (part 1)
```

**Key Point:** "A commit is tree + parents + author/committer + message — no branch name lives in it, and `git cat-file -p` shows you the raw truth."

---

### Q8: At the object level, what's the difference between an annotated tag and a lightweight tag?

**How to Answer:**

"A lightweight tag is not an object at all — it's just a ref, a file in `.git/refs/tags/` containing a commit hash. Zero metadata. An annotated tag is a real tag object with a tagger, date, message, and GPG signature capability, pointing at a commit.

That's why `git describe` and release workflows want annotated tags: they carry who cut the release and when. Lightweight tags are fine as personal bookmarks, but they tell CI nothing.

Quick tell in an interview: `git cat-file -t v1.2.0` prints `tag` for annotated and `commit` for lightweight. It's a one-command answer that shows you actually know."

**Key Point:** "Lightweight tags are just refs with no metadata; annotated tags are real objects with tagger, date, and message — use annotated for releases."

---

## Refs Branches and HEAD

### Q9: What is a ref, really? How is a branch different from a tag under the hood?

**How to Answer:**

"A ref is a pointer: a name mapped to an object hash, stored as a tiny file like `.git/refs/heads/main` containing one SHA. A branch is a ref that moves — every commit on that branch updates it. A tag is a ref that's meant to stay put.

That's genuinely all a branch is. There's no branch object, no branch metadata. `git branch feature` just writes a 40-char file. Deleting a branch with `-d` deletes the pointer, not the commits — which is why recovery is possible.

HEAD is the special ref that says 'where I am' — usually it points at a branch ref (`ref: refs/heads/main`), and that's what makes a branch 'checked out.' Follow that chain and the whole model clicks."

**Key Point:** "A branch is a movable pointer to a commit, a tag is a fixed pointer, HEAD is the pointer to your current pointer — it's pointers all the way down."

---

### Q10: What does "detached HEAD" mean, and when is it actually useful?

**How to Answer:**

"Detached HEAD means HEAD points directly at a commit hash instead of at a branch ref. You're on no branch — new commits you make here have no branch pointing at them, so they're easy to lose.

It happens when you check out a tag, a specific SHA, or a remote branch directly. Git warns you because commits made in this state are only reachable from the reflog.

But it's not always a mistake. CI systems build in detached HEAD all the time — they check out the exact commit they want and never commit anything. Bisecting also uses it: jump between commits, test, no branch needed. The rule is simple: detached HEAD is fine for reading history, dangerous for writing it."

**Key Point:** "Detached HEAD = HEAD points at a commit, not a branch. Safe for inspecting and bisecting, risky for committing — new work there has no branch to hold it."

---

## The Object Database in Practice

### Q11: Where do objects live on disk — loose objects vs packfiles?

**How to Answer:**

"Fresh objects start as loose files: `.git/objects/ab/cd...` where the first two hash chars are the directory. Each is zlib-compressed. Simple, but thousands of tiny files get slow, so Git periodically packs them.

Packfiles bundle many objects into one file with delta compression against similar objects, plus an index for random access. `git gc` triggers this, and clones/fetches transfer packfiles directly — that's why clone is fast.

So the snapshot model I described earlier stays true logically, but physically Git is smart: deltas exist, but only as a packing optimization the model never sees. If an interviewer says 'Git stores diffs,' this is the nuance that corrects them."

```bash
$ ls .git/objects/ab/          # loose objects: first 2 hash chars as dir
$ ls .git/objects/pack/        # pack-*.pack + pack-*.idx
$ git count-objects -v          # see loose vs packed counts
```

**Key Point:** "New objects are loose zlib files; `git gc` packs them with delta compression — the snapshot model stays true, packing is just physical optimization."

---

### Q12: You deleted a branch but remember the commit SHA. How do you get it back?

**How to Answer:**

"Deleting a branch only deleted the pointer — the commit objects are still in the database until `git gc` prunes them, which by default takes weeks. So recovery is usually trivial.

If I have the SHA, I just create a new branch pointing at it: `git branch recovery <sha>`. Done. If I've lost the SHA, `git reflog` shows where HEAD has been — every checkout, commit, and reset — and the hash is right there.

The deeper answer interviewers want: objects are only deleted when nothing references them AND they're older than the gc grace period. Branch deletion removes one reference, but the reflog keeps another for 90 days by default. That's your safety net."

```bash
$ git reflog                       # find the lost commit's SHA
$ git branch recovery a615b0bd      # re-point a branch at it
$ git fsck --lost-found             # last resort: find dangling commits
```

**Key Point:** "Deleting a branch deletes the pointer, not the objects — `git branch recovery <sha>` or the reflog brings it back, and gc won't touch it for weeks."

