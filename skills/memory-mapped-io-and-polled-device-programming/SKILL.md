---
name: memory-mapped-io-and-polled-device-programming
description: Reason about or write low-level I/O device programming — memory-mapped vs. port-mapped I/O, why I/O device timing differs from memory timing, polled I/O loops, GPIO register manipulation, and mapping physical device registers into a userspace process on Linux/ARM (/dev/mem + mmap). Use this when the user asks "why can't I just read/write an I/O device like memory," "what's the difference between polling and interrupts for a device," "how do I toggle a GPIO pin from C or assembly," "how do I access hardware registers directly from userspace on Linux," "what does /dev/mem + mmap do," "why do I need to check a status register before reading/writing a device," is writing a driver-adjacent program for a Raspberry Pi or other embedded Linux/ARM board, is debugging why a device write "did nothing" until polled/ready, or is choosing between polled I/O, interrupt-driven I/O, and DMA for a device interface. Distinct from arm64-macros-strings-and-project-structure (raw Linux syscalls / freestanding programs in general) — this is specifically about the hardware I/O programming model (registers, timing, polling, memory mapping), not general syscall/ABI mechanics.
---

# Memory-Mapped I/O and Polled Device Programming

I/O devices are not memory: they're orders of magnitude slower and, unlike memory, their timing
is unpredictable and event-driven rather than uniform. Every I/O programming technique — polling,
interrupts, DMA — exists to reconcile a fast, synchronous CPU with slow, asynchronous devices.
Getting this wrong looks like "my write to the device didn't take effect" or "my program hangs
waiting on a device that's actually fine."

## Why I/O isn't "just memory with a different address"

Memory access timing is uniform and independent of external events, so a CPU can issue a load or
store and know (architecturally) that it will complete in bounded time — no explicit check is
needed. I/O devices break this assumption:

- A keyboard at 120 words/minute produces one character roughly every 100ms — during which a
  2GHz CPU could execute on the order of 200 million instructions.
- A typical SSD moves data around 500MB/s; DDR4 main memory moves data around 20GB/s — roughly
  40x faster.
- Device timing also *varies* far more than memory: a keystroke's inter-arrival time is
  inconsistent; a disk's next-sector-ready time depends on rotational position.

Because of this, a program must never assume an I/O register access "just works" the way a memory
access does — it has to actively coordinate with the device's own readiness state.

## The device-controller register model

Software talks to a device through its controller (the electronics that do the actual work),
which typically exposes three kinds of registers:

- **Data** — the payload byte(s)/word(s) being sent to an output device or read from an input
  device.
- **Status** — readiness/error information about the controller and device.
- **Control** — commands from software to change the controller's or device's behavior (e.g.,
  "output now," "start read").

The universal I/O pattern is: **check status, then touch data, then (if required) tell the
controller to act via a control register.** Skipping the status check is the single most common
source of "my I/O didn't work" bugs — you wrote to a data register the controller wasn't ready to
consume, or read stale/garbage data the controller hadn't refreshed yet.

## Memory-mapped I/O vs. port-mapped I/O

Two ways the CPU can reach a device controller's registers:

- **Memory-mapped I/O**: a range of the normal memory address space is dedicated to device
  registers. Ordinary load/store instructions (`ldr`/`str`) read and write them — no special
  instructions needed. AArch64 supports **only** this scheme.
- **Port-mapped I/O**: device registers live in a separate address space reached only by special
  I/O instructions (`in`/`out` on x86). AArch64 has no equivalent; this is mentioned only because
  x86-derived code/documentation may assume it exists.

Because AArch64 is memory-mapped-only, device registers can usually be manipulated from
higher-level languages (C) without inline assembly — you just need the correct *virtual* address
of the mapped device register range, which is where userspace mapping comes in.

## Mapping device registers into a userspace process (Linux/ARM)

The kernel owns the *physical* addresses where device registers actually live; a userspace
program only ever sees *virtual* addresses. `/dev/mem` is a special file that is an image of
physical memory — opening it and `mmap`-ing a range gives userspace direct access to physical
addresses, including device registers, if the process has sufficient privilege (root):

```asm
// Open /dev/mem read/write with O_SYNC (so writes complete in hardware, not just
// cached), then mmap a window over the peripheral base address.
mov  w1, OPEN_FLAGS & 0xffff      // O_RDWR | O_SYNC, built via movz/movk
movk w1, OPEN_FLAGS / 0x10000, lsl 16
adr  x0, dev_mem_path             // "/dev/mem"
bl   open
cmp  w0, -1
b.eq error_return
mov  w19, w0                       // save fd

movz x5, PERIPHS & 0xffff, lsl 16  // SoC-specific peripheral physical base
movk x5, PERIPHS / 0x10000, lsl 32 //   (varies by chip revision/model!)
mov  w4, w19                       // fd
mov  w3, MAP_SHARED
mov  w2, PROT_RDWR                 // PROT_READ | PROT_WRITE
mov  w1, MEM_SIZE                  // bytes to map (cover all needed device regs)
mov  w0, NO_ADDR_PREF              // let the kernel choose the virtual address
bl   mmap
cmp  x0, -1
b.eq error_return
mov  x20, x0                       // save mapped virtual base address
mov  w0, w19
bl   close                         // fd no longer needed once mapped

// Device registers are now reachable at x20 + <device-specific offset>, using
// ordinary ldr/str just like normal memory.
```

