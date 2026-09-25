# Filesystem & Permissions — Hands-On Lab

*Pair this with the [Filesystem Hierarchy & Permissions interview guide](../02-filesystem-hierarchy-permissions.md). Do every exercise on a VM or container you can break — a disposable Ubuntu/Debian box is ideal. Run as a non-root user with sudo.*

---

## Exercise 1: Decode `ls -l` Like a Debugger

**Goal:** Read a permission string cold and know exactly what it means.

```bash
# Pick apart one line at a time
ls -l /etc/passwd /etc/shadow /tmp
stat -c '%A %a %U:%G %n' /etc/passwd /etc/shadow /tmp

# Now break it down by hand:
# -rw-r--r--  1 root root  ...  ->  file, rw- for owner, r-- for group, r-- for others
# -rw-r-----  1 root shadow ...  ->  group shadow can read, others get NOTHING (that's why it matters)
# drwxrwxrwt  ... /tmp           ->  directory, + sticky bit (t) — more on that in Exercise 4
```

**Observe:** `/etc/shadow` is `640` with group `shadow`. If it were `644`, any user on the box could read password hashes. That one permission bit is the entire reason hash-cracking attacks fail or succeed on a compromised host.

**Why this matters in interviews/ops:** "Why can't my app read its config?" starts here. If you can't decode `ls -l` in 5 seconds, you'll flail in every permissions debugging question.

---

## Exercise 2: chmod, chown, chgrp — Break It, Then Fix It

**Goal:** Change permissions and ownership three ways and feel the difference.

```bash
mkdir -p /tmp/permlab && cd /tmp/permlab
echo "secret" > app.conf

# Octal (absolute) — sets everything at once
chmod 600 app.conf && ls -l app.conf        # -rw------- : only YOU can read/write

# Symbolic (relative) — tweaks one slice
chmod g+r app.conf && ls -l app.conf        # -rw-r----- : group can now read
chmod o-rwx app.conf                        # strip others entirely

# Ownership
sudo chown root:root app.conf && ls -l app.conf
sudo chgrp adm app.conf && ls -l app.conf

# Recursive change (CAREFUL — never run this on / )
chmod -R 640 /tmp/permlab && ls -lR /tmp/permlab
```

**Observe:** Note how `stat -c '%a %U:%G'` gives you the compact truth: `640 root:adm`. Compare `chmod 640` vs `chmod g+r` — octal is deterministic, symbolic is surgical.

**Why this matters in interviews/ops:** Deploy scripts that `chmod -R 777` "to make it work" are how breaches happen. Interviewers will ask what permission a secrets file should have — the answer is `600` (or `640` with a dedicated group), and you should say it without hesitating.

---

## Exercise 3: umask — Predict Default Permissions

**Goal:** See how umask silently decides the permissions of every new file.

```bash
umask          # probably 0022 — write it down
touch f1 && mkdir d1 && stat -c '%a %n' f1 d1
# f1 = 644, d1 = 755. Why? 666 - 022 = 644 for files, 777 - 022 = 755 for dirs.

umask 0077
touch f2 && mkdir d2 && stat -c '%a %n' f2 d2
# f2 = 600, d2 = 700 — paranoid mode. Now restore:
umask 0022
```

**Observe:** umask is a *subtractive mask*, not a permission set. Files start from `666` (never executable by default — a security choice), dirs from `777`.

**Why this matters in interviews/ops:** "A cron job writes a log file that the monitoring agent can't read" — classic umask issue. Also comes up when CI artifacts or copied SSH keys (`~/.ssh` needs `700`/`600`) break with "bad permissions" errors.

---

## Exercise 4: SUID, SGID, Sticky Bit — The Special Three

**Goal:** See each special bit in action, not just in a table.

```bash
# SUID — runs with the FILE OWNER's privileges
ls -l /usr/bin/passwd
# -rwsr-xr-x  — the 's' where owner's x was. Normal users run it, but it executes AS ROOT,
# which is the only way a non-root user can edit /etc/shadow.

# SGID on a directory — new files inherit the directory's GROUP
mkdir /tmp/sgidlab && sudo chgrp adm /tmp/sgidlab
chmod 2775 /tmp/sgidlab && ls -ld /tmp/sgidlab
touch /tmp/sgidlab/testfile && ls -l /tmp/sgidlab/testfile
# testfile belongs to group 'adm' even though your primary group isn't adm.

# Sticky bit — only the file's OWNER can delete/rename inside a shared dir
ls -ld /tmp        # drwxrwxrwt — the 't'
# Without sticky on a world-writable dir, anyone could delete anyone else's files.

# Set them yourself
chmod u+s /tmp/permlab/app.conf && ls -l /tmp/permlab/app.conf   # 4640 — 's' appears
chmod g+s /tmp/permlab && chmod +t /tmp/permlab
stat -c '%a %A' /tmp/permlab/app.conf /tmp/permlab
chmod u-s,g-s,-t /tmp/permlab/app.conf /tmp/permlab   # clean up
```

