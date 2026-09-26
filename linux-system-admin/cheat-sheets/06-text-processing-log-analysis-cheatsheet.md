# Text Processing & Log Analysis — Cheat Sheet

One-page dense reference. Tape it to the wall until it's muscle memory.

## grep

```bash
grep -rn "ERROR" /var/log/app/        # recursive + line numbers
grep -i "timeout" app.log             # case-insensitive
grep -C 5 "panic" app.log             # 5 lines context (use -A/-B for after/before)
grep -v "health" app.log              # invert: drop noise lines
grep -E "ERROR|FATAL" app.log         # extended regex (alternation)
grep -F "192.168.1.10" access.log     # literal string, no regex, faster
grep -o "key=[^ ]*" app.log           # only the matched part
grep -oP 'id=\K\d+' app.log          # PCRE: \K drops the prefix
grep -c "WARN" app.log                # count matches
grep --include="*.log" -r "x" dir/    # limit file types
```

## regex (BRE vs ERE)

```bash
grep "a+b"        # BRE: literal "a+b"
grep -E "a+b"     # ERE: one-or-more a's + b
[a-z]            # class   [^"]*   # "not quote", repeatable
^start  end$     # anchors  \bword\b  # word boundary (-P)
.*               # greedy   .*?       # lazy (-P only)
\d+  \s+  \w+    # PCRE classes (-P)
(?<=k=)v         # lookbehind: match v after k= (-P)
```

## sed

```bash
sed 's/old/new/g' file                # substitute, global
sed 's|/a/b|/c/d|g' file              # alternate delimiter for slashes
sed -i.bak 's/x/y/g' file             # in-place WITH backup
sed -n '100,200p' file                # print lines 100-200 only
sed -n '/START/,/END/p' file          # print between patterns
sed -n '/21:1[0-5]/p' app.log         # regex line range
sed '/^$/d' file                      # delete blank lines
sed -e 's/a/b/g' -e 's/c/d/g' file    # multiple expressions
```

## awk

```bash
awk '{print $1, $7}' file             # columns 1 and 7 ($0 = whole line)
awk -F',' '{print $2}' file.csv       # comma delimiter
awk -F'[][]' '{print $2}' file        # bracket delimiter
awk '$9 >= 500 {print $7, $9}' log    # filter + print
awk '{s+=$NF} END {print s/NR}' log   # average of last column
awk '{c[$2]++} END {for (k in c) print c[k], k}' file  # histogram
awk 'NR<=10' file                     # first 10 lines (like head)
awk 'length > 200' file               # long lines only
```

## sort / uniq / cut / tr

```bash
sort -rn file        # numeric, reverse   sort -u  # dedupe
sort -t: -k3 -n      # sort by 3rd colon-field numerically
uniq -c | sort -rn   # count + rank (needs sorted input)
cut -d' ' -f1,7      # fields 1,7 space-delimited
tr 'A-Z' 'a-z' < f   # lowercase   tr -d '\r' < f  # strip CR
```

## journalctl

```bash
journalctl -u nginx.service            # logs for one unit
journalctl -u a -u b -f               # follow multiple units
journalctl -p err --since today       # errors only, today
journalctl --since "30 min ago" -u app # time-bounded
journalctl -k                         # kernel messages (OOM lives here)
journalctl --disk-usage               # journal size
dmesg -T                              # kernel ring buffer, human time
```

## kubectl logs (multi-pod)

```bash
kubectl logs -f -l app=api                    # follow all pods behind label
kubectl logs -l app=api --all-containers      # include sidecars
kubectl logs deploy/api --since=15m --tail=200
kubectl logs -f pod/x --prefix=true           # prefix pod names
stern -l app=api -t                           # multiplexed + timestamps
```

## the ranking idiom

```bash
<extract> | sort | uniq -c | sort -rn | head -10
# slowest endpoints:
awk '{print $NF, $7}' access.log | sort -rn | head -10
# error counts by component:
grep ERROR app.log | awk -F'[][]' '{print $2}' | sort | uniq -c | sort -rn
```
