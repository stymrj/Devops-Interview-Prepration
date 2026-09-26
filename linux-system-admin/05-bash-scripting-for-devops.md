# Bash Scripting for DevOps Interview Preparation Guide

*How to Answer Bash Scripting Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Script Safety and Strict Mode](#script-safety-and-strict-mode)
2. [Variables Quoting and Expansion](#variables-quoting-and-expansion)
3. [Conditionals and Exit Codes](#conditionals-and-exit-codes)
4. [Loops and File Processing](#loops-and-file-processing)
5. [Functions and Idempotent Scripts](#functions-and-idempotent-scripts)
6. [Debugging and Production Habits](#debugging-and-production-habits)

---

## Script Safety and Strict Mode

### Q1: What's the first thing you put at the top of a production bash script?

**How to Answer:**

"`#!/usr/bin/env bash` followed by `set -euo pipefail`. That's non-negotiable for me in production.

`set -e` exits on the first failing command so a broken step can't silently cascade into a disaster. `set -u` treats unset variables as errors — it catches typos like `$REIGON` before they wipe the wrong bucket. `set -o pipefail` makes a pipeline fail if any stage fails, not just the last one.

The interview trap is knowing what each flag does individually, because people memorize the line as a magic incantation. I can tell you: without pipefail, `deploy.sh | tee log.txt` reports success even when the deploy blew up."

```bash
#!/usr/bin/env bash
set -euo pipefail

REGION="${AWS_REGION:?AWS_REGION must be set}"
```

**Key Point:** "`set -euo pipefail` at the top of every production script — exit on failure, catch unset variables, and fail pipelines honestly."

### Q2: `set -e` doesn't always do what people expect. When does it fail silently?

**How to Answer:**

"`set -e` ignores failures in three places: conditions, `&&`/`||` lists, and anything inside `if`, `while`, or `until` tests. So `if ! deploy; then ...` is fine — the failure is handled, not fatal.

The classic production gotcha is command substitution: `RESULT=$(fetch_token)` — if `fetch_token` fails under `set -e`, bash versions behave inconsistently about whether the script dies. Older bash let it slide silently, which is how people end up deploying with an empty token.

My rule: never trust a substituted value without checking it. Append `|| exit 1` or use the `${VAR:?message}` guard from Q1 right after the assignment."

**Key Point:** "`set -e` doesn't fire inside conditions or `&&` lists — and command substitution failures are inconsistent across bash versions, so check substituted values explicitly."

---

## Variables Quoting and Expansion

### Q3: Double quotes vs single quotes vs no quotes — when does it actually bite you?

**How to Answer:**

"Double quotes allow expansion, single quotes are fully literal, and unquoted is where scripts die. The interview answer they want: always quote variable expansions unless you have a reason not to.

Unquoted `$FILES` word-splits on spaces and glob-expands wildcards. So `rm -rf $DIR/*` with a `DIR` containing a space becomes two different arguments — that's how you delete the wrong directory.

Single quotes are for literal strings — regex patterns, awk programs, things that must reach the tool untouched. Double quotes everywhere else. This one habit prevents an entire class of production incidents."

```bash
tar -czf "$BACKUP_NAME" "$SRC_DIR"   # quoted: spaces and globs stay safe
```

**Key Point:** "Quote every expansion by default. Unquoted variables split on spaces and expand globs — that's how wrong directories get deleted."

### Q4: How do you handle defaults, required values, and empty variables in one line?

**How to Answer:**

"Bash parameter expansion does all of it without if-statements. `${PORT:-8080}` means 'use PORT, or 8080 if it's unset or empty'. `${PORT-8080}` is the same but only when unset — subtle difference, worth knowing.

For required values, `${AWS_REGION:?Region is required}` prints my message and exits non-zero if it's unset or empty. I use this at the top of every deploy script instead of writing validation blocks.

The trap interviewers set: `:-` vs `-`. The colon version also catches empty strings, which is usually what you want — someone explicitly exporting `PORT=""` almost never means 'use port nothing'."

**Key Point:** "`${VAR:-default}` for defaults, `${VAR:?message}` to fail fast on missing values — one line replaces a whole validation block."

---

## Conditionals and Exit Codes

### Q5: `[[` vs `[` vs `((` — which do you use and why?

**How to Answer:**

"`[[` is always my choice inside bash. It's a shell keyword, so quoting is safer, `&&` and `||` work inside it, and it supports regex with `=~` and pattern matching with `==`.

`[` is the old test command — it needs careful quoting and trips people up with `-a`/`-o`. I only use it when a script must be POSIX `sh` compatible, which is rare for me now.

`((` is for arithmetic: `(( count++ ))`, `(( port > 1024 ))`. Using string comparison on numbers is a classic bug — `[ "10" \< "9" ]` is true as a string. That one has failed more health checks than I'd like to admit."

```bash
if [[ "$HTTP_CODE" =~ ^2[0-9]{2}$ ]]; then
  echo "healthy"
fi
```

**Key Point:** "`[[` for string/file tests with regex support, `((` for arithmetic — never compare numbers as strings."

### Q6: How do you make a script clean up after itself when it fails?

**How to Answer:**

"`trap` with an EXIT handler. Whatever kills the script — error, Ctrl-C, normal finish — the trap fires and cleans up temp files, releases locks, and removes the half-written artifact.

I always pair it with a temp dir from `mktemp -d` rather than hardcoded `/tmp/myscript`. Two runs on the same box shouldn't step on each other.

The detail interviewers probe: trap on EXIT, not just ERR, because EXIT catches everything including signals. And keep the cleanup function idempotent — if the temp dir was never created, removing it shouldn't error and trigger the trap recursively."

```bash
TMPDIR="$(mktemp -d)"
cleanup() { rm -rf "$TMPDIR"; }
trap cleanup EXIT
```

**Key Point:** "Trap on EXIT with an idempotent cleanup function — temp dirs, lock files, and partial artifacts never outlive the script."

---

## Loops and File Processing

### Q7: What's the safe way to loop over lines of a file?

**How to Answer:**

"`while IFS= read -r line` fed by input redirection. That's the only pattern I trust in production.

`IFS=` preserves leading and trailing whitespace, `-r` stops backslashes from being eaten, and the `< file` redirection keeps the loop in the current shell — unlike a pipe, which runs the loop in a subshell where variable assignments vanish.

The trap: `cat file | while read line; do count=$((count+1)); done` leaves `count` at zero after the loop because the pipe forked a subshell. I've debugged this exact bug in someone's log-parsing cron job. Redirection in, never pipe in."

```bash
while IFS= read -r line; do
  [[ "$line" =~ ^# ]] && continue
  process "$line"
done < servers.txt
```

**Key Point:** "`while IFS= read -r line ... done < file` — redirection, not pipes, so variables survive the loop and whitespace stays intact."

### Q8: When would you use a for loop over files vs find with -exec?

**How to Answer:**

"For a handful of files with simple names, `for f in *.log` is fine and readable. But the moment filenames can contain spaces or the count is large, I reach for `find`.

`find ... -print0 | xargs -0` handles any filename safely because null is the only character that can't appear in a path. `find ... -exec cmd {} +` batches arguments like xargs does, built in.

The interview point: `for f in $(ls)` is broken by design — command substitution word-splits. And `for f in *.log` silently runs once with the literal pattern if the glob matches nothing, unless `nullglob` is set. These are the bugs that delete the wrong files at 2 AM."

**Key Point:** "`for f in *.log` is fine for simple cases; `find -print0 | xargs -0` for anything untrusted or large — never `for f in $(ls)`."

---

## Functions and Idempotent Scripts

### Q9: How do you structure functions in a longer bash script?

**How to Answer:**

"Named functions with `local` for every variable inside. Without `local`, function variables leak into the global scope and two functions silently overwrite each other's state — nightmare to debug.

I keep the script as: strict mode, constants, helper functions, one `main` function, then `main "$@"` at the bottom. That gives a clear entry point and makes the file sourceable for testing.

Functions return exit codes, not values — capture stdout for data with `$(...)`. And one honest caveat I give in interviews: bash functions are great for orchestration glue, but if I'm writing real data structures I switch to Python. Knowing that boundary is part of the senior answer."

```bash
get_instance_id() {
  local tag="$1"
  aws ec2 describe-instances --filters "Name=tag:Name,Values=$tag" \
    --query 'Reservations[0].Instances[0].InstanceId' --output text
}
```

**Key Point:** "`local` every function variable, one `main "$@"` entry point — and know when the script has outgrown bash and belongs in Python."

### Q10: What makes a script idempotent, and why do interviewers care so much?

**How to Answer:**

"Idempotent means running it twice gives the same result as running it once. Interviewers care because every real automation runs repeatedly — cron retries, CI reruns, Terraform-style convergence.

In bash that means: check state before acting. `mkdir -p` instead of `mkdir`, `grep -q` before appending to a config, `systemctl is-active` before restarting. Downloads use `curl -z` or checksum comparison instead of blind re-fetch.

The trap is thinking 'exit 0 on second run' is enough. True idempotency means the second run changes nothing and still reports honestly. A deploy script that re-uploads an identical artifact and calls it a success is lying about what it did — diff first, act only on drift."

**Key Point:** "Check state before acting — `mkdir -p`, `grep -q` before appending, diff before deploying. Reruns must change nothing and still report honestly."

---

## Debugging and Production Habits

### Q11: A script works on your machine but fails in CI. How do you debug it?

**How to Answer:**

"First, reproduce with a clean environment: `env -i bash script.sh` strips inherited variables, which catches half of 'works on my machine' instantly. CI runs with a near-empty environment, so a variable I exported locally is the usual suspect.

Then `bash -n script.sh` checks syntax without running anything — catches quoting and bracket mistakes cheaply. For runtime, `set -x` traces every command as it executes, and I scope it with `set -x`/`set +x` around the suspicious section instead of drowning in output.

The CI-specific traps: different bash versions (macOS ships ancient bash 3.2), missing tools in the runner image, and relative paths — CI checks out to a different working directory than my laptop. I make scripts `cd` to their own directory first with `cd \"$(dirname \"$0\")\"`."

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"   # run from the script's own directory, everywhere
```

**Key Point:** "Reproduce with `env -i`, trace with scoped `set -x`, and never assume CI's environment, bash version, or working directory matches yours."

### Q12: How do you add logging that doesn't make a script unreadable?

**How to Answer:**

"Three tiny log functions at the top — `log`, `warn`, `die` — each prefixing a timestamp and level. Then the script reads naturally: `log 'Starting deploy'`, `die 'DB unreachable'`.

`die` prints to stderr and exits non-zero in one call, which replaces the `echo ... >&2; exit 1` dance everywhere. All logs go through the same functions so format stays consistent.

In production I redirect the whole script's output once: `exec >> logfile 2>&1` near the top, optionally teed. The interview point: structured-enough logs with timestamps turn a 3 AM page from guesswork into reading a timeline. And never log secrets — that's how tokens end up in CI artifacts."

```bash
log()  { echo "[$(date '+%F %T')] INFO  $*"; }
warn() { echo "[$(date '+%F %T')] WARN  $*" >&2; }
die()  { echo "[$(date '+%F %T')] ERROR $*" >&2; exit 1; }
```

**Key Point:** "Three functions — `log`, `warn`, `die` — with timestamps, one output redirect, and never log secrets."

### Q13: Give me a real scenario: write the skeleton of a deploy script you'd actually ship.

**How to Answer:**

"It follows everything above: strict mode, required-variable guards, a cleanup trap, logging, and idempotency checks. The flow is validate, lock, deploy, verify, unlock — and any step can fail safely.

I add a lock file with `flock` so two deploys can't run concurrently — overlapping deploys are how you get half-old half-new fleets. After deploying, the script verifies health before declaring success, because exit 0 on an unverified deploy is a lie.

The part interviewers love: the rollback path. If health checks fail, the script restores the previous version automatically instead of paging a human to do it manually. A deploy script without a rollback plan is just a hope with a shebang."

**Key Point:** "Validate, lock with `flock`, deploy, verify health, auto-rollback on failure — and never declare success on an unverified deploy."
