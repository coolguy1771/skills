---
name: bash-script-debugging-and-portability
description: Diagnose a bash script that's misbehaving — xtrace (set -x / PS4), targeted set -e / set -u instrumentation, the bash debugger (bashdb) — and write scripts that also run correctly under a POSIX /bin/sh (dash, POSIX mode ksh) or on a different Unix. Trigger for "why isn't my script doing what I expect," "my loop never terminates," "set -e isn't catching this," "debug this shell script," "make this script POSIX-compliant," "will this run on macOS/FreeBSD/dash," "checkbashisms," "shellcheck," or "this works on my machine but not on the deploy target." Companion skills: bash-defensive-scripting-fundamentals (the strict-mode header this diagnoses failures in), bash-script-security-hardening, bash-cli-and-automation-scripting.
---

# Bash Script Debugging and Portability

Two separate but related problems: figuring out why a script that's already written isn't doing what
you expect (debugging), and writing a script that behaves identically on every shell/OS it might run on
(portability). Both come down to not trusting your mental model of what bash "obviously" does, and
instead watching what it actually does.

## `set -x` / xtrace: watch every command as it runs

```bash
#!/bin/bash
set -x                     # turn tracing on from here...
risky_function
set +x                     # ...off again, to avoid drowning in noise for the rest of the script
```

Or trace only a specific run without touching the script:

```bash
bash -x ./myscript.sh
```

Every command bash executes gets printed to stderr, prefixed with `+`, *after* variable expansion —
so you see the actual arguments a command received, not the literal source line. This is usually faster
than sprinkling `echo` statements through a script, and it survives `case`/`if` branches you didn't
think to instrument. Customize the trace prefix with `PS4` when tracing something with function calls
or loops, to see *where* each traced line came from:

```bash
PS4='+ ${BASH_SOURCE}:${LINENO}:${FUNCNAME[0]:-main}: '
set -x
```

## Catching unset variables and silent command failures

```bash
set -u    # error immediately on reference to an unset variable, instead of substituting empty string
set -e    # exit immediately if a command exits non-zero, instead of continuing with stale state
```

Both are covered as a strict-mode baseline in `bash-defensive-scripting-fundamentals`; the debugging
angle is knowing exactly where `set -e` does *not* apply, because "why didn't `set -e` catch that" is one
of the most common bash debugging dead ends:

- A command's exit status is ignored under `-e` when it's the condition of `if`, `while`, or `until`.
- A command on the left side of `&&` or `||` doesn't trigger `-e` — the whole compound expression's
  final status is what matters.
- Every command in a pipeline **except the last** is exempt from `-e` unless `set -o pipefail` is also
  active — `grep pattern bad_file.txt | sort` can have `grep` fail to find the file while `sort` still
  exits 0 on empty input, and the script sails on.
- A function's internal failure doesn't propagate if the function's return value isn't itself checked at
  the call site, or if the call site is inside one of the exempt contexts above.

When a script seems to keep running past a command that should have killed it, check those four cases
before assuming `set -e` is broken — it almost never is; the failure is usually in one of those contexts.

## The bash debugger (`bashdb`)

For anything more than a handful of suspect lines, stepping through with an actual debugger beats
trace-and-guess:

```bash
bashdb ./myscript.sh
```

`bashdb` supports breakpoints, single-stepping, and inspecting variables mid-execution the way `gdb`
does for compiled code — the advantage over `set -x` is that you don't have to modify the script at all,
and you can pause at a specific line and inspect state interactively instead of scrolling through a full
trace looking for the moment things went wrong.

## Common categories of bash bugs worth checking first

- **Unquoted test comparisons that become syntax errors**: `[ $number = 1 ]` when `$number` is unset
  or empty collapses to `[ = 1 ]`, a syntax error at runtime rather than a caught condition. Quote it:
  `[ "$number" = 1 ]`, and pair with `set -u` so the unset case surfaces even earlier.
- **A loop counter or condition that never becomes false** ("wild loop") — usually an increment that
  got dropped, or a comparison operator that's backwards. Trace the loop variable specifically with a
  targeted `echo`/`set -x` rather than the whole script.
