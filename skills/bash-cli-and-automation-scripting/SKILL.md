---
name: bash-cli-and-automation-scripting
description: Build bash scripts with a real command-line interface and run them unattended — getopts option parsing (short flags, required arguments, non-option arguments), here documents for static/dynamic multi-line output, expect/autoexpect for driving other interactive programs, and the cron/systemd-timer gotchas that bite scripts the moment they leave an interactive terminal. Trigger for "parse command-line flags in bash," "how do I use getopts," "add -v/-h options to my script," "generate a file from a template in bash," "heredoc," "automate an interactive prompt," "why does my script work by hand but fail in cron," or "schedule this script." Companion skills: bash-defensive-scripting-fundamentals (core script safety), bash-text-processing-pipelines (processing the data these scripts consume/produce), bash-script-security-hardening (passwords needed by scheduled jobs).
---

# Bash CLI and Automation Scripting

A script that only ever gets run by its author, by hand, in the same terminal, doesn't need much of an
interface. A script that other people invoke with flags, that generates files for other tools, that has to
answer another program's prompts, or that runs unattended from cron needs to handle all of that
deliberately — none of it happens for free.

## Parsing options with `getopts`

`getopts` is the bash builtin for parsing short, single-dash options (`-a`, `-b value`), including
combined flags (`-ab`) the way real Unix utilities accept them — something hand-rolled `$1`/`$2`
parsing can't do cleanly.

```bash
#!/bin/bash
while getopts :a:b:c options; do
    case $options in
        a) var1=$OPTARG ;;                              # -a requires an argument, stored in OPTARG
        b) var2=$OPTARG ;;                              # -b requires an argument
        c) flag_c=1 ;;                                  # -c takes no argument
        \?) echo "Unknown option: -$OPTARG" >&2; exit 1 ;;
        :) echo "Option -$OPTARG requires an argument." >&2; exit 1 ;;
    esac
done
shift $((OPTIND - 1))                                    # drop parsed options, leaving non-option args

[[ -z $var1 ]] || echo "Option A is $var1"
[[ -n $flag_c ]] && echo "Flag C was set"
[[ -z $1 ]] || printf 'Remaining args: %s\n' "$@"
```

Key mechanics:
- The **optstring** (`a:b:c`) lists allowed option letters. A colon *after* a letter means that option
  requires an argument (delivered in `$OPTARG`); no colon means it's a bare flag.
- A **leading colon** in the optstring (`:a:b:c`) switches `getopts` into silent-error mode: instead of
  printing bash's own `illegal option --` message, it sets `$options` to `?` (unknown option, with the
  bad letter in `$OPTARG`) or `:` (known option, missing its required argument) so your `case` can
  produce a custom message and a controlled `exit`, instead of letting the script continue with
  empty variables.
- `getopts` only understands options already parsed in this invocation of the loop — non-option
  arguments (bare filenames, etc.) come *after* all the flags on the command line. `shift
  $((OPTIND - 1))` discards everything `getopts` already consumed so `$1`, `$2`, `"$@"` refer to what's
  left, exactly like a fresh invocation.
- `getopts` (bash builtin, POSIX) only does short single-letter options. Long options (`--verbose`) need
  either the separate `getopt(1)` external command (BSD/GNU implementations differ — a portability
  trap) or manual parsing of `"$@"` in a `while`/`case` loop.

## Here documents: multi-line text without a wall of `echo`

```bash
# Static text — cat << LIMIT ... LIMIT redirects everything between into cat's stdin:
cat << LICENSING
This software is released under the GNU GPL 3 license.
Anyone who uses this software must respect the terms of this license.
LICENSING
```

Variants that matter:
- `<<-LIMIT` (dash after `<<`) strips **leading tab characters** (not spaces) from each line of the
  heredoc body and from the closing limit string — lets you indent the heredoc to match the
  surrounding script's structure without the tabs leaking into the output.
- Quoting the limit string (`<<'LIMIT'` or `<<"LIMIT"`) disables variable expansion and command
  substitution inside the body — use this for literal text that happens to contain `$` or backticks.
  Leaving it unquoted (`<<LIMIT`) expands `$variables` and `$(commands)` normally, which is what
  makes dynamic generation work:

```bash
# Dynamic heredoc — variables and $(...) inside expand before cat ever sees the text,
# so this generates a real HTML file from live data:
title="System Information for $HOSTNAME"
current_date=$(date +%F)
cat <<- _system-info_ > sysinfo.html
    <title>$title</title>
    <body>
    $(uname -opr) <br>
    Updated on: $current_date
    </body>
_system-info_
```

- The closing limit string must start in column 1 with plain `<<` (no leading whitespace at all) or be
  a line of only tabs with `<<-`; a stray space before it means bash never finds the terminator and the
  heredoc swallows the rest of the script.
