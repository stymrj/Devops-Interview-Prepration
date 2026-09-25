# Process Management & systemd — Hands-On Lab

*Pair this with the [Process Management & systemd interview guide](../03-process-management-systemd.md). Do every exercise on a VM you can break — a disposable Ubuntu/Debian box is ideal. Some exercises need sudo.*

---

## Exercise 1: Read the Process Tree

**Goal:** See the parent-child tree and decode `ps` STAT codes cold.

```bash
ps --forest -o pid,ppid,stat,cmd | head -30
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10

# Find anything in state D (uninterruptible) or Z (zombie)
ps aux | awk '$8 ~ /[DZ]/ {print}'
```

**Observe:** PID 1 should be `systemd`. Every process has a PPID chain leading back to it. Note the STAT column — `Ss`, `R+`, `Sl` — the lowercase letters are modifiers (`s` = session leader, `l` = multithreaded, `+` = foreground group).

**Why this matters in interviews/ops:** "The app is slow" starts with `ps aux --sort=-%cpu`. If you can't read STAT codes, you can't tell a busy process from a stuck one.

---

## Exercise 2: Send Signals and Watch What Happens

**Goal:** Feel the difference between SIGTERM, SIGINT, and SIGKILL.

```bash
sleep 600 &
PID=$!
kill -0 $PID && echo "alive"

kill -TERM $PID        # polite request
sleep 1; kill -0 $PID 2>/dev/null && echo "still alive" || echo "gone"

sleep 600 &
PID=$!
kill -KILL $PID       # no negotiation
sleep 1; kill -0 $PID 2>/dev/null && echo "still alive" || echo "gone"

# List every signal and its number
kill -l
```

**Observe:** `sleep` dies on SIGTERM because it doesn't trap it — the default action runs. SIGKILL works even on processes that ignore SIGTERM. `kill -0` doesn't send anything; it just checks existence (great for health checks in scripts).

**Why this matters in interviews/ops:** Graceful shutdown in Docker/K8s is SIGTERM handling. If your app ignores SIGTERM, every deploy waits out the full grace period then gets SIGKILLed — slow rollouts and dropped connections.

---

## Exercise 3: Trap SIGTERM Like a Real App

**Goal:** Write a script that shuts down gracefully, then verify it.

```bash
cat > /tmp/graceful.sh <<'EOF'
#!/bin/bash
trap 'echo "SIGTERM received: flushing and exiting"; exit 0' TERM
echo "worker started, pid $$"
while true; do echo "$(date) working..."; sleep 2; done
EOF
chmod +x /tmp/graceful.sh
/tmp/graceful.sh &
PID=$!
sleep 3
kill -TERM $PID
wait $PID; echo "exit code: $?"
```

**Observe:** The trap runs, the script exits 0 — that's graceful shutdown. Without the trap, `sleep` inside would die mid-loop and any cleanup would be skipped. `wait` gives you the real exit code.

**Why this matters in interviews/ops:** This is exactly what your container's entrypoint must do. A PID 1 that doesn't trap SIGTERM is why Kubernetes pods take 30s to terminate.

---

## Exercise 4: Job Control — Suspend, Background, Foreground

**Goal:** Master Ctrl-Z, `bg`, `fg`, and `jobs` on a live process.

```bash
sleep 300
# press Ctrl-Z  ->  you should see: [1]+  Stopped  sleep 300

jobs              # lists suspended job
bg                # resumes it in the background
jobs              # now shows "Running"
fg                # brings it back to foreground
# press Ctrl-C to kill it
```

**Observe:** Ctrl-Z sends SIGTSTP — the process is frozen, holding memory and locks. `bg` resumes it without a controlling terminal need. All of this is per-shell: open another terminal and `jobs` shows nothing.

**Why this matters in interviews/ops:** Suspended ≠ backgrounded ≠ daemonized. Interviewers ask this to check you know the difference — and in real life, a Ctrl-Z'd migration holding a DB lock is a classic self-inflicted outage.

