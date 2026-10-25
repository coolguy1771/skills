---
name: linux-kernel-module-basics
description: Write, build, load, and debug a basic Loadable Kernel Module (LKM) — init/exit functions, module_init()/module_exit(), the out-of-tree Makefile (obj-m, KDIR, make -C ... M=$(PWD)), insmod/rmmod/lsmod/modinfo/dmesg workflow, module_param(), MODULE_LICENSE()/EXPORT_SYMBOL(_GPL) licensing and taint rules, and module stacking. Use when the user asks "how do I write a kernel module," "why won't insmod load my .ko," "how do I pass parameters to a kernel module," "why is my kernel tainted," "what's the Makefile for an out-of-tree module," "how do I cross-compile a kernel module," or is scaffolding/reviewing a Linux LKM's init/exit routines, Makefile, or module_param() usage. Foundation for device-driver, filesystem, or any other out-of-tree kernel code.
---

# Linux Kernel Module Basics

A kernel module (`.ko`) is kernel code compiled out-of-tree and inserted into a live kernel at
runtime — it runs with full kernel privilege, has no `main()`, no libc, and no user-space
libraries. Every mistake below either fails silently, taints the kernel, or crashes the machine,
so treat these conventions as load-bearing, not stylistic.

## Minimal LKM skeleton

```c
// helloworld_lkm.c
#include <linux/init.h>
#include <linux/module.h>

MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Hello, world - a minimal LKM");
MODULE_LICENSE("Dual MIT/GPL");
MODULE_VERSION("0.1");

static int __init helloworld_lkm_init(void)
{
	pr_info("Hello, world\n");
	return 0;               /* 0 == success */
}

static void __exit helloworld_lkm_exit(void)
{
	pr_info("Goodbye, world\n");
}

module_init(helloworld_lkm_init);
module_exit(helloworld_lkm_exit);
```

Key points:

- **No `main()`.** `module_init()`/`module_exit()` macros register your entry/exit functions as
  the constructor/destructor-like hooks the LKM framework calls on insmod/rmmod.
- **`static int __init foo_init(void)`** and **`static void __exit foo_exit(void)`** is the
  required naming/signature convention: `static` (private to this module), no parameters.
- **`__init`/`__exit`** are linker-section attributes. `__init` code/data is freed after init
  runs exactly once (`free_initmem()`); with modules built statically into the kernel image (or
  module support disabled), `__exit` has no effect.
- **Omitting `module_exit()`** makes the module impossible to `rmmod` (barring
  `CONFIG_MODULE_FORCE_UNLOAD`, which is off by default) — always pair init with exit.

### The 0/-E return convention (init function only)

- Return `0` on success.
- Return a **negative errno** on failure (e.g. `return -ENOMEM;`). This value gets negated again
  by glibc glue on the syscall return path, and user-space `errno` ends up positive. Never return
  a positive value or omit the return — modern compilers flag a missing return, and older kernels
  would abruptly unload the module.
- If a function must return a pointer instead of an int, use `ERR_PTR(-EFOO)` /
  `IS_ERR(ptr)` / `PTR_ERR(ptr)` (from `<linux/err.h>`) rather than inventing a sentinel:

```c
struct mystruct *myfunc(void)
{
	struct mystruct *m = kzalloc(sizeof(*m), GFP_KERNEL);
	if (!m)
		return ERR_PTR(-ENOMEM);
	return m;
}
/* caller */
retp = myfunc();
if (IS_ERR(retp)) {
	stat = PTR_ERR(retp);   /* -ENOMEM */
	goto out_fail;
}
```

## The out-of-tree Makefile

```makefile
# Makefile
PWD  := $(shell pwd)
KDIR := /lib/modules/$(shell uname -r)/build/
obj-m += helloworld_lkm.o

all:
	make -C $(KDIR) M=$(PWD) modules
install:
	make -C $(KDIR) M=$(PWD) modules_install
clean:
	make -C $(KDIR) M=$(PWD) clean
```

