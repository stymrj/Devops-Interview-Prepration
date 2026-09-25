# Linux Fundamentals for DevOps Interview Preparation Guide

*How to Answer Linux Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. Why Linux Matters in DevOps
2. Filesystem Hierarchy
3. File Permissions & Ownership
4. Process Management
5. systemd & Services
6. Package Management
7. Users & Groups
8. SSH & Remote Access
9. Disk Usage & Monitoring
10. Environment Variables & Shell Basics
11. Common Troubleshooting Scenarios

---

## Why Linux Matters in DevOps

### Q1: Why is Linux so important for DevOps roles?

**How to Answer:**

"Almost everything in DevOps runs on Linux. Your CI runners, Docker containers, Kubernetes nodes, cloud VMs — they're all Linux under the hood. When something breaks in production at 2 AM, you're not clicking through a GUI, you're SSH'd into a box running commands.

Linux matters because it's scriptable, stable, and transparent. Everything is a file, every action leaves a log, and you can automate literally anything. As a DevOps engineer, your job is to make infrastructure repeatable and observable, and Linux gives you the primitives for both — the shell, the filesystem, processes, networking — all controllable from the command line.

I'd say if you're not comfortable on the Linux command line, you can't really do DevOps. It's the foundation everything else sits on."

**Key Point:** "Linux is the operating system of DevOps — containers, CI/CD, and cloud all assume it. Command-line fluency isn't optional."

---

## Filesystem Hierarchy

### Q2: Explain the Linux filesystem hierarchy. What lives in the important directories?

**How to Answer:**

"Linux has a single unified directory tree starting at root, `/`. Unlike Windows there are no drive letters — everything mounts under `/`. The key directories I always keep in mind are:

- `/etc` — system configuration files. Things like `/etc/nginx/nginx.conf`, `/etc/ssh/sshd_config`, `/etc/fstab`. If you're debugging a service, you usually start here.
- `/var` — variable data that changes during runtime. `/var/log` is the big one — application and system logs live here. `/var/lib` holds state like Docker's data and databases.
- `/home` — user home directories.
- `/opt` — third-party software you install manually.
- `/tmp` — temporary files, wiped on reboot on most systems.
- `/proc` and `/sys` — these aren't real files on disk, they're virtual filesystems exposing kernel and process info. `/proc/cpuinfo`, `/proc/meminfo`, and per-process dirs like `/proc/1234/` are incredibly useful for debugging.
- `/usr` — user binaries and libraries, `/usr/bin`, `/usr/lib`.
- `/bin` and `/sbin` — essential binaries; on modern systems they're usually symlinks into `/usr`.

In a DevOps context, I mostly live in `/etc` for configs, `/var/log` for logs, and `/proc` for live system introspection."

**Key Point:** "`/etc` for config, `/var/log` for logs, `/proc` for live kernel truth — those three cover 90% of production debugging."

---

### Q3: What is the difference between absolute and relative paths? What do `.` and `..` mean?

**How to Answer:**

"An absolute path starts from root — like `/etc/nginx/nginx.conf` — and works no matter where you currently are. A relative path is resolved from your current directory, like `../logs/app.log`.

`.` means the current directory and `..` means the parent directory. So `cd ..` goes up one level. You'll see them in scripts a lot — for example `./deploy.sh` explicitly runs a script in the current directory, which matters because the current directory is usually not in your PATH for security reasons."

**Key Point:** "Absolute paths are unambiguous and preferred in automation; relative paths depend on where you stand."

---

## File Permissions & Ownership

### Q4: Explain Linux file permissions. What does `rwxr-xr--` mean?

**How to Answer:**

"Every file has three permission classes — user (owner), group, and others — and three permission types — read, write, execute. When you see `rwxr-xr--`, you read it in three chunks: `rwx` for the owner, `r-x` for the group, `r--` for others.

So the owner can read, write, and execute. Group members can read and execute but not write. Everyone else can only read.

