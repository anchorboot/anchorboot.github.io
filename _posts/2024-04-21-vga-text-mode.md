---
layout: post
title: "VGA Text Mode on ARM & RISC-V?"
author: "Alper Nebi Yasak"
date: 2024-04-21 17:00:00 +0300
categories: posts
tags: posts
---

I admit the title sounds like a bad joke. Who would want 1980s legacy
VGA on brand-new architectures? I don't particularly care for it. But
what I have been trying to do is enable coreboot support for QEMU
emulated display devices on architectures other than x86. This somehow
led me to experiment on VGA-related parts of coreboot where I managed
to get these displays working in text mode. 

This article is kind of a retrospective development log to document the
weird things I had to do and the interesting visuals that popped up, but
also serves as a long-form cover letter for patches that I should be
sending to coreboot upstream. I'm not the expert on any of this, so take
my words with a pinch of salt.


VGA on QEMU x86
---------------

I'm not going to go into [VGA](https://en.wikipedia.org/wiki/Video_Graphics_Array)
details here, but I want to at least visualize the state of affairs in
upstream coreboot to provide context. While we are configuring a
coreboot build, we can choose how we want our board's display to work,
either in VGA text mode or as a linear framebuffer.

<a href="/assets/vga-text/corebootfb-kconfig.png">
  <img src="/assets/vga-text/corebootfb-kconfig.png" alt="coreboot Kconfig choice for framebuffer mode">
</a>

Although coreboot may initialize the display, it doesn't really do much
with it itself. The point is to do just enough so that a payload like
SeaBIOS or TianoCore EDK II can use the display without much work.
For example, we can buid coreboot with SeaBIOS and invoke QEMU with
`-vga std` for a standard VGA display.

```sh
$ qemu-system-i386 \
     -display gtk,show-tabs=on \
     -bios build/coreboot.rom \
     -M pc -m 2G \
     -serial stdio \
     -vga std
```

<a href="/assets/vga-text/qemu-seabios.png">
  <img src="/assets/vga-text/qemu-seabios.png" alt="QEMU displaying SeaBIOS using VGA text mode">
</a>

With a linear framebuffer, we can set a splash screen image that
coreboot will render after initializing the display. Payloads will also
use the screen, so our splash image might show only for a split second.
It is easier to demonstrate that display initialization is working if we
build coreboot without a payload.

<a href="/assets/vga-text/qemu-bootsplash.png">
  <img src="/assets/vga-text/qemu-bootsplash.png" alt="QEMU displaying a splash image in linear framebuffer mode">
</a>

One thing I wanted to do is to make coreboot itself print something to
the VGA text mode display, so that I don't need to rely on payload
availability on other architectures to test if it works. I don't see a
config option that would print anything to the VGA text mode display,
but I could modify the VGA text mode initialization code to print some
text.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index 7f8ce6979582..eadb8ca6ac6b 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -341,6 +341,13 @@ vga_textmode_init(void)
 	vga_fb_clear();
 	vga_font_8x16_load();
 
+	const unsigned char *test_str = (const unsigned char *)"[VGA TEXT]";
+	vga_write_text(VGA_TEXT_LEFT, 0, test_str);
+	vga_write_text(VGA_TEXT_RIGHT, 0, test_str);
+	vga_write_text(VGA_TEXT_CENTER, VGA_LINES / 2, test_str);
+	vga_write_text(VGA_TEXT_LEFT, VGA_LINES - 1, test_str);
+	vga_write_text(VGA_TEXT_RIGHT, VGA_LINES - 1, test_str);
+
 	vga_sr_mask(0x00, 0x02, 0x02); /* take us out of reset */
 	vga_cr_mask(0x17, 0x80, 0x80); /* sync! */
 }
```

<a href="/assets/vga-text/qemu-vgatext.png">
  <img src="/assets/vga-text/qemu-vgatext.png" alt="QEMU displaying text from coreboot in VGA text mode">
</a>

All good, but the coreboot drivers for QEMU emulated display devices are
written for and restricted to x86 builds, so we can't use them like this
on others yet. We can still run QEMU with a display device, but it shows
a warning that the "guest" (coreboot in this case) has not initialized
the device. One more quirk is that we need `-device VGA` instead of
`-vga std` to attach the display device.

```sh
$ qemu-system-aarch64 \
    -display gtk,show-tabs=on \
    -bios build/coreboot.rom \
    -M virt,secure=on,virtualization=on \
    -cpu cortex-a72 -m 2G \
    -serial stdio \
    -device VGA
```

<a href="/assets/vga-text/qemu-uninitialized.png">
  <img src="/assets/vga-text/qemu-uninitialized.png" alt="QEMU warning that the display is not initialized yet">
</a>

It's possible for whatever payload that runs after coreboot to
initialize the display itself, but that requires the relevant drivers to
be implemented in the payload for each display device.


Bochs Display
-------------

Most other boards that I'm interested in have display initialization
already implemented in coreboot and there is a mechanism for passing a
working framebuffer for a payload to use. U-Boot supports this for x86,
but I need to extend that to other architectures, and I need at least
one QEMU display device working in coreboot to test my changes on QEMU.

There are already coreboot drivers for some QEMU display devices. The
build configuration has a single option to enable both and only allows
us to choose it on x86.

<a href="/assets/vga-text/coreboot-kconfig-bochs.png">
  <img src="/assets/vga-text/coreboot-kconfig-bochs.png" alt="coreboot kconfig help message for Bochs display">
</a>

Contrary to the configuration option's help message, the option enables
two drivers for Bochs and Cirrus emulated display devices and Cirrus is
no longer the default in QEMU.

Removing the config dependency on x86 is simple, but it results in a
build error giving us a hint at the first big problem we need to solve.

```diff
diff --git a/src/drivers/emulation/qemu/Kconfig b/src/drivers/emulation/qemu/Kconfig
index 11231ae52ee6..3cc47ff941f8 100644
--- a/src/drivers/emulation/qemu/Kconfig
+++ b/src/drivers/emulation/qemu/Kconfig
@@ -3,7 +3,6 @@
 config DRIVERS_EMULATION_QEMU_BOCHS
 	bool "bochs dispi interface vga driver"
 	default y
-	depends on CPU_QEMU_X86
 	depends on MAINBOARD_DO_NATIVE_VGA_INIT
 	select HAVE_VGA_TEXT_FRAMEBUFFER
 	select HAVE_LINEAR_FRAMEBUFFER
```

```
    CC         ramstage/drivers/emulation/qemu/bochs.o
src/drivers/emulation/qemu/bochs.c:3:10: fatal error: arch/io.h: No such file or directory
    3 | #include <arch/io.h>
      |          ^~~~~~~~~~~