The C equivalent (`open("/dev/mem", O_RDWR|O_SYNC)` then
`mmap(NULL, len, PROT_READ|PROT_WRITE, MAP_SHARED, fd, phys_addr)`) is the same idea and is what
most GPIO/register-poking libraries do under the hood. Key details that bite people:

- The peripheral base physical address is **chip-specific** (it changed across Raspberry Pi
  generations, for example) — hardcoding one board's base address silently breaks on another.
  Look it up from the SoC's datasheet/peripheral map, not from example code for a different board.
- This requires root, because writable access to raw physical memory bypasses all the OS's normal
  memory protection between processes.
- Always `munmap` when done, and always check `mmap`'s return against `-1` (technically
  `MAP_FAILED`), not `NULL` — 0 can be a legitimate mapped address on some setups.
- Register offsets and bit layouts (e.g., which bits in a function-select register choose a
  GPIO line's mode) come from the SoC datasheet, not from general ARM documentation — GPIO
  register layout is a peripheral-vendor detail, not part of the AArch64 ISA.

## Polled I/O

When a device controller needs processing time (the common case — anything more complex than a
single always-ready output line), software uses a **polling loop**: repeatedly read the status
register until it reports "ready," then touch the data register.

```asm
poll_loop:
    ldr   w0, [x_ctrl_base, STATUS_OFFSET]   // read status register
    tst   w0, READY_BIT
    beq   poll_loop                           // not ready yet — spin
    ldr   w1, [x_ctrl_base, DATA_OFFSET]     // now safe to read data
```

This is called **polled I/O** or **programmed I/O**, and its downside is exactly what the spin
loop shows: the CPU is fully occupied doing nothing useful while it waits, which is wasteful in
any environment running more than one program. It's acceptable for a CPU dedicated to one task
(a microwave oven controller) and unacceptable in a general multiprogrammed OS.

A GPIO line configured purely as an always-ready digital output (e.g., driving an LED) is the
degenerate case: there's nothing to poll because the line is always immediately ready, so a bare
`str` to its set/clear register is sufficient — no status check needed. That's what makes GPIO
blinking a good minimal example, but don't generalize "no polling needed" to devices that
actually buffer or process data.

## Interrupt-driven I/O and DMA (when polling doesn't scale)

- **Interrupt-driven I/O**: the device signals the CPU (an interrupt) only when it's actually
  ready, letting the CPU run other work in between instead of spinning. The CPU's response to the
  interrupt (jumping to a handler, changing privilege level) is covered by
  `arm64-exception-levels-and-interrupt-handling`.
- **DMA (direct memory access)**: for high-throughput devices (disks, etc.), a DMA controller
  moves data directly between the device and main memory *without* the CPU touching each byte;
  the CPU only sets up the transfer and gets one interrupt when it's complete. This removes both
  the polling overhead and the per-byte CPU involvement that interrupt-driven I/O still has.

Choose polled I/O only for simple, always-ready, or latency-insensitive single-purpose control;
prefer interrupt-driven I/O once the CPU has other work to do while waiting; reach for DMA once
per-byte CPU involvement itself becomes the bottleneck (bulk transfers).

## Common pitfalls

- **Reading/writing a device data register without checking status first**, on a controller that
  actually needs processing time — this reads stale data or clobbers a value the controller
  hasn't consumed yet.
- **Hardcoding a peripheral base physical address from example code** without verifying it
  matches your specific SoC/board revision — this is one of the most common "worked on my Pi 3,
  didn't work on my Pi 4/5" bugs.
- **Forgetting `O_SYNC` (or equivalent) when opening `/dev/mem`**, which can let writes sit in a
  cache instead of reaching the device immediately.
- **Busy-polling a device from a general-purpose multiprogrammed environment** instead of using
  interrupts, starving other work of the CPU.
- **Treating a GPIO line configured as instantly-ready output as the general model for I/O** —
  most real devices (UART, disk, network) need genuine status polling or interrupts; a blinking
  LED example is deliberately the simplest possible case.
- **Not unmapping (`munmap`) or leaking the `/dev/mem` file descriptor** in long-running
  programs that repeatedly map/unmap device regions.

## Checklist

- [ ] Confirmed whether the target CPU/ISA uses memory-mapped I/O, port-mapped I/O, or both
      before assuming `in`/`out`-style instructions exist
- [ ] Identified the device's status/data/control register layout from its actual datasheet, not
      copied blindly from a different board/chip revision
- [ ] Added a status-register check before every data register access on any device that isn't
      provably always-ready
- [ ] Chose polled vs. interrupt-driven vs. DMA based on how much idle CPU time spinning would
      waste and how much data volume is involved
- [ ] `/dev/mem` mapping code checks `mmap`'s return against failure correctly and calls
      `munmap` when finished
- [ ] Verified the peripheral base address and register offsets against the specific SoC's
      datasheet, especially when porting code between board revisions