You set these with `chmod`, either symbolically or numerically. Numeric is octal — read is 4, write is 2, execute is 1. So `chmod 755 script.sh` gives `rwxr-xr-x`, and `chmod 644 config.yaml` gives `rw-r--r--`. 755 is the standard for executables and directories, 644 for regular files.

One thing interviewers love to ask: why is a directory's execute bit important? For a directory, `x` doesn't mean 'run it' — it means 'you can traverse into it and access files inside'. A directory with `r` but no `x` lets you list names but not open anything inside."

For example:

```bash
# Make a deploy script executable by everyone, writable only by owner
chmod 755 deploy.sh

# Lock down a private key — only owner can read
chmod 600 ~/.ssh/id_rsa

# Change ownership
chown appuser:appgroup /opt/myapp -R
```

**Key Point:** "755 for executables and directories, 644 for files, 600 for secrets. For directories, `x` means traversable."

---

### Q5: What are `setuid`, `setgid`, and the sticky bit?

**How to Answer:**

"These are special permission bits beyond the standard rwx.

`setuid` on an executable means it runs with the privileges of the file's owner, not the person running it. The classic example is `/usr/bin/passwd` — it needs to write to `/etc/shadow` which only root can touch, so it's setuid root. You spot it as an `s` in the owner's execute slot: `rwsr-xr-x`.

`setgid` on a directory means new files created inside inherit the directory's group, not the creator's primary group. This is super useful for shared deploy directories where multiple team members or a CI user need consistent group ownership.

The sticky bit on a directory — like `/tmp` — means only the file's owner can delete or rename files inside it, even though everyone can write to the directory. You see it as a `t`: `drwxrwxrwt`.

From a security perspective, unexpected setuid binaries are a red flag — I always check for them during hardening with `find / -perm -4000`."

```bash
# Find all setuid binaries (security audit)
find / -perm -4000 -type f 2>/dev/null

# Set setgid on a shared directory
chmod 2775 /opt/shared-deploy
```

**Key Point:** "setuid runs as the file owner, setgid inherits group on directories, sticky bit protects shared dirs like /tmp."

---

## Process Management

### Q6: How do you inspect and manage processes on Linux?

**How to Answer:**

"My go-to is `ps aux` for a snapshot — it shows user, PID, CPU, memory, and the command. For a live view I use `top`, or better `htop` if it's installed. When I need the full picture including the process tree, `ps auxf` shows parent-child relationships, which is great for finding out what spawned a runaway process.

To kill a process, I start gentle: `kill <PID>` sends SIGTERM, which asks the process to shut down cleanly — flush buffers, close connections. If it ignores that, `kill -9 <PID>` sends SIGKILL which the kernel enforces immediately, no cleanup. I never start with -9 in production because it can leave corrupted state, half-written files, or dangling locks.

`pgrep` and `pkill` let me match by name instead of hunting PIDs:

```bash
# Find the PID of the app
pgrep -f "myapp.jar"

# Graceful, then forceful
kill $(pgrep -f "myapp.jar")
kill -9 $(pgrep -f "myapp.jar")

# What's listening on port 8080?
ss -tlnp | grep 8080
```

One more thing — zombie processes. A zombie is a process that finished but its parent never read its exit status, so it lingers in the process table. You can't kill a zombie; you have to fix or kill the parent."

**Key Point:** "SIGTERM first for clean shutdown, SIGKILL only as a last resort. Zombies are a parent-process problem, not something you can kill."

---

### Q7: What is the difference between a process and a thread? What about foreground vs background jobs?

**How to Answer:**

"A process is an independent execution unit with its own memory space. Threads are lightweight execution units inside a process that share its memory. So if one thread crashes a process badly, it can take the whole process down, but threads are cheaper to create and great for concurrent work — like a web server handling many connections.

In the shell, a foreground job occupies your terminal until it finishes. You can suspend it with `Ctrl+Z`, then resume it in the background with `bg`, or bring it back with `fg`. Starting a command with `&` runs it in the background immediately.

But background jobs still die when your shell exits — unless you detach them. `nohup` makes a process ignore the hangup signal, and `disown` removes it from the shell's job table. In practice though, for anything that needs to survive, I use systemd or run it in a container — nohup is a hack, not a strategy."