**Observe:** The leading octal digit: `4` = SUID, `2` = SGID, `1` = sticky. Uppercase `S`/`T` means the bit is set but the execute bit isn't (often a misconfiguration).

**Why this matters in interviews/ops:** "Find all SUID binaries on this host" is a real security-audit task (see Exercise 7). A rogue SUID binary = instant privilege escalation. Interviewers love asking why `/tmp` has a `t`.

---

## Exercise 5: ACLs — When user:group:other Isn't Enough

**Goal:** Grant one extra user access without touching the main permission bits.

```bash
# Check your filesystem supports ACLs (most do)
tune2fs -l /dev/sda1 2>/dev/null | grep -i acl || mount | grep ' / '

touch /tmp/permlab/shared.log
chmod 640 /tmp/permlab/shared.log

# Give user 'nobody' (or a second user you create) read access — without opening it to everyone
sudo setfacl -m u:nobody:r /tmp/permlab/shared.log
getfacl /tmp/permlab/shared.log
# Note the '+' in ls -l output now: -rw-r-----+  <- that + means "there's an ACL here"

# Default ACLs — auto-applied to new files in a dir
sudo setfacl -d -m u:nobody:r /tmp/permlab
touch /tmp/permlab/newfile && getfacl /tmp/permlab/newfile

# Clean up
sudo setfacl -b /tmp/permlab/shared.log /tmp/permlab
```

**Observe:** `ls -l` shows `+` when an ACL exists — if you only look at the permission string and miss the `+`, you'll misdiagnose access problems.

**Why this matters in interviews/ops:** Shared app dirs (`/var/log/myapp` read by both the app user and a log-shipper agent) are the textbook ACL case. "The permissions look right but access is denied" → check for the `+`, then `getfacl`.

---

## Exercise 6: chattr +i — The Immutable Flag

**Goal:** Make a file undeletable even by root, then undo it.

```bash
echo "do not touch" | sudo tee /tmp/permlab/locked.conf
sudo chattr +i /tmp/permlab/locked.conf
lsattr /tmp/permlab/locked.conf        # ----i--------e-------

# Try to break it — even as root
sudo rm -f /tmp/permlab/locked.conf    # Operation not permitted
sudo sh -c 'echo x > /tmp/permlab/locked.conf'   # Operation not permitted

# Unlock
sudo chattr -i /tmp/permlab/locked.conf
sudo rm /tmp/permlab/locked.conf && echo "gone"
```

**Observe:** Root is not omnipotent — file *attributes* sit below the permission model. Malware uses `+i` to persist; defenders use it to protect critical configs.

**Why this matters in interviews/ops:** "I can't delete this file even as root" is a great interview trap. The answer ladder: immutable flag (`lsattr`) → read-only mount (`findmnt -o ro`) → NFS weirdness. Knowing `chattr` exists puts you ahead of most candidates.

---

## Exercise 7: Hunt Risky Permissions with `find`

**Goal:** Run the same audit commands a security review would.

```bash
# All SUID binaries (privilege-escalation surface)
find /usr/bin /usr/sbin -perm -4000 -ls 2>/dev/null | head

# World-writable files (anyone can modify)
find /tmp/permlab /etc -xdev -type f -perm -0002 -ls 2>/dev/null | head

# Files with NO permissions for anyone (often a mistake)
find /tmp/permlab -type f -perm 000 -ls 2>/dev/null

# Files not owned by anyone (orphaned UID — user was deleted)
find / -xdev -nouser -o -nogroup 2>/dev/null | head

# Recently permission-changed files (post-incident forensics)
find /etc -xdev -type f -cmin -60 2>/dev/null | head
```

**Observe:** `-perm -4000` means "has the SUID bit among others" (match *any*), `-perm 4000` would mean *exactly* that. The `-` vs exact distinction is a classic gotcha.

**Why this matters in interviews/ops:** "How would you audit a host for permission issues?" — this is the answer. `-xdev` keeps you from descending into other mounts (and hanging on network filesystems), which is the kind of detail that signals real experience.

