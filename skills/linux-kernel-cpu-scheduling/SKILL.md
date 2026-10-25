---
name: linux-kernel-cpu-scheduling
description: Understand and control Linux CPU scheduling — the POSIX scheduling policies (SCHED_OTHER/NORMAL, SCHED_FIFO, SCHED_RR, SCHED_BATCH, SCHED_IDLE, SCHED_DEADLINE), the nice-value (-20..+19) vs real-time priority (1..99) scales, how CFS ("Completely Fair Scheduler") picks among SCHED_OTHER threads via vruntime, the scheduling-class hierarchy (stop_sched > deadline > RT > fair > idle) and why hardware/software interrupts always preempt even SCHED_FIFO, userspace tools (chrt, taskset, nice/renice) and syscalls (sched_setattr/sched_getattr, sched_setaffinity), and the kernel-internal APIs for setting a kthread's policy (sched_set_fifo(), sched_set_fifo_low(), sched_set_normal() — NOT sched_setscheduler()). Use when the user asks "SCHED_FIFO vs SCHED_RR," "what's the difference between nice value and real-time priority," "how does CFS decide which thread runs," "how do I pin a process/thread to a CPU core," "how do I set a kernel thread's scheduling policy," or is reasoning about latency, starvation, or scheduling behavior on Linux.
---

# Linux Kernel CPU Scheduling

Linux implements POSIX-style scheduling policies through a **scheduling-class hierarchy**, not a
single flat algorithm. Which class a thread belongs to determines both its priority scale and how
it competes for the CPU — get this model straight before reasoning about any scheduling behavior.

## The scheduling classes, highest to lowest priority

```
stop_sched_class   > dl_sched_class (SCHED_DEADLINE) > rt_sched_class (SCHED_FIFO/RR)
                   > fair_sched_class (SCHED_OTHER/NORMAL, SCHED_BATCH) > idle_sched_class
```

A thread belongs to exactly one class at a time. The scheduler always picks a runnable thread
from the highest-priority class that has one — real-time threads are checked before fair-class
threads even get a look-in, and stop/deadline classes exist above even that.

**Crucial exception that trips people up**: hardware and software interrupts always preempt
*any* thread, including a `SCHED_FIFO` priority-99 thread. Linux is a general-purpose OS (GPOS),
not a hard-RTOS — treat "real-time" policies here as *soft* (at best firm) real-time.

## Policies and their priority scales

| Policy | Priority scale | Behavior |
|---|---|---|
| `SCHED_OTHER`/`SCHED_NORMAL` | nice -20 (highest) to +19 (lowest), default 0 | Always the default; driven by CFS; goal is fairness + throughput |
| `SCHED_BATCH` | same nice range | Like `SCHED_OTHER` but less preemptive — for non-interactive batch jobs |
| `SCHED_IDLE` | lowest of all, below nice +19 | Only runs when nothing else wants the CPU |
| `SCHED_RR` | real-time 1 (lowest) to 99 (highest) | Finite timeslice (`/proc/sys/kernel/sched_rr_timeslice_ms`, default 100ms); same-priority `SCHED_RR` threads round-robin |
| `SCHED_FIFO` | real-time 1 to 99 | Effectively **infinite** timeslice — yields only on blocking I/O, stopping/dying, or preemption by a higher-priority RT thread. Same-priority threads do **not** round-robin — avoid multiple `SCHED_FIFO` threads at the same priority, one can starve the others |
| `SCHED_DEADLINE` | separate class, above RT | EDF-style: runtime/deadline/period parameters, not a simple priority number |

Non-real-time threads always have real-time priority 0 — they never compete on the same footing
as an RT thread regardless of how negative their nice value is. Nice value and real-time priority
are two *separate* scales that never mix.

## CFS in brief

Since 2.6.23, `SCHED_OTHER` threads are scheduled by CFS, tracked via a per-thread **vruntime**
(virtual runtime) stored in a red-black tree runqueue, ordered by vruntime. The scheduler always
picks the leftmost (lowest-vruntime = most CPU-starved) node — a thread that's run less gets
picked next, which is the fairness mechanism. `nice` values bias how fast vruntime accumulates
(a lower nice value accumulates vruntime more slowly, so effectively gets more CPU time), rather
than mapping to a fixed timeslice.

## Userspace tools

```sh
nproc                          # number of CPU cores
nice -n 10 mycommand           # start a process at nice +10 (lower priority)
renice -n 5 -p 1234            # change nice value of a running process

taskset -p 1234                # show a process's current CPU affinity mask
taskset -c 2 yes >/dev/null &  # pin a new process to core 2
taskset 03 gcc foo.c -o foo    # affinity mask 0x03 = cores 0-1

chrt -p 1234                   # query scheduling policy/priority of PID 1234
chrt -f -p 50 1234             # set SCHED_FIFO, priority 50
chrt -r -p 20 1234             # set SCHED_RR, priority 20
```

Under the hood, `chrt` queries/sets via the `sched_getattr()`/`sched_setattr()` syscalls (the
modern replacement for the older `sched_{get,set}scheduler()` pair), operating on `struct
sched_attr`. `taskset` uses `sched_getaffinity()`/`sched_setaffinity()`.

## Setting scheduling policy from inside the kernel — do NOT use sched_setattr()

A module or driver creating and configuring a **kthread**'s scheduling policy must not reach for
`sched_setscheduler()`/`sched_setattr()` with an arbitrary priority — as of kernel 5.9, the kernel
community removed that latitude deliberately (a module has no "systems knowledge" to know
what real-time priority is actually correct; only the administrator/integrator does). Use instead:

