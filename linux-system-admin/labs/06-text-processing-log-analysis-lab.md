# Text Processing & Log Analysis — Hands-On Lab

Ten exercises that drill the exact habits from the interview guide. Work in a throwaway directory — everything here is safe to run locally.

```bash
mkdir -p ~/textproc-lab && cd ~/textproc-lab
```

First, generate some realistic practice logs to work against:

```bash
cat > app.log << 'EOF'
2026-09-26 21:01:12 INFO  [api] request_id=abc123 user=u42 status=200 latency=34ms
2026-09-26 21:01:15 ERROR [api] request_id=abc124 user=u17 status=500 latency=2103ms msg="db connection timeout"
2026-09-26 21:01:18 WARN  [worker] queue_depth=847 retrying job_id=j991 in 5s
2026-09-26 21:01:22 ERROR [api] request_id=abc125 user=u42 status=503 latency=12ms msg="upstream unavailable"
2026-09-26 21:01:25 INFO  [api] request_id=abc126 user=u99 status=200 latency=41ms
2026-09-26 21:01:29 ERROR [worker] job_id=j992 failed after 3 attempts msg="OOM during transform"
2026-09-26 21:01:33 INFO  [health] probe ok latency=2ms
2026-09-26 21:01:36 INFO  [health] probe ok latency=2ms
EOF
```

---

## Exercise 1 — Find every error with context

**Goal:** Use `grep -n -C` to locate errors and see what surrounded them.

```bash
grep -n "ERROR" app.log
grep -C 1 "ERROR" app.log
```

**Expected output:** Line numbers 2, 4, 6 match. The `-C 1` version shows one line before and after each match, so you see the healthy lines bracketing the failure.

**Why it matters:** Context lines are where the cause hides — the error is the last domino.

---

## Exercise 2 — Exclude noise with -v

**Goal:** Strip health-check spam so real signal stands out.

```bash
grep -v "health" app.log
grep -c "ERROR" app.log
```

**Expected output:** Six lines without the two health probes; count of 3 errors.

**Why it matters:** Real logs are 90% noise. `-v` filtering before analysis is the first move in every incident.

---

## Exercise 3 — Pull a field with grep -oP

**Goal:** Extract only the `request_id` values from error lines.

```bash
grep "ERROR" app.log | grep -oP 'request_id=\K[^\s]+'
```

**Expected output:**
```
abc124
abc125
```

**Why it matters:** `\K` drops everything matched so far — a clean way to extract one field without awk.

---

## Exercise 4 — Greedy vs lazy regex

**Goal:** Feel the difference between `.*` and a bounded pattern.

```bash
echo '<b>hi</b>' | grep -oP '<.*>'      # greedy: one match
echo '<b>hi</b>' | grep -oP '<.*?>'     # lazy: two matches
echo '<b>hi</b>' | grep -oP '<[^>]*>'   # negated class: two matches
```

**Expected output:** First prints `<b>hi</b>`, the other two print `<b>` then `</b>`.

**Why it matters:** Greedy `.*` swallows repeated delimiters in logs — negated classes keep matches tight.

---

## Exercise 5 — Substitute a secret out of a log with sed

**Goal:** Redact a value in place, safely.

```bash
cp app.log redacted.log
sed -i.bak 's/user=u42/user=REDACTED/g' redacted.log
diff redacted.log.bak redacted.log
```

**Expected output:** The diff shows two lines changed (`u42` → `REDACTED`); the `.bak` file holds the original.

**Why it matters:** `-i.bak` is the production-safe habit — a bad sed without a backup can destroy a config.

---

## Exercise 6 — Carve a time window with sed

**Goal:** Print only log lines from a specific minute.

```bash
sed -n '/21:01:1/,/21:01:2/p' app.log
```

**Expected output:** The first four lines of the log (timestamps 21:01:12 through 21:01:22).

**Why it matters:** Pattern-addressed ranges turn a giant log into a slice around the incident.

---

## Exercise 7 — Average latency with awk

**Goal:** Compute a stat across a column in one pass.

```bash
grep '\[api\]' app.log | grep -oP 'latency=\K[0-9]+' | awk '{s+=$1; n++} END {print "avg:", s/n "ms", "over", n, "requests"}'
```

**Expected output:** `avg: 547.5ms over 4 requests`.

**Why it matters:** `awk` accumulates and reports — the one-liner version of a metrics query.

---

## Exercise 8 — Count errors per component with awk

**Goal:** Build a histogram of error sources.

```bash
grep "ERROR" app.log | awk -F'[][]' '{print $2}' | sort | uniq -c | sort -rn
```

**Expected output:**
```
2 api
1 worker
```

**Why it matters:** Counting by component tells you where to look first — `sort | uniq -c | sort -rn` is the universal ranking idiom.

---

## Exercise 9 — Find the slowest request

**Goal:** Rank lines by a numeric field.

```bash
awk '{for(i=1;i<=NF;i++) if($i ~ /^latency=/) {split($i,a,"="); gsub("ms","",a[2]); print a[2], $0}}' app.log | sort -rn | head -3
```

**Expected output:** The three highest-latency lines, starting with the 2103ms db-timeout error.

**Why it matters:** Extract → sort numerically → head is the skeleton of every "top N slowest" query.

---

## Exercise 10 — Simulate journalctl-style priority filtering

**Goal:** Practice the `-p` priority mindset on a flat file.

```bash
grep -E "ERROR|WARN" app.log              # everything at warning or worse
grep -E "ERROR|WARN" app.log | grep -v "retrying"   # drop the known-benign warning
```

**Expected output:** First: 4 lines (3 ERROR + 1 WARN). Second: 3 lines (the retry warning filtered).

**Why it matters:** Priority filtering plus exclusion of known-benign noise is exactly what `journalctl -p err` plus experience does on real systems.

---

**Done.** You can now grep with context, tame greedy regex, sed-edit safely, awk-aggregate, and rank log data — the full loop an interviewer expects you to demo.