```bash
# Run in background, survive shell exit, log output
nohup ./app --port 8080 > app.log 2>&1 &
```

**Key Point:** "Processes have isolated memory, threads share it. For persistent services use systemd, not nohup."

---

## systemd & Services

### Q8: What is systemd and how do you manage services with it?

**How to Answer:**

"systemd is the init system on basically all modern Linux distros — it's PID 1, the first process the kernel starts, and it manages everything else: services, mount points, timers, logging.

Service management goes through `systemctl`. The commands I use daily:

```bash
# Start, stop, restart, reload
sudo systemctl start nginx
sudo systemctl restart myapp
sudo systemctl reload nginx      # reload config without dropping connections

# Enable at boot vs start now — different things!
sudo systemctl enable myapp      # start on boot
sudo systemctl status myapp      # is it running? recent logs?

# View logs for a service
journalctl -u myapp -f           # follow live
journalctl -u myapp --since "1 hour ago"
```

A really common interview trap: `enable` vs `start`. `start` runs it now, `enable` makes it start on boot. You usually want both for a production service.

Unit files live in `/etc/systemd/system/` for custom services. Here's a minimal one for an app:

```ini
[Unit]
Description=My Application
After=network.target

[Service]
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --port 8080
Restart=always
RestartSec=5
Environment=ENV=production

[Install]
WantedBy=multi-user.target
```

The `Restart=always` is the key line for reliability — systemd will bring the app back if it crashes. After writing or editing a unit file, you must run `systemctl daemon-reload`."

**Key Point:** "`start` is now, `enable` is on boot. Always `daemon-reload` after editing unit files, and use `Restart=always` for resilience."

---

### Q9: How is `journalctl` different from reading log files in `/var/log`?

**How to Answer:**

"Traditional services write plain text logs to `/var/log` — like `/var/log/nginx/access.log`. systemd services log to the journal instead, which is a binary, indexed log store you query with `journalctl`.

The journal's advantages: it's structured — every entry is tagged with the unit name, PID, and timestamp — so `journalctl -u nginx` gives you exactly that service's logs without grepping. It's also time-aware: `--since`, `--until`, `-b` for current boot only. And `-f` follows live like `tail -f`.

One gotcha: by default the journal lives in memory and can be lost on reboot unless persistent storage is configured in `/etc/systemd/journald.conf`. For production I always make sure logs are either persisted or shipped to a central system — a reboot wiping your debug evidence is painful."

**Key Point:** "journalctl is structured and queryable by unit and time; ensure journal persistence or central shipping in production."

---

## Package Management

### Q10: How do you install and manage software on Linux? Compare apt and yum/dnf.

**How to Answer:**

"Debian/Ubuntu use `apt` with `.deb` packages, RHEL/CentOS/Amazon Linux use `yum` or `dnf` with `.rpm` packages. The workflows are parallel:

```bash
# Debian/Ubuntu
sudo apt update              # refresh package index (not upgrades!)
sudo apt install nginx -y
sudo apt upgrade             # actually upgrade packages

# RHEL/CentOS/Amazon Linux
sudo yum install nginx -y    # or dnf on newer versions
sudo yum update
```

The classic trap is `apt update` vs `apt upgrade` — update only refreshes the list of what's available, upgrade actually installs newer versions. Running install without update first can give you stale versions.

For DevOps, the important part isn't memorizing flags — it's that package installs should be in automation, not done by hand. If I'm installing something on a server manually more than once, it belongs in an Ansible playbook, a Dockerfile, or user-data script."

**Key Point:** "`apt update` refreshes the index, `apt upgrade` installs. Anything installed twice by hand belongs in automation."

---

## Users & Groups

### Q11: How do you manage users, and how does `sudo` actually work?

**How to Answer:**

"User management basics:

```bash
sudo useradd -m -s /bin/bash deploy      # create user with home dir
sudo passwd deploy                        # set password
sudo usermod -aG docker deploy             # add to group (append!)
sudo userdel -r olduser                   # remove user and home
```

