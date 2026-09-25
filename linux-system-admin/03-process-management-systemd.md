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

### Q8: A service is down. Walk me through your exact systemctl flow.

**How to Answer:**

"First `systemctl status myapp` — it shows active/inactive, recent log lines, the main PID, and whether it's enabled, all in one screen. If it's failed, I check the exit code and then `journalctl -u myapp -n 50 --no-pager` for the actual error.

Then the verbs: `start`, `stop`, `restart`, `reload` — and I use `reload` over `restart` when the app supports it, because reload keeps the process alive and just re-reads config. `enable`/`disable` control boot-time startup, `is-enabled` verifies it.

The classic trap: editing the unit file and forgetting `systemctl daemon-reload`. systemd caches unit files — your change does literally nothing until you reload the daemon."

```bash
systemctl status myapp && journalctl -u myapp -n 50 --no-pager
```

**Key Point:** "`status` then `journalctl -u` is the debugging loop. And after any unit edit: `daemon-reload`, or your change silently does nothing."

---

## Writing Your Own Unit Files

### Q9: Write a systemd unit file for a Node.js app. What are the directives that actually matter?

**How to Answer:**

"A unit file lives in `/etc/systemd/system/myapp.service` and has three sections. `[Unit]` holds `Description` and `After=network.target` so it starts after networking. `[Service]` is the meat: `ExecStart` with the full absolute path, `Restart=always`, `User=` so it never runs as root, `WorkingDirectory`, and `EnvironmentFile` for secrets.

`[Install]` with `WantedBy=multi-user.target` is what `systemctl enable` hooks into. Then `daemon-reload`, `enable --now`, and you're done.

Interviewers check two things here: that you never run app processes as root, and that you know `Type=simple` (the default, process stays foreground) vs `Type=forking` (old daemons that background themselves). For anything modern, simple."

```ini
[Unit]
Description=My Node.js API
After=network.target

[Service]
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node /opt/myapp/server.js
Restart=always
RestartSec=5
EnvironmentFile=/etc/myapp/env

[Install]
WantedBy=multi-user.target
```

**Key Point:** "Unit, Service, Install. Full paths in ExecStart, a non-root User, Restart=always — that's 90% of a production unit file."

### Q10: `Restart=always` sounds great. When does it bite you?

**How to Answer:**

"It bites when the app fails fast and loops — crash, restart, crash, restart, hammering the CPU and flooding logs. systemd has `StartLimitIntervalSec` and `StartLimitBurst` to cap that, but people don't set them.

The subtler trap: `Restart=always` restarts even on a clean exit. If your app exits 0 because its config is wrong and there's nothing to do, systemd keeps resurrecting a broken service. `Restart=on-failure` is usually what you actually want — restart on crashes, not on clean exits.

And `RestartSec` matters more than people think. Zero delay on a service that needs a database means it burns through its restart budget before the DB is even up. Five seconds of patience fixes a whole class of boot-order flakiness."

**Key Point:** "`always` restarts even clean exits — `on-failure` is usually right. Set `RestartSec` and start-limit guards, or a crashing service becomes a restart storm."

---

## systemd in Production: Timers, Journal, Resource Limits

### Q11: systemd timers vs cron — which do you pick and why?

**How to Answer:**

"Timers win on any systemd box and it's not close. A timer unit gives you logging in the journal per run, dependency ordering with `After=`, calendar or monotonic scheduling, and `Persistent=true` so missed runs fire on boot — cron does none of that.

The killer feature for me is observability: `systemctl list-timers` shows every timer, when it last ran, and when it runs next. With cron I'm grepping `/var/log/syslog` hoping the job logged something.

I still use cron for user-level quick hacks on my laptop. On servers, timers — because when a 3 AM job fails, I want `journalctl -u backup-job` to tell me why, not silence."

**Key Point:** "Timers give you logging, dependencies, and missed-run catch-up — everything cron lacks. `systemctl list-timers` alone is worth the switch."

