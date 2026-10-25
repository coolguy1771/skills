---
name: linux-kernel-oom-and-memory-overcommit
description: Understand and control Linux's virtual memory overcommit policy and OOM (out-of-memory) killer — the vm.overcommit_memory sysctl (0=OVERCOMMIT_GUESS default, 1=OVERCOMMIT_ALWAYS, 2=OVERCOMMIT_NEVER with its ratio/kbytes formula), why malloc() can succeed yet later crash the process instead of failing outright (demand paging), how to read /proc/meminfo's CommitLimit/Committed_AS, how the kernel picks an OOM victim (the 0-1000 oom_score plus per-process oom_score_adj tunable), and systemd-oomd/cgroups v2 interactions. Use when the user asks "why did the OOM killer kill my process," "why didn't malloc() fail when memory was low," "what does vm.overcommit_memory mean," "how do I protect a process from the OOM killer," "what is CommitLimit in /proc/meminfo," or is debugging a process mysteriously "Killed" with no error message. Distinct from linux-kernel-memory-allocation (kernel-space kmalloc/vmalloc API selection for module authors) — this is about userspace virtual memory accounting and the OOM killer, both kernel policy and its userspace-visible effects.
---

# Linux Kernel OOM Killer and Memory Overcommit

When a process is killed with no error, no crash log, and no failed syscall — just gone, logged as
`Killed` — that is almost always the OOM killer, not a bug in the process. Understanding why
requires separating two concepts: **memory overcommit policy** (does an allocation *request*
succeed?) and **demand paging** (when does that virtual memory actually become physical, and
who's blamed when there isn't enough).

## Why malloc() often doesn't fail — demand paging

Calling `malloc()`/`mmap()` reserves *virtual* address space; no physical page frame is allocated
until the memory is actually touched (read, written, or executed), at which point the MMU raises
a page fault and the kernel's fault handler allocates a physical frame on demand. This means:

- A process can `malloc()` far more than physical RAM+swap and have it "succeed," because
  nothing has been touched yet.
- The failure — if it comes — surfaces later, at the point of the *touch*, not the allocation call.
  When physical memory is genuinely exhausted at that point, the kernel doesn't fail the
  instruction; it invokes the **OOM killer** to free memory by killing a process.
- Reads of never-written pages are special-cased further: the kernel maps every such page to a
  single shared zero-filled physical frame rather than allocating a real frame per read — so even
  reading (not writing) huge amounts of sparse virtual memory costs almost nothing physically.

## The overcommit policy: vm.overcommit_memory

```sh
sudo sysctl -a | grep "vm\.overcommit"
# vm.overcommit_kbytes = 0
# vm.overcommit_memory = 0     <- the policy switch
# vm.overcommit_ratio  = 50
```

| Value | Macro | Behavior |
|---|---|---|
| `0` (default) | `OVERCOMMIT_GUESS` | Heuristic: an allocation request is only rejected up front if it alone exceeds `totalram_pages() + total_swap_pages`. Otherwise it's allowed to proceed — real enforcement happens later, if physical memory is actually exhausted, via the OOM killer. |
| `1` | `OVERCOMMIT_ALWAYS` | No accounting at all — any allocation request succeeds. Entirely on the caller if memory later runs out. |
| `2` | `OVERCOMMIT_NEVER` | Strict accounting: total *committed* virtual memory across the system is capped at `CommitLimit` (see below); a request that would exceed it fails **the allocation call itself** (`malloc()` returns `NULL`) instead of waiting for physical exhaustion. |

Set it (root):

```sh
sudo sh -c "echo 2 > /proc/sys/vm/overcommit_memory"
```

**Mode 2's formula**, and where `CommitLimit` in `/proc/meminfo` comes from:

```
CommitLimit = (total_RAM - total_huge_TLB) * (overcommit_ratio / 100) + total_swap
```

```sh
grep Commit /proc/meminfo
# CommitLimit:   1049192 kB   <- total allowed virtual commitment system-wide
# Committed_AS:   203344 kB   <- currently committed (requested, not necessarily touched) memory
```

`vm.overcommit_ratio`/`vm.overcommit_kbytes` only matter when `overcommit_memory=2` — they're
silently ignored under modes 0 and 1.

**Diagnostic difference between the modes**: with mode 0 (default), a memory-hogging process
runs for a long time, consuming real physical memory, and eventually gets `Killed` by the OOM
killer once physical memory is truly exhausted. With mode 2, the *same* process instead sees an
early, clean `malloc()` failure (`errno=ENOMEM`) once accounting hits `CommitLimit` — no OOM
kill, no ambiguity about which process to blame, but also less tolerance for legitimate transient
overcommit patterns (many programs allocate more virtual memory than they ever touch).

