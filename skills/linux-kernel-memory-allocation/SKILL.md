---
name: linux-kernel-memory-allocation
description: Choose and use the correct Linux kernel memory allocation API — kmalloc/kzalloc/krealloc/kcalloc (slab), vmalloc/vzalloc/kvmalloc (virtually-contiguous), the low-level page allocator (__get_free_pages/alloc_pages), and custom kmem_cache slab caches — including GFP flags (GFP_KERNEL vs GFP_ATOMIC), the ~4MB kmalloc size limit, and avoiding leaks/oversized-request bugs. Use when the user asks "how do I allocate memory in a kernel module/driver," "kmalloc vs vmalloc," "what GFP flag do I use," "why did kmalloc fail / return NULL," "how big can a single kmalloc be," "should I make my own slab cache," or is reviewing kernel/driver C code with kmalloc/vmalloc/kfree/vfree calls for correctness. Distinct from c-memory-management-safety (user-space malloc/free) — kernel space has different allocators, no libc, and a hard no-sleep-in-atomic-context rule that user-space code doesn't have.
---

# Linux Kernel Memory Allocation

Kernel-space memory allocation looks like `malloc`/`free` on the surface but has three constraints
user-space code never deals with: every allocator sits on top of the **page allocator**, so size
and physical-contiguity guarantees are architecturally limited; every allocation call must be
told, via a **GFP flag**, whether it's allowed to sleep; and getting either of those wrong doesn't
throw an exception — it panics the machine or corrupts memory silently.

## The allocator stack

```
kmem_cache_alloc() / kmalloc() / kzalloc()   <- slab allocator (object caching, sub-page granularity)
              |
__get_free_pages() / alloc_pages()            <- page allocator ("buddy system", the BSA)
              |
          physical RAM
```

- The **page allocator** (buddy system allocator, BSA) is the ultimate source of every byte of
  kernel memory. It hands out physically contiguous chunks sized as a power-of-two number of
  pages (`2^order` pages, `order` from 0 to `MAX_ORDER-1`). With a 4 KB page size and the typical
  `MAX_ORDER` of 11, the largest single chunk it can produce is `2^10` pages = 4 MB.
- The **slab allocator** sits above it purely to fix the page allocator's biggest weakness: asking
  for 12 bytes and getting a whole 4 KB page back is enormous internal fragmentation. The slab
  layer pre-carves pages into same-sized object caches (`kmalloc-8`, `kmalloc-16`, ... up to
  `kmalloc-8k`) and serves small requests from the best-fit cache instead.
- **`vmalloc()`** is a third, separate region of the kernel's virtual address space: it allocates
  memory that's virtually contiguous but *not* physically contiguous, side-stepping the page
  allocator's contiguity constraint entirely (at the cost of TLB/page-table overhead per access).

## kmalloc() / kzalloc() — the default choice

```c
#include <linux/slab.h>
void *kmalloc(size_t size, gfp_t flags);
void *kzalloc(size_t size, gfp_t flags);   /* kmalloc + zero the memory; prefer this */
void kfree(const void *ptr);
```

- Memory from `kmalloc`/`kzalloc` is **guaranteed physically contiguous** and hardware
  cacheline-aligned — this is why it's required for DMA-capable buffers (in conjunction with the
  proper DMA API, not used standalone for DMA setup).
- **Always prefer `kzalloc()` over `kmalloc()`.** Freshly-`kmalloc`'d memory has *random* leftover
  content (same as user-space `malloc`) — reading it before writing it is an uninitialized-memory-read
  bug, and it's a much more expensive class of bug in kernel space than in a user process.
- **Size limit**: a single `kmalloc()`/`kzalloc()` call is capped at the same ~4 MB the page
  allocator can produce in one chunk (`PAGE_SIZE * 2^(MAX_ORDER-1)`) — architecture-dependent, but
  4 MB on both x86_64 and ARM with the common `MAX_ORDER=11`/4 KB pages. Requesting more doesn't
  crash anything — the call simply fails and returns `NULL`.
- **Actual allocated size can exceed the request.** The slab layer rounds up to its nearest cache
  size (e.g. asking for 65 bytes actually allocates 128). Use `ksize(ptr)` to find the *actual*
  usable size of an already-allocated block if your code can benefit from it — never assume the
  requested size is the allocated size.
- `kcalloc(n, size, flags)` is the array-safe equivalent of `calloc()` — it checks for `n * size`
  overflow internally; never compute the multiplication yourself and pass it to `kmalloc`.
- `krealloc(ptr, newsize, flags)` behaves like `realloc()` — same "don't overwrite your only
  pointer with the return value before checking it" trap as user-space `realloc` applies here too.
- `kfree(NULL)` is a safe no-op, same as user-space `free(NULL)`. Use `kfree_sensitive()` (formerly
  `kzfree()`) instead of plain `kfree()` when the buffer held sensitive data (keys, credentials) —
  it wipes the contents before returning the block to its slab cache; plain `kfree()` does not
  clear memory, it just returns the slab to its free list with old content intact.

