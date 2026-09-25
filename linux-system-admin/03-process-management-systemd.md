# Process Management & systemd Interview Preparation Guide

*How to Answer Process Management & systemd Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Process Basics and PID 1](#process-basics-and-pid-1)
2. [Signals and Killing Processes](#signals-and-killing-processes)
3. [Background Jobs and Process Control](#background-jobs-and-process-control)
4. [systemd Units and systemctl](#systemd-units-and-systemctl)
5. [Writing Your Own Unit Files](#writing-your-own-unit-files)
6. [systemd in Production: Timers, Journal, Resource Limits](#systemd-in-production-timers-journal-resource-limits)
7. [Troubleshooting: Zombies, OOM, and Runaway Processes](#troubleshooting-zombies-oom-and-runaway-processes)

---

## Process Basics and PID 1

### Q1: What is a process, really? And what's so special about PID 1?

**How to Answer:**

"A process is just a running program — code plus its own memory, file descriptors, and state. Every process is spawned by a parent via fork and exec, and they form a tree you can see with `ps --forest`.

PID 1 is the first process the kernel starts, and it's the ultimate parent — orphaned processes get re-adopted by it. On modern distros that's systemd, which means systemd owns service supervision, boot ordering, and reaping zombies.

The interview trap: in a Docker container, PID 1 is *your app*, and most apps don't reap children. That's why you get zombie buildup in containers unless you use an init like `tini` or `--init`."

**Key Point:** "PID 1 is the ultimate parent and zombie reaper. In containers your app becomes PID 1, so either handle SIGTERM and reap children or run `--init`."

### Q2: How do you actually read what's happening with processes — ps, top, htop?

**How to Answer:**

"`ps aux` is my snapshot: user, PID, CPU%, MEM%, and the command line. The STAT column tells the real story — `R` running, `S` sleeping, `Z` zombie, `D` uninterruptible sleep which usually means stuck on I/O.

For live behavior I use `htop` over `top` — same data, but you can sort, filter, and kill without remembering keystrokes. And `/proc/<pid>/` is the raw source: cmdline, environ, open file descriptors in `fd/`, all plain text you can grep.

One thing interviewers love: `top` inside a container lies about total memory and CPU because it reads host-level `/proc`. I trust `docker stats` or cgroup files instead."

**Key Point:** "`ps` for snapshots, `htop` for live, `/proc/<pid>` for the raw truth. Inside containers, `top` lies — read the cgroup files instead."

---

## Signals and Killing Processes

### Q3: Explain signals. What actually happens when you kill a process?

**How to Answer:**

"Signals are the kernel's way of poking a process — they're async notifications, not function calls. The big ones: `SIGHUP` (1) reload config, `SIGINT` (2) what Ctrl-C sends, `SIGTERM` (15) the polite 'please shut down', and `SIGKILL` (9) the uncatchable 'die now'.

`kill` doesn't kill — it *sends a signal*, default SIGTERM. A well-behaved app catches SIGTERM, flushes buffers, closes connections, and exits cleanly. That's the whole graceful-shutdown contract your Dockerfile and Kubernetes `terminationGracePeriodSeconds` depend on.

SIGKILL and SIGSTOP are the two signals a process cannot catch or ignore. If SIGTERM doesn't work, that's the escalation path — but it skips cleanup, so it's a last resort, not a habit."

**Key Point:** "`kill` sends signals, default SIGTERM — a polite shutdown request the app can handle. SIGKILL (9) is uncatchable and skips cleanup, so use it last, not first."

### Q4: A process won't die even with kill -9. Now what?

**How to Answer:**

"First I check whether it's actually a process or a thread — `ps -L` shows threads, and killing a thread's TID does nothing. If it's real and SIGKILL truly didn't work, it's almost certainly in uninterruptible sleep, state `D`.

State `D` means the process is stuck inside a kernel syscall, usually waiting on storage — a dead NFS mount, a hung disk. You can't kill it because the kernel won't preempt that wait. `dmesg` and `/proc/<pid>/stack` will show you where it's stuck.

The honest answer: you don't kill it, you fix the underlying I/O problem — remount the NFS, replace the disk — and the process either completes or the box gets rebooted. Anyone who says 'just kill -9 harder' hasn't met state D."

```bash
ps aux | awk '$8 ~ /D/ {print}'   # find processes stuck in uninterruptible sleep
cat /proc/<pid>/stack            # see the exact kernel call it's blocked in
```

**Key Point:** "Unkillable processes are in state `D` — stuck in a kernel I/O wait. You can't signal your way out; fix the storage or reboot."

---

## Background Jobs and Process Control

### Q5: How do you run a long job so it survives you logging out?

**How to Answer:**

"Appending `&` backgrounds it, but that alone dies with your shell — the shell sends SIGHUP to its children on exit. `nohup` makes the process ignore SIGHUP, so `nohup ./long-job &` survives logout, with output going to `nohup.out`.

The cleaner approach: `disown` a running job, or just use `tmux`/`screen` so you can reattach later. But honestly, on a server the right answer is a systemd unit or a proper scheduler — nohup is for ad-hoc work, not production.

The interview trap: people say 'I used nohup in production.' That's a red flag. Supervision, restarts, and logs come from systemd, not from a backgrounded shell job."

**Key Point:** "`&` backgrounds, `nohup` survives SIGHUP, but neither belongs in production. Long-lived work gets a systemd unit; ad-hoc work gets tmux."

### Q6: Walk me through shell job control — Ctrl-Z, bg, fg, jobs.

**How to Answer:**

"Ctrl-Z sends SIGTSTP, which *suspends* the foreground process — it's frozen, not dead. `jobs` lists your shell's jobs, `bg` resumes the suspended one in the background, `fg` brings it back to the foreground. `fg %2` picks job number 2.

This is per-shell state, not system-wide — another terminal can't see your jobs. And a suspended job still holds its resources and file locks, which bites people who Ctrl-Z a database migration and forget about it.

I mostly mention this to show I know the difference between suspended, backgrounded, and daemonized — three different things people constantly mix up."

**Key Point:** "Ctrl-Z suspends (SIGTSTP), `bg`/`fg` move jobs around, `jobs` lists them. All per-shell — and suspended is not the same as backgrounded or daemonized."

---

## systemd Units and systemctl

### Q7: What is systemd, and what did it replace?

**How to Answer:**

"systemd is PID 1 on basically every modern distro — it's the init system, service manager, and a whole toolbox. It replaced SysV init scripts, where services were shell scripts in `/etc/init.d` with no real supervision, no dependency ordering, and parallel startup was a hack.

The core idea is *units*: declarative files describing services, sockets, timers, mounts, targets. You declare what you want — 'run this after the network is up, restart it if it dies' — and systemd handles the how. `systemctl` is the control plane for all of it.

Why it matters for DevOps: systemd gives you process supervision, structured logging via journald, resource limits via cgroups, and socket activation — all without extra tooling. It's the reason 'just run it as a service' is a complete answer now."

**Key Point:** "systemd replaced fragile SysV init scripts with declarative units plus supervision, logging, and cgroups built in. `systemctl` is how you drive all of it."
