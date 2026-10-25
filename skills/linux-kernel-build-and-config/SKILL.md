---
name: linux-kernel-build-and-config
description: Build the Linux kernel from source end to end — obtaining/extracting the source tree, generating a .config (localmodconfig vs make menuconfig vs defconfig), building the image and modules (make -jN), installing modules (make modules_install), generating the initramfs and updating GRUB (make install), customizing GRUB defaults/timeout, and cross-compiling for another architecture (ARCH=/CROSS_COMPILE=, e.g. Raspberry Pi AArch64). Use when the user asks "how do I build a custom kernel," "how do I configure the kernel with menuconfig," "what's localmodconfig," "why won't my new kernel show up in GRUB," "what is initramfs/initrd for," "how do I cross-compile the kernel for ARM64," "make modules_install vs make install," or is troubleshooting a kernel build/boot failure. Distinct from linux-kernel-module-basics (building an out-of-tree .ko against an *already-built* kernel) — this skill is about building the kernel image itself.
---

# Linux Kernel Build and Configuration

Building a kernel is a fixed 7-step pipeline. Skipping or reordering steps is the top cause of
"I built it but it won't boot" — follow the sequence, don't shortcut it.

## The 7 steps

1. **Obtain the source tree** — a tagged release tarball from kernel.org, or `git clone` the
   mainline/stable tree. Match the source you build against the source your out-of-tree modules
   will later build against.
2. **Extract it** — `tar xf linux-6.1.25.tar.xz`. Set an env var for convenience, e.g.
   `export LKP_KSRC=~/kernel/linux-6.1.25` — every later step assumes `cd ${LKP_KSRC}` first.
3. **Configure it** — produce a `.config` file (see below).
4. **Build the image and modules** — `make -jN` (parallelize; N ≈ number of CPU cores, e.g.
   `nproc`).
5. **Install the modules** — `sudo make modules_install`.
6. **Generate the initramfs image and update the bootloader** — `sudo make install`.
7. **Customize GRUB** if needed (default entry, timeout) and reboot into the new kernel.

## Step 3 in detail: configuring the kernel

Never hand-edit `.config`. Three practical starting points, roughly in order of preference for
"I just want to boot the modules I already use":

**localmodconfig** — start from the running kernel's actual module list, so the build only
includes what this exact machine uses (dramatically shorter build):

```sh
lsmod > /tmp/lsmod.now
cp /boot/config-$(uname -r) .config       # seed from the running kernel's known-good config
make LSMOD=/tmp/lsmod.now localmodconfig  # trims .config down to just what's loaded
make oldconfig                            # resolve any newly-introduced config options
```