- `obj-m` lists objects to build **as modules** (contrast `obj-y`, built into the kernel image).
- `-C $(KDIR)` re-enters the *kernel's own* top-level Makefile so the module is compiled with
  the exact same compiler flags/ABI as the running kernel — never invoke `gcc`/`clang` directly.
  `KDIR` is normally the `build` symlink under `/lib/modules/$(uname -r)/`, installed by the
  `linux-headers`/`kernel-headers` package.
- Always build via `make`, never by hand-invoking the compiler.

**Cross-compiling**: parameterize `KDIR` on `ARCH`, and pass `CROSS_COMPILE`:

```makefile
ifeq ($(ARCH),arm64)
KDIR ?= ~/rpi_work/kernel_rpi/linux   # target's kernel source/headers tree
endif
obj-m += ${FNAME_C}.o

all:
	make -C $(KDIR) M=$(PWD) modules
```
```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```
`KDIR` must point at the **target** kernel's source/headers tree, not the host's — a module
built against the wrong tree will fail to load (version/symbol mismatch).

## The dev loop: build, load, inspect, unload

```
make                                    # produces helloworld_lkm.ko
sudo insmod ./helloworld_lkm.ko         # requires root or CAP_SYS_MODULE
lsmod | grep helloworld_lkm             # confirm it's resident: name, size, use count
sudo dmesg | tail                       # see your printk output
sudo rmmod helloworld_lkm               # requires root; runs the exit function first
```

- `insmod`/`rmmod`/`lsmod`/`modinfo`/`modprobe`/`depmod` are all symlinks to the single `kmod`
  binary. `insmod` invokes the `finit_module()` syscall; `rmmod` invokes `delete_module()`.
- `insmod` fails when: `CONFIG_MODULES=n`, caller lacks root/`CAP_SYS_MODULE` (`EPERM`),
  `/proc/sys/kernel/modules_disabled == 1`, or a module of the same name is already loaded
  (`EEXIST`).
- `rmmod` fails when: caller lacks permission (`EPERM`), the module's use count is nonzero
  because something still depends on it or has it open (`EBUSY`), or it has no exit routine and
  forced unload is disabled.
- `lsmod` columns: name, resident size in bytes, use count, and (4th, optional) the list of
  modules that depend on it.
