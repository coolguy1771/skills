---
name: linux-kernel-synchronization
description: Choose and correctly use Linux kernel synchronization primitives — spinlocks (spin_lock/spin_lock_irq/spin_lock_irqsave/spin_lock_bh variants), mutexes, atomic_t/refcount_t, per-CPU variables, and RCU — with the exact rules for atomic vs process context and deadlock avoidance. Use when the user asks "spinlock vs mutex," "why can't I sleep while holding this lock," "how do I protect a critical section shared with my interrupt handler," "what's the difference between spin_lock and spin_lock_irqsave," "BUG: sleeping function called from invalid context," "how do I avoid a deadlock between process and interrupt context," or is writing/reviewing any kernel-space locking (module or driver code — this is kernel-space specific, distinct from userspace race-condition/mutex questions covered by concurrency-race-conditions-and-deadlocks).
---

# Linux Kernel Synchronization

Kernel-space races come from three independent sources userspace code doesn't fully share: SMP
concurrency, kernel preemption, and hardware interrupts that can preempt *any* code at any time.
The consequences of getting locking wrong in the kernel are also worse — not a wrong answer, but
a frozen or crashed machine. The single governing rule underlying almost every decision below:

> **The kernel must never sleep (call `schedule()`) in atomic context.**

Everything else — which lock type, which spinlock variant, which GFP flag — falls out of that
rule plus "how do I know what context I'm in."

## Recognizing a critical section

Any code touching shared, writable state that's reachable from more than one concurrent context
is a critical section — including a bare `i++`/`i--` on a global, and a **read** of shared data
(an unprotected read can still see a torn/dirty/inconsistent value). "I'm only reading it" is not
an exemption.

## Determining your context

```c
#include <linux/preempt.h>
if (in_task())
	/* process context — usually safe to sleep, UNLESS you're holding
	 * a spinlock or otherwise in a nested atomic section */
else
	/* atomic/interrupt context — never safe to sleep */
```

Prefer `in_task()` over the older `in_interrupt()` (bottom-half disabling can make
`in_interrupt()` unreliable in modern kernels). Being in process context is *necessary but not
sufficient* for "safe to sleep" — holding a spinlock puts you in an atomic section even in
process context.

## The core decision: spinlock or mutex?