- **`cut`/`read` piped from a command whose output the shell can't see afterward**: `some_command |
  read var` runs `read` in a *subshell* created by the pipeline, so `$var` is empty once the pipeline
  ends and control returns to the parent shell. Use process substitution (`read var < <(some_command)`)
  or a `while read` loop structured so the assignment happens in the current shell, not a piped-away
  subshell.
- **A `[[ $? == 0 ]]` check placed after something that already changed `$?`** — `$?` reflects the *most
  recent* command's exit status; anything between the command you care about and the check
  (including another `[` test) overwrites it. Capture it immediately: `status=$?`.

## POSIX portability: don't assume `/bin/sh` means bash

`#!/bin/sh` on Linux usually points to `dash`, not bash; on other Unix systems it can be an entirely
different, more limited shell. A script that relies on bash extensions ("bashisms") while declaring
`#!/bin/sh` will fail — sometimes loudly, sometimes by silently doing the wrong thing — on any system
where `sh` isn't secretly bash.

Bashisms to watch for when POSIX portability matters:

```bash
[[ $x == y ]]        # bashism — POSIX sh needs [ "$x" = "y" ] (single =, single brackets)
arr=(one two three)  # bashism — POSIX sh has no indexed arrays at all
echo -e "a\tb"        # portability trap — echo's flag/escape handling differs across shells;
                       # use printf '%s\n' instead for output you need to be identical everywhere
local var=value       # not in POSIX sh (works in bash, dash, and most others in practice, but not guaranteed)
function foo { ... }  # bashism — POSIX form is: foo() { ... }
```

Two ways to guarantee your script can always find bash rather than fighting `/bin/sh` differences:

```bash
#!/usr/bin/env bash          # searches PATH for bash instead of assuming /bin/bash exists at that path
```

or, on systems where bash isn't at a predictable path (older BSDs), create a symlink as part of setup
rather than hardcoding a path that varies by OS.

### Testing for portability before you ship

```bash
checkbashisms myscript.sh     # static check specifically for bashisms in a script claiming #!/bin/sh
shellcheck myscript.sh        # broader static analysis — quoting bugs, bashisms, common logic errors
```

`checkbashisms` and `shellcheck` are static checkers — they read the script without running it. For a
dynamic check (actually running the script under multiple real shells to see what breaks), test directly
against `dash`, `ksh`, and whatever POSIX-compliant shell is available, since a static checker can't catch
every runtime behavioral difference between implementations (e.g. `sed -i` syntax differences,
`echo` flag handling, array support entirely absent rather than just spelled differently).

```bash
dash myscript.sh    # dash is one of the most strictly POSIX-compliant shells commonly available —
                     # a script that runs cleanly under dash is a strong portability signal
```

## Common pitfalls

- Assuming `set -e` should have caught a failure that occurred inside an `if` condition, a `&&`/`||`
  chain, or a non-last pipeline stage without `pipefail` — check those four exempt contexts first.
- Piping into `read` (`cmd | read var`) and being confused why the variable is empty afterward — it's a
  subshell scoping issue, not a `read` bug.
- Shipping `#!/bin/sh` while using `[[ ]]`, arrays, or `local` — these silently fail or error out the
  moment `/bin/sh` isn't secretly bash on the target system.
- Trusting `echo` to behave identically across shells for anything involving escape sequences or a
  `-n`/`-e` flag — use `printf` when the output must be portable.
- Relying only on a static checker (`shellcheck`/`checkbashisms`) and never actually running the script
  under the target shell (`dash`, real BSD `sh`) before calling it portable.
- Debugging by adding `echo` statements everywhere instead of `set -x`/`PS4` or `bashdb`, and having
  to remove them all afterward (and occasionally leaving one behind).

## Checklist

- [ ] `set -x` (or `bash -x script.sh`) used to see actual expanded commands before adding manual
      `echo` instrumentation
- [ ] Every "why didn't `set -e` catch this" question checked against: `if` condition, `&&`/`||` chain,
      non-last pipeline stage without `pipefail`, or unchecked function return
- [ ] `$?` captured into a variable immediately after the command it refers to, never read after an
      intervening command
- [ ] Scripts declaring `#!/bin/sh` contain no `[[ ]]`, arrays, `local`, or other bash-only syntax —
      verified with `checkbashisms` and by actually running under `dash`
- [ ] `printf` used instead of `echo` wherever output must be byte-identical across shells
- [ ] `shellcheck` run on the script before considering it done
