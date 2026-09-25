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

"Linux uses a single unified directory tree starting at `/` — the root. There's no concept of C: or D: drives like Windows. Every disk, partition, network share, or container layer gets *mounted* somewhere under that one tree, which is why a path like `/var/lib/docker` can physically live on a completely different disk than `/` and you'd never know from the path alone.

The layout follows the Filesystem Hierarchy Standard, and the directories I actually care about day to day are: `/etc` for configuration, `/var` for data that changes at runtime — logs in `/var/log`, application state in `/var/lib` — `/home` for users, `/opt` for manually installed third-party software, `/srv` for data served by the system, and `/tmp` for throwaway files. Then there are the two virtual filesystems, `/proc` and `/sys`, which aren't real files on disk at all — they're live windows into the kernel.

The reason this matters for DevOps is that every tool we use assumes this layout. Docker expects container data under `/var/lib/docker`, systemd reads unit files from `/etc/systemd` and `/usr/lib/systemd`, and log pipelines tail `/var/log`. When you understand the hierarchy, you stop hunting around the filesystem and go straight to the right place."

**Key Point:** "One tree, everything mounted under `/`. Learn where configs, logs, and state live by convention and you'll navigate any Linux box — or container — blindfolded."

---

### Q2: What's the difference between /proc and /sys? When do you reach for each?

**How to Answer:**

"Both are virtual filesystems — they don't exist on disk, the kernel generates their contents on the fly. But they serve different purposes. `/proc` is process-centric and general kernel info: `/proc/cpuinfo`, `/proc/meminfo`, `/proc/loadavg`, and one directory per running process like `/proc/1234/` containing its cmdline, environ, file descriptors in `fd/`, and memory maps. When I'm debugging a live system, `/proc` is usually my first stop.

`/sys`, or sysfs, is about the kernel's device model — hardware and drivers. Things like `/sys/class/net/eth0/` for network interfaces, `/sys/block/sda/` for disks, and power management knobs. You also *write* to `/sys` to change kernel behavior at runtime — like toggling an interface or adjusting a device parameter — whereas `/proc/sys` is the older path for tunables, now mostly symlinked thinking toward `sysctl`.

A concrete example: if I want to check whether the OOM killer took out my app, I look at `dmesg` or `/proc`. If I want to check disk queue stats or network interface counters, I look under `/sys/class/`. And in containers, remember that `/proc` inside a container is namespaced — you see only that container's processes — which is exactly why `top` inside a container can lie to you about total system memory unless you know what you're looking at."

**Key Point:** "`/proc` is processes and kernel info, `/sys` is devices and drivers. Both are live kernel interfaces, not files — reading them is querying the kernel."

---

## The Directories That Matter in Production

### Q3: Where do configs, logs, and application data live by convention — and why does this matter for containers?

**How to Answer:**

"The convention is: configs in `/etc`, logs in `/var/log`, and persistent application state in `/var/lib`. So PostgreSQL data lives in `/var/lib/postgresql`, Docker's own storage in `/var/lib/docker`, Nginx configs in `/etc/nginx`. `/opt` is for self-contained third-party software you install manually, and `/srv` for data the machine serves, like web roots or FTP data.

This matters for containers in two big ways. First, container images follow the same layout, so when something breaks inside a container you already know where to look — same muscle memory. Second, it drives volume decisions: you mount volumes at the stateful paths. A database container gets a volume at `/var/lib/postgresql/data` because that's where the data lives by convention; you don't mount the whole `/var`.

The classic mistake I see is people treating containers like pets and exec'ing in to edit `/etc/app/config.yaml` by hand. The convention exists so that config is predictable — in the container world that means baking config into the image or injecting it via environment variables and mounted ConfigMaps, not hand-editing files."

**Key Point:** "`/etc` for config, `/var/log` for logs, `/var/lib` for state. Containers honor the same layout — that's what makes volume mounts and debugging predictable."

