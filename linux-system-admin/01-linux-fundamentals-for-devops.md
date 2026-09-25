# Linux Fundamentals for DevOps Interview Preparation Guide

*How to Answer Linux Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Linux Matters in DevOps](#why-linux-matters-in-devops)
2. [Filesystem Hierarchy](#filesystem-hierarchy)
3. [File Permissions & Ownership](#file-permissions--ownership)
4. [Process Management](#process-management)
5. [systemd & Services](#systemd--services)
6. [Package Management](#package-management)
7. [Users & Groups](#users--groups)
8. [SSH & Remote Access](#ssh--remote-access)
9. [Disk Usage & Monitoring](#disk-usage--monitoring)
10. [Environment Variables & Shell Basics](#environment-variables--shell-basics)
11. [Common Troubleshooting Scenarios](#common-troubleshooting-scenarios)

---

## Why Linux Matters in DevOps

### Q1: Why is Linux so important for DevOps roles?

**How to Answer:**

"Honestly? Everything in DevOps runs on Linux — containers, CI runners, K8s nodes, cloud VMs, all of it. When prod breaks at 2 AM there's no GUI, just you, SSH, and a terminal. If you're not comfortable on the command line, you can't do DevOps. It's that simple."

**Key Point:** "Linux is the OS of DevOps. Command-line fluency isn't optional."

---

## Filesystem Hierarchy

### Q2: Explain the Linux filesystem hierarchy. What lives in the important directories?

**How to Answer:**

"Single tree from `/` — no drive letters, everything mounts under one root. The three I live in: `/etc` for configs, `/var/log` for logs, `/proc` for live kernel truth. That covers 90% of production debugging. `/var/lib` holds app state like Docker data and databases, `/opt` is for manually installed software, `/tmp` is scratch that gets wiped. And `/proc` and `/sys` aren't real files — they're the kernel talking to you."

**Key Point:** "`/etc` for config, `/var/log` for logs, `/proc` for live kernel truth — those three cover 90% of production debugging."

---

### Q3: What is the difference between absolute and relative paths? What do `.` and `..` mean?

**How to Answer:**

"Absolute paths start at `/` and work from anywhere — that's what automation should always use. Relative paths depend on where you're standing. `.` is here, `..` is up one level. `./deploy.sh` runs the script in the current directory, which matters because `.` isn't in PATH — deliberately."

**Key Point:** "Absolute paths are unambiguous and preferred in automation; relative paths depend on where you stand."

---

## File Permissions & Ownership

### Q4: Explain Linux file permissions. What does `rwxr-xr--` mean?

**How to Answer:**

"Nine bits in three chunks — owner, group, others — each rwx. `rwxr-xr--` means the owner can do everything, the group can read and execute, everyone else just reads. Octal shorthand: r=4, w=2, x=1, so that's 754. My defaults: 755 for scripts and directories, 644 for files, 600 for secrets. One thing people miss: on a directory, x doesn't mean 'execute' — it means you can pass through. No x on a directory, nothing inside is reachable."

```bash
chmod 755 deploy.sh      # scripts and directories
chmod 644 config.yaml    # regular files
chmod 600 ~/.ssh/id_rsa  # secrets
chown -R appuser:appgroup /opt/myapp
```

**Key Point:** "755 for executables and directories, 644 for files, 600 for secrets. For directories, `x` means traversable."

---

### Q5: What are `setuid`, `setgid`, and the sticky bit?

**How to Answer:**

"Three extra bits beyond rwx. SUID on a binary: it runs as the file's *owner*, not you — that's how `/usr/bin/passwd` edits `/etc/shadow`. Shows as `rws`. SGID on a directory: new files inherit the *directory's* group — perfect for shared deploy folders. Sticky bit on a directory like `/tmp`: anyone can create files, but only the owner can delete them. Shows as `t`. Security angle: unexpected SUID binaries are a classic privilege-escalation path, so I hunt them during hardening."

```bash
find / -perm -4000 -type f 2>/dev/null   # audit SUID binaries
chmod 2775 /opt/shared-deploy            # SGID shared directory
```

**Key Point:** "SUID runs as the file owner, SGID inherits the group, sticky protects shared dirs. Audit SUID — it's the classic privesc path."

---

## Process Management

### Q6: How do you inspect and manage processes on Linux?

**How to Answer:**

"`ps aux` for a snapshot, `ps auxf` for the process tree, `top` or `htop` for live. To stop something: `kill PID` sends SIGTERM — a polite ask to shut down cleanly. `kill -9` is SIGKILL — the kernel ends it instantly with no cleanup, so I never lead with it in prod. `pgrep` and `pkill` match by name so I'm not hunting PIDs. And zombies? A zombie is a dead process whose parent never collected its exit status — you can't kill it, you fix the parent."

```bash
pgrep -f "myapp.jar"      # find by name, not PID
kill $(pgrep -f myapp)    # SIGTERM first, -9 only as last resort
ss -tlnp | grep 8080      # what's on this port?
```

**Key Point:** "SIGTERM first for clean shutdown, SIGKILL only as a last resort. Zombies are a parent-process problem, not something you can kill."

---

### Q7: What is the difference between a process and a thread? What about foreground vs background jobs?

**How to Answer:**

"A process gets its own memory; threads live inside a process and share its memory — cheaper, but one bad thread can take the whole thing down. In the shell: `&` backgrounds a job, Ctrl+Z suspends it, `bg` and `fg` move it around. But background jobs still die with your shell, and `nohup` only half-solves that. Anything that must survive gets systemd or a container. nohup is a hack, not a strategy."

**Key Point:** "Processes have isolated memory, threads share it. For persistent services use systemd, not nohup."

---

## systemd & Services

### Q8: What is systemd and how do you manage services with it?

**How to Answer:**

"systemd is PID 1 on every modern distro — it owns services, timers, mounts, and logs. Daily driver is `systemctl`. Don't mix up `start` (runs now) with `enable` (starts on boot) — for a production service you usually want both. Custom units live in `/etc/systemd/system/`, and after any edit you must run `daemon-reload` or nothing changes. The line I care about most in a unit file: `Restart=always` — that's what brings your app back after a crash."

```bash
systemctl enable --now myapp   # enable at boot AND start now
systemctl status myapp
journalctl -u myapp -f
```

**Key Point:** "`start` is now, `enable` is on boot. `daemon-reload` after edits, `Restart=always` for resilience."

---

### Q9: How is `journalctl` different from reading log files in `/var/log`?

**How to Answer:**

"Old-school services write text to `/var/log`; systemd services log to the journal — a binary, indexed store you query with `journalctl`. `journalctl -u nginx` gives me just that service, `--since` and `-b` slice by time, `-f` follows live. One gotcha: the journal is in-memory by default, so a reboot can wipe your evidence. Persist it or ship logs centrally."

**Key Point:** "journalctl is structured and queryable by unit and time; ensure journal persistence or central shipping in production."

---

## Package Management

### Q10: How do you install and manage software on Linux? Compare apt and yum/dnf.

**How to Answer:**

"Debian/Ubuntu: `apt` with `.deb` packages. RHEL family: `yum`/`dnf` with `.rpm`. The trap everyone falls into: `apt update` only refreshes the package *list* — `apt upgrade` is what actually installs new versions. And honestly the flags matter less than this: if you're installing something by hand twice, it belongs in Ansible, a Dockerfile, or user-data. Manual installs don't scale."

```bash
sudo apt update && sudo apt install -y nginx   # Debian/Ubuntu
sudo dnf install -y nginx                      # RHEL family
```

**Key Point:** "`apt update` refreshes the index, `apt upgrade` installs. Anything installed twice by hand belongs in automation."

---

## Users & Groups

### Q11: How do you manage users, and how does `sudo` actually work?

**How to Answer:**

"`useradd -m` creates the user with a home directory. `usermod -aG` appends groups — skip the `-a` and you *replace* all group memberships, which has locked people out of sudo before. Sudo rules live in `/etc/sudoers`, edited only via `visudo` because it validates syntax — one typo there and nobody gets root. For services I make dedicated users with no login shell, and scope sudo to exact commands, like letting `deploy` run only `systemctl restart myapp`. Least privilege, always."

```bash
sudo useradd -m -s /bin/bash deploy
sudo usermod -aG docker deploy   # -a appends, don't skip it
```

**Key Point:** "Always `visudo` for sudoers edits, always `-a` with `usermod -G`, and scope sudo to exact commands."

---

## SSH & Remote Access

### Q12: How does SSH key-based authentication work, and how do you harden SSH?

**How to Answer:**

"Key auth: the private key stays on your laptop, the public key goes in the server's `~/.ssh/authorized_keys`. The server challenges you to prove you hold the private key — it never crosses the wire. Permissions are make-or-break and SSH fails silently: `~/.ssh` must be 700, `authorized_keys` 600. Hardening in `sshd_config`: `PasswordAuthentication no`, `PermitRootLogin no`. That kills brute force outright. On cloud, I also lock port 22 to known IPs in the security group."

```bash
ssh-keygen -t ed25519 -C "satyam@laptop"
ssh-copy-id user@server
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

**Key Point:** "Keys over passwords always; 700 on .ssh, 600 on authorized_keys; disable password auth and root login."

---

## Disk Usage & Monitoring

### Q13: A server's disk is full. How do you diagnose and fix it?

**How to Answer:**

"`df -h` to find the full filesystem, then `du -sh /* | sort -rh | head` to find the hog — it's `/var/log` nine times out of ten. The sneaky one: a process holding a *deleted* file open. `du` won't show it, `df` still counts it. Find it with `lsof | grep deleted`, fix by restarting the process or truncating via `/proc/<pid>/fd/<fd>`. For active logs I truncate, never delete. Then the real fix: logrotate with `copytruncate` so it never happens again."

```bash
df -h; du -sh /var/log/* 2>/dev/null | sort -rh | head
lsof | grep -i deleted   # invisible disk hogs
```

**Key Point:** "Check `lsof | grep deleted` — deleted-but-open files are the classic invisible disk hog. Fix with logrotate, not manual deletes."

---

### Q14: How do you check memory and CPU pressure on a Linux box?

**How to Answer:**

"`free -h`, but read the `available` column — `used` includes disk cache the kernel will happily hand back, so high 'used' alone isn't a problem. For CPU I watch load average: sustained load above your core count means saturation. If the OOM killer fired, `dmesg` will say so — and that's a sizing or memory-leak problem, not a commands problem. In containers, check cgroup limits and the OOMKilled flag instead."

```bash
free -h               # trust 'available', not 'used'
dmesg | grep -i oom   # did the OOM killer fire?
```

**Key Point:** "Trust `available` memory, not `used`. Load average above core count means CPU saturation. OOM kills in dmesg mean a sizing problem."

---

## Environment Variables & Shell Basics

### Q15: How do environment variables work, and what's the difference between `.bashrc` and `.profile`?

**How to Answer:**

"Env vars are how config reaches your app without hardcoding — `export` makes a variable visible to child processes. That's the whole 12-factor config idea, and exactly how Docker `-e` and K8s env blocks work. `.profile` runs once at login, `.bashrc` runs for every interactive shell — env vars and PATH go in the former, aliases in the latter. Production gotcha: systemd services read *neither*. Their environment comes from `Environment=` in the unit file — I've watched people debug `.bashrc` on a server for an hour for nothing."

```bash
export NODE_ENV=production
env | grep NODE
```

**Key Point:** "export makes vars visible to children; 12-factor config rides on env vars. systemd ignores shell dotfiles — use Environment= in unit files."

---

## Common Troubleshooting Scenarios

### Q16: Walk me through debugging a service that won't start.

**How to Answer:**

"Fixed order, outside-in. `systemctl status` first — the last few log lines usually give it away. Then `journalctl -u myapp` for the full error. Then the usual suspects: config syntax (`nginx -t`), port already bound (`ss -tlnp`), permissions — `sudo -u appuser cat config.yaml` tells me if the service user can even read its own config — and dependencies. And always ask what changed recently: deploys, config edits, cert renewals. Most outages are a change, not a mystery."

**Key Point:** "Debug outside-in: status → logs → config → ports → permissions → dependencies. Most outages trace to a recent change."

---

### Q17: How do you find which process is listening on a port, and how do you check connectivity to a remote service?

**How to Answer:**

"`ss -tlnp` — the `-p` shows the owning process, though you need sudo to see other users' processes. For remote checks I layer it: `ping` for basic reachability (inconclusive if ICMP is blocked), `nc -zv host port` for 'is the port actually open', `curl -v` for the app layer — that shows the TLS handshake too, where I catch expired certs. If `nc` works but `curl` fails, the network's fine and the app is the problem."

```bash
ss -tlnp | grep 5432
nc -zv db.internal 5432
curl -v https://api.internal/health
```

**Key Point:** "`ss -tlnp` locally; `nc -zv` to isolate network vs app; `curl -v` to see TLS and HTTP behavior."

---

*Next: Filesystem Hierarchy & Permissions deep-dive → coming tomorrow.*