compilation terminated.
make: *** [Makefile:419: build/qemu_arm64/ramstage/drivers/emulation/qemu/bochs.o] Error 1
```

[This `<arch/io.h>`](https://github.com/coreboot/coreboot/blob/24.02.01/src/arch/x86/include/arch/io.h)
provides functions that compile to CPU instructions that send and
receive data over I/O ports on x86 systems, like `inb()` and `outb()`.
Looking at [that `bochs.c`](https://github.com/coreboot/coreboot/blob/24.02.01/src/drivers/emulation/qemu/bochs.c),
it seemed like there were an alternative to do whatever they are used
for, so I tried to naively skip calling them and see what happens.

```diff
diff --git a/src/drivers/emulation/qemu/bochs.c b/src/drivers/emulation/qemu/bochs.c
index 06504309e071..77d290f95b27 100644
--- a/src/drivers/emulation/qemu/bochs.c
+++ b/src/drivers/emulation/qemu/bochs.c
@@ -1,6 +1,10 @@
 /* SPDX-License-Identifier: GPL-2.0-only */
 
+#if CONFIG(CPU_QEMU_X86)
 #include <arch/io.h>
+#else
+#include <arch/mmio.h>
+#endif
 #include <console/console.h>
 #include <device/device.h>
 #include <device/mmio.h>
