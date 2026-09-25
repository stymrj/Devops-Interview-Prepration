# Process Management & systemd — Cheat Sheet

*Pair with the [interview guide](../03-process-management-systemd.md) and [hands-on lab](03-process-management-systemd-lab.md).*

## Inspecting processes

```bash
ps aux                          # snapshot: user, pid, %cpu, %mem, command
ps aux --sort=-%cpu | head       # top CPU consumers
ps aux --sort=-%mem | head       # top memory consumers
ps --forest -o pid,ppid,stat,cmd  # process tree
ps -o pid,ppid,stat,cmd -p <pid>  # one process, compact
ps -L -p <pid>                  # threads of a process
pgrep -af python                # find by name (full cmdline)
pidof nginx                     # pids by exact name
htop                            # interactive: sort/filter/kill
```

STAT codes: `R` running · `S` sleeping · `D` uninterruptible (stuck I/O) · `Z` zombie · `T` stopped · `s` session leader · `l` multithreaded · `+` foreground

## /proc — the raw source

```bash
ls /proc/<pid>/                 # cmdline, environ, status, fd/, task/
cat /proc/<pid>/cmdline | tr '\0' ' '   # exact command line
cat /proc/<pid>/status | grep -E 'State|VmRSS'  # state + resident memory
ls -l /proc/<pid>/fd            # open file descriptors (leak hunting)
cat /proc/<pid>/stack           # kernel stack — where a D-state process is stuck
cat /proc/loadavg               # 1/5/15min load + running/total threads
```

## Signals

| Signal | Num | Meaning |
|---|---|---|
| SIGHUP | 1 | Hangup — daemons reload config; shells send on logout |
| SIGINT | 2 | Ctrl-C |
| SIGQUIT | 3 | Quit + core dump |
| SIGKILL | 9 | Die now — cannot be caught or ignored |
| SIGTERM | 15 | Polite shutdown request (default for `kill`) |
| SIGSTOP | 19 | Freeze — cannot be caught or ignored |
| SIGTSTP | 20 | Ctrl-Z suspend (catchable, unlike STOP) |
| SIGCONT | 18 | Resume a stopped process |

```bash
kill -TERM <pid>     # graceful (default)
kill -9 <pid>        # last resort, skips cleanup
kill -0 <pid>        # existence check only (script health checks)
kill -l              # list all signals
pkill -f "gunicorn"  # kill by pattern (careful)
killall -u deploy    # kill all of a user's processes
```

## Job control (per-shell)

```bash
./job &        # background it
Ctrl-Z         # suspend foreground (SIGTSTP)
jobs           # list shell jobs
bg             # resume suspended job in background
fg             # bring job to foreground
fg %2          # bring job #2 to foreground
disown         # detach job from shell (no SIGHUP on exit)
nohup ./job &  # immune to SIGHUP; output -> nohup.out
```

## systemctl — daily verbs

```bash
systemctl status <svc>              # state, pid, recent logs — start here
systemctl start|stop|restart <svc>
systemctl reload <svc>              # re-read config, no process restart
systemctl enable|disable <svc>      # boot-time on/off
systemctl enable --now <svc>        # enable + start in one
systemctl is-enabled|is-active <svc>
systemctl daemon-reload             # REQUIRED after editing any unit file
systemctl list-units --failed       # everything failed right now
systemctl list-timers               # all timers: last run, next run
systemctl cat <svc>                 # show the merged unit file
systemctl show <svc> -p MainPID,MemoryCurrent  # machine-readable props
```

## Unit file anatomy (`/etc/systemd/system/<name>.service`)

```ini
[Unit]
Description=My app
After=network-online.target     # ordering (network.target = stack up, not usable)
Wants=postgresql.service        # weak dependency (try, don't fail)
Requires=docker.service         # strong dependency (fail together)

[Service]
Type=simple                     # default: ExecStart stays foreground
ExecStart=/usr/bin/myapp --flag # ALWAYS absolute paths
ExecReload=/bin/kill -HUP $MAINPID
User=appuser                    # never root
Group=appgroup
WorkingDirectory=/opt/myapp
Environment="PORT=8080"         # inline vars
EnvironmentFile=/etc/myapp/env  # file vars (Key=Value lines)
Restart=on-failure              # or: always | no | on-abnormal
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=3               # max 3 restarts per 60s, then give up
TimeoutStartSec=30
TimeoutStopSec=30               # SIGKILL after this on stop

[Install]
WantedBy=multi-user.target      # what `enable` hooks into
```

`Type=` values: `simple` (foreground, default) · `forking` (daemonizes itself, needs PIDFile) · `oneshot` (runs once, for timers) · `notify` (signals readiness via sd_notify)

## Resource limits (cgroups, per unit)

```ini
[Service]
MemoryMax=1G        # hard cap — OOM-kill past this
MemoryHigh=800M     # throttle zone before the wall
CPUQuota=50%        # half a core
CPUWeight=200       # relative share (default 100)
TasksMax=100        # cap threads/processes (fork-bomb guard)
```

```bash
systemd-cgtop                    # live cgroup resource view
systemctl show <svc> -p MemoryCurrent,CPUUsageNSec
```

## Timers (cron replacement)

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup

[Timer]
OnCalendar=daily                 # or: *-*-* 02:30:00 | hourly | weekly
OnBootSec=5min                   # monotonic: after boot
OnUnitActiveSec=1h               # monotonic: after last run
Persistent=true                  # run missed jobs on boot
AccuracySec=1min                 # allow coalescing (power saving)

[Install]
WantedBy=timers.target
```

Paired `<name>.service` needs `Type=oneshot`. Enable the **timer**, not the service: `systemctl enable --now backup.timer`.

`OnCalendar` examples: `*-*-* *:00:00` hourly · `Mon *-*-* 09:00:00` Mondays 9am · `*:0/15` every 15 min. Validate: `systemd-analyze calendar "Mon *-*-* 09:00:00"`.

## journalctl — logs

```bash
journalctl -u <svc>                 # this service's logs
journalctl -u <svc> -n 50           # last 50 lines
journalctl -u <svc> -f              # follow (like tail -f)
journalctl -u <svc> -b              # current boot only
journalctl -u <svc> --since "1 hour ago"
journalctl -u <svc> -p err          # errors and worse
journalctl -u <svc> -o json-pretty  # structured output
journalctl --disk-usage             # how much space logs use
journalctl --vacuum-time=7d         # prune older than 7 days
journalctl -k -p 3                  # kernel errors (dmesg equivalent)
```

## Debugging one-liners

```bash
systemctl status <svc> && journalctl -u <svc> -n 50 --no-pager  # the loop
dmesg | grep -i -E 'oom|killed process'   # did the OOM killer fire?
ps aux | awk '$8 ~ /D/'                  # stuck in uninterruptible I/O
ps aux | awk '$8 ~ /Z/'                  # zombies (fix the parent, not these)
cat /proc/<pid>/stack                    # where a D-state process is blocked
systemd-analyze blame                    # what slowed this boot
systemd-analyze critical-chain           # boot dependency chain
systemd-analyze verify <unit-file>       # lint a unit before enabling
```

## Gotchas checklist

- Edited a unit but nothing changed → forgot `daemon-reload`
- Fails on boot, works manually → missing `After=network-online.target` or env vars
- Container PID 1 zombies → app doesn't reap; use `--init`/`tini`
- `top` lies in containers → read cgroup files or `docker stats`
- `Restart=always` + fast crash → restart storm; use `on-failure` + `StartLimitBurst`
- Secrets in `Environment=` → visible in `systemctl show`; prefer `EnvironmentFile` with `640` perms
