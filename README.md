Anchorboot
==========

Anchorboot is a new platform firmware distribution for ARM-based
ChromeOS devices using coreboot and U-Boot, with the aim to make it easy
to install and use conventional Linux distributions on them through UEFI
support.

It is still in initial, proof-of-concept phases of development, so be
sure to check back later!


Why?
----

Despite their bad reputation as walled-garden systems, ChromeOS devices
have huge potential to be FOSS-friendly as most things that make them
work are published as free software. However, they use custom platform
firmware purpose-built to boot their operating system with non-standard
boot mechanisms, whose limitations make it significantly hard to run
other OSes on these devices through their stock firmware, stifling this
potential.

There are existing projects working on enabling conventional Linux
distributions to run on these devices. The most successful of these
efforts is the ["Chrultrabook"](https://chrultrabook.github.io/docs/)
project, which provides replacement platform firmware using coreboot and
TianoCore EDK II.

However, they have limited their focus on x86-based ChromeOS devices so
far. Meanwhile, although other developers have been doing firmware work
on ARM-based ones enough to support U-Boot on earlier models, there
isn't an easy-to-install distribution like theirs yet. We would like
to bridge the gap and replicate their success on the ARM side.


What's next?
------------

ChromeOS devices use coreboot-based firmware, and Google properly
releases their sources and upstreams code for their devices. We would
like to reuse this upstream support, in part to make maintenance easier
going forward. On the flip side, most ARM devices outside this niche use
U-Boot which has been steadily gaining UEFI capabilities, and most
conventional Linux distributions have support for its boot methods
otherwise. We would like to use it for this ease-of-use. There is some
support for combining the two on x86 systems, but not on ARM.

We will first improve and extend integration between coreboot and U-Boot
to the ARM architectures, then work on a selection of Chromebooks to fix
any issues and to port device drivers to either project where necessary.
As each board's work is complete, we will prepare and distribute
pre-built, tested firmware images ready to be flashed on these boards
along with sources, instructions on how to use the images, and other
documentation relevant to the devices.


Acknowledgements
----------------

<a href="https://NLnet.nl">
    <img src="/logo/banner.png" alt="Logo NLnet: abstract logo of four people seen from above">
</a>
<a href="https://NLnet.nl/NGI0">
    <img src="/image/logos/NGI0Entrust_tag.svg" alt="Logo NGI Zero: letterlogo shaped like a tag">
</a>

This project was funded through the [NGI0 Entrust](https://nlnet.nl/entrust) Fund,
a fund established by [NLnet](https://nlnet.nl) with financial support
from the European Commission's [Next Generation Internet](https://ngi.eu) programme,
under the aegis of DG Communications Networks, Content and Technology
under grant agreement N<sup>o</sup> 101069594.
