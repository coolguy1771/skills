---
name: concurrency-race-conditions-and-deadlocks
description: Diagnose or design around race conditions, deadlocks, and starvation in concurrent/multithreaded code using semaphores, mutexes, and read/write locks. Use this when the user asks "why did this counter/balance end up wrong under concurrent access", "what is a race condition and how do I fix it", "why did my program hang / deadlock", "how do semaphores/mutexes actually prevent races", "what order should I acquire multiple locks in", "what's the difference between a spin lock and a blocking lock", "how granular should my locking be", or is reviewing code with shared mutable state accessed from multiple threads/processes/requests (bank balances, counters, caches, in-memory queues). Also trigger for "why is test-and-set atomic and why does that matter", circular wait / lock ordering questions, or choosing between coarse-grained and fine-grained locking.
---

# Race Conditions and Deadlocks

Concurrency bugs come from one root cause: a processor (or thread) doesn't operate on shared
storage directly — it **copies** a value in, computes with the local copy, then writes the
result back. When two threads copy-compute-writeback on the same storage location without
coordination, whichever writeback happens last silently overwrites the other's work, even
though both computations were individually correct. This happens with multiple CPU cores, and
just as easily on a single core via preemptive multitasking — a thread can be paused
mid-computation and resumed later, interleaving with another thread exactly as if they ran on
separate processors. Never assume single-core execution makes shared mutable state safe.

## Anatomy of a race condition

The canonical example: two concurrent transactions both read a balance, each compute an
independent update from that same stale read, and whichever writes back last wins — the
other's update is lost entirely, not merged, not erroring, just silently gone.

```python
# UNSAFE: read-modify-write on shared state with no coordination.
def withdraw(account, amount):
    balance = account.balance        # read
    balance -= amount                # modify (local copy)
    account.balance = balance        # write back

# If two threads call withdraw() on the same account "simultaneously":
#   Thread A reads balance=1000
#   Thread B reads balance=1000        <- both read the SAME stale value
#   Thread A computes 1000-300=700, writes 700
#   Thread B computes 1000-200=800, writes 800   <- overwrites A's withdrawal entirely
# Final balance: 800 (should be 500). A's 300 withdrawal vanished with no error.
```

The bug is not in the arithmetic — each thread's read-modify-write sequence is individually
correct. The bug is that the sequence isn't **atomic**: another thread can interleave between
the read and the write. This is why race conditions are notoriously hard to reproduce in
testing — they depend on timing, and most runs won't hit the interleaving window at all.

## Fixing it: from too-strict to just-right

**Read-only sharing** eliminates races trivially but only works if nothing ever needs to
mutate the shared data — rarely viable for real systems, though worth calling out explicitly
whenever a "shared cache" or "shared config" turns out to genuinely never change after load.

**Serializing everything** (a single global lock around all access) is correct but throws away
concurrency entirely — every transaction waits behind every other transaction regardless of
whether they touch overlapping data. This is the right first fix to *reason about* correctness,
and the wrong one to ship if throughput matters.

**Fine-grained locking (semaphores/mutexes per resource)** is the actual production answer:
lock only the specific data a transaction touches, so unrelated transactions on unrelated data
never wait on each other.

```python
def withdraw(account, amount):
    with account.lock:               # acquire — blocks if another thread holds it
        balance = account.balance
        balance -= amount
        account.balance = balance
    # release happens automatically on exiting the `with` block
```

## Why the lock itself must be atomic (test-and-set)

A semaphore/lock is only safe if its own acquire operation can't be interrupted mid-check. A
naive acquire — "read the flag, if it's free set it to taken" — is *itself* a race condition if
those two steps aren't atomic: two threads can both read "free" before either writes "taken,"
and both proceed believing they hold the lock exclusively. This is why real locks are built on
a hardware-guaranteed **atomic test-and-set** (or compare-and-swap) instruction: read-and-write
happen as one indivisible hardware operation, so there's no window for another thread to
interleave. When implementing any synchronization primitive from scratch, the correctness of
the whole thing rests entirely on this one atomic primitive — never approximate it with
separate read and write steps, however briefly, "just for this one internal case."

## Spin locks vs. blocking locks

A **spin lock** repeatedly re-checks (busy-waits on) the lock in a tight loop until it's free.
Simple, but wastes CPU cycles doing nothing useful, and can cause **starvation**: if a
slower-checking thread (e.g., on a loaded core or slow network hop) keeps losing the race to
re-acquire against faster competitors, it can wait indefinitely even though it's "trying" the
whole time. A **blocking lock** instead suspends the waiting thread and queues its ID on the
lock, waking it in order once the lock frees — costs a context-switch but guarantees
first-come-first-served fairness and doesn't burn CPU while waiting. Default to blocking locks
for anything where wait times could be non-trivial or contention could be high; spin locks are
only appropriate when critical sections are extremely short (a few instructions) so the
busy-wait itself is cheaper than a context switch.

## Deadlock: circular waits from multi-lock acquisition