@@ -45,8 +49,10 @@ static int height = CONFIG_DRIVERS_EMULATION_QEMU_BOCHS_YRES;
 static void bochs_write(struct resource *res, int index, int val)
 {
 	if (res->flags & IORESOURCE_IO) {
+#if CONFIG(CPU_QEMU_X86)
 		outw(index, res->base);
 		outw(val, res->base + 1);
+#endif
 	} else {
 		write16(res2mmio(res, 0x500 + index * 2, 0), val);
 	}
@@ -55,8 +61,12 @@ static void bochs_write(struct resource *res, int index, int val)
 static int bochs_read(struct resource *res, int index)
 {
 	if (res->flags & IORESOURCE_IO) {
+#if CONFIG(CPU_QEMU_X86)
 		outw(index, res->base);
 		return inw(res->base + 1);
+#else
+		return 0;
+#endif
 	} else {
 		return read16(res2mmio(res, 0x500 + index * 2, 0));
 	}
@@ -64,9 +74,11 @@ static int bochs_read(struct resource *res, int index)
 
 static void bochs_vga_write(struct resource *res, int index, uint8_t val)
 {
-	if (res->flags & IORESOURCE_IO)
+	if (res->flags & IORESOURCE_IO) {
+#if CONFIG(CPU_QEMU_X86)
 		outb(val, index + 0x3c0);
-	else
+#endif
+	} else
 		write8(res2mmio(res, 0x400 + index, 0), val);
 }
 
```

```
    CC         ramstage/drivers/emulation/qemu/bochs.o
    CC         ramstage/drivers/emulation/qemu/cirrus.o
src/drivers/emulation/qemu/cirrus.c: In function 'write_hidden_dac':
src/drivers/emulation/qemu/cirrus.c:176:9: error: implicit declaration of function 'inb'; did you mean 'isb'? [-Werror=implicit-function-declaration]
  176 |         inb(0x3c8);
      |         ^~~
      |         isb
src/drivers/emulation/qemu/cirrus.c:181:9: error: implicit declaration of function 'outb' [-Werror=implicit-function-declaration]
  181 |         outb(data, 0x3c6);
      |         ^~~~
cc1: all warnings being treated as errors
make: *** [Makefile:419: build/qemu_arm64/ramstage/drivers/emulation/qemu/cirrus.o] Error 1
```

The Bochs driver compiles with that much. But [the `cirrus.c` file](https://github.com/coreboot/coreboot/blob/24.02.01/src/drivers/emulation/qemu/bochs.c)
also has the same problem, except with a lot more I/O calls and no
apparent alternative for what it is using those calls for. I decided to
just focus on Bochs for now, achieved simply by not building the Cirrus
code.

```diff
diff --git a/src/drivers/emulation/qemu/Makefile.mk b/src/drivers/emulation/qemu/Makefile.mk
index c9d94bdca0c2..186067e7d49f 100644
--- a/src/drivers/emulation/qemu/Makefile.mk
+++ b/src/drivers/emulation/qemu/Makefile.mk
@@ -6,4 +6,3 @@ postcar-$(CONFIG_CONSOLE_QEMU_DEBUGCON) += qemu_debugcon.c
 ramstage-$(CONFIG_CONSOLE_QEMU_DEBUGCON) += qemu_debugcon.c
 
 ramstage-$(CONFIG_DRIVERS_EMULATION_QEMU_BOCHS) += bochs.c
-ramstage-$(CONFIG_DRIVERS_EMULATION_QEMU_BOCHS) += cirrus.c
```

```
Built emulation/qemu-aarch64 (QEMU AArch64)

        ** WARNING **
coreboot has been built without a payload. Writing
a coreboot image without a payload to your board's
flash chip will result in a non-booting system. You
can use cbfstool to add a payload to the image.
```

That gets us a display device working in linear framebuffer mode if we
use `-device bochs-display` or `-device secondary-vga` when invoking
QEMU.

Unfortunately it does not work with `-device VGA`. Looking closer, that
makes the driver try to use the I/O functions that I remove above, so
that is to be expected. We can force it to prefer non-VGA mode for the
VGA device on other architectures.

```diff
diff --git a/src/drivers/emulation/qemu/bochs.c b/src/drivers/emulation/qemu/bochs.c
index 77d290f95b27..ac66dd2f03df 100644
--- a/src/drivers/emulation/qemu/bochs.c
+++ b/src/drivers/emulation/qemu/bochs.c
@@ -115,7 +115,7 @@ static void bochs_init_linear_fb(struct device *dev)
 
 	/* MMIO bar supported since qemu 3.0+ */
 	res_io = probe_resource(dev, PCI_BASE_ADDRESS_2);
-	if (((dev->class >> 8) == PCI_CLASS_DISPLAY_VGA) ||
+	if ((CONFIG(CPU_QEMU_X86) && ((dev->class >> 8) == PCI_CLASS_DISPLAY_VGA)) ||
 	    !res_io || !(res_io->flags & IORESOURCE_MEM)) {
 		printk(BIOS_DEBUG, "QEMU VGA: Using legacy VGA\n");
 		res_io = &res_legacy;
```

After that last modification, the display appears to work fine as a
linear framebuffer with `-device VGA`, confirmed by being able to show a
boot splash image. Obviously not in a VGA compliant way, but I assumed
the standard doesn't make sense in the non-x86 world and kept iterating
on the changes to the point I was fine with submitting them upstream.

However, the question of implementing I/O functions and VGA support kept
occupying my mind. I wouldn't need to restrict the code to avoid them if
I could implement them correctly and the result would be a lot cleaner.


Port I/O
--------

x86 architectures have an I/O address space distinct from the memory
address space and has explicit instructions to communicate to devices
mapped into this I/O space as "ports". This is in contrast to mapping
devices into the memory address space (MMIO) and treating them as if
they are a volatile part of memory. That's as far as I understood all
that.

I wanted to see if it was even possible to implement the I/O functions
on other architectures, and looked at `io.h` headers that I could find
[in U-Boot](https://source.denx.de/u-boot/u-boot/-/blob/v2024.04/arch/arm/include/asm/io.h)
and [in Linux](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/io.h?h=v6.8)
for a better understanding. In essence, those implementations are
calculating a memory address from a board-specific I/O base address and
the port's index, then reading from or writing to that memory address to
take input from or output to the port.

The obvious thing to try was to copy the U-Boot implementation into
coreboot. Sadly, I could not find the proper value for that I/O base
address by the time. I decided to [provide empty stubs for the I/O functions](https://review.coreboot.org/c/coreboot/+/80372/3)
instead of removing the calls with a preprocessor, in hope that someone
else can manage to fully implement them, and called it a day.

```c
static inline void outb(uint8_t value, uint16_t port)
{
	printk(BIOS_ERR, "arch/io.h: %s() not implemented\n", __func__);
}

static inline uint8_t inb(uint16_t port)
{
	printk(BIOS_ERR, "arch/io.h: %s() not implemented\n", __func__);
	return 0;
}
```

The final piece I needed was revealed by a review comment pointing me to
[the `VIRT_PCIE_PIO` value](https://gitlab.com/qemu-project/qemu/-/blob/v8.2.2/hw/arm/virt.c#L164)
from QEMU sources as the proper one to use.

I didn't understand what purpose those I/O functions would serve until
then. PCI also has the concept of an I/O address space, which appears to
be the same as x86 one. CPUs of other architectures still need to talk
to PCI devices over the PCI I/O space, which they achieve through
something mapped to memory that handles the translation. These I/O
functions talk to that something.

I tinkered a bit more with what I could copy from U-Boot. But I didn't
really have a known-good way to test what I had. After all, I was adding
them for the QEMU display drivers and those would be the first users, I
needed to work on both sides at the same time. Simplifying the I/O code
was the less interesting part, so let me skip to the end result for
ARM64 `io.h` below.

```c
/* SPDX-License-Identifier: GPL-2.0-only */

/*
 * I/O device access primitives. Simplified based on related U-Boot code,
 * which is in turn based on early versions from the Linux kernel:
 *
 *   Copyright (C) 1996-2000 Russell King
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License version 2 as
 * published by the Free Software Foundation.
 */

#ifndef __ARCH_IO_H__
#define __ARCH_IO_H__

#include <stdint.h>
#include <stddef.h>
#include <endian.h>
#include <arch/mmio.h>

/*
 * From QEMU hw/arm/virt.c:
 *   [VIRT_PCIE_PIO] =           { 0x3eff0000, 0x00010000 },
 */
#define PCI_PIO_BASE    0x3eff0000
#define PCI_PIO_LIMIT   ((size_t)0xffff)
#define __io(a)         (void *)(uintptr_t)(PCI_PIO_BASE + ((a) & PCI_PIO_LIMIT))

static inline void outb(uint8_t value, uint16_t port)
{
	write8(__io(port), value);
}

static inline void outw(uint16_t value, uint16_t port)
{
	write16(__io(port), cpu_to_le16(value));
}

static inline void outl(uint32_t value, uint16_t port)
{
	write32(__io(port), cpu_to_le32(value));
}

static inline uint8_t inb(uint16_t port)
{
	return read8(__io(port));
}

static inline uint16_t inw(uint16_t port)
{
	return le16_to_cpu(read16(__io(port)));
}

static inline uint32_t inl(uint16_t port)
{
	return le32_to_cpu(read32(__io(port)));
}

static inline void outsb(uint16_t port, const void *addr, unsigned long count)
{
	uint8_t *buf = (uint8_t *)addr;
	while (count--)
		write8(__io(port), *buf++);
}

static inline void outsw(uint16_t port, const void *addr, unsigned long count)
{
	uint16_t *buf = (uint16_t *)addr;
	while (count--)
		write16(__io(port), *buf++);
}

static inline void outsl(uint16_t port, const void *addr, unsigned long count)
{
	uint32_t *buf = (uint32_t *)addr;
	while (count--)
		write32(__io(port), *buf++);
}

static inline void insb(uint16_t port, void *addr, unsigned long count)
{
	uint8_t *buf = (uint8_t *)addr;
	while (count--)
		*buf++ = read8(__io(port));
}

static inline void insw(uint16_t port, void *addr, unsigned long count)
{
	uint16_t *buf = (uint16_t *)addr;
	while (count--)
		*buf++ = read16(__io(port));
}

static inline void insl(uint16_t port, void *addr, unsigned long count)
{
	uint32_t *buf = (uint32_t *)addr;
	while (count--)
		*buf++ = read32(__io(port));
}

#endif
```

I'm not sure everything here is correct, but it was enough to get the
rest of my display experiments working. This `PCI_PIO_BASE` definitely
needs to be configurable since it depends on the board we are compiling
for, but I don't see why it wouldn't work as an architecture-generic
implementation otherwise.


VGA Display
-----------

At this point, I had the port I/O implementation I copied from U-Boot
and the Bochs display driver as the thing to test it on, working on both
at the same time. But reverting the previous hacks that avoid port I/O
and then invoking QEMU with `-device VGA` wasn't enough make it work
immediately.

```
[DEBUG]  PCI: 00:00:02.0 init
[DEBUG]  QEMU VGA: Using legacy VGA
[DEBUG]  QEMU VGA: bochs dispi: ID mismatch.
[DEBUG]  PCI: 00:00:02.0 init finished in 0 msecs
```

The device ID check indeed uses the I/O functions, so my first instinct
was to look at what result I was getting and what it should've been.

```diff
diff --git a/src/drivers/emulation/qemu/bochs.c b/src/drivers/emulation/qemu/bochs.c
index 06504309e071..6a8ed6e8c159 100644
--- a/src/drivers/emulation/qemu/bochs.c
+++ b/src/drivers/emulation/qemu/bochs.c
@@ -115,6 +115,8 @@ static void bochs_init_linear_fb(struct device *dev)
 	id = bochs_read(res_io, VBE_DISPI_INDEX_ID);
 	if ((id & 0xfff0) != VBE_DISPI_ID0) {
 		printk(BIOS_DEBUG, "QEMU VGA: bochs dispi: ID mismatch.\n");
+		printk(BIOS_DEBUG, "QEMU VGA: expected %#x, got id=%#x\n",
+				   VBE_DISPI_ID0, id);
 		return;
 	}
 	mem = bochs_read(res_io, VBE_DISPI_INDEX_VIDEO_MEMORY_64K) * 64 * 1024;
```

```
[DEBUG]  PCI: 00:00:02.0 init
[DEBUG]  QEMU VGA: Using legacy VGA
[DEBUG]  QEMU VGA: bochs dispi: ID mismatch.
[DEBUG]  QEMU VGA: Expected 0xb0c0, got id=0xffff
[DEBUG]  PCI: 00:00:02.0 init finished in 0 msecs
```

I would've expected it to be zero if the I/O function wasn't doing
anything and just reading an unused address in memory. And this is
indeed the case if we change the `PCI_PIO_BASE` in the `io.h` to some
wrong value. Or in the worst case I guess it might've been random
data. If it looked random while being consistent across multiple runs,
I would've thought of endianness issues.

This `0xffff` result led me to look more into what the driver code is
doing and what it should be doing. Luckily, [QEMU VGA device documentation](https://www.qemu.org/docs/master/specs/standard-vga.html#io-ports-used)
had the answer I needed.

<a href="/assets/vga-text/qemu-doc-ioports.png">
  <img src="/assets/vga-text/qemu-doc-ioports.png" alt="QEMU documenting different IO ports for x86 and others">
</a>

The Bochs driver uses a data port at `01cf` for x86, but that is
unavailable for other architectures. We need to use the one at `01d0`.
There is a constant defined at the top of the Bochs driver, but the
value is also implicitly calculated again from the index port, so I had
to take care to update those as well.

```diff
diff --git a/src/drivers/emulation/qemu/bochs.c b/src/drivers/emulation/qemu/bochs.c
index 06504309e071..2ef9cfbc5fdc 100644
--- a/src/drivers/emulation/qemu/bochs.c
+++ b/src/drivers/emulation/qemu/bochs.c
@@ -14,7 +14,7 @@
 
 /* VGA init. We use the Bochs VESA VBE extensions  */
 #define VBE_DISPI_IOPORT_INDEX          0x01CE
-#define VBE_DISPI_IOPORT_DATA           0x01CF
+#define VBE_DISPI_IOPORT_DATA           0x01D0
 
 #define VBE_DISPI_INDEX_ID              0x0
 #define VBE_DISPI_INDEX_XRES            0x1
@@ -46,7 +46,7 @@ static void bochs_write(struct resource *res, int index, int val)
 {
 	if (res->flags & IORESOURCE_IO) {
 		outw(index, res->base);
-		outw(val, res->base + 1);
+		outw(val, res->base + 2);
 	} else {
 		write16(res2mmio(res, 0x500 + index * 2, 0), val);
 	}
@@ -56,7 +56,7 @@ static int bochs_read(struct resource *res, int index)
 {
 	if (res->flags & IORESOURCE_IO) {
 		outw(index, res->base);
-		return inw(res->base + 1);
+		return inw(res->base + 2);
 	} else {
 		return read16(res2mmio(res, 0x500 + index * 2, 0));
 	}
```

```
[DEBUG]  PCI: 00:00:02.0 init
[DEBUG]  QEMU VGA: Using legacy VGA
[DEBUG]  QEMU VGA: bochs dispi interface found, 16 MiB video memory
[DEBUG]  QEMU VGA: framebuffer @ 3d000000 (pci bar 0)
[INFO ]  framebuffer_info: bytes_per_line: 3200, bits_per_pixel: 32
[INFO ]                     x_res x y_res: 800 x 600, size: 1920000 at 0x3d000000
[DEBUG]  PCI: 00:00:02.0 init finished in 1 msecs
         [...]
[INFO ]  Setting up bootsplash in 800x600@32
[DEBUG]  FMAP: area COREBOOT found @ Bochs 20200 (16645632 bytes)
[INFO ]  CBFS: Found 'bootsplash.jpg' @0x11500 size 0xa76c in mcache @0xbfffeb6c
[DEBUG]  Bootsplash image resolution: 256x256
[INFO ]  Bootsplash loaded
```

And we get a working display out of it! I'm assuming this time it's more
compliant with standard VGA, but who knows at this point?


VGA Text Mode
-------------

Well, the next fun step was to see if VGA text mode works with this
much. Changing the build configuration for text mode is easy, but it
causes a build error.

```
    CC         cbfs/fallback/ramstage.debug
.../bin/aarch64-elf-ld.bfd: warning: build/qemu_arm64/cbfs/fallback/ramstage.debug has a LOAD segment with RWX permissions
.../bin/aarch64-elf-ld.bfd: build/qemu_arm64/ramstage/drivers/emulation/qemu/bochs.o: in function `bochs_init_text_mode':
.../src/drivers/emulation/qemu/bochs.c:160:(.text.bochs_init+0x8): undefined reference to `vga_misc_write'
.../src/drivers/emulation/qemu/bochs.c:160:(.text.bochs_init+0x8): relocation truncated to fit: R_AARCH64_CALL26 against undefined symbol `vga_misc_write'
.../bin/aarch64-elf-ld.bfd: .../src/drivers/emulation/qemu/bochs.c:161:(.text.bochs_init+0x10): undefined reference to `vga_textmode_init'
.../src/drivers/emulation/qemu/bochs.c:161:(.text.bochs_init+0x10): relocation truncated to fit: R_AARCH64_JUMP26 against undefined symbol `vga_textmode_init'
make: *** [src/arch/arm64/Makefile.mk:136: build/qemu_arm64/cbfs/fallback/ramstage.debug] Error 1
```

Now we run deeper into our second big problem: VGA support itself.
Apparently I got lucky with the linear framebuffer mode, because
[coreboot's legacy VGA support code](https://github.com/coreboot/coreboot/tree/24.02.01/src/drivers/pc80/vga)
is restricted to build only on x86 and we haven't been using it at all.
We can enable building it, which exchanges one build error for another.

```diff
diff --git a/src/drivers/pc80/vga/Makefile.mk b/src/drivers/pc80/vga/Makefile.mk
index 63ec6ba6ae7a..db103d82a996 100644
--- a/src/drivers/pc80/vga/Makefile.mk
+++ b/src/drivers/pc80/vga/Makefile.mk
@@ -1,7 +1,5 @@
 ## SPDX-License-Identifier: GPL-2.0-only
 
-ifeq ($(CONFIG_ARCH_X86),y)
-
 romstage-$(CONFIG_ROMSTAGE_VGA) += vga_io.c
 romstage-$(CONFIG_ROMSTAGE_VGA) += vga_palette.c
 romstage-$(CONFIG_ROMSTAGE_VGA) += vga_font_8x16.c
@@ -11,5 +9,3 @@ ramstage-$(CONFIG_VGA) += vga_io.c
 ramstage-$(CONFIG_VGA) += vga_palette.c
 ramstage-$(CONFIG_VGA) += vga_font_8x16.c
 ramstage-$(CONFIG_VGA) += vga.c
-
-endif
```

```
src/drivers/pc80/vga/vga.c: In function 'vga_write_text':
src/drivers/pc80/vga/vga.c:286:1: error: stack usage is 2064 bytes [-Werror=stack-usage=]
  286 | vga_write_text(enum VGA_TEXT_ALIGNMENT alignment, unsigned int line,
      | ^~~~~~~~~~~~~~
cc1: all warnings being treated as errors
make: *** [Makefile:419: build/qemu_arm64/ramstage/drivers/pc80/vga/vga.o] Error 1
```

I didn't know why high stack usage is a problem, but there is an array in
the `vga_write_text()` function that can hold a string big enough to
fill the text mode screen. I assumed we don't need it specifically to be
on the stack and moved it out of the function. Making it `static` also
works.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index eadb8ca6ac6b..dcfbd29940ec 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -287,7 +287,8 @@ vga_write_text(enum VGA_TEXT_ALIGNMENT alignment, unsigned int line,
 	       const unsigned char *ustring)
 {
 	const char *string = (const char *)ustring;
-	char str[VGA_COLUMNS * VGA_LINES] = {0};
+	static char str[VGA_COLUMNS * VGA_LINES] = {0};
+	memset(str, 0, sizeof(str) - 1);
 	memcpy(str, string, strnlen(string, sizeof(str) - 1));
 
 	char *token = strtok(str, "\n");
```

Moving the array out of stack got me to a successful build. But when
running QEMU I got strange behavior.

<video controls loop style="max-width: 100%; width: 100%; margin: 0 0 20px 0;">
  <source src="/assets/vga-text/qemu-vgatext-pause.mp4" type="video/mp4">
</video>

It takes four seconds to try to initialize the display and there's no
text on it, despite including the test code from the first section. The
good news is: the final state of the display looks exactly the same as
an empty VGA text mode screen would look. So, this felt like validation
that my port I/O functions work and a hint that using the QEMU VGA
device in text mode is possible.


Breaking the Standard
---------------------

Time to explore what the VGA support code is doing, at least to find
something to experiment with. In the test code I used `vga_write_text()`
to print text, so I started reading from there.

```c
void
vga_write_text(enum VGA_TEXT_ALIGNMENT alignment, unsigned int line,
	       const unsigned char *ustring)
{
	const char *string = (const char *)ustring;
	static char str[VGA_COLUMNS * VGA_LINES] = {0};
	memset(str, 0, sizeof(str) - 1);
	memcpy(str, string, strnlen(string, sizeof(str) - 1));

	char *token = strtok(str, "\n");

	while (token != NULL) {
		size_t offset = VGA_COLUMNS - strnlen(token, VGA_COLUMNS);
		switch (alignment) {
		case VGA_TEXT_CENTER:
			vga_write_at_offset(line++, offset/2, token);
			break;
		case VGA_TEXT_RIGHT:
			vga_write_at_offset(line++, offset, token);
			break;
		case VGA_TEXT_LEFT:
		default:
			vga_write_at_offset(line++, 0, token);
			break;
		}
		token = strtok(NULL, "\n");
	}
}
```

This just splits long text by lines, finds out where on the screen the
pieces should be printed, then the actual printing work is handled by
`vga_write_at_offset()`.

```c
static void
vga_write_at_offset(unsigned int line, unsigned int offset, const char *string)
{
	if (!string)
		return;

	unsigned short *p = (unsigned short *)VGA_FB + (VGA_COLUMNS * line) + offset;
	size_t i, len = strlen(string);

	for (i = 0; i < (VGA_COLUMNS - offset); i++) {
		if (i < len)
			p[i] = 0x0F00 | (unsigned char)string[i];
		else
			p[i] = 0x0F00;
	}
}
```

And this one calculates a position in memory and writes some 16-bit data
for each character from that point on. The [VGA text mode](https://en.wikipedia.org/wiki/VGA_text_mode)
Wikipedia page explains the how those 16 bits are interpreted: the lower
8 bits choose the character and the higher 8 bits choose attributes like
background and text colors. This is also why the function does bitwise
OR with `0x0F00` before writing the character, to set white text on
black background.

Other than that, the [`VGA_FB` constant](https://github.com/coreboot/coreboot/blob/24.02.01/src/include/pc80/vga.h)
here immediately stuck out to me, as it's a "framebuffer". The simplest
explanation I could think of for why we couldn't display anything was
that we are putting the data in the wrong place. And since I got linear
framebuffer mode working, I already had a candidate for the right value
from the log messages: `framebuffer @ 3d000000 (pci bar 0)`.

```diff
diff --git a/src/include/pc80/vga.h b/src/include/pc80/vga.h
index 7a97afe5e59a..1b78378a256c 100644
--- a/src/include/pc80/vga.h
+++ b/src/include/pc80/vga.h
@@ -3,7 +3,7 @@
 #ifndef VGA_H
 #define VGA_H
 
-#define VGA_FB 0xB8000
+#define VGA_FB 0x3D000000
 #define VGA_FB_SIZE 0x4000 /* char + attr = word sized so 0x8000 / 2 */
 #define VGA_COLUMNS 80
 #define VGA_LINES 25
```

<a href="/assets/vga-text/qemu-vgatext-corrupt.png">
  <img src="/assets/vga-text/qemu-vgatext-corrupt.png" alt="QEMU display showing colorful blocks of corrupt text">
</a>

Finally, we get to see something exciting on the VGA display! This
block-and-glyph nature is further proof of the emulated device operating
in some text mode. We also see three white lines of glyphs which are
caused by my test print calls. The stride problem with this picture is
easier to notice with hindsight, but instead I tried putting some data
on the framebuffer manually similar to `vga_write_at_offset()` does.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index dcfbd29940ec..976578212122 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -3,6 +3,7 @@
 #include <pc80/vga.h>
 #include <pc80/vga_io.h>
 
+#include <delay.h>
 #include <string.h>
 #include "vga.h"
 
@@ -349,6 +350,13 @@ vga_textmode_init(void)
 	vga_write_text(VGA_TEXT_LEFT, VGA_LINES - 1, test_str);
 	vga_write_text(VGA_TEXT_RIGHT, VGA_LINES - 1, test_str);
 
+	uint16_t *p = (uint16_t *)VGA_FB;
+	for (uint16_t i = 0; i < UINT16_MAX; i++) {
+		p[i % (VGA_COLUMNS * VGA_LINES)] = i;
+		if (i % (VGA_COLUMNS * VGA_LINES) == 0)
+			mdelay(500);
+	}
+
 	vga_sr_mask(0x00, 0x02, 0x02); /* take us out of reset */
 	vga_cr_mask(0x17, 0x80, 0x80); /* sync! */
 }
```

<video controls loop style="max-width: 100%; width: 100%; margin: 0 0 20px 0;">
  <source src="/assets/vga-text/qemu-vgatext-fill16.mp4" type="video/mp4">
</video>

When we write 16 bits of data for each character enough to fill the text
mode screen, we only end up filling half of it. Which means we need
32 bits of data per character here. I tried increasing the data stride
and that was enough to cover the whole display.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index 976578212122..2f096255fdb2 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -350,7 +350,7 @@ vga_textmode_init(void)
 	vga_write_text(VGA_TEXT_LEFT, VGA_LINES - 1, test_str);
 	vga_write_text(VGA_TEXT_RIGHT, VGA_LINES - 1, test_str);
 
-	uint16_t *p = (uint16_t *)VGA_FB;
+	uint32_t *p = (uint32_t *)VGA_FB;
 	for (uint16_t i = 0; i < UINT16_MAX; i++) {
 		p[i % (VGA_COLUMNS * VGA_LINES)] = i;
 		if (i % (VGA_COLUMNS * VGA_LINES) == 0)
```

<video controls loop style="max-width: 100%; width: 100%; margin: 0 0 20px 0;">
  <source src="/assets/vga-text/qemu-vgatext-fill32.mp4" type="video/mp4">
</video>

We have the colors cycling as before. The corrupt glyphs are reduced
down to a single smiley, which is concerning. We still see that single
glyph periodically. So I guessed we likely have the lower 16 bits same
as in the standard.

At the time I couldn't think of the higher bits having any importance. I
tried a few other things like changing the upper and lower bits based on
coordinates, using different bit shifts, and changing the buffer byte by
byte for more precise test values. The key thing to try was setting just
the standard bits and not touching others.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index 549494829152..68a87008b962 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -350,6 +350,17 @@ vga_textmode_init(void)
 	vga_write_text(VGA_TEXT_LEFT, VGA_LINES - 1, test_str);
 	vga_write_text(VGA_TEXT_RIGHT, VGA_LINES - 1, test_str);
 
+	uint8_t x, y;
+	uint8_t *p = (uint8_t *)VGA_FB;
+	for (uint16_t i = 0; i < VGA_LINES * VGA_COLUMNS * 4; i += 4) {
+		x = (i/4) % VGA_COLUMNS;
+		y = ((i/4) / VGA_COLUMNS) % VGA_LINES;
+		p[i] = x + y;
+		p[i+1] = y / 2;
+		/* p[i+2] = x; */
+		/* p[i+3] = y; */
+	}
+
 	vga_sr_mask(0x00, 0x02, 0x02); /* take us out of reset */
 	vga_cr_mask(0x17, 0x80, 0x80); /* sync! */
 }
```

<a href="/assets/vga-text/qemu-vgatext-badfont.png">
  <img src="/assets/vga-text/qemu-vgatext-badfont.png" alt="QEMU display showing colorful lines of corrupt text">
</a>

Again, the conclusion is easy to see here in retrospect. The higher bits
hold font data, they were being cleared when writing the standard bits
as 32-bit data and being corrupted when I treated the buffer as 16-bit.
I could not make the connection, but I could identify this picture had a
problem with fonts. There is a `vga_font_8x16_load()` function that is
called just before my test code, so I explored further from there.

```c
static void
vga_font_8x16_load(void)
{
	unsigned char *p;
	size_t i, j;
	unsigned char sr2, sr4, gr5, gr6;

#define height 16
#define count 256

	sr2 = vga_sr_read(0x02);
	sr4 = vga_sr_read(0x04);
	gr5 = vga_gr_read(0x05);
	gr6 = vga_gr_read(0x06);

	/* disable odd/even */
	vga_sr_mask(0x04, 0x04, 0x04);
	vga_gr_mask(0x05, 0x00, 0x10);
	vga_gr_mask(0x06, 0x00, 0x02);

	/* plane 2 */
	vga_sr_write(0x02, 0x04);
	p = (unsigned char *)VGA_FB;
	for (i = 0; i < count; i++) {
		for (j = 0; j < 32; j++) {
			if (j < height)
				*p = vga_font_8x16[i][j];
			else
				*p = 0x00;
			p++;
		}
	}

	vga_gr_write(0x06, gr6);
	vga_gr_write(0x05, gr5);
	vga_sr_write(0x04, sr4);
	vga_sr_write(0x02, sr2);

	/* set up font size */
	vga_cr_mask(0x09, 16 - 1, 0x1F);
}
```

The `unsigned char` is 8 bits, so with `VGA_FB` being 32-bit I knew what
to experiment on. After fiddling with that for loop for a while, I got
something that works without fully understanding why.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index 68a87008b962..304cfbd0e8df 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -189,12 +189,13 @@ vga_font_8x16_load(void)
 	vga_sr_write(0x02, 0x04);
 	p = (unsigned char *)VGA_FB;
 	for (i = 0; i < count; i++) {
-		for (j = 0; j < 32; j++) {
-			if (j < height)
-				*p = vga_font_8x16[i][j];
+		for (j = 0; j < 64; j++) {
+			if (j < 32)
+				*p = vga_font_8x16[i][j/2];
 			else
 				*p = 0x00;
 			p++;
+			p++;
 		}
 	}
 
```

<a href="/assets/vga-text/qemu-vgatext-okfont.png">
  <img src="/assets/vga-text/qemu-vgatext-okfont.png" alt="QEMU display showing colorful lines of text with minor errors">
</a>

It may look like everything is finally working, but it turns out there
was still more to fix. You might recall there were three lines of white
glyphs in the first colorful display we got. I mentioned that those were
caused by our test messages. This is how that screen looked after the
font fix.

<a href="/assets/vga-text/qemu-vgatext-badtest.png">
  <img src="/assets/vga-text/qemu-vgatext-badtest.png" alt="QEMU display showing corrupt test message without colors">
</a>

The colorful blocks were in fact caused by font loading spilling over to
codepoint and attribute bytes. The test messages were slightly better,
at least rendering some letters. Of course, the key to fixing that was
`VGA_FB` being 32-bit again. The fix appeared simple but subtly broke
the colored text.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index f69e96be7c10..53ec96cd1430 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -264,7 +264,7 @@ vga_write_at_offset(unsigned int line, unsigned int offset, const char *string)
 	if (!string)
 		return;
 
-	unsigned short *p = (unsigned short *)VGA_FB + (VGA_COLUMNS * line) + offset;
+	uint32_t *p = (uint32_t *)VGA_FB + (VGA_COLUMNS * line) + offset;
 	size_t i, len = strlen(string);
 
 	for (i = 0; i < (VGA_COLUMNS - offset); i++) {
```

<a href="/assets/vga-text/qemu-vgatext-missfont.png">
  <img src="/assets/vga-text/qemu-vgatext-missfont.png" alt="QEMU display showing colorful text with few glyphs missing">
</a>

It is shortly after this that I made the mental connection that the
higher 16 bits were font data. With that, I could simplify and improve
the previous fixes.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index 68a87008b962..a5e16d3de60b 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -168,7 +168,7 @@ vga_mode_set(int hdisplay, int hblankstart, int hsyncstart, int hsyncend,
 static void
 vga_font_8x16_load(void)
 {
-	unsigned char *p;
+	uint32_t *p;
 	size_t i, j;
 	unsigned char sr2, sr4, gr5, gr6;
 
@@ -187,11 +187,11 @@ vga_font_8x16_load(void)
 
 	/* plane 2 */
 	vga_sr_write(0x02, 0x04);
-	p = (unsigned char *)VGA_FB;
+	p = (uint32_t *)VGA_FB;
 	for (i = 0; i < count; i++) {
 		for (j = 0; j < 32; j++) {
 			if (j < height)
-				*p = vga_font_8x16[i][j];
+				*p = vga_font_8x16[i][j] << 16;
 			else
 				*p = 0x00;
 			p++;
@@ -263,14 +263,16 @@ vga_write_at_offset(unsigned int line, unsigned int offset, const char *string)
 	if (!string)
 		return;
 
-	unsigned short *p = (unsigned short *)VGA_FB + (VGA_COLUMNS * line) + offset;
+	uint32_t *p = (uint32_t *)VGA_FB + (VGA_COLUMNS * line) + offset;
 	size_t i, len = strlen(string);
 
 	for (i = 0; i < (VGA_COLUMNS - offset); i++) {
 		if (i < len)
-			p[i] = 0x0F00 | (unsigned char)string[i];
+			p[i] = (p[i] & 0xFFFF0000) |
+			       0x0F00 | (unsigned char)string[i];
 		else
-			p[i] = 0x0F00;
+			p[i] = (p[i] & 0xFFFF0000) |
+			       0x0F00;
 	}
 }
 
```

<a href="/assets/vga-text/qemu-vgatext-goodfont.png">
  <img src="/assets/vga-text/qemu-vgatext-goodfont.png" alt="QEMU display showing colorful lines of text">
</a>

I guess I can finally call it a working text mode display. With some
extra effort we can design a nicer test screen, although it's quite late
at this point.

```diff
diff --git a/src/drivers/pc80/vga/vga.c b/src/drivers/pc80/vga/vga.c
index c3bdb7ceab8e..652969fc3603 100644
--- a/src/drivers/pc80/vga/vga.c
+++ b/src/drivers/pc80/vga/vga.c
@@ -314,6 +314,51 @@ vga_write_text(enum VGA_TEXT_ALIGNMENT alignment, unsigned int line,
 	}
 }
 
+static void
+vga_textmode_testscreen(void)
+{
+	uint32_t i;
+	uint8_t c, x, y, row, col;
+	uint32_t *p = (uint32_t *)VGA_FB;
+	const unsigned char *test_str = (const unsigned char *)"-- VGA TEXT MODE --\n";
+
+	vga_write_text(VGA_TEXT_CENTER, 1, test_str);
+
+	for (c = 0x00; c < 0xFF; c++) {
+		x = 8 + (c & 0xF) * 2;
+		y = 5 + (c >> 4);
+		col = (c & 0x0F);
+		row = (c & 0xF0) >> 4;
+
+		i = y * VGA_COLUMNS + x;
+		p[i] = (p[i] & 0xFFFF0000) |
+		       (0x0 << 12) | (0xF << 8) | c;
+
+		i = y * VGA_COLUMNS + (34 + x);
+		p[i] = (p[i] & 0xFFFF0000) |
+		       (col << 12) | (row << 8) | 0x04;
+
+		i = 3 * VGA_COLUMNS + x;
+		p[i] = (p[i] & 0xFFFF0000) |
+		       (0x0 << 12) | (0xF << 8) |
+		       ((col < 10) ? (0x30 + col) : (0x41 + col % 10));
+
+		i = 3 * VGA_COLUMNS + (34 + x);
+		p[i] = (p[i] & 0xFFFF0000) |
+		       (0x0 << 12) | (0xF << 8) |
+		       ((col < 10) ? (0x30 +col) : (0x41 + col % 10));
+
+		i = y * VGA_COLUMNS + 5;
+		p[i] = (p[i] & 0xFFFF0000) |
+		       (0x0 << 12) | (0xF << 8) |
+		       ((row < 10) ? (0x30 + row) : (0x41 + row % 10));
+	}
+
+	vga_write_text(VGA_TEXT_CENTER, VGA_LINES - 2, test_str);
+
+	mdelay(5000);
+}
+
 /*
  * set up everything to get a basic 80x25 textmode.
  */
@@ -345,6 +390,10 @@ vga_textmode_init(void)
 	vga_fb_clear();
 	vga_font_8x16_load();
 
+	vga_textmode_testscreen();
+	vga_fb_clear();
+	vga_font_8x16_load();
+
 	vga_sr_mask(0x00, 0x02, 0x02); /* take us out of reset */
 	vga_cr_mask(0x17, 0x80, 0x80); /* sync! */
 }
```

<a href="/assets/vga-text/qemu-vgatext-testscreen.png">
  <img src="/assets/vga-text/qemu-vgatext-testscreen.png" alt="QEMU display showing a custom text mode test screen">
</a>


Cirrus VGA Display
------------------

Well, we have an entire other emulated VGA display to work on. I had
disabled building coreboot's Cirrus display device driver while working
on the Bochs one because it uses a lot of port I/O functions, but after
implementing those I thought I might as well try to make it work. After
re-adding its driver to the Makefile, we still get an error about `io.h`
functions.

```
    CC         ramstage/drivers/emulation/qemu/bochs.o
    CC         ramstage/drivers/emulation/qemu/cirrus.o
src/drivers/emulation/qemu/cirrus.c: In function 'write_hidden_dac':
src/drivers/emulation/qemu/cirrus.c:176:9: error: implicit declaration of function 'inb'; did you mean 'isb'? [-Werror=implicit-function-declaration]
  176 |         inb(0x3c8);
      |         ^~~
      |         isb
src/drivers/emulation/qemu/cirrus.c:181:9: error: implicit declaration of function 'outb' [-Werror=implicit-function-declaration]
  181 |         outb(data, 0x3c6);
      |         ^~~~
cc1: all warnings being treated as errors
make: *** [Makefile:419: build/qemu_arm64/ramstage/drivers/emulation/qemu/cirrus.o] Error 1
```

Because it's missing the `#include <arch/io.h>` statement. Adding that
and building coreboot for linear framebuffer mode makes the display work
when we invoke QEMU with `-device cirrus-vga`.

```diff
diff --git a/src/drivers/emulation/qemu/Makefile.mk b/src/drivers/emulation/qemu/Makefile.mk
index 186067e7d49f..c9d94bdca0c2 100644
--- a/src/drivers/emulation/qemu/Makefile.mk
+++ b/src/drivers/emulation/qemu/Makefile.mk
@@ -6,3 +6,4 @@ postcar-$(CONFIG_CONSOLE_QEMU_DEBUGCON) += qemu_debugcon.c
 ramstage-$(CONFIG_CONSOLE_QEMU_DEBUGCON) += qemu_debugcon.c
 
 ramstage-$(CONFIG_DRIVERS_EMULATION_QEMU_BOCHS) += bochs.c
+ramstage-$(CONFIG_DRIVERS_EMULATION_QEMU_BOCHS) += cirrus.c
diff --git a/src/drivers/emulation/qemu/cirrus.c b/src/drivers/emulation/qemu/cirrus.c
index 1dc8ac9e3ef8..21af0a4b6e33 100644
--- a/src/drivers/emulation/qemu/cirrus.c
+++ b/src/drivers/emulation/qemu/cirrus.c
@@ -1,6 +1,7 @@
 /* SPDX-License-Identifier: GPL-2.0-or-later */
 
 #include <stdint.h>
+#include <arch/io.h>
 #include <console/console.h>
 #include <device/device.h>
 #include <device/pci.h>
```

```
[DEBUG]  PCI: 00:00:02.0 init
[DEBUG]  QEMU VGA: cirrus framebuffer @ 3c000000 (pci bar 0)
[INFO ]  framebuffer_info: bytes_per_line: 3200, bits_per_pixel: 32
[INFO ]                     x_res x y_res: 800 x 600, size: 1920000 at 0x3c000000
[DEBUG]  PCI: 00:00:02.0 init finished in 1 msecs
	 [...]
[INFO ]  Setting up bootsplash in 800x600@32
[DEBUG]  FMAP: area COREBOOT found @ 20200 (16645632 bytes)
[INFO ]  CBFS: Found 'bootsplash.jpg' @0x11680 size 0xa76c in mcache @0xbfffeb6c
[DEBUG]  Bootsplash image resolution: 256x256
[INFO ]  Bootsplash loaded
```

<a href="/assets/vga-text/qemu-cirrus-bootsplash.png">
  <img src="/assets/vga-text/qemu-cirrus-bootsplash.png" alt="QEMU cirrus display showing a boot splash image">
</a>

Notice how Cirrus' framebuffer address is `3c000000` instead of the
earlier `3d000000`. I plugged that as the new `VGA_FB` address, built
coreboot for text mode and my standard-breaking text mode changes worked
equally as nice for Cirrus as well.

```diff
diff --git a/src/include/pc80/vga.h b/src/include/pc80/vga.h
index 1b78378a256c..554d92402b56 100644
--- a/src/include/pc80/vga.h
+++ b/src/include/pc80/vga.h
@@ -3,7 +3,7 @@
 #ifndef VGA_H
 #define VGA_H
 
-#define VGA_FB 0x3D000000
+#define VGA_FB 0x3C000000
 #define VGA_FB_SIZE 0x4000 /* char + attr = word sized so 0x8000 / 2 */
 #define VGA_COLUMNS 80
 #define VGA_LINES 25
```

<a href="/assets/vga-text/qemu-cirrus-vgatext.png">
  <img src="/assets/vga-text/qemu-cirrus-vgatext.png" alt="QEMU cirrus display showing a custom text mode test screen">
</a>


VGA on RISC-V
-------------

From what I saw in U-Boot, I already knew the Bochs display device could
work on QEMU RISC-V virtual machines. So while trying to port coreboot's
driver to be architecture-generic, I thought I should try testing it on
RISC-V as well. 

Getting the driver to work on ARM64 didn't immediately make it work on
RISC-V, so I compared the coreboot code concerning both boards to see
what could be missing on the RISC-V side. Long story short, PCI support
was not yet implemented there. Based on the ARM64 code I managed to
[enable PCI support on QEMU RISC-V virtual machines](https://review.coreboot.org/c/coreboot/+/77486).
Then enabling relevant config options for emulated display drivers made
it work in linear framebuffer mode.

After doing all these VGA text mode experiments, I checked if it would
work also on RISC-V without much effort. It did after copying the same
`io.h` and replacing board-specific values. The `PCI_PIO_BASE` address
needs to be `0x3000000` for port I/O functions to work, same as
[the `VIRT_PCIE_PIO` value](https://gitlab.com/qemu-project/qemu/-/blob/v8.2.2/hw/riscv/virt.c#L92)
from QEMU. And the `VGA_FB` address needs to be `0x7F000000` for
`-device VGA` or `0x7E000000` for `-device cirrus-vga` to show anything
in text mode.


VGA on ARMv7
------------

There is a QEMU ARMv7 port in coreboot intended for Versatile Express
hardware, but QEMU doesn't support PCI on those emulation models, which
prevents us from adding the relevant display devices to it.

For other reasons, I've been also working on a [coreboot port for QEMU ARMv7 'virt' platform](https://review.coreboot.org/c/coreboot/+/80381).
It's almost the same as the ARM64 one on the QEMU side, which resulted
in everything working on it without any changes. 


Conclusion
----------

It is possible to use QEMU's emulated VGA devices in text mode, even
on ARM and RISC-V architectures, by going beyond the VGA standard a bit.

We need to communicate with the device through PCI I/O space, which
apparently is possible via a memory mapped translator. Standard
behavior is enough to get the display device into text mode, but we
can't put anything on the display.

The standard VGA text buffer at `0xB8000` in memory does not work, but
the framebuffer memory exposed over PCI regions can be used instead.
That buffer appears to hold 32-bit data: lower 16 bits are interpreted
as codepoint and attributes while higher bits are interpreted as font
data, so care must be taken to avoid overwriting the latter when writing
the former. I don't know if there is a way to map the relevant parts of
this buffer back at the standard address.

Firmware that is aware of these quirks can use these devices in this
non-standard text mode. Though, I would not expect it to work on any
actual hardware. It's likely how the framebuffer memory is being
interpreted as text-mode data is specific to QEMU and wouldn't work the
same way on other graphics hardware.

If we wanted to go forward with this in coreboot, the payloads would
also need to handle this non-standard behavior in their code to be able
to use the display. But guessing it may be very specific to QEMU for
some legacy functionality, I'm not sure it would be worth it.

Overall, most of what I did here looks like wasted effort to me. But at
least it gave me something interesting to write about, I hope you
enjoyed it. And I think having all of this written out will help my
attempts to improve QEMU display support in coreboot.