```c
#include <linux/sched.h>

void sched_set_fifo(struct task_struct *p);       // SCHED_FIFO, RT priority 50 (MAX_RT_PRIO/2)
void sched_set_fifo_low(struct task_struct *p);   // SCHED_FIFO, RT priority 1 (lowest RT)
void sched_set_normal(struct task_struct *p, int nice);  // SCHED_NORMAL at the given nice value
```

These are the only kernel-internal entry points intended for module/driver use; they're thin
wrappers over `sched_setscheduler_nocheck()`/`sched_setattr_nocheck()`. Do not try to hand-roll
a specific `SCHED_FIFO` priority from kernel code — that's exactly the anti-pattern this API
replaced.

## Setting CPU affinity from a kernel thread

`sched_setaffinity()` is **not exported** to modules (and `kallsyms_lookup_name()`, the old
workaround for calling unexported symbols, was itself unexported from kernel 5.7 — see commit
`0bd476e6c671`). There is no clean in-tree way for an out-of-tree module to set a kthread's CPU
affinity. The documented (if inelegant) workaround: look the symbol's address up via
`/proc/kallsyms` (root-only) at load time, pass it into the module as a parameter, and call it
through a function pointer:

```c
static unsigned long func_ptr;
module_param(func_ptr, ulong, 0);
unsigned long (*schedsa_ptr)(pid_t, const struct cpumask *) = NULL;

schedsa_ptr = (unsigned long (*)(pid_t, const struct cpumask *))func_ptr;

cpumask_clear(&mask);
cpumask_set_cpu(cpu, &mask);      /* param is CPU number, not a bitmask */
ret = (*schedsa_ptr)(0, &mask);   /* 0 => affine the calling (kernel) thread itself */
```

This is explicitly a hack, not a supported pattern — acceptable for learning/experimentation
(e.g. demonstrating per-CPU variables), not for production drivers. Prefer letting userspace pin
kthread-backed work via existing kernel affinity APIs (`kthread_bind()` at creation time is the
clean, exported alternative when you control kthread creation).

## Visualizing scheduling activity

```sh
gnome-system-monitor          # GUI per-core visualization
perf sched                    # detailed scheduler event tracing
top -H                        # per-thread view with which CPU each is on
```

## cgroups v2, briefly

Modern systemd-managed systems bias CPU/memory allocation per-cgroup (slices/scopes) rather
than pure per-thread nice values — `CONFIG_SCHED_AUTOGROUP` groups interactive terminal
sessions for better foreground responsiveness, and systemd unit files can set CPU quotas
(`CPUQuota=`) that constrain a whole cgroup regardless of individual thread nice values. If a
process's observed CPU share doesn't match what its nice value alone would predict, check
`systemctl show <unit> -p CPUQuota,CPUWeight` before assuming the nice value isn't taking effect.

## Common pitfalls

- Assuming a lower nice value always beats *any* real-time thread — nice values only rank
  `SCHED_OTHER` threads against each other; every RT thread outranks every `SCHED_OTHER`
  thread regardless of nice value.
- Running multiple `SCHED_FIFO` threads at the *same* priority expecting fairness — unlike
  `SCHED_RR`, same-priority `SCHED_FIFO` threads do not round-robin; one can monopolize the CPU.
  Use `SCHED_RR` (or distinct priorities) if fairness among them matters.
- A `SCHED_FIFO` priority-99 thread stuck in a tight loop on a single-core, low-preemption kernel
  hanging the machine — always ensure an RT thread has a genuine yield/blocking point.
- Calling `sched_setscheduler()`/`sched_setattr()` from kernel/module code with a hardcoded RT
  priority — use `sched_set_fifo()`/`sched_set_fifo_low()`/`sched_set_normal()` instead.
- Expecting `sched_setaffinity()` to be callable directly from an out-of-tree module — it isn't
  exported; either use `kthread_bind()` at kthread-creation time or accept the `/proc/kallsyms` +
  function-pointer hack is explicitly non-production.
- Debugging a scheduling-latency mystery without checking whether cgroups/systemd quotas
  (`CPUQuota=`, `CPUWeight=`) are the actual constraint, rather than the thread's own nice value
  or policy.
- Confusing `taskset`'s CPU affinity *mask* (bitmask, e.g. `03` = cores 0–1) with a CPU *index*.

## Checklist

- [ ] Correctly distinguished which scale a stated priority number is on — nice (-20..+19,
      `SCHED_OTHER`/`BATCH`) vs real-time priority (1..99, `SCHED_FIFO`/`RR`) — before reasoning
      about who wins
- [ ] No same-priority `SCHED_FIFO` threads relied upon for round-robin fairness (used `SCHED_RR`
      or distinct priorities instead)
- [ ] Kernel-space policy changes on a kthread use `sched_set_fifo()`/`sched_set_fifo_low()`/
      `sched_set_normal()`, never a raw `sched_setscheduler()`/`sched_setattr()` call with an
      arbitrary hardcoded priority
- [ ] CPU affinity for a kthread set via `kthread_bind()` at creation where possible, rather than
      calling the unexported `sched_setaffinity()` from module code
- [ ] Any real-time (`SCHED_FIFO`/`RR`) thread has a genuine, reachable yield/blocking condition —
      not an unconditional tight loop
- [ ] Unexplained CPU-share behavior checked against cgroup/systemd quotas
      (`CPUQuota`/`CPUWeight`) before concluding the nice value/policy itself is wrong
- [ ] Used `chrt`/`taskset`/`nice`/`renice` for userspace policy/affinity work rather than writing a
      custom syscall wrapper