---

### Q4: What is tmpfs, and why do /tmp and /run behave the way they do?

**How to Answer:**

"tmpfs is a filesystem that lives in RAM — or swap — instead of on disk. Files written there are fast, but they vanish on reboot and they consume memory, so you don't put anything important there. On modern systems, both `/tmp` and `/run` are typically tmpfs mounts.

`/tmp` is the traditional scratch space — any user can write there, and it's world-writable with the sticky bit set, which I'll get to. Files there may be cleaned on reboot, or even periodically by `systemd-tmpfiles`. `/run` is newer: it's for runtime state like PID files and sockets — `/run/nginx.pid`, `/run/docker.sock`. It replaced the old `/var/run`, which is now just a symlink to `/run`, and it's guaranteed to be empty at boot.

The DevOps relevance: I've seen outages where an app wrote gigabytes of temp files to `/tmp` on a small VM and effectively ate all RAM, because tmpfs counts against memory. If you see memory pressure with no obvious process using it, check `df -h /tmp`. And in containers, `/tmp` being an in-memory filesystem is why some apps that buffer large uploads there get OOM-killed — the fix is mounting an emptyDir with a size limit or writing to a real volume."

**Key Point:** "tmpfs is RAM-backed storage. `/tmp` and `/run` are fast and ephemeral — great for scratch and sockets, dangerous for anything large or important."

---

## Permission Bits and Ownership

### Q5: Decode `-rwxr-xr--` for me. What are permission bits, really?

**How to Answer:**

"That string is 10 characters. The first one is the file type — `-` for a regular file, `d` for directory, `l` for symlink. The remaining nine are three groups of three: owner, group, and others. Each triple is read, write, execute in that order.

So `-rwxr-xr--` means: it's a regular file, the owner can read, write, and execute it, the group can read and execute, and everyone else can only read. In octal that's 754 — each triple maps to a number: r=4, w=2, x=1, so rwx is 7, r-x is 5, r-- is 4.

The execute bit means different things for files versus directories, which trips people up. On a file, x means you can run it as a program. On a directory, x means you can traverse into it and access files inside — you can `cd` through it. That's why directories are almost always 755 or 775: without the execute bit on a directory, you can't reach anything inside it even if you know the exact path."

**Key Point:** "Nine bits, three triples: owner, group, others — each rwx. And remember: on directories, x means 'you may pass through', which is why stripping it breaks everything below."

---

### Q6: How do permissions behave differently on directories versus files?

**How to Answer:**

"This is one of the most misunderstood things in Linux, and it comes up in interviews constantly. On a file: r means you can read its contents, w means you can modify it, x means you can execute it. Simple. On a directory: r means you can *list* its contents with `ls`, w means you can *create, delete, or rename* entries inside it, and x means you can *traverse* it — cd into it and access files inside by name.

The tricky combinations are what interviewers love. A directory with `--x` but no `r`: you can't list it, but if you know a filename inside, you can open it directly. That's actually a useful pattern — like a dropbox. A directory with `r` but no `x`: you can see filenames with `ls` but can't open or stat any of them — `ls -l` shows question marks. And here's the big one: to delete a file, you need write permission on the *directory*, not on the file itself. I can delete a file I'm not allowed to read, as long as I can write to its parent directory.

In practice: web roots are typically 755 on directories and 644 on files. Upload directories where the app needs to write get owned by the app user. And when someone reports 'permission denied' on a deep path, I check every level of the path with `namei -l` because a missing x on any parent directory blocks everything below it."

**Key Point:** "Deleting a file needs write on the directory, not the file. And one missing x anywhere up the path blocks the whole path — `namei -l` is how you find it."

---

## Special Bits: SUID, SGID, Sticky

### Q7: What do the SUID, SGID, and sticky bits do? Give me real examples.

**How to Answer:**

"These are three extra bits beyond the standard nine, shown as a fourth octal digit in front — like 4755 or 1755.