### Q12: How do you debug a service that fails on boot but works when you start it manually?

**How to Answer:**

"That pattern screams ordering or environment problem. `journalctl -u myapp -b` shows this boot's logs only — I look for what wasn't ready yet: database unreachable, DNS not up, a mount missing.

The usual suspects: the unit is missing `After=network-online.target` (not just `network.target`, which only means the stack started), or it depends on an env var that's set in my shell but not in the unit's `Environment`. Systemd services get a minimal environment — no `~/.bashrc`, no exported vars.

My fix flow: add the right `After=` and `Wants=`, put env in `EnvironmentFile`, then `daemon-reload` and reboot-test. If it survives a reboot, it's fixed — testing with manual `systemctl start` proves nothing about boot."

**Key Point:** "Fails on boot but not manually = ordering or environment. `journalctl -u myapp -b`, fix `After=` and `EnvironmentFile`, then prove it with a real reboot."

### Q13: How do you cap a service's CPU and memory with systemd?

**How to Answer:**

"systemd sits on top of cgroups, so resource limits are just unit directives. `MemoryMax=1G` hard-caps RAM — the kernel OOM-kills the service's processes past it. `CPUQuota=50%` limits it to half a core. No sidecar tooling needed.

I set these on everything in production, because one runaway worker shouldn't take the box down with it. The app gets killed, systemd restarts it per the restart policy, and I get paged — that's a controlled failure instead of a 3 AM full outage.

One nuance: `MemoryMax` is a hard wall, `MemoryHigh` throttles first. For most services I set both — throttle as a warning zone, hard cap as the wall. And these same knobs are what Kubernetes requests/limits translate to under the hood."

```ini
[Service]
MemoryMax=1G
MemoryHigh=800M
CPUQuota=50%
```

**Key Point:** "`MemoryMax` and `CPUQuota` in the unit file cap a service via cgroups. One runaway process gets killed and restarted — it doesn't take the whole box down."

---

## Troubleshooting: Zombies, OOM, and Runaway Processes

### Q14: What are zombie processes? Should I panic when I see them?

**How to Answer:**

"A zombie is a dead process whose parent hasn't called `wait()` to collect its exit status yet. It's not running, not using CPU or memory — just an entry in the process table holding the exit code. The `Z` in `ps` STAT.

A couple of zombies are harmless — parents reap them eventually. Hundreds of them mean the parent is broken or stuck and never reaping, which can eventually exhaust the PID table. You can't kill a zombie because it's already dead; you fix or restart the *parent*.

The container angle is the real interview point: if your app is PID 1 and spawns children without reaping them, zombies accumulate forever. That's the `--init` / `tini` conversation again — PID 1 has reaping duties."

**Key Point:** "Zombies are dead processes awaiting reaping — harmless in small numbers, a broken-parent symptom in large ones. Kill the parent, not the zombie. In containers, that's why PID 1 must reap."

### Q15: The OOM killer fired in production. What happened, and what do you do?

**How to Answer:**

"When the box runs out of memory, the kernel's OOM killer picks a victim — usually the biggest RSS scorer — and kills it to save the system. `dmesg | grep -i oom` shows you exactly what died and why, with the memory score.

First response: confirm it was OOM and not a crash — the dmesg line is unambiguous. Then figure out if it was a leak or a spike: `journalctl` memory graphs, or the app's metrics if you have them. A slow climb is a leak; a sudden spike is usually a bad deploy or a traffic surge.

The fix depends on the cause. Leaks get fixed in code. Spikes get guardrails: `MemoryMax` on the unit so the service dies alone instead of the kernel picking victims at random, and proper sizing so the box has headroom. Random OOM kills are a capacity-planning failure, not bad luck."

**Key Point:** "`dmesg` proves it was OOM. Leaks get fixed, spikes get `MemoryMax` guardrails. The kernel picking random victims means you under-provisioned — plan capacity instead."

---

*End of guide — practice saying these out loud until they sound like you.*
