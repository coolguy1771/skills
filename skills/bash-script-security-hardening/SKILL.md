---
name: bash-script-security-hardening
description: Harden bash scripts against the ways they leak secrets or grant more access than intended — safe temp file creation with mktemp, avoiding embedded plaintext passwords, understanding SUID/SGID on scripts, restricting who can read/execute a script via permissions and ACLs, and the command-injection risk of eval and unsanitized input. Trigger for "how do I store a password in a script safely," "temp file security," "is eval dangerous," "restrict who can run/read this script," "this script needs root but I don't want everyone to see the password," "command injection in bash," or any script that touches untrusted input, credentials, or runs with elevated privileges. Companion skills: bash-defensive-scripting-fundamentals (quoting is the first defense against injection), bash-script-debugging-and-portability.
---

# Bash Script Security Hardening

A script that works is not the same as a script that's safe to leave lying around on a shared system,
run with elevated privileges, or hand to someone with less trust than you have. The failure modes here
are quieter than a crashing script — a predictable temp filename, a password grep-able in plaintext, or
an `eval` fed user input can sit unnoticed for years before someone exploits it.

## Temporary files: predictable names and races are the vulnerability

```bash
# WRONG — predictable name in a world-writable directory, and no atomicity:
tmpfile="/tmp/myscript_$$"
echo "sensitive data" > "$tmpfile"
```

That pattern has two separate problems: `/tmp` is world-writable, so another local user can
pre-create `/tmp/myscript_<pid>` as a symlink to a file they want overwritten (a classic symlink
attack), and the predictable name means an attacker doesn't even need to race you — they can just
guess it ahead of time.

```bash
# RIGHT — mktemp creates the file atomically with a random name and 0600 permissions, and fails
# loudly instead of silently if it can't:
tmpfile=$(mktemp) || exit 1
trap 'rm -f "$tmpfile"' EXIT

tmpdir=$(mktemp -d) || exit 1
trap 'rm -rf "$tmpdir"' EXIT
```

`mktemp` guarantees the file didn't already exist when it created it (no race window for another
process to have gotten there first) and sets restrictive permissions on it by default — always prefer it
over constructing a temp path by hand, and always pair it with a `trap ... EXIT` cleanup (see
`bash-defensive-scripting-fundamentals`) so the file doesn't outlive the script even on an early exit.

## Passwords embedded in scripts

Two situations force a password into a script: an unattended job (cron/systemd timer) that can't stop
to prompt, or a wrapper script that needs a service credential without exposing it to the humans
running the wrapper. Neither justifies a plaintext password sitting in the script body.

```bash
# A partial mitigation, not a full solution — encrypt the password at rest with openssl,
# decrypt it only in memory at the moment it's needed:
encrypted_pw=$(cat encrypted_password.bin)
password=$(openssl enc -d -aes-256-cbc -pbkdf2 -in encrypted_password.bin -pass pass:"$key")
sshpass -p "$password" ssh user@remote-host
```

This is *layered* protection, not a guarantee — whatever key decrypts the password has to live
somewhere too, and if that key is also readable by whoever can read the script, you've mostly just
added an extra step for an attacker rather than closed the hole. The more robust answers are outside
plain bash entirely: a secrets manager (Vault, cloud KMS, `pass`), SSH key-based auth instead of
password auth so there's no password to embed at all, or a service account whose credential is
injected by the orchestration layer (systemd `LoadCredential=`, a CI secret store) rather than
committed anywhere on disk.

If you do keep a secret in a file, lock down who can read it *and* the directory it lives in:

```bash
chmod 700 "$HOME"                 # the containing directory must also deny access, not just the file
chmod 600 secret_config.sh        # owner read/write only, nobody else
```

A `chmod 600` on the file alone isn't sufficient if the parent directory is world-readable/searchable in
a way that leaks the filename or allows it to be copied elsewhere before permissions are checked —
harden the whole path, not just the leaf file.

## Restricting who can read or run a script

```bash
chmod 750 admin_script.sh          # owner rwx, group rx, others nothing
chown root:admins admin_script.sh  # pair with group ownership so "group" means the right people
```

For finer-grained access than the standard owner/group/other model allows (one specific user needs
execute but not read, for example), use an ACL instead of trying to force it through Unix permission
bits and group membership alone:

```bash
setfacl -m u:horatio:r-x admin_script.sh   # grant horatio specifically read+execute
getfacl admin_script.sh                     # verify what's actually granted
```

