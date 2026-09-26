# Text Processing & Log Analysis Interview Preparation Guide

*How to Answer Text Processing & Log Analysis Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Grep Essentials](#grep-essentials)
2. [Regular Expressions](#regular-expressions)
3. [Sed Stream Editing](#sed-stream-editing)
4. [Awk Field Processing](#awk-field-processing)
5. [Journalctl and System Logs](#journalctl-and-system-logs)
6. [Log Analysis Pipelines](#log-analysis-pipelines)

---

## Grep Essentials

### Q1: Walk me through how you'd search logs with grep. What flags do you actually use?

**How to Answer:**

"I live in `grep -rn 'ERROR' /var/log/app/` — recursive, line numbers, done. `grep -i` when case doesn't matter, which is most of the time with messy logs.

`-C 5` is my favorite: it shows five lines of context around each match, so I see what led up to the error. `-v` inverts — great for filtering out noisy health-check lines.

And `grep -E` for extended regex when I need alternation like `ERROR|FATAL|CRITICAL`. Plain grep vs egrep matters less than knowing which flag to reach for."

```bash
grep -rn "ERROR" /var/log/app/ --include="*.log" -C 3 | head -50
```

**Key Point:** "`grep -rn` for recursive search with line numbers, `-C` for context around matches, `-E` for extended regex."

---

### Q2: What's the difference between grep, egrep, and fgrep? Which do you use?

**How to Answer:**

"They're all grep now — `egrep` is `grep -E` and `fgrep` is `grep -F`. The binaries were deprecated years ago but the names stuck around.

`grep -E` enables extended regex: `+`, `?`, `|`, and parentheses without backslashes. Plain grep treats those as literal characters unless you escape them.

`grep -F` is fixed-string mode — no regex at all. It's faster and it saves you from accidentally interpreting a `.` or `[` in what you're searching. I use `-F` whenever I'm matching an exact string like an IP or a pod name."

```bash
grep -F "192.168.1.10" access.log        # literal dots, no escaping
grep -E "ERROR|FATAL" app.log           # alternation needs -E
```

**Key Point:** "egrep and fgrep are just `grep -E` and `grep -F` — use `-E` for regex features, `-F` for fast literal matching."

---

## Regular Expressions

### Q3: Explain the difference between `.*` and `.*?` — why does greedy matching bite people?

**How to Answer:**

"`.*` is greedy — it grabs as much as it can. So `<.*>` on `<b>hi</b>` matches the entire string, not just the first tag. Classic surprise.

`.*?` is lazy — it grabs as little as possible, so `<.*?>` matches `<b>` first, then `</b>` on the next pass. But note: lazy quantifiers only exist in extended regex and PCRE, not basic grep.

My rule for log parsing: avoid `.*` when the line has repeated delimiters. `[^"]*` — 'everything except a quote' — is almost always what I actually want for quoted fields."

```bash
grep -oP '"status":"[^"]*"' app.log   # exact field, no greedy spillover
```

**Key Point:** "Greedy `.*` eats too much on repeated delimiters — use lazy `.*?` or a negated class like `[^"]*` to bound it."

---

### Q4: What's the difference between basic and extended regex? Give a real example where it matters.

**How to Answer:**

"Basic regex — plain grep — treats `+`, `?`, `|`, and `()` as literal characters. Extended regex — `grep -E` — treats them as operators. Same pattern, totally different behavior.

So `grep "a+b"` matches the literal string `a+b`, while `grep -E "a+b"` matches one or more a's followed by b. This is the number-one reason a grep 'doesn't work' and people blame the log.

My habit: I default to `grep -E` because readable regex beats backslash soup. And if I ever need PCRE lookarounds, that's `grep -P`."

```bash
grep -E "timeout after [0-9]+ms" app.log | grep -Ev "after (3[0-9]|4[0-9])ms"
```

**Key Point:** "BRE treats `+?|()` as literals, ERE treats them as operators — default to `grep -E` and stop escaping everything."

---

## Sed Stream Editing

### Q5: How do you use sed in practice? Walk me through substitution.

**How to Answer:**

"`sed 's/old/new/g'` — substitute, global. That's 90% of my sed usage: fixing config files, mass-renaming, redacting secrets before sharing logs.

The delimiter doesn't have to be `/`. When I'm touching paths or URLs, I switch to `sed 's|/old/path|/new/path|g'` so I don't drown in backslashes.

`-i` edits the file in place, and I always pair it with a backup extension like `-i.bak` in production. Losing a config file to a bad sed is a rite of passage I don't want twice."

```bash
sed -i.bak 's|image: myapp:1.2|image: myapp:1.3|g' deployment.yaml
```

**Key Point:** "Substitution is `s/old/new/g` — swap delimiters when matching slashes, and always use `-i.bak` for safe in-place edits."

---

### Q6: How would you print only a range of lines with sed? When is that useful?

**How to Answer:**

"`sed -n '100,200p'` prints just lines 100 to 200, and the `-n` suppresses everything else. Without `-n` it prints the whole file plus your range again — doubling output, classic mistake.

It's my go-to when a stack trace or a log section lives at known line numbers. `grep -n` to find the line, then sed to carve out the region.

I can also match by pattern: `sed -n '/START/,/END/p'` prints between two markers. Handy for dumping one request's block out of a noisy shared log."

```bash
sed -n '/2026-09-26 21:1[0-5]/p' app.log   # everything from 21:10 to 21:15
```

**Key Point:** "`sed -n 'START,ENDp'` carves out a line or pattern range — use `-n` or the whole file prints too."