## How the OOM killer picks a victim

Every process has an **OOM score** from 0 (using no memory) to 1000 (using ~100% of available
memory), visible per-process:

```sh
cat /proc/<pid>/oom_score
```

The kernel also protects certain classes by heuristic regardless of score: root-owned processes,
kernel threads, and any process holding a hardware device open are never selected.

Adjust a process's effective score with the tunable `oom_score_adj` (root-only, default 0, range
-1000 to +1000):

```
net_oom_score = oom_score + oom_score_adj
```

```sh
echo -1000 | sudo tee /proc/<pid>/oom_score_adj    # never select this process as the victim
echo  1000 | sudo tee /proc/<pid>/oom_score_adj    # guarantee this process is picked first
choom -p 1234                                       # query/set both score and adjustment in one tool
```

Once a victim is chosen, diagnose it from the kernel log:

```sh
dmesg | tail -n1
# Out of memory: Killed process 812 (myproc) total-vm:1791976kB, anon-rss:1711176kB, ...

sudo dmesg | grep -A5 "Tasks state"   # full per-process memory table the kernel dumped before killing
```

Key fields in the tasks-state dump: `rss` (resident set size — actual physical memory in pages,
the best at-a-glance indicator of who's the real hog), `pgtables_bytes` and `swapents` (overhead
that also counts toward the score — page-table metadata and swap usage add up on large
mappings). `oom-kill:constraint=...` in the log line tells you the scope the OOM decision was
made within (global vs. a specific cgroup/NUMA node).

## systemd-oomd and cgroups v2 — a second, earlier OOM layer

On systemd-managed distros (Ubuntu 22.04+ defaults to this; Arch does not), a userspace daemon,
`systemd-oomd`, can intervene *before* the kernel OOM killer does, based on cgroup memory
pressure (PSI — Pressure Stall Information, kernel 4.20+) rather than waiting for hard exhaustion.
This means a process can be killed for memory pressure even when the kernel-level OOM killer
never would have fired yet. If a process was killed and `dmesg` shows *no* kernel OOM-killer log
entry at all, check `systemd-oomd` next:

```sh
systemctl status systemd-oomd
journalctl -u systemd-oomd
man systemd-oomd.service    # PSI thresholds, per-cgroup memory accounting requirements
```

## Common pitfalls

- Treating a process `Killed` with no error/log as a bug in the process itself, without first
  checking `dmesg` for an OOM-killer log entry.
- Assuming `malloc()` failing (`NULL`/`ENOMEM`) or a silent `Killed` behave the same way — which
  one happens is entirely determined by `vm.overcommit_memory`: mode 2 fails the call cleanly,
  modes 0/1 let it succeed and defer the reckoning to the OOM killer later.
- Forgetting that `vm.overcommit_ratio`/`vm.overcommit_kbytes` do nothing unless
  `vm.overcommit_memory=2` — tuning them under the default mode 0 has zero effect.
- Assuming `CommitLimit` is a hard cap on *physical* memory usage — it's a cap on virtual
  *commitment* (requested, whether or not touched), and it only applies at all under mode 2.
- Trying to make a critical process OOM-immune by giving it more memory instead of setting
  `oom_score_adj=-1000` — the score reflects usage, not exemption; a heavy but critical process
  will still be near the top of the candidate list without an explicit adjustment.
- Not checking `systemd-oomd` when a process was killed for memory pressure but the kernel's
  own OOM killer log is absent — on modern systemd distros this is a second, earlier trigger.
- Confusing `rss` (physical memory actually resident) with total virtual memory requested
  (`total-vm` in the OOM log line) when deciding who the "real" hog is — a process can have a
  huge `total-vm` while touching very little of it.

## Checklist

- [ ] `dmesg`/`journalctl -k` checked for an OOM-killer log line before assuming any other cause
      for an unexplained process death
- [ ] `vm.overcommit_memory` value confirmed before reasoning about whether an allocation
      *should* have failed outright vs. succeeded and been settled later by the OOM killer
- [ ] Diagnosed via `/proc/meminfo`'s `CommitLimit`/`Committed_AS` when working under
      overcommit mode 2, not assumed to apply under the default mode 0
- [ ] A process that must never be killed has `oom_score_adj` explicitly set to `-1000` (or
      `choom`), not just "seems important"
- [ ] `systemd-oomd` checked (`systemctl status systemd-oomd`, `journalctl -u systemd-oomd`) when
      a kill occurred with no matching kernel-level OOM log entry
- [ ] Victim analysis reads `rss` (actual physical usage) rather than `total-vm` (virtual
      commitment) when judging who was the genuine memory hog