The `-aG` flag is one people get wrong — without `-a` (append), `usermod -G` *replaces* all group memberships, which has locked people out of sudo before.

`sudo` lets permitted users run commands as root. Who can do what is defined in `/etc/sudoers` — which you should only edit with `visudo` since it validates syntax before saving. A syntax error in sudoers can lock everyone out of root, so never edit it directly.

For automation, I create service-specific users with no login shell and no password — like `appuser` — and scope their sudo to exactly the commands they need:

```
# /etc/sudoers.d/deploy
deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp
```

That follows least privilege: the deploy user can restart the app and nothing else."

**Key Point:** "Always `visudo` for sudoers edits, always `-a` with `usermod -G`, and scope sudo to exact commands."

---

## SSH & Remote Access

### Q12: How does SSH key-based authentication work, and how do you harden SSH?

**How to Answer:**

"SSH key auth uses a key pair: the private key stays on your machine, the public key goes into `~/.ssh/authorized_keys` on the server. When you connect, the server challenges you to prove you hold the private key without ever sending it over the wire. It's both more secure and more automation-friendly than passwords.

Setup is:

```bash
ssh-keygen -t ed25519 -C "satyam@laptop"
ssh-copy-id user@server
```

Permissions matter enormously here — SSH will silently refuse to work if they're wrong: `~/.ssh` must be 700, `authorized_keys` must be 600, and both owned by the user.

For hardening, in `/etc/sshd_config` I always set:

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

And restart sshd. Disabling password auth kills brute-force attacks outright, and `PermitRootLogin no` means attackers can't target root directly — they need a valid username *and* key. On cloud VMs I also restrict port 22 to known IPs in the security group."

**Key Point:** "Keys over passwords always; 700 on .ssh, 600 on authorized_keys; disable password auth and root login."

---

## Disk Usage & Monitoring

### Q13: A server's disk is full. How do you diagnose and fix it?

**How to Answer:**

"This is one of the most common production incidents, and there's a method to it. First, confirm and locate:

```bash
df -h                          # which filesystem is full?
du -sh /* 2>/dev/null | sort -rh | head   # biggest top-level dirs
du -sh /var/log/* 2>/dev/null | sort -rh | head  # check logs specifically
```

Nine times out of ten it's `/var/log` — a runaway log file. But there's a classic trap: a process holding open a *deleted* file. If someone did `rm` on a log that a process is still writing to, `du` won't show it but `df` still shows the space as used, because the kernel only frees space when the last file descriptor closes. You find those with:

```bash
lsof | grep -i deleted
```

And the fix isn't deleting more files — it's restarting the holding process or truncating via the fd: `: > /proc/<pid>/fd/<fd>`.

For the immediate fix on a log directory, I truncate rather than delete active logs: `> /var/log/app.log` or `truncate -s 0`. Then the real fix: set up logrotate — which most distros include — so it never happens again:

```
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    missingok
    copytruncate
}
```

`copytruncate` matters for apps that can't reopen log files — it copies then truncates in place instead of moving the file."

**Key Point:** "Check `lsof | grep deleted` — deleted-but-open files are the classic invisible disk hog. Fix with logrotate, not manual deletes."

---

### Q14: How do you check memory and CPU pressure on a Linux box?

**How to Answer:**

"`free -h` for memory at a glance — but read it correctly. On Linux, 'used' includes disk cache, which the kernel will happily give back. The `available` column is the real number: memory actually free for new processes. People panic about high 'used' memory all the time, but if `available` is healthy, the system is fine.

For CPU, `top` or `htop` shows per-core usage, and I watch the load average — those three numbers are 1, 5, and 15-minute averages of runnable processes. Rule of thumb: sustained load above your core count means the CPUs are saturated.

For deeper diagnosis:

```bash
free -h                        # memory, watch 'available'
vmstat 1                       # context switches, swap activity
iostat -x 1                    # disk IO pressure
dmesg | grep -i oom            # did the OOM killer fire?
```