## GFP flags: the sleep/no-sleep contract

Every allocation call takes a `gfp_t` flags argument telling the allocator whether it's allowed to
sleep (block) while trying to free up memory to satisfy the request:

- **`GFP_KERNEL`** — use whenever you're in process context and it's safe to sleep (the overwhelmingly
  common case for module/driver init paths, ioctl handlers, etc.). Internally expands to
  `__GFP_RECLAIM | __GFP_IO | __GFP_FS` — meaning it can trigger I/O and filesystem operations to
  reclaim memory, which is itself why it must never be used from a context where I/O/FS reentrancy
  would deadlock (e.g. inside a filesystem's own writeback path — use `GFP_NOFS`/`GFP_NOIO` there).
- **`GFP_ATOMIC`** — **required** whenever you're in interrupt/softirq context, or holding a
  spinlock, or anywhere else sleeping is illegal (see `linux-kernel-internals-essentials` for how
  to determine this). It will not sleep, will not trigger reclaim I/O, and is more likely to fail
  under memory pressure than `GFP_KERNEL` — but failing fast beats sleeping where sleep is illegal.
- `__GFP_ZERO` — bitwise-OR'd with `GFP_KERNEL`/`GFP_ATOMIC` to zero the returned memory (this is
  what `kzalloc`/`vzalloc` do internally).
- `GFP_DMA` — restricts the allocation to `ZONE_DMA` (physical memory reachable by older,
  address-limited DMA-capable hardware).
- `GFP_NOWAIT` — like `GFP_ATOMIC` but does not even dip into emergency memory pools; use only when
  you specifically want "succeed immediately or fail immediately, no exceptions."

**The rule that matters in practice**: process context + safe to sleep → `GFP_KERNEL`. Interrupt
context, or holding a spinlock, or anywhere blocking is illegal → `GFP_ATOMIC`. Getting this
backwards — using `GFP_KERNEL` inside a spinlock-held region or an interrupt handler — is a classic
"sleeping function called from invalid context" kernel bug, often only surfacing under load.

## vmalloc() / vzalloc() and friends — when contiguity doesn't matter

```c
#include <linux/vmalloc.h>
void *vmalloc(unsigned long size);
void *vzalloc(unsigned long size);   /* zeroed, like kzalloc */
void vfree(const void *addr);
```

- Returns **virtually contiguous, page-aligned** memory with **no guarantee of physical
  contiguity** — the kernel maps potentially scattered physical pages into one contiguous virtual
  range.
- Can service allocations far larger than kmalloc's ~4 MB ceiling (limited only by the size of the
  kernel's vmalloc address region — can be terabytes on 64-bit systems).
- **Must only be called from process context where it's safe to sleep** — `vmalloc()` can never be
  used from atomic/interrupt context, unlike `kmalloc(..., GFP_ATOMIC)`.
- Costs more per access than kmalloc'd memory: each access potentially walks extra page-table
  levels (TLB pressure) since the underlying physical pages aren't contiguous. Use it for large,
  software-only buffers where you don't need DMA-style physical contiguity, not as a casual
  default.
- **Never treat vmalloc'd memory as physically contiguous** — passing it to a DMA-mapping API
  as if it were one physical buffer is a real, book-documented bug class. DMA buffers belong to
  the dedicated DMA Engine API (`dma_alloc_coherent()` etc.), not the slab or vmalloc layer directly.

**`kvmalloc(size, flags)`** is the pragmatic "I don't know how big this will be at runtime" choice:
it tries `kmalloc()` first (fast, physically contiguous) and transparently falls back to
`vmalloc()` if the slab allocator can't satisfy the size. Free with `kvfree()` regardless of which
path was actually taken internally. Prefer `kvzalloc()`/`kvmalloc_array()` for the zeroed/array
variants, same rationale as `kzalloc` over `kmalloc`.

## Custom slab caches — kmem_cache

When a module/driver frequently allocates and frees many instances of the *same* fixed-size
struct, creating a dedicated slab cache for it beats repeated generic `kmalloc()` calls (same
underlying mechanism the kernel itself uses for its own hot structures like `task_struct`,
inodes, and dentries):

```c
#include <linux/slab.h>

struct kmem_cache *cachep;

cachep = kmem_cache_create("my_ctx_cache", sizeof(struct my_ctx),
                           0,                 /* alignment */
                           SLAB_HWCACHE_ALIGN, /* flags */
                           NULL);              /* optional constructor */
if (!cachep)
	return -ENOMEM;

/* per allocation: */
struct my_ctx *obj = kmem_cache_alloc(cachep, GFP_KERNEL);
/* ... use obj ... */
kmem_cache_free(cachep, obj);

/* at module unload, after every object has been freed: */
kmem_cache_destroy(cachep);
```

`kmem_cache_create()` may only be called from process context. Reach for a custom cache when
you're allocating/freeing the same struct at high frequency and want to amortize the setup cost
and improve cache locality — not for one-off or rarely-allocated structures, where plain
`kzalloc()` is simpler and just as correct.

