---
name: linux-kernel-printk-and-dynamic-debug
description: Control and debug kernel logging beyond basic pr_info() calls — printk log levels and format specifiers, where output goes (kernel log buffer / logfile / console) and the /proc/sys/kernel/printk sysctl that governs console routing, turning on KERN_DEBUG output via -DDEBUG vs the kernel's dynamic debug framework (CONFIG_DYNAMIC_DEBUG, /sys/kernel/debug/dynamic_debug/control), and printk indexing (CONFIG_PRINTK_INDEX) for tracking log-message provenance. Use when the user asks "why doesn't my pr_debug() show up," "how do I enable debug prints without rebuilding," "how do I get printk to also show on the console," "what does /proc/sys/kernel/printk mean," "how do I turn on dynamic debug for a specific file/module/function," or is instrumenting/debugging a module or driver with printk-family calls. Distinct from linux-kernel-module-basics (which covers only the basic pr_*() macro forms) — this is the full logging/debug-instrumentation toolchain.
---

# Linux Kernel printk and Dynamic Debug

`printk()` output can land in up to three places — the in-memory kernel log buffer (always), a
persistent log file (typically, via the logging daemon), and the console device (conditionally) —
and getting debug-level (`KERN_DEBUG`) messages to appear at all requires one of two distinct
mechanisms. Neither is obvious from the API surface alone; this skill covers both.

## Log levels and the pr_*() macros

```c
pr_emerg(fmt, ...)    // KERN_EMERG  (0) - system is unusable
pr_alert(fmt, ...)    // KERN_ALERT  (1) - action must be taken immediately
pr_crit(fmt, ...)     // KERN_CRIT   (2) - critical conditions
pr_err(fmt, ...)      // KERN_ERR    (3) - error conditions
pr_warn(fmt, ...)     // KERN_WARNING(4) - warning conditions
pr_notice(fmt, ...)   // KERN_NOTICE (5) - normal but significant
pr_info(fmt, ...)     // KERN_INFO   (6) - informational
pr_debug(fmt, ...)    // KERN_DEBUG  (7) - debug-level (OFF by default, see below)
```

Each expands to `printk(KERN_<LEVEL> pr_fmt(fmt), ...)`. Define `pr_fmt()` before any includes to
auto-prefix every message in the file, which is what makes kernel logs greppable per-module:

```c
#define pr_fmt(fmt) "%s:%s(): " fmt, KBUILD_MODNAME, __func__
#include <linux/init.h>
#include <linux/module.h>
```

`pr_cont(fmt, ...)` continues the *previous* printk on the same line — useful for building up one
log line across several calls; only the final `pr_cont()` in the chain should carry the trailing `\n`.

**Driver authors must use `dev_*()`, not `pr_*()`** — `dev_err(dev, fmt, ...)`, `dev_dbg(dev, fmt,
...)`, etc. They take a `struct device *` first argument, and the driver core auto-prefixes the
device name/bus/address, e.g. `ks3231 1-0068: IRQ! #5` — far more useful in a multi-device log
than a bare message.

Always terminate a `printk`/`pr_*()` format string with `\n` — a missing newline can make output
appear to not take effect at all (it gets buffered pending a continuation).

## Where output goes: console routing

```sh
cat /proc/sys/kernel/printk
4 4 1 7
```

Four integers: **current console log level**, **default level for unleveled messages**, **minimum
allowed level**, **boot-time default level**. Any message with a level *numerically less than* the
first value is also sent to the console device (in addition to always going to the log buffer). With
the default `4` (`KERN_WARNING`), `EMERG`/`ALERT`/`CRIT`/`ERR` reach the console; `WARNING` and
below do not. `KERN_EMERG` (0) always reaches the console regardless of this setting.

Force everything to the console (make printk behave like printf), at runtime, without a reboot:

```sh
sudo bash -c "echo Y > /sys/module/printk/parameters/ignore_loglevel"   # on
sudo bash -c "echo N > /sys/module/printk/parameters/ignore_loglevel"   # off
```

Or pass `ignore_loglevel` as a kernel boot parameter for the whole session. `dmesg --console-level`
and console-level kernel command-line parameters (`loglevel=n`) are the other knobs.

## Turning on KERN_DEBUG output: two mechanisms

`pr_debug()`/`dev_dbg()`/`printk(KERN_DEBUG ...)` are **compiled out and silent by default** —
unlike every other level, they only fire when `DEBUG` is defined for that translation unit.

**Mechanism 1 — compile-time, per-module, requires rebuild+reload:**

```makefile
# in the module's Makefile — uncomment one:
ccflags-y += -DDEBUG                      # whole module (preferred; EXTRA_CFLAGS is deprecated)
# CFLAGS_myfile.o := -DDEBUG              # single source file only
```

Not practical in production — every debug session means rebuild, `rmmod`, `insmod`.