SUID, that's 4, on an executable means it runs with the *owner's* privileges, not the caller's. The classic example is `/usr/bin/passwd` — it's owned by root with the SUID bit set, which is how a normal user can update `/etc/shadow`, a file they otherwise can't touch. In `ls -l` you see it as an `s` in the owner's execute position: `-rwsr-xr-x`.

SGID, that's 2, has two meanings. On an executable, the process runs with the *group's* privileges. On a directory — and this is the one I use constantly — new files created inside inherit the *directory's group* instead of the creator's primary group. So for a shared deploy directory owned by the `deployers` group with SGID set, every file anyone drops in stays group-accessible. You see it as an `s` in the group execute position.

The sticky bit, that's 1, on a directory means only the file's owner — or root — can delete or rename files inside, even though the directory is world-writable. That's exactly how `/tmp` works: everyone can create files there, but you can't delete someone else's. It shows as a `t` in the others execute position: `drwxrwxrwt`.

The security angle interviewers want: SUID binaries are a privilege-escalation surface, so auditors hunt for unexpected ones — a SUID bit on `vim` or `find` is basically a root shell waiting to happen."

**Key Point:** "SUID runs as the file's owner, SGID on directories inherits the group, sticky on directories protects everyone's files from each other. Audit SUID binaries — they're the classic privesc path."

---

### Q8: Why is `chmod 777` a red flag, and what should you use instead?

**How to Answer:**

"`chmod 777` gives everyone — every user on the system — read, write, and execute on that file or directory. It's the 'I don't understand permissions so I'll just open everything' move, and in an interview it signals you reach for the sledgehammer instead of diagnosing.

The problems are real: any compromised service or user can now modify or replace that file. If it's a web directory, anyone can drop in a malicious script. If it's a config, anyone can rewrite it. And it's lazy — it papers over the actual issue, which is almost always that the *wrong user* owns the file or the *group* is wrong.

What I do instead: first figure out which user actually needs access — usually the app runs as a dedicated service user. Then `chown` the files to that user, or put the humans who need access in a shared group and use `chgrp` plus 770 or 750. For the classic 'web server can't write to upload dir' case: `chown www-data:www-data uploads/` and `chmod 755` — or 775 with SGID if multiple deploy users need in. And if the requirement genuinely doesn't fit user/group/other, that's what ACLs are for — not 777.

If I ever see 777 in a production audit, I treat it as a finding, not a configuration."

**Key Point:** "777 means you stopped thinking. Fix ownership and groups instead — `chown` to the service user, or a shared group with SGID. Reach for ACLs before you ever reach for 777."

---
## umask and Default Permissions

### Q9: How does umask work? How do default file and directory permissions get decided?

**How to Answer:**

"When you create a file or directory, the system doesn't just pick permissions at random — it starts from a base and subtracts the umask. The base is 666 for files and 777 for directories. Directories get the execute bit by default because without it you couldn't traverse into them; files don't, because you don't want every new file to be executable.

So with the classic umask of 022: files become 666 minus 022 = 644, directories become 777 minus 022 = 755. With a stricter umask of 027, files are 640 and directories 750 — group can read, others get nothing. That's the standard for multi-user servers where you don't want every user's files world-readable.

A few things interviewers like to probe: umask is per-process and inherited — you set it in `/etc/profile`, `~/.bashrc`, or for services via the `UMask=` directive in a systemd unit. And here's the trap: if your app explicitly calls `chmod` or `open()` with a mode, umask doesn't matter — the app's mode wins, masked by umask. So 'I set umask 077 but my app still creates world-readable files' usually means the app sets its own permissions and you need to fix it in the app config, not the shell."

```bash
umask          # show current mask, e.g. 0022
umask 027      # set stricter default
touch f && mkdir d && ls -ld f d   # observe 640 / 750
# systemd service unit:
# [Service]
# User=appuser
# UMask=0027
```