## Which API to use — decision guide

| Need | Use |
|---|---|
| Small (< 1 page), physically contiguous, typical module/driver case | `kmalloc()` / `kzalloc()` / `kcalloc()` / `krealloc()` |
| Same, but in a device driver's `probe()` (auto-freed on driver detach) | `devm_kzalloc()` / `devm_kmalloc()` |
| Physically contiguous, general low-level page-sized allocation | `__get_free_page[s]()`, `get_zeroed_page()`, `alloc_page[s]()` |
| Physically contiguous, for DMA | The dedicated DMA Engine API (`dma_alloc_coherent()`, etc.) — never slab/page allocator directly |
| Large, virtually-contiguous, software-only buffer | `vmalloc()` / `vzalloc()` |
| Unsure of runtime size, want the best of both | `kvmalloc[_array]()` / `kvzalloc()`, freed with `kvfree()` |
| Frequently allocated/freed custom struct | A dedicated `kmem_cache` |

As a default rule of thumb: reach for `kzalloc()` first for anything under a page; only move to
`vmalloc()`/`kvmalloc()` once you have a concrete reason (large or runtime-unknown size) to need it.

## Common pitfalls

- **Using `GFP_KERNEL` in atomic context** (interrupt handler, softirq, or while holding a
  spinlock) — must be `GFP_ATOMIC` there. This is the single most common kernel-memory-allocation
  bug class.
- **Not checking the return value.** Every one of these APIs can return `NULL`; a NULL deref two
  lines later is a much worse debugging experience than a clean early return. (You generally don't
  need to `pr_err`/`WARN` on failure yourself — the kernel already logs extensively when memory
  allocation fails — but you must still branch on the failure.)
- **Computing `num * size` yourself and passing it to `kmalloc`** instead of using `kcalloc()` —
  same integer-overflow-to-undersized-allocation risk as user-space `malloc(num*size)` (see
  `c-memory-management-safety`).
- **Leaking on error paths.** Kernel init functions commonly allocate several resources in
  sequence; every early-return/`goto` failure path must free everything allocated *before* the
  failure point, in reverse order — untested error paths are where kernel memory leaks
  overwhelmingly live.
- **Treating `vmalloc()`-returned memory as physically contiguous** (e.g., trying to DMA into/out
  of it directly) — it isn't, and hardware will read/write the wrong physical pages.
- **Calling `vmalloc()` from atomic/interrupt context** — it can sleep internally and must not be
  called from anywhere `GFP_ATOMIC` would be required.
- **Oversized single-call `kmalloc()` requests** (approaching or exceeding ~4 MB) failing and
  being mistaken for a bug in the caller rather than an inherent page-allocator/`MAX_ORDER` limit —
  switch to `vmalloc()`/`kvmalloc()` instead of trying to force a larger single kmalloc through.
- **Declaring large local arrays/structs on the kernel stack** instead of heap-allocating them —
  the kernel-mode stack is small and fixed-size (see `linux-kernel-internals-essentials`); a
  "just allocate it on the stack" instinct from user-space code is a stack-overflow bug here.
- **Assuming the requested size equals the allocated size** — use `ksize()` if your logic actually
  depends on how much memory a slab allocation gave you, rather than assuming it's exactly what
  you asked for.

## Debugging tools worth knowing exist

- `/proc/slabinfo`, `slabtop`, and the kernel-source `tools/vm/slabinfo` utility — inspect which
  slab caches are active and how much memory they're using.
- `/proc/buddyinfo` and `/proc/vmallocinfo` — free page-frame counts per zone/order, and active
  vmalloc-region allocations respectively.
- `kmemleak` — kernel memory leak detector; KASAN (Kernel Address Sanitizer) generally supersedes
  it for catching use-after-free/out-of-bounds bugs during development.

## Checklist

- [ ] Every allocation site uses `GFP_KERNEL` in sleepable process context and `GFP_ATOMIC` in any
      atomic/interrupt/spinlock-held context — never the reverse
- [ ] `kzalloc()`/`vzalloc()`/`kvzalloc()` preferred over their non-zeroing counterparts unless the
      caller immediately overwrites the entire buffer anyway
- [ ] Every allocation's return value is checked for `NULL` before use
- [ ] Array allocations use `kcalloc()`/`kvmalloc_array()`, never manual `num * size` multiplication
- [ ] No single `kmalloc()`/`kzalloc()` call requests anywhere near the ~4 MB page-allocator limit;
      switched to `vmalloc()`/`kvmalloc()` if it does
- [ ] `vmalloc()`-family calls only occur in sleepable process context, never atomic/interrupt context
- [ ] Memory that held sensitive data is freed with `kfree_sensitive()`, not plain `kfree()`
- [ ] Every error/early-return path in an init or setup function frees everything allocated before
      the failure point, in reverse order
- [ ] Custom `kmem_cache`s are reserved for structs allocated/freed at real frequency, not used
      reflexively in place of `kzalloc()`