| | Mutex | Spinlock |
|---|---|---|
| Loser threads | sleep (context-switched off CPU) | spin (busy-wait, or on ARM: `WFE` low-power wait) |
| Legal in interrupt/atomic context | **No — never** | Yes (this is what it's for) |
| Legal in process context | Yes | Yes, if critical section is short and non-blocking |
| Critical section may block/sleep | Yes | **No — never** |
| Relative overhead | Higher (≥ 2 context switches) | Lower |

The practical rule the book gives, verbatim in spirit:

- **Is the critical section running in atomic/interrupt context, or in process context where it
  must not sleep? Use the spinlock.**
- **Is the critical section running in process context where sleeping/blocking I/O is possible?
  Use the mutex.**

Theoretical justification: a mutex's minimum cost is ~2 context switches; if the critical
section's runtime is shorter than that (very common — "increment two integers"), a mutex is pure
overhead ("thrashing"). Spinlocks only make sense as spinlocks on SMP (spinning on another core
while the owner runs); on UP kernels the spinlock degenerates to disabling
interrupts/preemption, which is exactly the primitive you need there anyway.

## Mutex

```c
#include <linux/mutex.h>
struct mutex mymtx;
mutex_init(&mymtx);                 /* dynamic init */
/* or: DEFINE_MUTEX(mymtx);            static init */

mutex_lock(&mymtx);
/* ... critical section: may sleep/block ... */
mutex_unlock(&mymtx);
```

Hard rules (enforced under `CONFIG_DEBUG_MUTEXES`):

- Only the current owner may unlock; unlocking twice, or a different task unlocking, is a bug.
- Recursive locking (re-acquiring a mutex you already hold) is **not supported** — self-deadlock.
- Must never be used in interrupt/softirq/tasklet/timer context, or in any context where
  blocking is disallowed.
- A task may not exit while holding a mutex; the lock's backing memory must outlive its use.
- Prefer keeping the lock as a **member of the struct it protects** — avoids ambiguity about
  which lock guards which data, especially as a codebase grows.

Variants: `mutex_trylock()` (non-blocking, returns 1=acquired/0=contended — never spin-poll this
to "check" lock state, it's racy), `mutex_lock_interruptible()` (abortable by a user signal,
returns `-EINTR`/`-ERESTARTSYS` on interruption), `mutex_lock_killable()` (only fatal signals
abort it). Prefer mutexes over the older, more generic `semaphore`/`down()`/`up()` API — kernel
semaphores lack the mutex's ownership semantics and are considered legacy for pure mutual
exclusion.

## Spinlock

```c
#include <linux/spinlock.h>
spinlock_t lock;
spin_lock_init(&lock);              /* or: DEFINE_SPINLOCK(lock); */

spin_lock(&lock);
/* ... critical section: must NOT sleep or block ... */
spin_unlock(&lock);
```

**Never call anything that can sleep while holding a spinlock** — `copy_to_user()`,
`GFP_KERNEL` allocation, `mutex_lock()`, `vmalloc()`, `msleep()`, etc. Doing so on a debug kernel
trips "BUG: scheduling while atomic" / "sleeping function called from invalid context." If a
critical section needs both a fast non-blocking path and a possibly-blocking one, use a spinlock
for the former and a **separate** mutex for the latter (both protecting the same struct is a
common, legitimate pattern) — never try to make one lock serve both purposes.

### Spinlock variant selection (this is the part people get wrong)

| Variant | Use when |
|---|---|
| `spin_lock()` / `spin_unlock()` | Process context only; no hardware interrupt on the same core ever touches this data |
| `spin_lock_irq()` / `spin_unlock_irq()` | Process context critical section that **races with your own hardware interrupt handler**; masks interrupts (and kernel preemption) on the local core for the duration |
| `spin_lock_irqsave(&lock, flags)` / `spin_unlock_irqrestore(&lock, flags)` | Same as above, but interrupts may already be selectively masked elsewhere — saves/restores the actual prior interrupt-mask state instead of unconditionally re-enabling everything |
| `spin_lock_bh()` / `spin_unlock_bh()` | Critical section shared between process context and a softirq/tasklet (bottom half) |

Why the `_irq` variants exist: if a hardware interrupt fires on the *same* CPU core while a
plain `spin_lock()` critical section is running, the interrupt handler preempts it; if the
handler then tries to take the same lock, it spins forever waiting for an unlock that can never
happen (the owner is preempted and can't run) — **self-deadlock**, even though it "looks" like
correct locking. `spin_lock_irq()` prevents this by masking local interrupts for the duration.

Why `_irqsave`/`_irqrestore` over plain `_irq`: `spin_lock_irq()` unconditionally re-enables all
interrupts on unlock. If some interrupts were deliberately masked *before* you took the lock
(e.g. via `disable_irq()` elsewhere in the system), `spin_unlock_irq()` would wrongly reenable
them. `_irqsave` captures the actual prior CPU interrupt-mask state into `flags` and
`_irqrestore` puts back exactly that state — always prefer this pair unless you're certain
nothing else manipulates the interrupt mask.

Overhead ordering (lowest to highest): `spin_lock()` < `spin_lock_irq()`/`spin_lock_bh()` <
`spin_lock_irqsave()`. Use the cheapest variant that's still correct for your actual race
partners — but when genuinely unsure whether something else in the system is manipulating IRQ
masks, `_irqsave`/`_irqrestore` is the safe default.

On UP (uniprocessor) systems, spinlock APIs largely degrade to "just disable interrupts/
preemption" (no real spinning occurs) — you still must use the correct API; the abstraction
means you never have to special-case UP vs SMP yourself.

## Atomic integer operations

For simple counters, prefer atomics over a lock entirely:

```c
#include <linux/atomic.h>
atomic_t v = ATOMIC_INIT(1);
atomic_read(&v); atomic_set(&v, n);
atomic_inc(&v); atomic_dec(&v);
atomic_add(n, &v); atomic_sub(n, &v);
atomic_add_return(n, &v);           /* returns new value */
```

Never read/write an `atomic_t` with plain `v` / `v = n` — always go through the accessor
functions. For **reference counting specifically**, prefer the newer `refcount_t` API
(`refcount_set/read/inc/dec`, `<linux/refcount.h>`) over the older `atomic_t` — it enforces a
strict valid range `[1 .. INT_MAX-1]`, saturates instead of wrapping on overflow, and emits a
loud one-time `WARN_ONCE()` on underflow/overflow, catching use-after-free bugs that a bare
`atomic_t` counter would silently corrupt through. 64-bit counterparts: `atomic64_t`.

## Per-CPU variables and RCU: prefer lock-free when possible

The book's own priority order for protecting shared data, most-preferred first:

1. **Avoid the shared mutable state entirely** if the architecture allows it.
2. **Lock-free techniques**: per-CPU variables, RCU.
3. **Locking**: mutex (blocking-capable, process context) or spinlock (atomic context / short
   non-blocking sections).

**Per-CPU variables** eliminate the critical section altogether by giving every CPU its own copy:

```c
#include <linux/percpu.h>
DEFINE_PER_CPU(int, pcpa);
get_cpu_var(pcpa)++;   /* disables preemption for the duration — do not sleep in here */
put_cpu_var(pcpa);
```

**RCU (Read-Copy-Update)** is the modern preferred replacement for reader-writer locks on
read-mostly data — readers pay almost no synchronization cost:

```c
/* reader */
rcu_read_lock();
p = rcu_dereference(shared_ptr);
/* use *p; must stay inside the read-side critical section */
rcu_read_unlock();

/* writer (must serialize itself against other writers — RCU does not do that) */
new = kzalloc(sizeof(*new), GFP_KERNEL);
/* ... populate new ... */
rcu_assign_pointer(shared_ptr, new);
synchronize_rcu();      /* blocks until all pre-existing readers are done */
kfree(old);
```

Core APIs: `rcu_read_lock()`/`rcu_read_unlock()` (mark reader critical section),
`synchronize_rcu()` (writer: block until a grace period — all readers that started before this
call have finished — then it's safe to free the old object), `call_rcu()` (async/non-blocking
equivalent, needed inside interrupt/atomic context), `rcu_assign_pointer()` (safely publish a
new pointer value with correct memory ordering), `rcu_dereference()` (safely read the
RCU-protected pointer inside a read-side critical section). **RCU protects readers only —
concurrent writers must still serialize against each other with an ordinary lock** (commonly a
spinlock).

## Common deadlock scenarios

- **Self-deadlock**: re-acquiring a lock you already hold (recursive locking is disallowed and
  not silently safe).
- **AB-BA deadlock**: thread A holds lock A, wants lock B; thread B holds lock B, wants lock A —
  each waits forever. Prevent by a strict, documented, always-followed **lock ordering** across
  the whole codebase (the order chosen doesn't matter; consistency does). Lock *release* order
  doesn't matter.
- **Spinlock-vs-interrupt-handler deadlock**: process context holds a plain `spin_lock()`; the
  same lock's interrupt handler fires on the same core, preempts the owner, and spins forever
  waiting for an unlock the owner can't perform. Fixed by using `spin_lock_irq[save]()` instead
  of plain `spin_lock()` whenever a critical section races with your own interrupt handler.
- **Priority inversion**: a low-priority thread holds a mutex a high-priority thread needs, and a
  medium-priority thread preempts the low-priority one — the high-priority thread is blocked
  indefinitely by a thread with *lower* priority than its actual blocker. (This exact bug froze
  NASA's Mars Pathfinder rover in 1997; fixed via priority-inheritance mutexes.) The kernel's
  RT-mutex provides priority inheritance for this scenario.

General guidelines: keep critical sections as short as possible ("lock data, not code"); only
the current lock owner may release it; document and follow one lock-ordering rule everywhere;
never assume a `free()`/`kfree()`-style API nulls out anything for you; use `lockdep` (the
kernel's runtime lock dependency validator, enabled on debug kernels) to catch violations of all
the above automatically rather than relying on manual review.

## Common mistakes

- Not recognizing a bare `i++`/`i--` on shared data as a critical section.
- Treating a read-only access to shared data as exempt from locking.
- Using `GFP_KERNEL` or any blocking call while holding a spinlock.
- Using a mutex inside an interrupt handler, tasklet, or softirq — never legal.
- Using plain `spin_lock()` where the same data is also touched by a hardware interrupt handler
  on the same core — must be `spin_lock_irq()`/`spin_lock_irqsave()`.
- Using `spin_lock_irq()` when something else in the system may have already selectively masked
  interrupts — use `spin_lock_irqsave()`/`spin_unlock_irqrestore()` instead so the prior mask
  state is correctly restored rather than unconditionally cleared.
- Directly reading/writing an `atomic_t`/`refcount_t` instead of going through its accessors.
- Assuming RCU protects writers — it doesn't; writers still need their own lock.

## Checklist

- [ ] Every shared, writable data access (including bare increments and reads) is recognized as
      a critical section and protected
- [ ] Lock type matches context: mutex only in process context where sleeping is fine; spinlock
      for atomic/interrupt context or short non-blocking sections
- [ ] Nothing that can sleep (allocation with `GFP_KERNEL`, `copy_to_user()`, `mutex_lock()`,
      `vmalloc()`, `msleep()`, ...) runs inside a spinlock-protected critical section
- [ ] The correct spinlock variant is used: plain / `_irq` / `_irqsave` / `_bh` matched to what
      actually races with the critical section (own interrupt handler? bottom half? nothing?)
- [ ] A single, documented lock-ordering rule is followed everywhere multiple locks are taken
- [ ] Simple counters use `atomic_t`/`refcount_t` accessors, never a bare read-modify-write
- [ ] Lock-free options (per-CPU variables, RCU) were considered before reaching for a lock
- [ ] Tested (if possible) on a debug kernel with lockdep/`CONFIG_DEBUG_ATOMIC_SLEEP` enabled