---

## Exercise 5: Survive Logout — nohup vs tmux

**Goal:** Compare the ad-hoc ways to keep work alive after you disconnect.

```bash
# Method 1: nohup (survives SIGHUP)
nohup bash -c 'while true; do date >> /tmp/nohup-test.log; sleep 5; done' &
echo $! > /tmp/nohup.pid
# log out and back in — the loop is still running, output in nohup.out / the log

# Method 2: tmux (reattachable session)
tmux new -s work
# run: while true; do date; sleep 5; done
# detach with Ctrl-b then d. Log out, log back in:
tmux attach -t work

# Cleanup
kill $(cat /tmp/nohup.pid)
```

**Observe:** `nohup` output defaults to `nohup.out` in the cwd — a classic disk-fill surprise. `tmux` keeps the whole terminal session, scrollback included, which is strictly better for interactive work.

**Why this matters in interviews/ops:** Both are fine for ad-hoc jobs. Neither is a production answer — if someone says they run services with nohup, that's a supervision gap waiting to page you.

---

## Exercise 6: Write and Ship a Real systemd Unit

**Goal:** Create a service unit from scratch and run it under systemd.

```bash
sudo useradd -r -s /usr/sbin/nologin appuser 2>/dev/null
sudo mkdir -p /opt/demo && cat > /tmp/demo.py <<'EOF'
import time
print("demo service starting", flush=True)
while True:
    print("heartbeat", flush=True); time.sleep(10)
EOF
sudo cp /tmp/demo.py /opt/demo/ && sudo chown -R appuser:appuser /opt/demo

sudo tee /etc/systemd/system/demo.service > /dev/null <<'EOF'
[Unit]
Description=Demo heartbeat service
After=network.target

[Service]
User=appuser
ExecStart=/usr/bin/python3 /opt/demo/demo.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now demo
systemctl status demo --no-pager
journalctl -u demo -n 5 --no-pager
```

**Observe:** `enable --now` enables boot-start AND starts immediately. `status` shows the main PID, memory, and recent log lines in one view. The service runs as `appuser`, not root — verify with `ps -o user= -p $(systemctl show demo -p MainPID --value)`.

**Why this matters in interviews/ops:** This is the single most practical systemd skill: take any long-lived process and supervise it properly. Every "how do you run X in production" answer ends here.

---

## Exercise 7: Break It, Then Debug With journalctl

**Goal:** Practice the real failure-debugging loop on a broken unit.

```bash
# Break the service: point ExecStart at a missing binary
sudo sed -i 's|/usr/bin/python3|/usr/bin/python3-MISSING|' /etc/systemd/system/demo.service
sudo systemctl daemon-reload
sudo systemctl restart demo || true

systemctl status demo --no-pager          # shows failed + exit code
journalctl -u demo -n 20 --no-pager      # the actual error
journalctl -u demo -b --no-pager         # this boot only

# Fix it back
sudo sed -i 's|python3-MISSING|python3|' /etc/systemd/system/demo.service
sudo systemctl daemon-reload && sudo systemctl restart demo
systemctl is-active demo
```

**Observe:** `status` tells you *that* it failed; `journalctl -u` tells you *why*. The `-b` flag filters to the current boot — essential on long-lived boxes with rotated logs. Note the failure would have been invisible without `daemon-reload`.

**Why this matters in interviews/ops:** `status` → `journalctl -u` is the debugging loop you'll run hundreds of times. Interviewers love "service fails on boot but works manually" — this exercise builds the muscle for it.

---

## Exercise 8: systemd Timer vs cron

**Goal:** Build a timer unit and compare its observability to cron.