Compiling a script to a binary with `shc` obscures the source from casual reading (`strings`/opening it
in an editor no longer shows the plaintext) but is not real protection against a determined attacker —
`strace`/`dtrace`/`truss` can trace the compiled binary's actual system calls and reveal embedded
secrets or logic at runtime regardless of whether the source text is visible. Treat `shc` as
obfuscation, not encryption, and never let it be the only thing standing between a script and a secret
it contains.

## SUID/SGID on scripts: understand before you set

```bash
chmod u+s /path/to/program   # SUID — runs with the FILE OWNER's privileges, not the invoking user's
chmod g+s /path/to/program   # SGID — runs with the FILE's GROUP privileges
```

Setting SUID root on a *shell script* (as opposed to a compiled binary) is widely considered unsafe on
modern Linux — the kernel deliberately ignores the SUID bit on scripts on many systems precisely
because of the window between the kernel opening the interpreter and the interpreter opening the
script, which historically allowed the target of execution to be swapped out from under it. If a task
genuinely needs privilege escalation, prefer `sudo` with a tightly scoped `/etc/sudoers` entry (a specific
command, not blanket `ALL`) over trying to make a script SUID:

```
# /etc/sudoers — Horatio may run this ONE command as root, nothing else:
horatio ALL=(root) /usr/local/bin/specific_backup_script.sh
```

The three `ALL`s in a raw `horatio ALL=(ALL:ALL) ALL` sudoers line mean "any host, as any user, as
any group, running any command" — that's effectively full root for Horatio, not a scoped grant. Narrow
every field explicitly for anything beyond a fully trusted administrator.

## `eval` and command injection

```bash
# DANGEROUS — if $user_input can be influenced by anyone else, this executes arbitrary commands:
eval "$user_input"
```

`eval` re-parses its argument as a brand-new command line — any shell metacharacter in the string
(`;`, `|`, `` ` ``, `$(...)`, `&&`) is interpreted, not treated as literal text. An input like
`"; rm -rf ~ #"` doesn't get passed as data, it gets *executed*. The same risk applies more subtly to
any construct that re-interprets a variable as code or as a filename pattern the shell expands —
unquoted variable expansion feeding into a command is a milder version of the same class of bug (see
the quoting discipline in `bash-defensive-scripting-fundamentals`).

If a script's design seems to require `eval`, look for a way to avoid it first — arrays for building
dynamic argument lists, `case` statements for dynamic dispatch instead of building command strings.
When `eval` is genuinely unavoidable, validate the input against a strict allowlist (known-safe
characters or known-safe values only) before it ever reaches `eval`, and never trust "it's just an
internal tool" as a reason to skip validation — internal tools get copy-pasted into contexts that accept
untrusted input more often than anyone plans for.

## Common pitfalls

- Building a temp filename by hand (`/tmp/script_$$` or similar) instead of `mktemp` — predictable
  names in a shared, world-writable directory are a symlink-attack vector, not just an aesthetic issue.
- Storing a plaintext password directly in a script's source rather than even the partial mitigation of
  encrypting it at rest.
- `chmod 600` on a secrets file while leaving its parent directory world-readable or world-searchable.
- Reaching for SUID on a shell script to solve a "needs more privilege" problem instead of a scoped
  `sudoers` entry — many systems ignore the SUID bit on scripts anyway, so it may not even work.
- Writing a `sudoers` entry with `ALL=(ALL:ALL) ALL` when the actual need is "run this one command as
  root" — that grants far more than intended.
- Treating `shc`-compiled scripts as secure storage for secrets — it obscures, it doesn't encrypt, and
  runtime tracing still exposes what the script does.
- Passing any string that could contain shell metacharacters to `eval` without first validating it
  against a strict allowlist.

## Checklist

- [ ] All temp files/dirs created with `mktemp`/`mktemp -d`, never a hand-built path, and cleaned up
      via `trap ... EXIT`
- [ ] No plaintext password committed to a script; anything embedded is at minimum encrypted at rest,
      ideally replaced with key-based auth or a real secrets manager
- [ ] Permissions restricted on both the secrets file and its containing directory, not just the file
- [ ] Access to sensitive scripts scoped with `chmod`/`chown` and, where per-user granularity is
      needed, ACLs (`setfacl`) rather than loosening group permissions for everyone
- [ ] Privilege escalation done via a narrowly scoped `sudoers` entry (specific command, specific user),
      not a SUID bit on a script or a blanket `ALL=(ALL:ALL) ALL` grant
- [ ] No `eval` (or unquoted variable used as a command) on a string influenced by external/untrusted
      input without a strict allowlist validating it first
