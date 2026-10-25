---
name: bash-defensive-scripting-fundamentals
description: Write bash scripts that fail loudly instead of silently corrupting data — set -euo pipefail, quoting discipline, trap-based cleanup, proper test constructs ([[ ]] vs [ ]), array handling, and command substitution. Trigger for "write a bash script that," "make this shell script more robust," "why did my script keep going after a command failed," "how do I clean up a temp file even if the script errors," "quoting in bash," "set -e gotchas," "shellcheck warns about unquoted variable," or any request to author, review, or harden a new .sh script's core structure. Companion skills: bash-script-debugging-and-portability (diagnosing existing bugs), bash-script-security-hardening (untrusted input / privilege boundaries), bash-cli-and-automation-scripting (argument parsing, scheduling).
---

# Bash Defensive Scripting Fundamentals

A shell script that "works when I tested it" and a shell script that's safe to run unattended are
different things. Bash's defaults favor keep-going-no-matter-what: a failed command in the middle
of a pipeline doesn't stop the script, an unset variable silently expands to nothing, and an
unquoted variable gets word-split and glob-expanded by the shell before your command ever sees it.
Every technique here exists to turn one of those silent defaults into a loud, early failure.

## Start every script with a strict-mode header

```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'
```

- **`set -e`** — exit immediately if any command returns non-zero, instead of plowing ahead with
  stale state. Exceptions you must know or `-e` will surprise you: a command's exit status is
  ignored when it's the condition of an `if`/`while`/`until`, part of `&&`/`||`, or (critically)
  **every command in a pipeline except the last** unless `pipefail` is also set.
- **`set -u`** (a.k.a. `nounset`) — reference an unset variable and the script exits with an error
  instead of silently substituting an empty string. This catches typo'd variable names
  (`$fiel` instead of `$file`) that would otherwise run with the wrong value.
- **`set -o pipefail`** — makes a pipeline (`cmd1 | cmd2 | cmd3`) return the exit status of the
  *first* failing command instead of always returning cmd3's status. Without it, `grep pattern
  huge.log | sort | uniq -c` reports success even if `grep` couldn't read the file at all.
- **`IFS=$'\n\t'`** — narrows word-splitting to newlines and tabs only, so a filename containing a
  space doesn't get split into two words during unquoted expansion or `for x in $list`. This is a
  belt-and-suspenders habit on top of quoting, not a substitute for it.

None of `-e`/`-u`/`pipefail` protects against a command whose failure you actually want to allow —
add `|| true`, or an explicit `if`, at that specific call site instead of dropping strict mode for
the whole script.

```bash
# Command that's allowed to fail — don't let it kill the whole script:
rm -f "$lockfile" || true

# A pipeline where you deliberately want the earlier stage's failure ignored:
set +o pipefail
grep pattern "$file" | head -1
set -o pipefail
```

## Quote every variable expansion, every time

Unquoted `$var` is subject to word-splitting (on `$IFS`) and pathname expansion (globbing) *before*
the command runs. This is the single most common source of "works on my machine, breaks on files
with spaces" bugs.

```bash
# Wrong — breaks the moment $file contains a space, or expands to multiple files via globbing:
if [ -f $file ]; then
    cp $file $dest
fi

# Right — quoting disables both word-splitting and globbing for this expansion:
if [ -f "$file" ]; then
    cp "$file" "$dest"
fi
```

Rules of thumb:
- Quote every `$variable` and every `$(command substitution)` unless you specifically want
  word-splitting (rare — usually only when deliberately building a word list from unquoted output).
- `"${array[@]}"` (quoted, with `@`) preserves each array element as a separate word even if an
  element contains spaces; `${array[*]}` joins everything into one string using `$IFS` — pick `@`
  unless you specifically want the joined form.
- Quote `$@` as `"$@"` to pass positional parameters through to another command as separate
  arguments, exactly as received — `$@` unquoted or `"$*"` both mangle multi-word arguments.

## Prefer `[[ ]]` over `[ ]` for tests in bash

```bash
[[ -d "$dir" ]] && echo "exists"      # bash builtin, no subshell, supports pattern/regex matching
[ -d "$dir" ] && echo "exists"        # POSIX test — works everywhere, but has quoting sharp edges
```

`[[ ]]` doesn't word-split or glob-expand what's inside it, so `[[ $string == *.txt ]]` glob-matches
safely without quoting `$string`, and `[[ $string =~ ^[0-9]+$ ]]` gives you regex matching directly.
`[ ]` requires careful quoting of every operand (`[ "$string" = "$other" ]`) and can't do regex
matching at all. The trade-off: `[[ ]]` is a bash/ksh/zsh extension, not POSIX — if the script must
run under `dash`/`sh`, use `[ ]` instead (see `bash-script-debugging-and-portability`).

```bash
# Numeric comparison operators, same in both — never use ==/< for numbers, they compare strings:
[[ $count -eq 0 ]]      # equal
[[ $count -ne 0 ]]      # not equal
[[ $count -lt 10 ]]     # less than
[[ $count -ge 10 ]]     # greater or equal