- Loading an out-of-tree/unsigned module taints the kernel (`dmesg` shows "loading out-of-tree
  module taints kernel") — expected and generally harmless in development.

## Logging: printk and the pr_*() macros

Never use `printf`/libc — there is no libc in kernel space. Use `printk()` or, preferably, the
`pr_<level>()` convenience macros:

```c
pr_emerg(...)  pr_alert(...)  pr_crit(...)  pr_err(...)
pr_warn(...)   pr_notice(...) pr_info(...)  pr_debug(...)
```

These expand to `printk(KERN_<LEVEL> pr_fmt(fmt), ...)`. Always terminate a printk/pr_* format
string with `\n` — unterminated output can appear to "not take effect." `pr_cont()` continues the
previous line without its own level prefix (only the final segment should carry `\n`). Driver
code should prefer `dev_*()` variants (`dev_info(dev, ...)`, etc.) — they auto-prefix device
identity, which is essential once you have more than one instance of a driver.

Output goes to (up to) three places: the in-RAM ring buffer (`dmesg`), a non-volatile system log
(`journalctl -k`, or `/var/log/syslog`/`messages`), and the console. The ring buffer is finite
(default ~256 KB via `CONFIG_LOG_BUF_SHIFT`) and volatile — it does not survive a crash or power
cycle, so don't rely on it alone for post-mortem debugging of a panic.

## Module parameters

Expose tunables as `insmod`-time (and, if writable, runtime) parameters instead of hardcoding:

```c
static int mp_debug_level;
module_param(mp_debug_level, int, 0660);
MODULE_PARM_DESC(mp_debug_level, "Debug level [0-2]");

static char *mp_strparam = "default";
module_param(mp_strparam, charp, 0660);
MODULE_PARM_DESC(mp_strparam, "A demo string parameter");
```

- `module_param(name, type, perm)`: `name` must be `static`; `type` is one of `byte, hexint,
  short, ushort, int, uint, long, ulong, charp, bool, invbool`; `perm` is the octal sysfs
  permission (`0` = no sysfs node; nonzero creates `/sys/module/<mod>/parameters/<name>`,
  readable/writable per the mode — use octal literals like `0660`, not symbolic `S_IRUSR|...`,
  per kernel `checkpatch.pl` style).
- Pass values at load time: `sudo insmod ./mod.ko mp_debug_level=2 mp_strparam="hi"`.
- If `perm` is nonzero, the parameter can be read/changed live: `cat`/`echo` against
  `/sys/module/<mod>/parameters/<name>` (root only by default).
- **Don't initialize a static parameter variable to `0` explicitly** (`static int x = 0;`) —
  kernel style (`checkpatch.pl`) flags this; rely on default static zero-initialization.
- There is no `(argc, argv)` — parameters are wired via linker "trickery" over the declared
  static globals, not a conventional parameter list.
- To make a parameter mandatory, check it against its default/sentinel value in `init` and
  `return -EINVAL` if unset/out of range — there's no built-in "required" flag.
- `modinfo -p ./mod.ko` lists a module's parameters, types, and descriptions without needing the
  source.
- `module_param_array()` handles arrays; `module_param_named()` lets the sysfs/insmod-visible
  name differ from the internal variable name.

## Licensing, taint, and symbol visibility

- `MODULE_LICENSE()` is metadata for the module loader/tooling, **not** a substitute for a real
  SPDX license header. Valid free-software idents include `"GPL"`, `"GPL v2"`,
  `"Dual BSD/GPL"`, `"Dual MIT/GPL"`, `"Dual MPL/GPL"`; `"Proprietary"` marks a non-free module.
- A module without a recognized free-software license is flagged proprietary and **cannot call
  symbols exported via `EXPORT_SYMBOL_GPL()`** — only `EXPORT_SYMBOL()` symbols are visible to
  it. This is the mechanism, not a courtesy: get the license string right before debugging a
  mysterious "unknown symbol" load failure.
- Loading an out-of-tree or unsigned module taints the kernel; this is expected in development
  and does not itself indicate a bug.
- To expose your own module's symbols to other modules ("module stacking" — a library-like
  pattern), mark them with `EXPORT_SYMBOL(sym);` (or `EXPORT_SYMBOL_GPL(sym);` to restrict to
  GPL-licensed consumers) right after the definition. An un-exported symbol is invisible outside
  its own module, full stop.

## Common pitfalls

- Forgetting to check `kmalloc`/allocation return values in `init` and returning `0` anyway —
  always propagate `-ENOMEM` (or the relevant `-E*`) and let cleanup run.
- Missing `module_exit()` — makes the module effectively permanent until reboot.
- Hand-invoking `gcc` instead of `make` — breaks compiler-flag parity with the running kernel and
  silently produces an incompatible `.ko`.
- Using symbolic permissions (`S_IRUSR|S_IWUSR`) in `module_param()` — kernel style wants octal.
- Assuming `malloc`/`printf`-style semantics carry over — kernel space has neither; use
  `kmalloc`/`kzalloc` (see `linux-kernel-memory-allocation`) and `pr_*()`.
- Building against the host's `KDIR` when cross-compiling for a different target kernel.
- Treating `MODULE_LICENSE("GPL")` as decorative — it gates access to `EXPORT_SYMBOL_GPL()` API
  surface at link/load time.

## Checklist

- [ ] `init` returns `0`/`-Eerrno`; `exit` is `void` and always paired with `module_init()`
- [ ] Built via the provided `Makefile` (`obj-m`, `KDIR` pointing at the correct target kernel
      tree), never by invoking the compiler directly
- [ ] All logging uses `pr_*()`/`dev_*()`, never `printf`, and every format string ends in `\n`
- [ ] Every `module_param()` uses octal permissions and has a matching `MODULE_PARM_DESC()`
- [ ] `MODULE_LICENSE()` correctly reflects intent — proprietary modules cannot use
      `EXPORT_SYMBOL_GPL()` symbols
- [ ] Verified with the real dev loop: `make` → `insmod` → `lsmod` → `dmesg` → `rmmod`, not just
      "it compiled"
