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
