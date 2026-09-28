# Git Internals Interview Preparation Guide

*How to Answer Git Internals Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

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
