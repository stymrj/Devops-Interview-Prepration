# Git Internals — Hands-On Lab

*Open up the hood: inspect real objects, refs, and packfiles in a scratch repo. Everything here is safe — you'll work in `/tmp/git-internals-lab`.*

**Setup:**

```bash
mkdir -p /tmp/git-internals-lab && cd /tmp/git-internals-lab
git init -q && git config user.name "Lab" && git config user.email "lab@example.com"
echo "hello git" > file.txt && git add file.txt && git commit -qm "first commit"
echo "second line" >> file.txt && git add file.txt && git commit -qm "second commit"
```

---

## Exercise 1 — Prove content addressing with `hash-object`

**Goal:** Show that identical content always produces the same blob hash, regardless of filename.

**Commands:**

```bash
printf 'hello git' | git hash-object --stdin
echo 'hello git' > totally-different-name.txt
git hash-object totally-different-name.txt
```

**Expected output:** Both commands print the exact same 40-char SHA.

**Why it matters:** This is the dedup interviewers ask about — a thousand identical files cost one blob. The filename never enters the hash.

---

## Exercise 2 — Read a raw commit object with `cat-file`

**Goal:** See the actual fields inside a commit object.

**Commands:**

```bash
git cat-file -p HEAD
git cat-file -t HEAD
```

**Expected output:** `tree <sha>`, `parent <sha>` (missing on the first commit), `author`/`committer` lines, and the message. `-t` prints `commit`.

**Why it matters:** "What's inside a commit?" stops being a memorized list once you've seen the raw object. Author vs committer is visible right here.

---

## Exercise 3 — Walk the tree: from commit to blob

**Goal:** Trace HEAD → tree → blob by hand, the way Git does.

**Commands:**

```bash
TREE=$(git cat-file -p HEAD | awk '/^tree/ {print $2}')
git cat-file -p $TREE
BLOB=$(git cat-file -p $TREE | awk '/file.txt/ {print $3}')
git cat-file -p $BLOB
```

**Expected output:** The tree listing shows `100644 blob <sha> file.txt`; the final `cat-file` prints the file's content.

**Why it matters:** This is the commit → tree → blob chain from the guide, done live. Once you can walk it, "how does checkout work" answers itself.

---

## Exercise 4 — Inspect a ref: branches are just files

**Goal:** Prove a branch is a 41-byte pointer, not an object.

**Commands:**

```bash
cat .git/refs/heads/main
cat .git/HEAD
git cat-file -t $(cat .git/refs/heads/main)
```

**Expected output:** A 40-char SHA in the branch file, `ref: refs/heads/main` in HEAD, and `commit` from `cat-file -t`.

**Why it matters:** "Deleting a branch doesn't delete commits" makes sense when you see the branch is literally a text file holding a hash.

---

## Exercise 5 — Detached HEAD on purpose

**Goal:** Enter detached HEAD safely and see exactly what changes.

**Commands:**

```bash
git checkout HEAD~1 -q
cat .git/HEAD
git checkout main -q
cat .git/HEAD
```

**Expected output:** In detached state HEAD contains a raw SHA; back on `main` it contains `ref: refs/heads/main` again.

**Why it matters:** Detached HEAD demystified in two commands. You'll never again wonder what the warning means — you can describe the exact file change.

---

## Exercise 6 — Annotated vs lightweight tag at the object level

**Goal:** Feel the difference between a tag object and a bare ref.

**Commands:**

```bash
git tag light-tag
git tag -a v1.0 -m "release 1.0"
git cat-file -t light-tag
git cat-file -t v1.0
git cat-file -p v1.0
```

**Expected output:** `light-tag` reports type `commit` (it's just a ref to the commit); `v1.0` reports type `tag`, and `-p` shows the tagger, date, and message.

**Why it matters:** This is the one-command interview answer: `git cat-file -t <tag>` tells you which kind it is. Release pipelines want the annotated kind.

---

## Exercise 7 — Watch packfiles appear with `gc`

**Goal:** See loose objects get packed and compare counts.

**Commands:**

```bash
git count-objects -v
# make some history to pack
for i in $(seq 1 50); do echo "change $i" >> file.txt && git commit -qam "commit $i"; done
git gc -q
git count-objects -v
ls .git/objects/pack/
```

**Expected output:** Before: dozens of loose objects (`in-pack: 0`). After: `count: 0`, `in-pack: 50+`, and a `pack-*.pack` + `pack-*.idx` in the pack dir.

**Why it matters:** You now have a lived answer for "loose objects vs packfiles" — and you know delta compression is a storage detail, not the data model.

---

## Exercise 8 — Recover a "deleted" branch

**Goal:** Delete a branch, then bring it back from the object database.

**Commands:**

```bash
git checkout -qb doomed && echo x >> file.txt && git commit -qam "doomed work"
SHA=$(git rev-parse HEAD)
git checkout -q main && git branch -D doomed
git branch recovered $SHA
git log --oneline recovered | head -3
```

**Expected output:** `recovered` points at the "doomed" commit history — nothing was lost.

**Why it matters:** The classic interview scenario. Deleting the pointer never deletes the objects; recovery is one command if you kept the SHA.

---

## Exercise 9 — Find a lost commit with the reflog

**Goal:** Recover work when you *didn't* save the SHA.

**Commands:**

```bash
git checkout -qb temp && echo y >> file.txt && git commit -qam "temp work"
git checkout -q main && git branch -D temp
git reflog | head -8
# copy the SHA of "temp work", then:
git branch found <paste-sha-here>
```

**Expected output:** The reflog lists the temp branch's commit with its SHA; the new `found` branch restores it.

**Why it matters:** Reflog is the 90-day safety net behind every "I deleted something" story. Knowing it's there — and its default expiry — is the complete answer.

---

## Exercise 10 — Verify integrity with `fsck`

**Goal:** Run Git's corruption checker and understand what it validates.

**Commands:**

```bash
git fsck --full
echo "---"
git fsck --lost-found | head -5
```

**Expected output:** `dangling commit`/`dangling blob` lines at most (normal leftovers, e.g. from rebases); no `missing` or `broken` objects in a healthy repo.

**Why it matters:** Content addressing isn't just theory — `fsck` re-hashes every object and compares. This is the integrity half of the interview answer.

---

**Cleanup:** `cd /tmp && rm -rf /tmp/git-internals-lab`

*Next: the [cheat sheet](../cheat-sheets/09-git-internals-cheatsheet.md) for quick revision before the interview.*
