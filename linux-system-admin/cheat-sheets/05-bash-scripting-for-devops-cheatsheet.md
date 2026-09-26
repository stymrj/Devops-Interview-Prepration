# Bash Scripting for DevOps — Cheat Sheet

One-page dense reference. Tape it to the wall until it's muscle memory.

## Strict mode & shebang

```bash
#!/usr/bin/env bash
set -euo pipefail      # exit on error, unset vars fatal, pipelines fail honestly
set -x / set +x       # trace on/off (scope around the suspicious section)
bash -n script.sh     # syntax check without running
```

## Variables & expansion

```bash
"${VAR}"              # always quote expansions
${VAR:-default}       # default if unset OR empty
${VAR-default}        # default only if unset
${VAR:?message}       # exit 1 with message if unset/empty (fail fast)
${VAR:+alt}           # alt if VAR is set and non-empty
${#VAR}               # length
${VAR#prefix}         # strip shortest leading match
${VAR##prefix}        # strip longest leading match
${VAR%suffix}         # strip shortest trailing match
${VAR%%suffix}        # strip longest trailing match
${VAR/old/new}        # replace first; ${VAR//old/new} replace all
"${@:2}"              # args from $2 on; "$#" count; "$?" last exit; "$$" pid
```

## Tests — `[[ ]]` strings/files, `(( ))` arithmetic

```bash
[[ -f f ]] [[ -d d ]] [[ -x f ]] [[ -s f ]]   # file / dir / executable / non-empty
[[ -z "$s" ]] [[ -n "$s" ]]                    # empty / non-empty
[[ "$a" == "$b" ]]  [[ "$a" != "$b" ]]         # equality (== supports globs)
[[ "$c" =~ ^2[0-9]{2}$ ]]                      # regex match; groups in ${BASH_REMATCH[1]}
[[ -v VAR ]]                                   # is VAR set (bash 4.2+)
(( n > 1024 ))  (( count++ ))  (( total += n ))# arithmetic — no $ inside
```

## Loops & iteration

```bash
while IFS= read -r line; do ...; done < file   # safe line loop (no subshell)
for f in *.log; do ...; done                   # glob loop; set nullglob for empty
for i in {1..10}; do ...; done                 # brace range
for ((i=0; i<n; i++)); do ...; done            # C-style
find . -name '*.log' -print0 | xargs -0 rm     # filenames with spaces, safe
find . -type f -mtime +30 -delete              # older than 30 days
```

## Functions & structure

```bash
name() { local x="$1"; ...; }   # local every function variable
main() { ...; }; main "$@"      # single entry point at the bottom
return 0                        # exit code only — use stdout for data
result="$(get_id "$tag")"       # capture stdout, never "return" a string
```

## Error handling & cleanup

```bash
TMPDIR="$(mktemp -d)"; trap 'rm -rf "$TMPDIR"' EXIT   # always clean up
trap 'die "failed"' ERR                                # hook on any error
cmd || die "cmd failed"          # explicit check (works under set -e)
exec 200>/var/lock/app.lock; flock -n 200 || exit 1   # no overlapping runs
```

## Logging pattern

```bash
log()  { echo "[$(date '+%F %T')] INFO  $*"; }
warn() { echo "[$(date '+%F %T')] WARN  $*" >&2; }
die()  { echo "[$(date '+%F %T')] ERROR $*" >&2; exit 1; }
exec >> /var/log/app.log 2>&1   # redirect everything once (add | tee -a to also print)
```

## Debugging CI failures

```bash
env -i bash script.sh           # reproduce CI's empty environment
cd "$(dirname "$0")"            # run from the script's own dir
command -v jq || die "jq missing"  # assert dependencies up front
```

## Quick gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Pipeline "succeeds" but step failed | `pipefail` off | `set -o pipefail` |
| Variable empty after piped loop | loop ran in subshell | `done < file`, not `cat file \|` |
| `[ "10" < "9" ]` true | string comparison on numbers | `(( 10 > 9 ))` |
| Script deletes wrong dir | unquoted `$DIR` word-split | quote every expansion |
| Works locally, fails in CI | inherited env / old bash / cwd | `env -i`, `bash -n`, `cd $(dirname)` |
| Second cron run duplicates config | not idempotent | `grep -q` before append, `mkdir -p` |