```bash
sudo tee /etc/systemd/system/demo-tick.service > /dev/null <<'EOF'
[Unit]
Description=Demo periodic tick

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo "tick at $(date)" >> /tmp/tick.log'
EOF
sudo tee /etc/systemd/system/demo-tick.timer > /dev/null <<'EOF'
[Unit]
Description=Run demo tick every minute

[Timer]
OnCalendar=*:0/1
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now demo-tick.timer
sleep 70
systemctl list-timers demo-tick.timer --no-pager
cat /tmp/tick.log
journalctl -u demo-tick --no-pager | tail -5
```

**Observe:** `list-timers` shows last run, next run, and time left — try getting that from cron. Every run is logged in the journal automatically. `Persistent=true` means a missed run during downtime fires on boot.

**Why this matters in interviews/ops:** This is the concrete demo behind "timers over cron." When the 3 AM job fails, `journalctl -u` gives you the answer instead of silence.

---

## Exercise 9: Cap a Runaway With cgroups

**Goal:** Limit a service's memory and watch the limit enforced.

```bash
# A memory hog: allocates ~500MB
cat > /tmp/hog.py <<'EOF'
data = bytearray(500 * 1024 * 1024)
import time; time.sleep(3600)
EOF

sudo cp /tmp/hog.py /opt/demo/ && sudo chown appuser:appuser /opt/demo/hog.py
sudo tee /etc/systemd/system/demo-hog.service > /dev/null <<'EOF'
[Unit]
Description=Demo memory hog (capped)

[Service]
User=appuser
ExecStart=/usr/bin/python3 /opt/demo/hog.py
MemoryMax=200M
Restart=no

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start demo-hog || true
sleep 3
systemctl status demo-hog --no-pager | head -8
dmesg | tail -5   # look for the OOM-killer line naming python3
```

**Observe:** The hog asked for 500MB, `MemoryMax=200M` killed it, and `dmesg` shows the kernel OOM-killer doing it. The rest of the box never noticed. Remove the limit and re-run to feel the difference — without it, the kernel picks victims on its own terms.

**Why this matters in interviews/ops:** This is the hands-on version of "one runaway shouldn't take the box down." Same mechanism as Kubernetes memory limits — now you've seen it at the systemd layer.

---

## Exercise 10: Hunt Zombies and Read dmesg

**Goal:** Create a zombie on purpose and learn why you can't kill it.

```bash
cat > /tmp/zombie-parent.py <<'EOF'
import os, time
pid = os.fork()
if pid == 0:
    os._exit(0)          # child dies immediately -> becomes zombie
print(f"parent {os.getpid()} spawned dead child {pid}, now sleeping")
time.sleep(300)          # parent never calls wait() -> zombie persists
EOF
python3 /tmp/zombie-parent.py &
ps aux | awk '$8 ~ /Z/ {print}'   # the zombie shows as [python3] <defunct>

# Try to kill the zombie (it won't work — it's already dead)
ZPID=$(ps aux | awk '$8 ~ /Z/ {print $2}' | head -1)
kill -9 $ZPID; sleep 1
ps -p $ZPID -o pid,stat,cmd      # still there

# Kill the PARENT instead — init reaps the zombie
PPID=$(ps -o ppid= -p $ZPID | tr -d ' ')
kill $PPID; sleep 1
ps aux | awk '$8 ~ /Z/ {print}' || echo "zombie reaped"
```

**Observe:** `kill -9` on the zombie does nothing — there's no process left to signal. Killing the parent orphans it to PID 1, which reaps it instantly. That's the whole zombie lifecycle in one exercise.

**Why this matters in interviews/ops:** "You can't kill a zombie, you kill its parent" is a classic interview one-liner — now you've proven it. And the container takeaway: a PID 1 that never reaps accumulates these forever.

---

*Cleanup: `sudo systemctl disable --now demo demo-tick.timer; sudo rm /etc/systemd/system/demo*.service /etc/systemd/system/demo*.timer; sudo systemctl daemon-reload; rm -rf /tmp/permlab /tmp/*.py /tmp/*.sh /tmp/tick.log /tmp/nohup*.log`*