**Mechanism 2 — the kernel's dynamic debug framework, no rebuild required:**

Every `KERN_DEBUG` call site in the kernel and in modules is compiled in (when
`CONFIG_DYNAMIC_DEBUG=y`) and individually toggled at runtime through a debugfs (or procfs)
control file:

```sh
grep "DYNAMIC_DEBUG=y" /boot/config-$(uname -r)     # confirm it's available
head -n1 /sys/kernel/debug/dynamic_debug/control    # or /proc/dynamic_debug/control if no debugfs
# filename:lineno [module]function flags format
```

Each call site shows its current state via `flags`: `=_` means off (the default). Turn one on by
writing `+p` (print) to the control file; `-p` turns it back off. Scope can be by file, module,
function, or line:

```sh
# all debug prints in files matching *usb*
echo -n 'file *usb* +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
echo -n 'file *usb* -p' | sudo tee /sys/kernel/debug/dynamic_debug/control

# every pr_debug()/dev_dbg() call site in every currently loaded module
echo -n 'module * +pflmt' | sudo tee /sys/kernel/debug/dynamic_debug/control

# a single named module
echo -n 'module e1000 +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
```

Watch results live in another terminal with `journalctl -k -f` (or `dmesg -w`). This is the
practical, production-safe way to get debug output out of a running system — reach for it before
reaching for mechanism 1.

## printk format specifiers worth knowing

- `%pI4` / `%pI6` — IPv4/IPv6 addresses, formatted correctly without manual byte-splitting.
- `%*ph` — raw buffer as hex bytes (`*` replaced by byte count; keep it under ~64 bytes, use
  `print_hex_dump_bytes()` for larger buffers).
- Bare `%p` for a pointer is **deprecated and a security issue** (leaks kernel addresses) — the
  kernel hashes it by default; use the specific `%pX` variant for what you're actually printing, per
  `Documentation/printk-formats.txt`.

## printk indexing (CONFIG_PRINTK_INDEX)

Solves a real production problem: a monitoring daemon greps kernel logs for a specific message
string; a later kernel/driver update silently reworks that message's wording and the daemon's
match breaks invisibly. With `CONFIG_PRINTK_INDEX=y` (off by default; `General setup > Printk
indexing debugfs interface` in menuconfig), every printk call site's format string, source location,
and log level is recorded into a `.printk_index` section and exposed per-binary:

```sh
head -n3 /sys/kernel/debug/printk/index/vmlinux
# <level/flags> filename:line function "format"
grep -Hn -i -w "fire" /sys/kernel/debug/printk/index/*     # find where a message actually lives
```

Lets a monitoring tool verify the exact message it's grepping for still exists at the expected
source location before relying on it, rather than discovering a silent break in production. Zero
overhead when disabled.

## Common pitfalls

- Using `pr_debug()`/`dev_dbg()` and concluding it's "broken" when nothing appears — it's silent
  by design until `DEBUG` is defined or dynamic debug enables that specific call site.
- Rebuilding a module with `-DDEBUG` as the go-to debug workflow in a live/production system —
  use dynamic debug instead; no rebuild, no reload, finer-grained scoping.
- Forgetting the trailing `\n` on a printk/pr_* format string.
- Using `pr_*()` in driver code instead of `dev_*()` — loses the auto-prefixed device context that
  makes multi-device logs actually diagnosable.
- Printing a raw pointer with bare `%p` — security-sensitive (kernel address disclosure) and
  deprecated; use the specific `%pX` format for the actual data type.
- Assuming `/proc/sys/kernel/printk`'s first number is a log *filter* on the log buffer — it only
  controls what additionally reaches the *console*; everything still goes to the log buffer/`dmesg`
  regardless of this setting.
- Checking for dynamic debug support by looking for `pr_debug` output directly instead of first
  confirming `CONFIG_DYNAMIC_DEBUG=y` and locating the correct control file
  (`/sys/kernel/debug/dynamic_debug/control` vs `/proc/dynamic_debug/control`, whichever exists
  on that system).

## Checklist

- [ ] Every log call site uses `pr_fmt()` for consistent per-file prefixing, and driver code uses
      `dev_*()` rather than `pr_*()`
- [ ] Every printk/pr_* format string ends in `\n`
- [ ] Debug output needed on a live system goes through dynamic debug
      (`/sys/kernel/debug/dynamic_debug/control`), not a `-DDEBUG` rebuild
- [ ] Confirmed `CONFIG_DYNAMIC_DEBUG=y` before assuming dynamic debug is available
- [ ] No bare `%p` in printk calls — replaced with the specific `%pX` specifier for the data type
- [ ] Console-visibility issues are diagnosed via `/proc/sys/kernel/printk` / `ignore_loglevel`, not
      assumed to be a log-buffer problem