- Functions called inside a heredoc body run and their output is substituted in, same as any other
  command substitution — useful for building a templated report out of small, testable
  `show_uptime()`/`drive_space()`-style functions rather than one giant inline block.

## Driving other interactive programs with `expect`

When the program you need to automate is *not* your own script — a `passwd` prompt, an `ssh`
login, any CLI that insists on interactive answers — `expect` scripts it from the outside instead.

```bash
which expect || echo "install the 'expect' package first"
```

The fastest way to build an `expect` script is to record a real session with `autoexpect`, then read
and trim the result rather than hand-writing it:

```bash
autoexpect -f interactive_script.exp ./interactive_script.sh
# ... answer the prompts interactively once; autoexpect records the exchange ...
```

The generated script has its own shebang and a request/response pattern:

```expect
#!/bin/expect -f
spawn ./interactive_script.sh
expect -exact "Hello. What is your name?\r\n"
send -- "Donnie\r"
expect -exact "Donnie\r\nWhat's your favorite operating system?\r\n"
send -- "Linux\r"
expect eof
```

`spawn` starts the target program under `expect`'s control; each `expect` waits for that exact text
before `send`-ing the next response; `expect eof` waits for the child process to exit. Because
`autoexpect`-generated scripts embed the *exact* prompt text, a wording change in the target program
breaks the automation silently (it just hangs waiting for text that never arrives) — re-record after any
upstream change, and prefer `expect -re` with a regex over `-exact` for prompts likely to vary slightly
(timestamps, hostnames).

## Running unattended: cron and systemd timers

A script that works perfectly by hand can fail silently under cron because cron jobs run in a minimal,
non-interactive shell with none of your interactive login environment:

- **`PATH` is short or absent.** A cron job's `PATH` is typically just `/usr/bin:/bin` (or less) — a
  script that calls a tool installed in `/usr/local/bin` or a user's `~/bin` without an absolute path, or
  without setting `PATH` explicitly at the top of the script, works in your terminal and fails from cron.
- **No TTY, no `$DISPLAY`, no interactive prompts.** A script that calls `read` for input, needs a
  password interactively, or expects an X11 `$DISPLAY` for a GUI tool will hang or fail outright when
  cron runs it — this is exactly the case that needs the password-handling techniques in
  `bash-script-security-hardening`, not an interactive `read`.
- **Output isn't a terminal.** cron mails stdout/stderr to the job owner by default (or discards it,
  depending on config) — redirect explicitly to a log file (`>> /var/log/myjob.log 2>&1`) instead of
  relying on cron's default handling, and check the log rather than assuming silence means success.
- **Relative paths resolve against the wrong directory.** cron doesn't `cd` into the script's directory
  first — `cd "$(dirname "$0")"` or use absolute paths throughout.
- **Overlapping runs.** A long job scheduled more often than it can finish will pile up concurrent
  instances unless the script takes its own lock (`flock` on a lock file is the standard approach).

For anything more complex than "run this at this time" — retries, dependency ordering, resource
limits, or better logging — a systemd timer unit paired with a service unit is generally a more
debuggable alternative to a crontab entry (`systemctl status`/`journalctl -u` beat parsing cron's mail).

## Common pitfalls

- Forgetting the leading colon in the `getopts` optstring, then being surprised that invalid options
  print bash's generic `illegal option --` message instead of your own error handling.
- Reading non-option arguments as `$1`/`$2` without `shift $((OPTIND - 1))` first — they're still
  shifted past the options `getopts` already consumed.
- Combining `getopts`, which only handles one dash and single letters, with a manual attempt to also
  support `--long-options` in the same loop — pick one parsing strategy per script.
- Using an unquoted heredoc limit string when the body contains literal `$` or backtick characters you
  didn't mean to expand — quote the limit string.
- Indenting a heredoc body with spaces and expecting `<<-` to strip them — it only strips leading tabs.
- Hand-writing an `expect` script's exact prompt text from memory instead of recording it with
  `autoexpect`, then debugging a mismatch character by character.
- Assuming a script that runs correctly by hand will run correctly under cron without checking `PATH`,
  working directory, and whether it needs a TTY it won't have.

## Checklist

- [ ] Options parsed with `getopts` using a leading colon in the optstring, with explicit `\?`/`:` cases
      for unknown options and missing arguments
- [ ] `shift $((OPTIND - 1))` applied before touching non-option positional parameters
- [ ] Heredoc limit string quoted when literal `$`/backticks must not expand, left unquoted when
      dynamic values should be substituted
- [ ] `expect` automation recorded with `autoexpect` against the real program, not guessed by hand
- [ ] Any script destined for cron/systemd sets `PATH` (or uses absolute paths), redirects
      stdout/stderr to a log file, and avoids interactive `read`/GUI calls
- [ ] Scheduled jobs that could overlap take a lock (`flock`) instead of assuming one run finishes
      before the next starts
