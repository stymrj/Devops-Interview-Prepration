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

*Part 2 continues with umask, ACLs, inodes, links, and production permission debugging.*