**make menuconfig** — the ncurses UI over Kconfig; use it after `localmodconfig` to tweak
specific options (it will ask about anything `oldconfig` couldn't resolve automatically):

```sh
make menuconfig
```

Inside it: `/` opens a live search across every Kconfig symbol (find `CONFIG_FOO` fast); `<Help>`
on a highlighted item shows the Kconfig help text — read it, it usually says exactly what you
need to know. A useful tweak while you're in there: set **Local version – append to kernel
release** (`CONFIG_LOCALVERSION`) to something like `-lkp-kernel`, so `uname -r` clearly
distinguishes your custom build from the distro kernel — this also keeps its `/lib/modules/<ver>/`
and GRUB entry separate rather than colliding with the stock kernel.

**defconfig** — a board/arch-provided known-good baseline (`make bcm2711_defconfig` for
Raspberry Pi 4, for example); the right starting point when cross-compiling for a specific board
rather than tuning your own running system.

Verify a specific option landed correctly with a direct grep, rather than trusting memory:

```sh
grep -E "CONFIG_LOCALVERSION|CONFIG_DYNAMIC_DEBUG" .config
```

## Step 4 in detail: building

```sh
make -j8                          # native build; adjust -jN to core count
make -j8 2>&1 | tee out.txt       # keep a full log — invaluable when a build fails deep in a subdir
```

Useful introspection targets that don't build anything:

```sh
make kernelrelease   # e.g. 6.1.25-lkp-kernel  (== `uname -r` after install)
make kernelversion   # e.g. 6.1.25
make image_name      # e.g. arch/x86/boot/bzImage
```

## Step 5 in detail: installing modules

```sh
sudo make modules_install
```

This copies every module built as `m` in `.config` (the `.ko` files, found anywhere under the
source tree via `find . -name "*.ko"`) into `/lib/modules/$(uname -r)/kernel/...`, optionally
signs them (if `CONFIG_MODULE_SIG=y`), then runs `depmod` to resolve inter-module dependencies.
`sudo` is required — the destination is root-owned.

To stage modules somewhere other than `/lib/modules/` (e.g. building a rootfs for an embedded
target — critical so you never clobber the *host's* own modules by mistake):

```sh
export STG_MYKMODS=../staging/rootfs/my_kernel_modules
make INSTALL_MOD_PATH=${STG_MYKMODS} modules_install   # sudo not needed if the path is user-writable
```

## Step 6 in detail: initramfs + bootloader

```sh
sudo make install
```

On x86/Ubuntu this one command does three things: copies `vmlinuz-<ver>`, `System.map-<ver>`,
and `config-<ver>` into `/boot`; builds and installs the **initramfs** image
(`/boot/initrd.img-<ver>`); and regenerates `/boot/grub/grub.cfg` so the new kernel appears as a
boot entry. Override the destination with `INSTALL_PATH` if not building for the running system.

**What initramfs is for and why you can't skip it**: at boot, the kernel is in RAM but the root
filesystem isn't mounted yet — and the driver module needed to mount it (say, `f2fs.ko`) lives
*inside* that same not-yet-mounted filesystem. Classic chicken-and-egg. initramfs solves it: it's
a small, separately-loaded cpio archive (`newc` format, gzip-compressed) containing just enough
(`/etc`, `/lib`, the needed `.ko` files, scripts) to bring up whatever's needed to mount the real
root filesystem. Conceptually:

```sh
find my_initramfs/ | sudo cpio -o --format=newc -R root:root | gzip -9 > initramfs.img
```

Controlled by `CONFIG_BLK_DEV_INITRD` (on by default). Skip it only when every driver the boot
path needs (storage controller, root filesystem type) is compiled directly into the kernel image
(`=y`, not `=m`) — otherwise the modules it needs are unreachable at the point they're needed.

## Step 7: customizing GRUB

Edit `/etc/default/grub`, then regenerate the config:

```sh
# /etc/default/grub
GRUB_TIMEOUT_STYLE=menu     # was GRUB_TIMEOUT_STYLE=hidden — actually show the menu
GRUB_TIMEOUT=3
GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.1.25-lkp-kernel"

sudo update-grub            # required after ANY edit to /etc/default/grub — it does not self-apply
```

`GRUB_DEFAULT=0` means "boot the first menu entry" — after installing a new kernel, that's often
no longer the one you want, hence explicitly naming it as above.

## Cross-compiling for another architecture (e.g. Raspberry Pi / AArch64)

Two environment variables drive it everywhere — the kernel Makefile, and any out-of-tree
module Makefile built against this kernel, both key off them:

```sh
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-   # note trailing hyphen; invoked as ${CROSS_COMPILE}gcc, etc.

make ARCH=arm64 bcm2711_defconfig         # board-specific baseline config
make ARCH=arm64 menuconfig                # tweak from there if needed
make -j8 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- all
```

Install cross-built modules to a staging root, never the host's `/lib/modules/`:

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
     INSTALL_MOD_PATH=${STG_MYKMODS} modules_install
```

## Common pitfalls

- Hand-editing `.config` directly instead of via `menuconfig`/`oldconfig` — dependent options get
  left in an inconsistent state; always let Kconfig resolve dependencies.
- Running `make menuconfig`/`make` as a fresh checkout without first seeding `.config` from either
  `localmodconfig` or a `defconfig` — you'll be asked hundreds of questions with no good defaults.
- Forgetting `sudo` on `modules_install`/`install` (fails opaquely on permission) — or, the reverse
  danger, forgetting `INSTALL_MOD_PATH` when cross-compiling and *actually* overwriting the
  host's own `/lib/modules/$(uname -r)/`, which can break the host.
- Editing `/etc/default/grub` and forgetting to run `sudo update-grub` afterward — the file alone
  has no effect until regenerated.
- Not setting `CONFIG_LOCALVERSION` — a custom kernel silently shares `/lib/modules/<ver>/` and
  the GRUB label with a same-numbered stock kernel, causing confusing collisions.
- Assuming `initramfs`/`initrd` is optional cruft and disabling `CONFIG_BLK_DEV_INITRD` — fine
  only if literally everything the boot path needs is built `=y`, not `=m`.
- Cross-compiling without exporting *both* `ARCH` and `CROSS_COMPILE` for every invocation
  (including `modules_install`) — a build that silently reverts to host-native tools mid-pipeline.
- Not parallelizing (`make` with no `-jN`) on a full kernel build — painfully slow for no benefit.

## Checklist

- [ ] `.config` was generated via `localmodconfig`/`defconfig` + `menuconfig`/`oldconfig`, never
      hand-edited
- [ ] `CONFIG_LOCALVERSION` set to something distinguishing this build from the stock kernel
- [ ] Build parallelized with `-jN` matched to available cores
- [ ] `modules_install` run with `sudo` (native) or with `INSTALL_MOD_PATH` pointed at a staging
      root (cross-compiling) — never cross-compiled modules installed straight into the host's
      `/lib/modules/`
- [ ] `sudo make install` completed (initramfs regenerated, GRUB updated) before rebooting
- [ ] `/etc/default/grub` changes followed by `sudo update-grub`
- [ ] `GRUB_DEFAULT` points at the intended kernel entry, not left at a stale index
- [ ] Cross-builds export both `ARCH` and `CROSS_COMPILE` consistently across every `make`
      invocation in the pipeline
