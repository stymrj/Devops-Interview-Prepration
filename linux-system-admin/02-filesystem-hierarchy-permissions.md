# Filesystem Hierarchy & Permissions Interview Preparation Guide

*How to Answer Filesystem & Permissions Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Filesystem Hierarchy Deep Dive](#filesystem-hierarchy-deep-dive)
2. [The Directories That Matter in Production](#the-directories-that-matter-in-production)
3. [Permission Bits and Ownership](#permission-bits-and-ownership)
4. [Special Bits: SUID, SGID, Sticky](#special-bits-suid-sgid-sticky)
5. [umask and Default Permissions](#umask-and-default-permissions)
6. [ACLs and Extended Attributes](#acls-and-extended-attributes)
7. [Links, Inodes, and Deleted-but-Open Files](#links-inodes-and-deleted-but-open-files)
8. [Permission Debugging in Production](#permission-debugging-in-production)

---

## Filesystem Hierarchy Deep Dive

### Q1: Walk me through the Linux filesystem hierarchy. Why does it look the way it does?

**How to Answer:**

"Linux has one unified tree starting at `/` — no C: or D: drives. Every disk, partition, or network share gets *mounted* somewhere under it, so `/var/lib/docker` can physically live on a different disk than `/` and you'd never know from the path.

The conventions I actually use: `/etc` for config, `/var/log` for logs, `/var/lib` for app state, `/opt` for manually installed software. `/proc` and `/sys` aren't real files — they're live windows into the kernel.

This matters because every tool assumes the layout. Docker, systemd, your log pipeline — they all look in the same places. Know the conventions and you stop hunting around the filesystem."

**Key Point:** "One tree, everything mounted under `/`. Know where configs, logs, and state live by convention and you'll navigate any box or container blindfolded."

### Q2: What's the difference between /proc and /sys? When do you reach for each?

**How to Answer:**

"`/proc` and `/sys` are both virtual — the kernel generates them on the fly, nothing on disk. `/proc` is process-centric: cpuinfo, meminfo, and a directory per process with its cmdline and file descriptors. `/sys` is the device model: network interfaces, disks, driver knobs you can even write to. In a container, `/proc` is namespaced — which is why `top` inside a container lies about total memory."

**Key Point:** "`/proc` is processes and kernel info, `/sys` is devices and drivers. Both are live kernel queries, not files."

---

## The Directories That Matter in Production

### Q3: Where do configs, logs, and application data live by convention — and why does this matter for containers?

**How to Answer:**

"Configs in `/etc`, logs in `/var/log`, persistent state in `/var/lib` — that's the deal. Containers follow the same layout, so two things come free: your debugging muscle memory transfers straight in, and volume decisions get obvious — a database volume goes at `/var/lib/postgresql/data`, not all of `/var`.

One thing I never do: exec into a container and hand-edit `/etc/app/config.yaml`. Bake config into the image or inject it via env vars and ConfigMaps — that's the whole point of the convention being predictable."

**Key Point:** "`/etc` config, `/var/log` logs, `/var/lib` state. Containers follow the same layout — that's what makes volumes and debugging predictable."

### Q4: What is tmpfs, and why do /tmp and /run behave the way they do?

**How to Answer:**

"tmpfs lives in RAM, not on disk — fast, gone on reboot, and it eats memory. `/tmp` is world-writable scratch space (with the sticky bit), cleaned on reboot. `/run` holds runtime state like PID files and sockets — `/run/docker.sock` — and starts empty every boot.

The gotcha I've actually hit: an app dumping gigabytes into `/tmp` on a small VM ate all RAM, because tmpfs counts against memory. If you see memory pressure with no obvious process behind it, check `df -h /tmp`."

```bash
df -h /tmp   # tmpfs usage counts against RAM
```

**Key Point:** "tmpfs is RAM-backed storage. `/tmp` and `/run` are fast and ephemeral — great for scratch and sockets, dangerous for anything large or important."

---

## Permission Bits and Ownership

### Q5: Decode `-rwxr-xr--` for me. What are permission bits, really?

**How to Answer:**

"First character is the file type — `-` file, `d` directory, `l` symlink. Then three triples: owner, group, others, each read-write-execute. So this is a regular file: owner gets rwx, group gets r-x, others get r--. In octal, r=4, w=2, x=1 — that's 754.

The part people miss: on a directory, x means *traverse*. Without it you can't reach anything inside, even if you know the exact path. That's why directories are basically always 755 or 775."

**Key Point:** "Owner, group, others — three rwx triples. On directories, x means 'you may pass through' — strip it and everything below breaks."

### Q6: How do permissions behave differently on directories versus files?

**How to Answer:**

"On files it's simple: r reads, w modifies, x executes. On directories: r lists contents, w creates/deletes/renames entries, x lets you traverse through.

Two things interviewers love: a directory with `--x` but no `r` works like a dropbox — you can't list it, but you can open files if you know their names. And deleting a file needs write on the *directory*, not the file — I can delete a file I'm not allowed to read.

When 'permission denied' hits a deep path, I check every level — one missing x on any parent blocks the whole path."

```bash
namei -l /var/log/myapp   # permissions at every level of the path
```

**Key Point:** "Deleting a file needs write on its directory, not the file. One missing x up the path blocks everything — `namei -l` finds it."

---

## Special Bits: SUID, SGID, Sticky

### Q7: What do the SUID, SGID, and sticky bits do? Give me real examples.

**How to Answer:**

"Three extra bits, shown as a fourth octal digit. SUID (4): the binary runs as its *owner* — `/usr/bin/passwd` is root-owned with SUID, which is how normal users update `/etc/shadow`. SGID (2): on directories, new files inherit the *directory's group* — perfect for shared deploy dirs. Sticky (1): on directories, only the file's owner can delete their own files — that's how `/tmp` works.

The security angle: SUID binaries are a privilege-escalation surface. A SUID bit on `vim` or `find` is basically a root shell waiting to happen — auditors hunt for these."

```bash
ls -l /usr/bin/passwd   # -rwsr-xr-x  (s = SUID)
ls -ld /tmp             # drwxrwxrwt  (t = sticky)
```

**Key Point:** "SUID runs as the file's owner, SGID on directories inherits the group, sticky on directories protects everyone's files from each other. Audit SUID binaries — they're the classic privesc path."

### Q8: Why is `chmod 777` a red flag, and what should you use instead?

**How to Answer:**

"777 gives literally everyone full access — it's the 'I gave up debugging' move. Any compromised user or service can now rewrite your configs or drop in scripts. It also papers over the real problem, which is almost always wrong ownership or a wrong group.

What I actually do: `chown` the files to the service user, or put the humans in a shared group with 770/750. For the classic 'web server can't write to uploads' case — `chown www-data:www-data uploads/`, done. If the need genuinely doesn't fit user/group/other, that's what ACLs are for. Never 777."

**Key Point:** "777 means you stopped thinking. Fix ownership and groups instead — `chown` to the service user, or a shared group with SGID. Reach for ACLs before you ever reach for 777."

---

## umask and Default Permissions

### Q9: How does umask work? How do default file and directory permissions get decided?

**How to Answer:**

"New files start at 666, directories at 777, then umask subtracts. umask 022 gives you the classic 644 files and 755 dirs. umask 027 gives 640/750 — group can read, others get nothing, which is what I want on shared servers.

Two things worth knowing: umask is per-process and inherited, so services set it via systemd's `UMask=`. And the trap — if the app sets its own mode explicitly, umask is ignored entirely. 'I set umask 077 but my app still creates world-readable files' means fix the app config, not the shell."

```bash
umask 027
touch f && mkdir d && ls -ld f d   # 640 / 750
```

**Key Point:** "Base 666 for files, 777 for dirs, minus umask. But an app that sets its own mode bypasses it — umask is a default, not a policy."

### Q10: What's the difference between chown and chgrp — and what breaks when a container writes files as root onto a host volume?

**How to Answer:**

"`chown` changes the user (and optionally group), `chgrp` only the group. In practice `chown user:group` covers both.

The container trap: containers run as root by default, so anything written to a bind-mounted host directory lands owned by root. Then your CI user can't clean it up and the pipeline fails on the next step — I've seen exactly this. Fix it with `docker run --user $(id -u):$(id -g)` or a `USER` directive in the image. And never 'fix' it with 777 on the host mount."

```bash
docker run --user $(id -u):$(id -g) -v $PWD/data:/data myimage
```

**Key Point:** "Root in a container is root on the host's filesystem for bind mounts. Run containers as a non-root UID or the host inherits a permission mess."

---

## ACLs and Extended Attributes

### Q11: What are ACLs, and when do standard Unix permissions fall short?

**How to Answer:**

"user/group/other gives you exactly one owner and one group. The moment backend needs rw and qa needs read-only on the same directory, you're stuck — that's what ACLs solve. `setfacl -m u:qa:r-x` adds a per-user entry without touching owner or group, and `-d` sets defaults that new files inherit.

You'll spot them by the `+` in `ls -l`, and `getfacl` shows the full picture. One warning: `cp` and `tar` silently drop ACLs unless you pass `-a` / `--acls` — so make sure your backup pipeline preserves them."

```bash
setfacl -m u:qa:r-x /srv/releases
setfacl -d -m g:backend:rwx /srv/releases   # inherited by new files
getfacl /srv/releases
```

**Key Point:** "One user, one group isn't enough for shared directories. ACLs give per-user/per-group entries plus inheritance — but verify your backup tooling preserves them."

### Q12: What are file attributes (chattr/lsattr)? When would you make a file immutable?

**How to Answer:**

"Beyond permission bits, there are attributes: `+i` makes a file immutable — not even root can touch it until you remove the flag. `+a` is append-only, great for tamper-evident logs. I use `+i` on files that must never change unexpectedly, like `/etc/resolv.conf` on boxes where DHCP keeps overwriting DNS.

This answers the classic interview trap: 'I'm root and still get permission denied.' Most people stare at `ls -l`. The answer is `lsattr` — that little `i` is your culprit."

```bash
chattr +i /etc/resolv.conf        # immutable
chattr +a /var/log/app/audit.log # append-only
lsattr /etc/resolv.conf
```

**Key Point:** "When root gets 'permission denied', check `lsattr` before anything else. `+i` and `+a` are enforcement below the permission bits — powerful, so document them."

---

## Links, Inodes, and Deleted-but-Open Files

### Q13: What are inodes? "Disk has free space but writes fail" — how do you debug it?

**How to Answer:**

"Inodes hold a file's metadata — permissions, ownership, timestamps, block pointers. Filenames just point at inode numbers, and each filesystem has a fixed count of them.

So here's the failure: `df -h` shows gigabytes free, but writes fail with 'No space left on device.' That's inode exhaustion — millions of tiny files from a runaway session dir or mail queue. Debug it with `df -i`, then hunt the directory hoarding files. I alert on `df -i` exactly like I alert on `df -h` — nobody does until the first outage."

```bash
df -i /var   # inode usage, not just space
```

**Key Point:** "`df -h` lies by omission — always check `df -i` too. Inode exhaustion from millions of tiny files is a classic 'but there's free space!' outage."

### Q14: What's the difference between hard links and soft links — and what breaks with each?

**How to Answer:**

"A hard link is just another name for the same inode — no original, no copy. Delete either name and the data survives until the link count hits zero. They can't cross filesystems. Backup tools use them for space-efficient incrementals.

A symlink is a tiny file storing a *path* — it crosses filesystems, but dangles if the target moves. Deploys love them: `current -> release-123`, repoint to roll back, atomic and instant. One landmine: `rm -rf link/` with a trailing slash follows into the target directory. That's caused real disasters."

```bash
ln file.txt hardlink.txt
ln -s /opt/myapp/release-123 /opt/myapp/current
```

**Key Point:** "Hard links share an inode — no original, no copy. Symlinks store a path — flexible, but they dangle. Know which one `cp` and `rm` follow before you touch them."

---

## Permission Debugging in Production

### Q15: How do you audit a system for risky permissions?

**How to Answer:**

"`find` does the heavy lifting: `-perm -4000` for SUID binaries, `-2000` for SGID, `-0002` for world-writable. The dangerous combo is world-writable directories *without* the sticky bit. I run this against golden images in CI and diff against a baseline — new SUID binary or new world-writable path fails the build.

On live boxes I scope it to `/usr`, `/opt`, `/srv` and the app dirs — full-filesystem finds are slow. And the mindset: permissions drift after every 2 AM debug session. Audits exist to catch the drift."

```bash
find / -xdev -perm -4000 -type f 2>/dev/null   # SUID binaries
find / -xdev -perm -0002 -type d ! -perm -1000 2>/dev/null  # world-writable dirs WITHOUT sticky
```

**Key Point:** "Permissions drift after every 2 AM debug session. Audit SUID/SGID and world-writable paths against a baseline — in CI for images, on a schedule for live boxes."

### Q16: Scenario — your app gets "Permission denied" writing to /var/log/myapp. Walk me through your triage.

**How to Answer:**

"I go outside-in. First: who does the app actually run as? `ps -o user=` — half these cases are 'I assumed myapp, it's actually nobody.' Second: `namei -l` on the full path — a missing x on any parent is the most common root cause. Third: `ls -ld` the target — right owner? right group? (And group changes need a service restart to take effect.)

Then the extended checks: `getfacl`, `lsattr`, is the mount read-only, `df -h` *and* `df -i`. On RHEL, check SELinux denials too — I've seen perfect Unix permissions blocked by SELinux more than once. The fix is always minimal: chown, group + SGID, or an ACL. Never 777."

```bash
ps -o user= -C myapp
namei -l /var/log/myapp
ls -ld /var/log/myapp && getfacl /var/log/myapp && lsattr -d /var/log/myapp
```

**Key Point:** "Triage outside-in: process user → every path level with `namei -l` → ownership → ACLs/attributes → mounts → SELinux. Fix with the minimal correct change, never 777."

---

*Continue practicing with the [hands-on lab](labs/02-filesystem-permissions-lab.md) and the [cheat sheet](cheat-sheets/02-filesystem-permissions-cheatsheet.md) for this topic.*