**Key Point:** "Base 666 for files, 777 for dirs, minus umask. But an app that sets its own mode bypasses it — umask is a default, not a policy."

---

### Q10: What's the difference between chown and chgrp — and what breaks when a container writes files as root onto a host volume?

**How to Answer:**

"`chown` changes the owner user and optionally the group — `chown alice:devs file`. `chgrp` changes only the group — `chgrp devs file`. In practice `chown` with the `user:group` syntax covers both, and `-R` makes it recursive. One handy trick is `chown --reference=goodfile target` which copies ownership from another file — useful when you've fixed one file and want the rest to match.

The container angle is where this bites people. If your container runs as root — which is the Docker default — and writes to a bind-mounted host directory, those files land on the host owned by root. Then your host user, or the next container running as non-root, can't modify them. I've seen CI pipelines fail on cleanup for exactly this reason: the build container ran as root, wrote artifacts, and the next step running as the `jenkins` user couldn't delete them.

The fixes: run the container with `--user $(id -u):$(id -g)` so files are created with your UID, or design the image with a `USER` directive and a known UID. For named volumes, Docker initializes the volume with the image's directory ownership, which sidesteps it. And never 'fix' it with 777 on the host mount — you're one misconfiguration away from a host compromise."

```bash
chown -R appuser:appgroup /var/lib/myapp
chgrp -R devs /srv/shared && chmod -R g+s /srv/shared
docker run --user $(id -u):$(id -g) -v $PWD/data:/data myimage
```

**Key Point:** "Root in a container is root on the host's filesystem for bind mounts. Run containers as a non-root UID or the host inherits a permission mess."

---

## ACLs and Extended Attributes

### Q11: What are ACLs, and when do standard Unix permissions fall short?

**How to Answer:**

"Standard Unix permissions give you exactly one owner, one group, and everyone else. That breaks down the moment two different teams need different access to the same directory — say the `backend` team needs read-write on `/srv/releases` and the `qa` team needs read-only, plus the deploy service user needs full access. You can't express that with user/group/other.

That's what POSIX ACLs are for. `setfacl -m u:qa:r-x /srv/releases` gives the qa user read-execute without touching the owner or group. `setfacl -m g:backend:rwx` for the group. And default ACLs — `setfacl -d -m g:backend:rwx dir` — act like inheritance: new files created inside automatically get those ACL entries, which solves the 'new file has wrong group access' problem more flexibly than SGID.

You can spot ACLs because `ls -l` shows a `+` after the permission bits, and `getfacl` shows the full picture. The gotchas I always mention: `cp` without `-a`, `mv` across filesystems, and `tar` without `--acls` can silently drop ACLs — so your backup/restore pipeline needs to preserve them. And on NFS, both client and server need ACL support or they get quietly ignored, which is a nasty surprise."

```bash
setfacl -m u:qa:r-x /srv/releases
setfacl -m g:backend:rwx /srv/releases
setfacl -d -m g:backend:rwx /srv/releases   # default/inherited ACL
getfacl /srv/releases
ls -ld /srv/releases   # drwxr-x---+  <- the + means ACLs present
```

**Key Point:** "One user, one group isn't enough for shared directories. ACLs give per-user/per-group entries plus inheritance — but verify your backup tooling preserves them."

---

### Q12: What are file attributes (chattr/lsattr)? When would you make a file immutable?

**How to Answer:**

"Beyond the permission bits, ext4 and xfs support extended attributes you manage with `chattr` and view with `lsattr`. The two I actually use: `+i` for immutable and `+a` for append-only.

Immutable means the file cannot be modified, deleted, or renamed — not even by root — until you remove the flag with `chattr -i`. I use it on files that must never change unexpectedly: `/etc/resolv.conf` on boxes where DHCP keeps overwriting DNS, or a pinned `authorized_keys`. Append-only is perfect for logs: with `+a`, processes can append but nobody can truncate or overwrite — it gives you tamper-evident logs even if an attacker gets in.

