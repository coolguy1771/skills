---
name: linux-kernel-internals-essentials
description: Understand the kernel-internal concepts a module/driver author needs before writing real code — task_struct and the `current` macro, process vs interrupt (atomic) context and why you can't sleep in the latter, and how the kernel's virtual address space (lowmem, zones, page frames, VMAs) relates to physical RAM. Use when the user asks "what is task_struct," "how does `current` work," "am I in process or interrupt context," "can I sleep/block here," "what's the difference between a kernel logical address and a virtual address," "what are memory zones / ZONE_DMA / ZONE_NORMAL," or "what is a page frame / PFN." This is background/conceptual grounding — pair it with linux-kernel-memory-allocation (which APIs to call) and linux-kernel-synchronization (locking rules tied to context).
---

# Linux Kernel Internals Essentials

Two pieces of background knowledge quietly gate almost everything else in kernel/driver
development: knowing **what context your code is running in** (and therefore what you're allowed
to do), and knowing **where memory actually comes from** when you allocate it. Neither is deep
kernel-hacker material — both are things every module author needs before their first real driver.

## The task structure and `current`

Every thread alive on the system — user or kernel — has exactly one `struct task_struct` (defined
in `include/linux/sched.h`) representing it in kernel memory. Fields worth knowing:

```c
struct task_struct {
	unsigned int   __state;      /* runnable, sleeping, etc. */
	void          *stack;        /* kernel-mode stack location */
	struct mm_struct *mm;        /* memory management info (NULL for kernel threads) */
	pid_t          pid;          /* unique per-thread ID (yes — every thread, not just process) */
	pid_t          tgid;         /* thread-group ID; == pid for the main thread of a process */
	char           comm[TASK_COMM_LEN]; /* task name */
	struct files_struct *files;  /* open file table */
	struct thread_struct thread; /* CPU-specific hardware context */
	/* ... plus scheduling fields (see linux-kernel-cpu-scheduling) ... */
};
```

Counterintuitive but important: **every thread has a unique PID**, including every thread of a
multithreaded process. What userspace `ps`/`top` display as "the process's PID" is actually the
*TGID* (thread-group ID) — it equals the PID of that group's main thread. `TGID == PID` for a
single-threaded process or a group's main thread; every other thread in the group has its own PID
with the same TGID.