# String comparison:
[[ $name == "root" ]]
[[ -z $name ]]           # true if empty
[[ -n $name ]]           # true if non-empty
```

## `trap` for cleanup — the only reliable way to run code on exit

A script can end via normal completion, `exit`, an error under `set -e`, or a signal (Ctrl-C,
`kill`). `trap ... EXIT` runs your cleanup code in every one of those cases, which manual "delete
the temp file at the end of the script" code does not — that code never runs if something earlier
exits early.

```bash
tmpfile=$(mktemp) || exit 1
trap 'rm -f "$tmpfile"' EXIT

# ... script body uses $tmpfile, may exit early on error at any point ...
# the trap still fires and removes it.
```

For multi-step cleanup, or to react differently to a signal versus normal exit, register on the
specific signals too:

```bash
cleanup() {
    local status=$?
    rm -f "$tmpfile"
    trap - EXIT INT TERM     # avoid double-invocation if cleanup itself triggers a signal
    exit "$status"
}
trap cleanup EXIT INT TERM
```

`trap - EXIT INT TERM` clears the trap once cleanup has run, and `$?` captured at the top of the
function preserves the original exit status so cleanup doesn't accidentally turn a failing script
into an apparently-successful one.

## Command substitution: `$( )`, never backticks

```bash
count=$(who | wc -l)          # correct — nests cleanly, escaping is straightforward
count=`who | wc -l`           # deprecated — nesting backticks requires escaping and is error-prone
```

`$( )` nests without special escaping (`$(cat "$(dirname "$0")/config")`) while backtick
substitutions require backslash-escaping any backticks or `$` inside. Always quote the result when
assigning or using it in a test: `if [[ -n "$(pgrep -f myapp)" ]]`.

## Test before you act — don't assume the happy path

```bash
[ -d graphics ] || mkdir graphics        # create it only if it's missing — idempotent, no clobber
[ -f "$outfile" ] && { echo "already done today"; exit 0; }
cd "$target_dir" || exit 1               # cd can fail (deleted dir, permissions) — always check it
```

`cd` failing silently is a classic bug: if the directory doesn't exist, the script keeps running in
the *wrong* directory and can write output or delete files somewhere unintended. Under `set -e`
this is caught automatically, but `cd ... || exit 1` documents the intent even without strict mode.

## Arrays: declare explicitly, expand with quotes

```bash
declare -a hosts=(web1 web2 "db server")   # quote elements that contain spaces
echo "${hosts[0]}"          # first element
echo "${hosts[@]}"          # all elements, unquoted here loses the multi-word element
echo "${#hosts[@]}"         # element count
for h in "${hosts[@]}"; do  # quoted @ — each element stays a single word in the loop
    echo "Host: $h"
done
```

Building an array from command output needs the same discipline:

```bash
declare -a ips
mapfile -t ips < <(grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' access.log)   # -t strips trailing newlines
```

Prefer `mapfile`/`readarray` over `ips=($(command))` — the latter still word-splits and
glob-expands the substitution before array assignment, defeating the point of using an array for
safety.

## Common pitfalls

- Writing `if [ $status -eq 0 ]` without quotes when `$status` could be empty (unset or a command
  that produced no output) — becomes `if [ -eq 0 ]`, a syntax error at runtime instead of a caught
  bug. Quote it: `[ "$status" -eq 0 ]`, or better, use `set -u` so the unset case is caught earlier.
- Assuming `set -e` catches everything — it does not fire inside `&&`/`||` chains, `if` conditions,
  or non-last pipeline stages without `pipefail`. Test the actual failure path, not just the happy
  path.
- Using `$*` where `"$@"` was needed when forwarding arguments to another command — breaks on any
  argument containing whitespace.
- Relying on end-of-script cleanup code instead of `trap ... EXIT` — any early `exit` or error skips
  it.
- Building an array with unquoted command substitution (`arr=($(cmd))`) and being surprised when a
  filename with a space becomes two array elements.
- Comparing numbers with `==`/`<`/`>` inside `[[ ]]` (those do string/glob comparison, not numeric)
  instead of `-eq`/`-lt`/`-gt`.

## Checklist

- [ ] `set -euo pipefail` at the top, with explicit `|| true` at any call site where failure is
      genuinely expected and acceptable
- [ ] Every `$variable` and `$(...)` expansion is double-quoted unless word-splitting is deliberate
- [ ] `[[ ]]` used for bash-only scripts (or `[ ]` with careful quoting if POSIX portability matters)
- [ ] Temp files/dirs created with `mktemp`/`mktemp -d` and removed via `trap ... EXIT`, not manual
      end-of-script cleanup
- [ ] `cd`, and any other command whose failure would leave the script in a bad state, is checked
      explicitly (`cd "$dir" || exit 1`) even under `set -e`
- [ ] Arrays built with `mapfile`/`readarray` or explicit quoted elements, never bare
      `arr=($(command))`
- [ ] `"$@"` (not `$@` or `$*`) used whenever forwarding all arguments to another command