The classic interview trap this answers: 'I'm root and I still get permission denied writing to a file.' Most people check `ls -l` and get confused. The answer is `lsattr` — if you see that little `i`, that's your culprit. Same story with 'I can't delete this file as root.' Attributes sit below permissions in the enforcement stack, and they survive reboots, so they're a legitimate hardening layer — but document them, because the next on-call engineer will be confused otherwise."

```bash
lsattr /etc/resolv.conf
chattr +i /etc/resolv.conf        # immutable
chattr +a /var/log/app/audit.log # append-only for tamper-evident logs
chattr -i /etc/resolv.conf        # remove when you need to change it
```

**Key Point:** "When root gets 'permission denied', check `lsattr` before anything else. `+i` and `+a` are enforcement below the permission bits — powerful, so document them."

---

## Links, Inodes, and Deleted-but-Open Files

### Q13: What are inodes? "Disk has free space but writes fail" — how do you debug it?

**How to Answer:**

"An inode is the filesystem's metadata record for a file — it holds permissions, ownership, timestamps, and pointers to the data blocks. The filename itself is not in the inode; it's just a directory entry pointing at an inode number. One filesystem has a fixed number of inodes, decided at format time.

So here's the failure mode: `df -h` shows gigabytes free, but writes fail with 'No space left on device.' That's inode exhaustion — millions of tiny files, like a runaway session directory, a mail queue, or an app cache that never cleans up. Each tiny file eats one inode regardless of size. The diagnosis is `df -i` — if IUse% is at 100%, that's it. Then `find` the culprit: something like hunting which directory holds the most files.

The fix is deleting the small files — and the prevention is monitoring `df -i` alongside `df -h`, because almost nobody alerts on inodes until the first outage. You can also format with more inodes via `mkfs -N`, but you can't change it on a live filesystem, so plan ahead for workloads you know create millions of files."

```bash
df -i /var              # check inode usage, not just space
df -h /var              # space looks fine...
# find directories with the most files:
find /var/spool -xdev -type f | cut -d/ -f1-4 | sort | uniq -c | sort -rn | head
```

**Key Point:** "`df -h` lies by omission — always check `df -i` too. Inode exhaustion from millions of tiny files is a classic 'but there's free space!' outage."

---

### Q14: What's the difference between hard links and soft links — and what breaks with each?

**How to Answer:**

"A hard link is just another directory entry pointing at the same inode — same file, two names. There's no 'original' versus 'link'; delete either name and the data survives until the link count hits zero. Constraints: hard links can't cross filesystems, and you generally can't hard-link directories. Where I use them: backup tools like rsnapshot use hard links for space-efficient incremental backups — unchanged files are just additional links to the same inode.

A soft link — symlink — is its own tiny file that stores a *path* to the target. It can cross filesystems, it can point at directories, and `ls -l` shows it with an arrow. But it breaks if the target moves or is deleted — you get a dangling symlink. That's also its power: versioned deploys. `/opt/myapp/current -> /opt/myapp/release-123` — to roll back, you just repoint the symlink. Atomic, instant.

The interview traps: `cp` follows symlinks by default and copies the *target's* content, which surprises people; `rm` on a symlink removes the link, not the target — but `rm -rf link/` with a trailing slash follows into the target directory, which has caused real disasters. And on modern distros, `/bin` is a symlink to `/usr/bin` — the usrmerge — which is why those 'duplicate' directories exist."

```bash
ln file.txt hardlink.txt        # same inode
ln -s /opt/myapp/release-123 /opt/myapp/current
ls -li                          # see inode numbers; hard links share one
readlink -f ./current           # resolve a symlink chain
find / -xtype l                 # find dangling symlinks
```

**Key Point:** "Hard links share an inode — no original, no copy. Symlinks store a path — flexible, but they dangle. Know which one `cp` and `rm` follow before you touch them."

