---
layout: post
title: "News #3: OutRun"
author: "Alper Nebi Yasak"
date: 2024-02-25 16:30:00 +0300
categories: news
tags: news
---

This is Anchorboot News #3, a semi-regular update post about the state
of the project. I've had another problematic month where I could do
little to no work. But in the time I could, I've continued to work on
the coreboot side of things: finishing and sending patches upstream for
review, and deliberating on how to fix things based on reviews. Here's
the details:


Coreboot Gerrit
---------------

Coreboot uses the [Gerrit Code Review](https://www.gerritcodereview.com/)
software to manage code contributions. Their processes are a bit
different than the usual mailing list or pull request workflows that I
have been accustomed to. I took a bit of time to learn it as best as I
could, and integrate what it expects from me into the way I work with
git repositories.

One weird thing is that Gerrit essentially treats each commit as
independent regardless of history, and merely shows a "relation chain"
in the sidebar. There's no easy way to talk about or link to an entire
series like a cover letter or a pull request description, so I'll just
try to link to the most prominent patch in each.


QEMU Bochs Display
------------------

I had managed to get the Bochs display to work on other architectures,
and I have sent those patches upstream as [CB:80376](https://review.coreboot.org/c/coreboot/+/80376/1).
Part of that relation chain is [CB:80372](https://review.coreboot.org/c/coreboot/+/80372/3)
which adds stub implementations for x86-style port I/O instructions, to
help clean things up. I tried to avoid using legacy VGA resources on
non-x86 architectures so that I don't have to implement those and port
the entire VGA stack to three different architectures, but I'm having a
feeling that I'll be required to do it.


RAM Detection on QEMU
---------------------

For RAM detection, I have managed to write an ad-hoc parser to get
memory information from the flat device-tree binary as a utility
function, and uploaded it as [CB:80322](https://review.coreboot.org/c/coreboot/+/80322/3),
along with patches for using it on ARM64 and RISC-V virtual machines.
Most of the reviews for that are about refactoring useful parts into
separate functions (whereas I wanted to avoid extending an API I didn't
like), so there's more to do.

There's also the question of using device-tree information in other
places, so maybe I should try to push for a switch to a more common
implementation like [libfdt](https://git.kernel.org/pub/scm/utils/dtc/dtc.git/tree/README.md).

There was another RAM detection issue on ARM64 QEMU VMs where the
existing RAM probing mechanism could only detect up to 1GiB of RAM due
to how MMU is initialized, which I got fixed with [CB:80321](https://review.coreboot.org/c/coreboot/+/80321).


QEMU Firmware Configuration Driver
----------------------------------

Using device-tree for RAM detection means I don't have an immediate need
for the QEMU Firmware Configuration device, but I didn't want to leave
that work incomplete. I managed to incorporate the changes I needed for
other architectures into the existing code, and uploaded it as
[CB:80369](https://review.coreboot.org/c/coreboot/+/80369/1).


ARMv7 "virt" mainboard
----------------------

I had ended up creating a new mainboard for the ARMv7 "virt" platform,
which I uploaded upstream as a work-in-progress [CB:80831](https://review.coreboot.org/c/coreboot/+/80381/1),
at least until I can get the other patches merged.


PCIe Support on QEMU RISC-V
---------------------------

I had managed to get PCIe support working on RISC-V QEMU virtual machine
enough to use the Bochs display with my other patches, which I uploaded
as [CB:80378](https://review.coreboot.org/c/coreboot/+/80378/1).


U-Boot Payload for QEMU RISC-V
------------------------------

I have found an earlier attempt to use U-Boot as a coreboot payload for
RISC-V QEMU builds, [CB:77486](https://review.coreboot.org/c/coreboot/+/77486).
It's a nice base for coreboot build integration which can easily be
adapted to ARM architectures as well.

However, that still relies on U-Boot being built for a specific board,
where it's necessary to port U-Boot to each board one wants to use it
on, which I'm trying to avoid by making generic ports on the U-Boot side
for ARM. It also lacks support for a framebuffer that would be passed on
to the payload, which my patches above will help fix.