If `dmesg` shows OOM killer activity, the kernel killed your process because the box genuinely ran out of memory — that's a sizing or leak problem, not something you fix with commands. In containers, the equivalent is checking cgroup limits and the OOMKilled status on the pod."

**Key Point:** "Trust `available` memory, not `used`. Load average above core count means CPU saturation. OOM kills in dmesg mean a sizing problem."

---

## Environment Variables & Shell Basics

### Q15: How do environment variables work, and what's the difference between `.bashrc` and `.profile`?

**How to Answer:**

"Environment variables are key-value pairs inherited by child processes — they're how we pass configuration to applications without hardcoding it. `export DATABASE_URL=...` makes it visible to child processes; without `export` it's just a shell-local variable.

```bash
export NODE_ENV=production
echo $NODE_ENV
env | grep NODE            # list matching vars
```

This is the foundation of 12-factor app config, and it's how containers get their config too — Docker `-e` flags and Kubernetes env blocks all become process environment variables.

As for the shell files: `.profile` (or `.bash_profile`) runs once at login, `.bashrc` runs for every interactive non-login shell. The practical rule: put environment variables and PATH changes in `.profile`, put aliases and prompt customization in `.bashrc`. And on most systems `.profile` sources `.bashrc` anyway.

One production note: systemd services don't read any of these — you set their environment in the unit file with `Environment=` lines or an `EnvironmentFile`. I've seen people debug for an hour wondering why their app can't see a variable they put in `.bashrc` on the server."

**Key Point:** "export makes vars visible to children; 12-factor config rides on env vars. systemd ignores shell dotfiles — use Environment= in unit files."

---

## Common Troubleshooting Scenarios

### Q16: Walk me through debugging a service that won't start.

**How to Answer:**

"I follow a fixed order so I don't skip steps under pressure:

1. **`systemctl status myapp`** — is it running, failed, or never started? The recent log lines here often give it away immediately.
2. **`journalctl -u myapp --since '15 min ago'`** — full error output. Most startup failures print the actual reason here — bad config, missing file, port in use.
3. **Config check** — if it's nginx, `nginx -t`; if it's my own app, verify the config file syntax and that referenced paths exist.
4. **Port conflicts** — `ss -tlnp | grep <port>` — is something already bound to the port?
5. **Permissions** — can the service user read its config and write its log dir? `sudo -u appuser cat /opt/myapp/config.yaml` is a quick test.
6. **Dependencies** — `systemctl list-dependencies myapp` — did the database or network target actually come up?

I work outside-in: service state → logs → config → resources → dependencies. And I always check whether *anything* changed recently — deploys, config edits, cert renewals. Most outages are caused by a change, not by spontaneous failure."

**Key Point:** "Debug outside-in: status → logs → config → ports → permissions → dependencies. Most outages trace to a recent change."

---

### Q17: How do you find which process is listening on a port, and how do you check connectivity to a remote service?

**How to Answer:**

"Locally, `ss` replaced `netstat` — it's faster and preinstalled everywhere:

```bash
ss -tlnp              # TCP listening sockets with process names
ss -tlnp | grep 5432  # who's on postgres's port?
```

The `-p` flag shows the process, but you need sudo to see processes owned by other users.

For remote connectivity, I layer the checks:

```bash
ping db.internal               # is the host reachable at all? (ICMP may be blocked, so failure isn't conclusive)
nc -zv db.internal 5432        # can I open a TCP connection to the port?
curl -v https://api.internal/health   # does the application respond?
```

`nc -zv` is the key one — it tells you specifically whether the port is open, separating network issues from application issues. And `curl -v` shows the TLS handshake, which is where I catch expired certificates and SNI problems.

If `nc` succeeds but `curl` fails, the network is fine and the app is misbehaving. If `nc` fails, I'm looking at security groups, firewalls, or the service not listening."

**Key Point:** "`ss -tlnp` locally; `nc -zv` to isolate network vs app; `curl -v` to see TLS and HTTP behavior."

---

*Next: Filesystem Hierarchy & Permissions deep-dive → coming tomorrow.*