---

## Permission Debugging in Production

### Q15: How do you audit a system for risky permissions?

**How to Answer:**

"I treat permission auditing as a checklist I can run on any box or bake into image builds. The `find` one-liners do the heavy lifting:

SUID binaries — `find / -perm -4000 -type f` — any surprise here is a privilege-escalation candidate. SGID — `-perm -2000`. World-writable files — `-perm -0002 -type f`. World-writable directories *without* the sticky bit — that's the dangerous combination, someone else's files deletable. SSH material with loose permissions — `~/.ssh` should be 700, keys 600, or sshd refuses them, which is its own debugging session.

In practice I run this against golden images in CI: build the image, run the audit, diff against a known-good baseline, fail the build on new SUID binaries or new world-writable paths. On live systems I scope it — full-filesystem finds are slow, so I target `/usr`, `/opt`, `/srv`, and the app directories rather than scanning all of `/proc` and `/sys`.

The mindset I communicate in interviews: permissions drift. Someone debugs at 2 AM with a chmod, forgets to revert it, and six months later it's an incident. Audits exist to catch the drift."

```bash
find / -xdev -perm -4000 -type f 2>/dev/null   # SUID binaries
find / -xdev -perm -2000 -type f 2>/dev/null   # SGID binaries
find / -xdev -perm -0002 -type d ! -perm -1000 2>/dev/null  # world-writable dirs WITHOUT sticky bit
ls -ld ~/.ssh ~/.ssh/*   # 700 on dir, 600 on keys
```

**Key Point:** "Permissions drift after every 2 AM debug session. Audit SUID/SGID and world-writable paths against a baseline — in CI for images, on a schedule for live boxes."

---

### Q16: Scenario — your app gets "Permission denied" writing to /var/log/myapp. Walk me through your triage.

**How to Answer:**

"I debug this layer by layer, outside-in, because the failure is rarely where you first look.

First: who is the app actually running as? `ps -o user= -C myapp`, or check the `User=` directive in its systemd unit. Half of these cases are 'I assumed it runs as myapp but it runs as nobody.'

Second: trace the full path with `namei -l /var/log/myapp`. This shows permissions at *every* level — a missing execute bit on `/var` or `/var/log` blocks everything below it, and this is the single most common root cause.

Third: check the target itself — `ls -ld /var/log/myapp`. Is the app user the owner? In the group? (`id appuser` to confirm group membership — and remember, group changes need a re-login or service restart to take effect.)

Fourth: extended checks. `getfacl` for ACLs that might be denying. `lsattr` for immutable flags. `findmnt` / `mount | grep` — is the filesystem mounted read-only? Is it full — `df -h` *and* `df -i`?

Fifth: the security layer. On RHEL-family systems, SELinux contexts — `ls -Z`, denials in the audit log. On Ubuntu, AppArmor profiles. I've seen perfectly correct Unix permissions blocked by SELinux more than once, and the fix is a context label or boolean, not a chmod.

And the fix is always the *minimal* correct one: chown to the service user, or a group + SGID, or an ACL entry. Never 777 — if I see that in a postmortem, that's a second finding."

```bash
ps -o user=,group= -C myapp
namei -l /var/log/myapp
ls -ld /var/log/myapp && getfacl /var/log/myapp && lsattr -d /var/log/myapp
findmnt -T /var/log/myapp
df -h /var/log && df -i /var/log
# SELinux (RHEL): 
getenforce; ausearch -m avc -ts recent | grep myapp
```

**Key Point:** "Triage outside-in: process user → every path level with `namei -l` → ownership → ACLs/attributes → mounts → SELinux. Fix with the minimal correct change, never 777."

---

*Continue practicing with the [hands-on lab](labs/02-filesystem-permissions-lab.md) and the [cheat sheet](cheat-sheets/02-filesystem-permissions-cheatsheet.md) for this topic.*