`current` is a macro yielding a pointer to the `task_struct` of whatever is running kernel code on
*this* CPU core right now (loosely, kernel space's version of `this`). Its implementation is
architecture-specific and deliberately fast — a dedicated register on AArch64/PowerPC, a
lock-free per-CPU variable on x86_64. Usage:

```c
#include <linux/sched.h>
pr_info("running as PID %d (%s)\n", current->pid, current->comm);
```

Prefer kernel-provided accessors over raw field access where they exist — e.g. `task_pid_nr(current)`
over `current->pid` — since some fields require a lock to read safely, have a more optimal
accessor, or only make sense within a namespace. `current` is only valid to dereference in
**process context** (see below); using it from a hard-interrupt handler is a bug.

Every user-space thread has a kernel-mode stack in addition to its user-mode stack; kernel threads
have only a kernel-mode stack (no user mapping to speak of). **The kernel-mode stack is fixed-size
and small** (2–4 pages depending on architecture/config) — unlike a user-mode stack, it does not
grow. This is why large local arrays, deep recursion, and generous local structs are dangerous in
kernel/driver code: overflowing it corrupts adjacent memory silently or crashes. `CONFIG_FRAME_WARN`
makes the compiler warn at build time about oversized individual stack frames — heed that warning.

## Process context vs. interrupt (atomic) context

Kernel code always runs in exactly one of two mutually exclusive contexts:

- **Process (task) context** — code running on behalf of a specific thread (a syscall, a kernel
  thread's normal execution, your module's init/cleanup code — which runs in the context of
  whatever process called `insmod`/`rmmod`, *not* some abstract "the kernel"). Usually safe to
  sleep, unless something else about the moment makes it unsafe (see below).
- **Interrupt (atomic) context** — hardware interrupt handlers, softirqs, tasklets. **Always**
  atomic; sleeping here is a kernel bug, full stop.

**The golden rule: never sleep (block) in atomic context.** Sleeping means invoking the scheduler
to context-switch away — if the "thread" doing so isn't a real schedulable entity (a hard-IRQ
handler isn't), or is holding a lock that must not be released by falling asleep, the result is
undefined behavior, typically a hang or a kernel panic.

Check which context you're in with:

```c
#include <linux/preempt.h>
if (in_task())
	foo();   /* process context; usually safe to sleep */
else
	bar();   /* atomic/interrupt context; never sleep here */
```

Prefer `in_task()` over the older `in_interrupt()` macro (bottom-half disabling can confuse
`in_interrupt()`'s result on modern kernels).

**The trap**: `in_task()` returning true does *not* by itself guarantee it's safe to sleep. You can
be in process context while holding a spinlock (see `linux-kernel-synchronization`) — the code
between lock and unlock must still run atomically, and issuing a blocking call there is just as
much a bug as sleeping in a hardware interrupt handler. Context alone isn't the whole answer;
what you're currently holding matters too.

## Virtual addresses, physical RAM, and zones — just enough to allocate memory correctly

A user-space or kernel virtual address is not a raw offset into RAM — the MMU translates it via
page tables into a physical address, and the two need not have any numeric relationship. The
kernel's virtual address space is split from user space (on a typical 64-bit system there's ample
room for both; older 32-bit systems used a 3:1 GB split and had to worry much more about this).

Within the kernel's virtual address space, one particularly important region is **lowmem**: at
boot, the kernel direct-maps most/all physical RAM into a fixed region of its own address space,
one-to-one — physical page frame 0 maps to kernel virtual page 0 at a fixed offset (`PAGE_OFFSET`),
frame 1 to page 1, and so on. Addresses in this region are called **kernel logical addresses**, and
only for *these* addresses do the simple conversions hold:

```c
pa = kva - PAGE_OFFSET;   /* only valid for lowmem/direct-mapped addresses */
kva = pa + PAGE_OFFSET;
```

This does **not** work for user addresses, for kernel addresses outside lowmem (module code,
`vmalloc`-region memory, ioremap/MMIO, KASAN shadow memory, any legacy highmem) — physical RAM
also isn't guaranteed contiguous (it can have holes), so always use the portable kernel APIs
(`virt_to_phys()`, `phys_to_virt()`, `virt_to_page()`, `page_address()`) rather than hand-rolled
arithmetic outside lowmem.

Physical RAM is organized as **nodes** (one per NUMA memory bank; even non-NUMA/UMA systems are
represented as a single fake node) → **zones** (`ZONE_DMA`, `ZONE_DMA32`, `ZONE_NORMAL`, and the
now largely legacy `ZONE_HIGHMEM` on old 32-bit systems) → **page frames** (physical pages, each
tracked by one `struct page`). Inspect this on a live system:

```sh
cat /proc/buddyinfo        # free page-frame counts per node/zone/order
journalctl -b -k | grep -A7 NUMA   # zone ranges the kernel detected at boot
```

Why this matters practically: the allocation APIs covered in `linux-kernel-memory-allocation`
service requests out of specific zones (e.g. `GFP_DMA` restricts an allocation to `ZONE_DMA`, for
hardware that can only address a limited physical range), and the physical-contiguity guarantees
those APIs make are meaningful only because of this direct-mapped/zoned structure.

## Checklist

- [ ] Before calling a blocking/sleeping API, confirmed the code path is in process context
      (`in_task()`) *and* not currently holding a spinlock or otherwise in a locked/atomic region
- [ ] `current` is only dereferenced from process context, never from a hard-interrupt handler
- [ ] Large local variables, deep recursion, and big stack-allocated arrays are avoided — the
      kernel-mode stack is small and fixed-size, with no guard-page growth like a user stack
- [ ] Prefer kernel accessor functions (`task_pid_nr()`, etc.) over raw `current->field` access
      where an accessor exists
- [ ] Any hand-written virtual-to-physical address arithmetic is restricted to addresses known to
      be in the direct-mapped lowmem region; everywhere else, use `virt_to_phys()`/`page_address()`/etc.