Locking a single resource prevents races on that resource but introduces a new failure mode
once a transaction needs **multiple** locks at once. If two transactions acquire the same two
locks in opposite orders, each can end up holding one lock while waiting for the other — a
**circular wait**, one form of deadlock, where neither transaction can ever proceed:

```python
# Transaction A                          # Transaction B
with lock_main:                          with lock_secondary:
    with lock_secondary:  # blocks           with lock_main:      # blocks
        ...                                      ...
# A holds lock_main, waiting for lock_secondary (held by B)
# B holds lock_secondary, waiting for lock_main (held by A)
# Neither can ever proceed. This is not a slow path -- it hangs forever.
```

**The fix: a global, consistent lock-ordering rule.** Assign every lockable resource a fixed
order (e.g., by ID) and require every code path to acquire locks in that order, never any
other. If both transactions above always acquire `lock_main` before `lock_secondary`
regardless of which "direction" the business operation conceptually goes, only one can ever
get past the first acquisition, and the other simply waits its turn — no cycle is possible.

```python
def transfer(account_a, account_b, amount):
    first, second = sorted([account_a, account_b], key=lambda a: a.id)  # canonical order
    with first.lock:
        with second.lock:
            account_a.balance -= amount
            account_b.balance += amount
```

This is the single most important habit for any code that ever needs to hold more than one
lock at a time: **never acquire locks in an order that depends on argument order or call site**
— always normalize to a canonical order first.

## Read/write locks: recovering performance for read-heavy workloads

If most access is read-only and only occasional writes mutate the data, a plain mutex
needlessly serializes reads against each other. **Read/write locks** allow any number of
concurrent readers, but a writer needs exclusive access with no readers or other writers
present:

- A read lock can be acquired only when no thread holds (or, in the fairer variant, is
  *waiting for*) the write lock.
- A write lock can be acquired only when no thread holds any lock (read or write) on that data.

The naive version reintroduces starvation: if readers arrive continuously, a waiting writer
can be starved indefinitely. The fix is a priority rule — once a writer requests the lock,
subsequent readers queue behind it rather than continuing to cut in line, guaranteeing the
writer eventually gets exclusive access.

## Granularity: the actual engineering tradeoff

- **Too coarse** (one lock protects a large group of unrelated data): a transaction touching
  one field blocks every other transaction touching any field in the group, even completely
  unrelated ones. Simple to reason about, poor concurrency.
- **Too fine** (a separate lock per tiny piece of data): acquiring/releasing many locks per
  operation adds real overhead, and more locks in play means more opportunities to violate
  lock-ordering discipline and create a deadlock.

There's no universal right answer — it's a measured tradeoff specific to actual contention
patterns. Start coarser than you think you need (correctness first), then narrow granularity
only where profiling shows real contention, verifying the ordering rule still holds after every
split.

## Common pitfalls

- **Assuming single-threaded-looking code is race-free because "it's all one function."**
  Preemptive multitasking can interleave *any* two operations on shared state, including two
  invocations of the same function from different requests/threads — races aren't limited to
  code that explicitly spawns threads.
- **Implementing a custom lock/flag with separate read and write steps** instead of an atomic
  primitive — reintroduces the exact race the lock exists to prevent, just one level up.
- **Acquiring multiple locks in an order that depends on call-site argument order** (e.g.,
  `transfer(a, b)` locks `a` then `b`, but `transfer(b, a)` locks `b` then `a`) — this is the
  single most common cause of deadlock in real systems. Always normalize to a canonical
  acquisition order.
- **Using a spin lock for a critical section that isn't guaranteed short**, wasting CPU and
  risking starvation under contention.
- **Locking a whole object/table when only one field/row is actually contended** — kills
  throughput for no correctness benefit; profile actual contention before deciding granularity.
- **"Fixing" a race by adding a lock around only some of the access paths.** If any code path
  reads or writes the shared data without acquiring the same lock, the race persists on that
  path — locking must be applied consistently everywhere the data is touched, not just where
  the bug was first noticed.
- **Confusing deadlock with a plain slow path.** A genuine circular wait never resolves on its
  own — no amount of extra time fixes it. If a "just wait longer" retry appears to fix a hang,
  that's a symptom of contention or an unrelated slow dependency, not a deadlock, and the fix
  should be understood accordingly.

## Checklist

- [ ] Identified every code path that reads or writes the shared data, and confirmed all of
      them acquire the same lock — not just the path where the bug was noticed
- [ ] Confirmed the lock/semaphore's own acquire operation is backed by a real atomic
      primitive, not separate read-then-write steps
- [ ] Any code that holds more than one lock at a time acquires them in a single, canonical
      order regardless of call-site argument order
- [ ] Chose blocking locks over spin locks unless the critical section is provably very short
- [ ] Considered read/write locks (with writer-priority) for read-heavy shared data instead of
      a plain mutex
- [ ] Locking granularity matches measured contention — not so coarse that unrelated
      operations block each other, not so fine that lock overhead or ordering complexity
      becomes its own risk
- [ ] Distinguished an actual deadlock (permanent, no self-resolution) from ordinary
      contention or an unrelated slow dependency before choosing a fix
