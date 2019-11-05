# ACPI/GPIO controller
GPIO controller related notes on ACPI-enabled Intel Edison board.

## Enumeration issue

As has been reported by Georgii Staroselskii:

> I bumped into a very strange issue that I'm pretty sure you can just
crack from the first look. Even though I have the kernel,
[U-Boot](u-boot-update) and
[meta-acpi](https://github.com/westeri/meta-acpi/) 
layer checked out at the default branches I'm seeing this output:

```
root@edison:~# dmesg | grep df5

[    0.134373] pci 0000:00:0c.0: reg 0x14: [mem 0x000df570-0x000df57f]
[    0.164101] pci 0000:00:0c.0: can't claim BAR 1 [mem 0x000df570-0x000df577]: no compatible bridge window

root@edison:~# dmesg | grep gpio

[    0.739599] gpio-merrifield 0000:00:0c.0: can't enable device: BAR 1 [memsize 0x00000008] not assigned
[    0.740114] gpio-merrifield: probe of 0000:00:0c.0 failed with error -22
```

> I'm 100% sure that it wasn't the case when I started working on the
BSP. Do you have any idea what could've gone wrong? It seems like
addresses just don't match but at the moment I don't understand how
exactly reg#N have their addresses assigned. These messages are very
early in the boot process so I guess that initramfs doesn't have
anything to do with those, does it?

> The southcluster.asl file in [U-Boot](https://edison.internet-share.com/wiki/U-Boot)
is exactly the same as you left it. I'm using the branch I have left a
link to in the previous letter. But now Wi-Fi, rfkill, and pretty much
everything that's GPIO-based doesn't work.

JFYI, I found in the internal document what this BAR is for.

The document states that two 32-bit fields provide couple of numbers,
while I can't disclose them, but which we are not using in the Linux.
So, basically it might be anywhere in the address space as firmware
provides.

The problem is caused by the IFWI firmware providing the address range,
and the address range varies between IFWI versions. A
[U-Boot](u-boot-update) ASL fix (adding another resource in the _CRS table) sounds plausible.

However, <https://git.yoctoproject.org/cgit/cgit.cgi/meta-intel-edison/log/utils/flash/ifwi/edison>
shows only 3 revisions of IWFI have been released, of which only the
first (installed on the original factory image) and the last are found
on the wild. The original (3.10) kernel shows SFI table revisions in
dmesg:

|                            |2015-03-13 release | 2015-03-25 release |
|----------------------------|:-------:|:-------:|
|OEMB iafw version           | 002.001 | 002.012 |
|OEMB val_hooks version      | 002.001 | 002.004 |
|OEMB ia suppfw version      | 000.000 | 000.000 |
|OEMB scu runtime version    | 176.073 | 176.073 |
|OEMB ifwi version           | 237.011 | 237.015 |

By using flashall --recovery or using Flash Tool Lite
(<https://edison-fw.github.io/meta-intel-edison/2.0-Building-and-installing-the-image.html#flash-tool-lite>)
IFWI is updated simultaneously with U-Boot and the latest IFWI is
installed so this issue does not occur.