---

## Exercise 8: Inode Exhaustion — Disk "Full" With Free Space

**Goal:** Reproduce the "df says there's space but writes fail" mystery.

```bash
df -i /tmp            # note IUse% — inodes are a SEPARATE budget from bytes
df -h /tmp            # byte budget

# Simulate exhaustion in a small sandbox (use a ramdisk so you don't hurt the host)
sudo mkdir -p /mnt/inodelab
sudo mount -t tmpfs -o size=50m,nr_inodes=1000 tmpfs /mnt/inodelab
i=0; while touch /mnt/inodelab/f$((i++)) 2>/dev/null; do :; done
echo "created $i files before failing"
df -i /mnt/inodelab   # IUse% = 100%
df -h /mnt/inodelab   # still has MBs free — but nothing can be created
touch /mnt/inodelab/one_more   # No space left on device  <- THE error

# Cleanup
sudo umount /mnt/inodelab
```

**Observe:** Every file costs one inode regardless of size. A million tiny files (runaway logs, exploded tarballs, container layers) kills the inode budget while gigabytes sit free.

**Why this matters in interviews/ops:** This is a famous production incident pattern. The debugging move is `df -i`, and the fix is finding and deleting the file-spam (`find /var -xdev -type f | wc -l` per directory, or the classic `/var/spool` mail queue explosion).

---

## Exercise 9: Deleted-But-Open Files — The `df` vs `du` Mystery

**Goal:** See why disk space doesn't come back after deleting a "huge" log.

```bash
# Create a big file and hold it open
dd if=/dev/zero of=/tmp/permlab/big.log bs=1M count=100 2>/dev/null
tail -f /tmp/permlab/big.log & TAILPID=$!

# Delete it while the process holds it open
rm /tmp/permlab/big.log
ls -lh /tmp/permlab/          # file is GONE from the directory...
df -h /tmp | tail -1          # ...but space is NOT freed

# Find the ghost
sudo lsof +L1 | grep -i deleted | head
# or: ls -l /proc/$TAILPID/fd | grep deleted

# Kill the holder — NOW the space returns
kill $TAILPID
df -h /tmp | tail -1          # freed
```

**Observe:** Deleting a filename removes the *directory entry*; the inode and blocks survive until the last file descriptor closes. `df` (filesystem view) and `du` (directory-tree view) disagree exactly by the size of these ghosts.

**Why this matters in interviews/ops:** "We deleted 50GB of logs but `df` still shows the disk full" — the fix is restarting the holding process or truncating instead of deleting (`: > app.log` / `truncate -s 0 app.log`). This question appears in nearly every SRE interview loop.

---

## Exercise 10: Trace the Full Path with `namei -l`

**Goal:** Find which directory in a path is blocking access.

```bash
# Set up a trap: file is 644 but a parent dir blocks traversal
mkdir -p /tmp/permlab/lockeddir/sub
echo data > /tmp/permlab/lockeddir/sub/data.txt
chmod 644 /tmp/permlab/lockeddir/sub/data.txt
chmod 700 /tmp/permlab/lockeddir     # owner-only — group/others can't even traverse

# The money command — shows EVERY component's permissions vertically
namei -l /tmp/permlab/lockeddir/sub/data.txt

# Now run as a different user to feel the failure
sudo -u nobody cat /tmp/permlab/lockeddir/sub/data.txt   # Permission denied
# The file itself is world-readable — the DIRECTORY blocks it. Classic.

# Bonus: same trick for debugging web servers and NFS
namei -l /var/www/html/index.html 2>/dev/null || echo "(path doesn't exist here — try it on a real web host)"
chmod 755 /tmp/permlab/lockeddir   # restore
```

**Observe:** File access requires **execute (`x`) on every parent directory**. A `644` file under a `700` directory is unreadable to everyone but the owner. `namei -l` lays the whole chain out — it's the fastest way to find the broken link.

**Why this matters in interviews/ops:** "Nginx returns 403, the file is 644, what's wrong?" — nine times out of ten it's a parent directory (`/home/deploy` at `700` is the classic). `namei -l` is the one-command answer, and interviewers light up when you reach for it.

---

## Cleanup

```bash
sudo rm -rf /tmp/permlab /tmp/sgidlab
sudo setfacl -b /tmp/permlab 2>/dev/null
echo "lab cleaned"
```

*If you could predict the outcome of every exercise before running it, the permission model is yours. If any surprised you — that's the one to re-read in the guide.*
