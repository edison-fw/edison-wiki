# Using a vanilla Linux kernel with Intel Edison

The Linux kernel used in that build is version 3.10.17 which is patched
with a gigantic 150000+ lines patch to add drivers necessary for the
[Edison](https://edison-fw.github.io/meta-intel-edison/).
Unfortunately, by now, the 3.10.17 kernel is rather old and lacking
features that Ionic/OpenWrt] rely upon.

The first step in porting Ionic/OpenWrt to the Intel®
[Edison](https://edison-fw.github.io/meta-intel-edison/) is therefor getting a newer kernel running on the Edison. Searching Google, I came across this site:

<https://habrahabr.ru/post/254247/>

Unfortunately it is all in Russian --- of which I understand very little
--- but with a little help from Google Translate it looked promising.

The author, [Andy Shevchenko](https://habrahabr.ru/users/andy_shev/)
have managed to boot the [Edison](https://edison-fw.github.io/meta-intel-edison/)
using the latest vanilla Linux kernel without using any of the patches.

I decided to try to replicate Andy's effort in order to gain more
knowledge of the inner workings of the [Edison](https://edison-fw.github.io/meta-intel-edison/)
and to have a platform/framework to port remaining drivers.

## Kernel

First I cloned the same kernel (this also works using the latest
release kernel [Linux 4.1-rc7](https://www.kernel.org/pub/linux/kernel/v4.x/testing/linux-4.1-rc7.tar.xz)
that Andy was using in his effort:

```
git clone <git://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git>
```

After a considerable time (my Internet sucks big time), I created a
copy of the original [[i386_defconfig]:

```
 cp arch/x86/configs/{i386,edison}_defconfig
```
Next, edit the `arch/x86/configs/edison_defconfig` to contain
the following:
```
    # CONFIG_DRM_I915 is not set
    CONFIG_BACKLIGHT_LCD_SUPPORT=y
    CONFIG_USB_XHCI_HCD=y
    CONFIG_USB_DWC3=y
    CONFIG_USB_DWC3_GADGET=y
    CONFIG_GPIOLIB=y
    CONFIG_GPIO_INTEL_MID=y
    CONFIG_INTEL_MID_WATCHDOG=y
    CONFIG_X86_EXTENDED_PLATFORM=y
    CONFIG_X86_INTEL_MID=y
    CONFIG_EFI_STUB=y
    CONFIG_EARLY_PRINTK_EFI=y
    CONFIG_HSU_DMA=y
    CONFIG_HSU_DMA_PCI=y
    CONFIG_SERIAL_8250_DMA=y
    CONFIG_SERIAL_8250_MID=y
    CONFIG_SERIAL_8250_PCI=y
    Finally, build the kernel using this configuration:
    $ make edison_defconfig
    $ make -j4
    $ make bzImage
```

The `-j4` essentially allows four files to be
build in parallel. Change the number 4 to the numbers of
processors/cores available on the build system.

### 64-bit mode

Intel Atom, which is installed on Intel Edison, is known x86_64 CPU.
To enable 64-bit build in Linux kernel v4.4 [the patch](https://www.spinics.net/lists/kernel/msg2163592.html)
should be applied.

```
Linux buildroot 4.4.0-next-20160115+ #25 SMP Fri Jan 15 22:03:19 EET 2016 x86_64 GNU/Linux
```

NOTE: To run it directly from U-Boot one has to update it to at least
version 2015.05.

The newest U-Boot sources are available 
[here](https://github.com/andy-shev/u-boot/tree/edison).
I compile them successfully on Debian unstable with whatever default GCC
is there.


## Root Filesystem

In order to build the root filesystem, first clone the buildroot
environment:

```
git clone <git://git.buildroot.net/buildroot>
```

In the buildroot directory, create a `configs/edison_defconfig`, containing the
following:

```
    # Architecture
    BR2_i386=y
    BR2_x86_i586=y

    # Misc
    BR2_ROOTFS_DEVICE_CREATION_DYNAMIC_MDEV=y
    BR2_TARGET_GENERIC_GETTY_PORT=«ttyS2»

    # Root FS
    # BR2_TARGET_ROOTFS_TAR is not set
    BR2_TARGET_ROOTFS_CPIO=y
    BR2_TARGET_ROOTFS_CPIO_BZIP2=y

    # Packages
    BR2_PACKAGE_KEXEC=y
    BR2_PACKAGE_KEXEC_ZLIB=y
    BR2_PACKAGE_LRZSZ=y
    BR2_PACKAGE_SCREEN=y
    BR2_PACKAGE_PCIUTILS=y
    BR2_PACKAGE_DMIDECODE=y
    BR2_PACKAGE_BUSYBOX_WATCHDOG=y
```

Finally, build the root filesystem:

```
    $ make edison_defconfig
    $ make
```

After a while, the resulting image is available as:

```
    output/images/rootfs.cpio.bz2
```

which is roughly 1.6 MB of size.

## Copying file to [Edison](https://edison-fw.github.io/meta-intel-edison/)

Before restarting the [Edison](https://edison-fw.github.io/meta-intel-edison/)
while still running the original Intel® provided firmware, create a FAT
file system on device 0:9. (If you don't, and save files on the
originally-available file system, they will not be visible from U-Boot.)

```
    # mkfs.vfat -F32 -I /dev/mmcblk0p9
```

Reboot the [Edison](https://edison-fw.github.io/meta-intel-edison/).


Mount the [Edison](https://edison-fw.github.io/meta-intel-edison/)'s
now-visible drive on the host computer.

Copy the just-built Linux image and root file system to the Edison
drive:

```
    $ cp linux-next/arch/x86/boot/bzImage/<path-to-Edison-drive>/vmlinuz.efi
    $ cp buildroot/output/images/rootfs.cpio.bz2/<path-to-Edison-drive>/initrd
```
## U-Boot

While still running the original Intel® provided firmware, configure a
few variables in the U-Boot environment:

```
    setenv boot_edsboot 'zboot 0x100000 0 0x3000000 0x1000000'
    setenv bootargs_edsboot 'console=tty1 console=ttyS2,115200n8 rootfstype=ramfs rw'
    setenv bootcmd_edsboot 'setenv bootargs ${bootargs_edsboot}; run load_edsboot; run boot_edsboot'
    setenv load_edsboot 'load mmc 0:9 0x100000 vmlinuz.efi; load mmc 0:9 0x3000000 initrd'
```

The [Edison](https://edison-fw.github.io/meta-intel-edison/)
device will still boot as normal, but *if* the normal boot
is interrupted by pressing a key during boot (at the right time), it is
now possible to boot the new system:

```
U-Boot 2014.04 (Jun 04 2015 - 12:37:07)

       Watchdog enabled

DRAM:  980.6 MiB
MMC:   tangier_sdhci: 0
In:    serial
Out:   serial
Err:   serial

Hit any key to stop autoboot:  0

boot > run bootcmd_edsboot
```

### New U-Boot

The older U-Boot doesn't support kernels starting from v4.7. Read some
details here [Intel Edison does not work on Linux 4.7+kernels](https://github.com/andy-shev/linux/issues/3).

Newer [U-Boot](u-boot-update)
is located in edison branch of my [repository on
GitHub](https://github.com/andy-shev/u-boot/tree/edison).


The goal is to get Intel Edison supported by U-Boot
out-of-the-box.

See [discussion](https://communities.intel.com/thread/107935)
for more details regarding U-Boot upstream approach.

## Result
### Boot Console
```
* * * * * * * * * * * * * * * * * * * * * * * * * * * * * *

PSH KERNEL VERSION: b0182b2b

                WR: 20104000

* * * * * * * * * * * * * * * * * * * * * * * * * * * * * *
SCU IPC: 0x800000d0  0xfffce92c
PSH miaHOB version: TNG.B0.VVBD.0000000c
microkernel built 11:24:08 Feb  5 2015


 * * * * * * * PSH loader  * * * * * * *

PCM page cache size = 192 KB
Cache Constraint = 0 Pages
Arming IPC driver ..
Adding page store pool ..
PagestoreAddr(IMR Start Address) = 0x04899000
pageStoreSize(IMR Size)          = 0x00080000

 * * * Ready to receive application  * * *

U-Boot 2014.04 (Jun 04 2015 - 12:37:07)
       Watchdog enabled

DRAM:  980.6 MiB
MMC:   tangier _sdhci: 0
In:    serial
Out:   serial
Err:   serial
Hit any key to stop autoboot:  0

boot  > run bootcmd _edsboot

reading vmlinuz.efi

5737808 bytes read in 144 ms (38 MiB/s)

reading initrd

1645487 bytes read in 54 ms (29.1 MiB/s)
Valid Boot Flag
Setup Size = 0x00003e00
Magic signature found
Using boot protocol version 2.0d
Linux kernel version 4.1.0-rc6+ (lth @ncpws04)  #1 SMP Tue Jun 9
14:10:06 MYT 2015
Building boot _params at 0x00090000
Loading bzImage at address 00100000 (5721936 bytes)
Magic signature found
Initial RAM disk at linear address 0x00800000, size 8388608 bytes
Kernel command line:  "console=tty1 console=ttyS2,115200n8
root=/dev/ram0 rw initrd=0x800000,8M "
Starting kernel  ...
 [    0.000000 ] Initializing cgroup subsys cpuset
 [    0.000000 ] Initializing cgroup subsys cpu
 [    0.000000 ] Initializing cgroup subsys cpuacct
 [    0.000000 ] Linux version 4.1.0-rc6+ (lth @ncpws04) (gcc version 4.9.2 (Debian 4.9.2-16) )  #1 SMP Tue Jun 9 14:10:06 MYT 2015
 [    0.000000 ] e820: BIOS-provided physical RAM map:
 [    0.000000 ] BIOS-e820:  [mem 0x0000000000000000-0x0000000000097fff ] usable
 [    0.000000 ] BIOS-e820:  [mem 0x0000000000100000-0x0000000003ffffff ] usable
 [    0.000000 ] BIOS-e820:  [mem 0x0000000004000000-0x0000000005ffffff ] reserved
 [    0.000000 ] BIOS-e820:  [mem 0x0000000006000000-0x000000003f4fffff ] usable
 [    0.000000 ] BIOS-e820:  [mem 0x000000003f500000-0x000000003fffffff ] reserved
 [    0.000000 ] BIOS-e820:  [mem 0x00000000fec00000-0x00000000fec00fff ] reserved
 [    0.000000 ] BIOS-e820:  [mem 0x00000000fec04000-0x00000000fec07fff ] reserved
 [    0.000000 ] BIOS-e820:  [mem 0x00000000fee00000-0x00000000fee00fff ] reserved
 [    0.000000 ] BIOS-e820:  [mem 0x00000000ff000000-0x00000000ffffffff ] reserved
 [    0.000000 ] Notice: NX (Execute Disable) protection cannot be enabled: non-PAE kernel!
 [    0.000000 ] SMBIOS 2.6 present.
 [    0.000000 ] e820: last _pfn = 0x3f500 max _arch _pfn = 0x100000
 [    0.000000 ] PAT configuration  [0-7 ]: WB  WC  UC- UC  WB  WC  UC- UC
 [    0.000000 ] Scanning 1 areas for low memory corruption
 [    0.000000 ] init _memory _mapping:  [mem 0x00000000-0x000fffff ]
 [    0.000000 ] init _memory _mapping:  [mem 0x37000000-0x373fffff ]
 [    0.000000 ] init _memory _mapping:  [mem 0x00100000-0x03ffffff ]
 [    0.000000 ] init _memory _mapping:  [mem 0x06000000-0x36ffffff ]
 [    0.000000 ] init _memory _mapping:  [mem 0x37400000-0x377fdfff ]
 [    0.000000 ] RAMDISK:  [mem 0x00800000-0x00ffffff ]
 [    0.000000 ] ACPI: Early table checksum verification disabled
 [    0.000000 ] ACPI BIOS Error (bug): A valid RSDP was not found (20150410/tbxfroot-243)
 [    0.000000 ] 125MB HIGHMEM available.
 [    0.000000 ] 887MB LOWMEM available.
 [    0.000000 ]   mapped low ram: 0 - 377fe000
 [    0.000000 ]   low ram: 0 - 377fe000
 [    0.000000 ] Zone ranges:
 [    0.000000 ]   DMA       [mem 0x0000000000001000-0x0000000000ffffff ]
 [    0.000000 ]   Normal    [mem 0x0000000001000000-0x00000000377fdfff ]
 [    0.000000 ]   HighMem   [mem 0x00000000377fe000-0x000000003f4fffff ]
 [    0.000000 ] Movable zone start for each node
 [    0.000000 ] Early memory node ranges
 [    0.000000 ]   node   0:  [mem 0x0000000000001000-0x0000000000097fff ]
 [    0.000000 ]   node   0:  [mem 0x0000000000100000-0x0000000003ffffff ]
 [    0.000000 ]   node   0:  [mem 0x0000000006000000-0x000000003f4fffff ]
 [    0.000000 ] Initmem setup node 0  [mem 0x0000000000001000-0x000000003f4fffff ]
 [    0.000000 ] Using APIC driver default
 [    0.000000 ] SFI: Simple Firmware Interface v0.81 http://simplefirmware.org
 [    0.000000 ] SFI: SYST E31F0, 0060 (v1  INTEL INTELFDK)
 [    0.000000 ] SFI: CPUS E3296, 0020 (v1  INTEL INTELFDK)
 [    0.000000 ] SFI: FREQ E32C2, 0030 (v1  INTEL INTELFDK)
 [    0.000000 ] SFI: MMAP E32FE, 01A4 (v1  INTEL INTELFDK)
 [    0.000000 ] SFI: XSDT E34B0, 002C (v1  INTEL INTELFDK)
 [    0.000000 ] SFI: APIC E353E, 0020 (v1  INTEL INTELFDK)
 [    0.000000 ] SFI: WAKE E356A, 0020 (v2  INTEL INTELFDK)
 [    0.000000 ] SFI: DEVS E359E, 047D (v1  INTEL INTELFDK)
 [    0.000000 ] SFI: GPIO E3A27, 0964 (v1  INTEL INTELFDK)
 [    0.000000 ] SFI: OEMB E4397, 0060 (v5 UMGFDK CFGINFO!)
 [    0.000000 ] SFI: registering lapic [0 ]
 [    0.000000 ] SFI: registering lapic [2 ]
 [    0.000000 ] IOAPIC [0 ]: apic _id 0, version 32, address 0xfec00000, GSI 0-54
 [    0.000000 ] smpboot: Allowing 2 CPUs, 0 hotplug CPUs
 [    0.000000 ] PM: Registered nosave memory:  [mem 0x00000000-0x00000fff ]
 [    0.000000 ] PM: Registered nosave memory:  [mem 0x00098000-0x000fffff ]
 [    0.000000 ] PM: Registered nosave memory:  [mem 0x04000000-0x05ffffff ]
 [    0.000000 ] e820:  [mem 0x40000000-0xfebfffff ] available for PCI devices
 [    0.000000 ] clocksource refined-jiffies: mask: 0xffffffff max _cycles: 0xffffffff, max _idle _ns: 1910969940391419 ns
 [    0.000000 ] setup _percpu: NR _CPUS:8 nr _cpumask _bits:8 nr _cpu _ids:2 nr _node _ids:1
 [    0.000000 ] PERCPU: Embedded 19 pages/cpu  @f6fe2000 s45516 r0 d32308 u77824
 [    0.000000 ] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 249255
 [    0.000000 ] Kernel command line: console=tty1 console=ttyS2,115200n8 root=/dev/ram0 rw initrd=0x800000,8M
 [    0.000000 ] PID hash table entries: 4096 (order: 2, 16384 bytes)
 [    0.000000 ] Dentry cache hash table entries: 131072 (order: 7, 524288 bytes)
 [    0.000000 ] Inode-cache hash table entries: 65536 (order: 6, 262144 bytes)
 [    0.000000 ] Initializing CPU #0
 [    0.000000 ] Initializing HighMem for node 0 (000377fe:0003f500)
 [    0.000000 ] Initializing Movable for node 0 (00000000:00000000)
 [    0.000000 ] Memory: 975356K/1004124K available (7204K kernel code, 708K rwdata, 2200K rodata, 600K init, 596K bss, 28768K reserved, 0K cma-reserved, 128008K highmem)
 [    0.000000 ] virtual kernel memory layout:
 [    0.000000 ]     fixmap  : 0xfff15000 - 0xfffff000   ( 936 kB)
 [    0.000000 ]     pkmap  : 0xff800000 - 0xffc00000   (4096 kB)
 [    0.000000 ]     vmalloc : 0xf7ffe000 - 0xff7fe000   ( 120 MB)
 [    0.000000 ]     lowmem  : 0xc0000000 - 0xf77fe000   ( 887 MB)
 [    0.000000 ]       .init : 0xc19e4000 - 0xc1a7a000   ( 600 kB)
 [    0.000000 ]       .data : 0xc1709731 - 0xc19e22c0   (2914 kB)
 [    0.000000 ]       .text : 0xc1000000 - 0xc1709731   (7205 kB)
 [    0.000000 ] Checking if this processor honours the WP bit even in supervisor mode ...Ok.
 [    0.000000 ] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
 [    0.000000 ] Hierarchical RCU implementation.
 [    0.000000 ]  Additional per-CPU info printed with stalls.
 [    0.000000 ]  RCU restricting CPUs from NR _CPUS=8 to nr _cpu _ids=2.
 [    0.000000 ] RCU: Adjusting geometry for rcu _fanout _leaf=16, nr _cpu _ids=2
 [    0.000000 ] NR _IRQS:2304 nr _irqs:512 0
 [    0.000000 ] Console: colour dummy device 80x25
 [    0.000000 ] console  [tty1 ] enabled
 [    0.000000 ] console  [ttyS2 ] enabled
 [    0.000000 ] tsc: Detected 499.200 MHz processor
 [    0.000009 ] Calibrating delay loop (skipped), value calculated using timer frequency.. 998.40 BogoMIPS (lpj=499200)
 [    0.000311 ] pid _max: default: 32768 minimum: 301
 [    0.000541 ] Security Framework initialized
 [    0.000674 ] SELinux:  Initializing.
 [    0.000848 ] Mount-cache hash table entries: 2048 (order: 1, 8192 bytes)
 [    0.001040 ] Mountpoint-cache hash table entries: 2048 (order: 1, 8192 bytes)
 [    0.002066 ] Initializing cgroup subsys freezer
 [    0.002276 ] CPU: Physical Processor ID: 0
 [    0.002402 ] CPU: Processor Core ID: 0
 [    0.002522 ] ENERGY _PERF _BIAS: Set to  'normal ', was  'performance '
 [    0.002691 ] ENERGY _PERF _BIAS: View and update with x86 _energy _perf _policy(8)
 [    0.018477 ] mce: CPU supports 6 MCE banks
 [    0.018616 ] CPU0: Thermal monitoring enabled (TM1)
 [    0.018760 ] process: using mwait in idle threads
 [    0.018910 ] Last level iTLB entries: 4KB 48, 2MB 0, 4MB 0
 [    0.019068 ] Last level dTLB entries: 4KB 128, 2MB 16, 4MB 16, 1GB 0
 [    0.019909 ] Freeing SMP alternatives memory: 32K (c1a7a000 - c1a82000)
 [    0.020110 ] SFI: MCFG E34F6, 003C (v1  INTEL INTELFDK)
 [    0.020512 ] Enabling APIC mode:  Flat.  Using 1 I/O APICs
 [    0.020720 ] smpboot: CPU0: Genuine Intel(R) CPU   4000  @  500MHz (fam: 06, model: 4a, stepping: 08)
 [    0.021086 ] Performance Events: PEBS fmt2+, generic architected perfmon, full-width counters, Intel PMU driver.
 [    0.021415 ]  ... version:                3
 [    0.021537 ]  ... bit width:              40
 [    0.021659 ]  ... generic registers:      2
 [    0.021819 ]  ... value mask:             000000ffffffffff
 [    0.021975 ]  ... max period:             000000ffffffffff
 [    0.022127 ]  ... fixed-purpose events:   3
 [    0.022248 ]  ... event mask:             0000000700000003
 [    0.024499 ] x86: Booting SMP configuration:
 [    0.024632 ]  .... node   #0, CPUs:       #1
 [    0.035782 ] Initializing CPU #1
 [    0.051678 ] Skipped synchronization checks as TSC is reliable.
 [    0.051968 ] x86: Booted up 1 node, 2 CPUs
 [    0.052099 ] smpboot: Total of 2 processors activated (1996.80 BogoMIPS)
 [    0.053680 ] devtmpfs: initialized
 [    0.055857 ] clocksource jiffies: mask: 0xffffffff max _cycles: 0xffffffff, max _idle _ns: 1911260446275000 ns
 [    0.055865 ] kworker/u4:0 (18) used greatest stack depth: 7520 bytes left
 [    0.056812 ] SFI: SFI sysfs interfaces init success
 [    0.057031 ] RTC time:  0:00:11, date: 01/01/00
 [    0.058027 ] NET: Registered protocol family 16
 [    0.058871 ] kworker/u4:1 (20) used greatest stack depth: 7048 bytes left
 [    0.066445 ] cpuidle: using governor ladder
 [    0.075479 ] cpuidle: using governor menu
 [    0.076803 ] PCI: MMCONFIG for domain 0000  [bus 00-00 ] at  [mem 0x3f500000-0x3f5fffff ] (base 0x3f500000)
 [    0.077078 ] PCI: MMCONFIG at  [mem 0x3f500000-0x3f5fffff ] reserved in E820
 [    0.077268 ] PCI: Using MMCONFIG for extended config space
 [    0.077423 ] PCI: Using configuration type 1 for base access
 [    0.120030 ] ACPI: Interpreter disabled.
 [    0.120577 ] vgaarb: loaded
 [    0.121309 ] SCSI subsystem initialized
 [    0.122302 ] usbcore: registered new interface driver usbfs
 [    0.122560 ] usbcore: registered new interface driver hub
 [    0.122815 ] usbcore: registered new device driver usb
 [    0.123225 ] pps _core: LinuxPPS API ver. 1 registered
 [    0.123377 ] pps _core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti  <giometti @linux.it >
 [    0.123684 ] PTP clock support registered
 [    0.124480 ] Advanced Linux Sound Architecture Driver Initialized.
 [    0.124871 ] Intel MID platform detected, using MID PCI ops
 [    0.125056 ] PCI: Probing PCI hardware
 [    0.125377 ] PCI host bridge to bus 0000:00
 [    0.125520 ] pci _bus 0000:00: root bus resource  [io  0x0000-0xffff ]
 [    0.125704 ] pci _bus 0000:00: root bus resource  [mem 0x00000000-0xffffffff ]
 [    0.125903 ] pci _bus 0000:00: No busn resource found for root bus, will use  [bus 00-ff ]
 [    0.146027 ] cfg80211: Calling CRDA to update world regulatory domain
 [    0.146324 ] NetLabel: Initializing
 [    0.146438 ] NetLabel:  domain hash size = 128
 [    0.146591 ] NetLabel:  protocols = UNLABELED CIPSOv4
 [    0.146808 ] NetLabel:  unlabeled traffic allowed by default
 [    0.147843 ] Switched to clocksource refined-jiffies
 [    0.189437 ] pnp: PnP ACPI: disabled
 [    0.206085 ] NET: Registered protocol family 2
 [    0.207157 ] TCP established hash table entries: 8192 (order: 3, 32768 bytes)
 [    0.207455 ] TCP bind hash table entries: 8192 (order: 4, 65536 bytes)
 [    0.207814 ] TCP: Hash tables configured (established 8192 bind 8192)
 [    0.208076 ] UDP hash table entries: 512 (order: 2, 16384 bytes)
 [    0.208281 ] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes)
 [    0.208715 ] NET: Registered protocol family 1
 [    0.209279 ] RPC: Registered named UNIX socket transport module.
 [    0.209460 ] RPC: Registered udp transport module.
 [    0.209601 ] RPC: Registered tcp transport module.
 [    0.209739 ] RPC: Registered tcp NFSv4.1 backchannel transport module.
 [    0.212895 ] Unpacking initramfs ...
 [    1.928263 ] Initramfs unpacking failed: junk in compressed archive
 [    1.935823 ] Freeing initrd memory: 8192K (c0800000 - c1000000)
 [    1.937111 ] microcode: CPU0 sig=0x406a8, pf=0x1, revision=0x81f
 [    1.937313 ] microcode: CPU1 sig=0x406a8, pf=0x1, revision=0x81f
 [    1.937748 ] microcode: Microcode Update Driver: v2.00  <tigran @aivazian.fsnet.co.uk >, Peter Oruba
 [    1.939259 ] Scanning for low memory corruption every 60 seconds
 [    1.941624 ] futex hash table entries: 512 (order: 3, 32768 bytes)
 [    1.941911 ] audit: initializing netlink subsys (disabled)
 [    1.942175 ] audit: type=2000 audit(946684812.709:1): initialized
 [    1.943790 ] HugeTLB registered 4 MB page size, pre-allocated 0 pages
 [    1.953369 ] kworker/u4:0 (584) used greatest stack depth: 6944 bytes left
 [    1.958323 ] VFS: Disk quotas dquot _6.6.0
 [    1.958765 ] VFS: Dquot-cache hash table entries: 1024 (order 0, 4096 bytes)
 [    1.963405 ] NFS: Registering the id _resolver key type
 [    1.963627 ] Key type id _resolver registered
 [    1.963758 ] Key type id _legacy registered
 [    1.967465 ] bounce: pool size: 64 pages
 [    1.967909 ] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 250)
 [    1.968147 ] io scheduler noop registered
 [    1.968280 ] io scheduler deadline registered
 [    1.968896 ] io scheduler cfq registered (default)
 [    1.971213 ] pci _hotplug: PCI Hot Plug PCI Core version: 0.5
 [    1.973607 ] hsu _dma _pci 0000:00:05.0: Found HSU DMA, 12 channels
 [    1.974238 ] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
 [    1.976378 ] serial: probe of 0000:00:04.0 failed with error -16
 [    1.977005 ] 0000:00:04.1: ttyS0 at MMIO 0xff010080 (irq = 28, base _baud = 1843200) is a TI16750
 [    1.978131 ] 0000:00:04.2: ttyS1 at MMIO 0xff010100 (irq = 29, base _baud = 1843200) is a TI16750
 [    1.979118 ] console  [ttyS2 ] disabled
 [    1.979289 ] 0000:00:04.3: ttyS2 at MMIO 0xff010180 (irq = 54, base _baud = 1843200) is a TI16750
 [    3.127872 ] console  [ttyS2 ] enabled
 [    3.132778 ] Non-volatile memory driver v1.3
 [    3.137593 ] Linux agpgart interface v0.103
 [    3.142225 ]  [drm ] Initialized drm 1.1.0 20060810
 [    3.154806 ] loop: module loaded
 [    3.160193 ] e100: Intel(R) PRO/100 Network Driver, 3.5.24-k2-NAPI
 [    3.166339 ] e100: Copyright(c) 1999-2006 Intel Corporation
 [    3.172025 ] e1000: Intel(R) PRO/1000 Network Driver - version 7.3.21-k8-NAPI
 [    3.179130 ] e1000: Copyright (c) 1999-2006 Intel Corporation.
 [    3.185067 ] e1000e: Intel(R) PRO/1000 Network Driver - 3.2.5-k
 [    3.190944 ] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
 [    3.197064 ] sky2: driver version 1.30
 [    3.602087 ] clocksource tsc: mask: 0xffffffffffffffff max _cycles: 0xe642f98297, max _idle _ns: 881590439301 ns
 [    3.603171 ] xhci-hcd xhci-hcd.1.auto: xHCI Host Controller
 [    3.603549 ] xhci-hcd xhci-hcd.1.auto: new USB bus registered, assigned bus number 1
 [    3.603725 ] xhci-hcd xhci-hcd.1.auto: hcc params 0x0220f06c hci version 0x100 quirks 0x00010010
 [    3.603803 ] xhci-hcd xhci-hcd.1.auto: irq 34, io mem 0xf9100000
 [    3.604036 ] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002
 [    3.604047 ] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
 [    3.604056 ] usb usb1: Product: xHCI Host Controller
 [    3.604065 ] usb usb1: Manufacturer: Linux 4.1.0-rc6+ xhci-hcd
 [    3.604074 ] usb usb1: SerialNumber: xhci-hcd.1.auto
 [    3.605034 ] hub 1-0:1.0: USB hub found
 [    3.605088 ] hub 1-0:1.0: 1 port detected
 [    3.605539 ] xhci-hcd xhci-hcd.1.auto: xHCI Host Controller
 [    3.605844 ] xhci-hcd xhci-hcd.1.auto: new USB bus registered, assigned bus number 2
 [    3.606107 ] usb usb2: New USB device found, idVendor=1d6b, idProduct=0003
 [    3.606118 ] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
 [    3.606321 ] usb usb2: Product: xHCI Host Controller
 [    3.606330 ] usb usb2: Manufacturer: Linux 4.1.0-rc6+ xhci-hcd
 [    3.606338 ] usb usb2: SerialNumber: xhci-hcd.1.auto
 [    3.607467 ] hub 2-0:1.0: USB hub found
 [    3.607517 ] hub 2-0:1.0: 1 port detected
 [    3.608316 ] ehci _hcd: USB 2.0  'Enhanced ' Host Controller (EHCI) Driver
 [    3.608328 ] ehci-pci: EHCI PCI platform driver
 [    3.608524 ] ohci _hcd: USB 1.1  'Open ' Host Controller (OHCI) Driver
 [    3.608550 ] ohci-pci: OHCI PCI platform driver
 [    3.608825 ] uhci _hcd: USB Universal Host Controller Interface driver
 [    3.609276 ] usbcore: registered new interface driver usblp
 [    3.609526 ] usbcore: registered new interface driver usb-storage
 [    3.609869 ] mousedev: PS/2 mouse device common for all mice
 [    3.612697 ] intel _mid _wdt intel _mid _wdt: Intel MID watchdog device probed
 [    3.613581 ] device-mapper: ioctl: 4.31.0-ioctl (2015-3-12) initialised: dm-devel @redhat.com
 [    3.613908 ] hidraw: raw HID events driver (C) Jiri Kosina
 [    3.617356 ] usbcore: registered new interface driver usbhid
 [    3.617361 ] usbhid: USB HID core driver
 [    3.621574 ] Netfilter messages via NETLINK v0.30.
 [    3.621642 ] nf _conntrack version 0.5.0 (15368 buckets, 61472 max)
 [    3.622865 ] ctnetlink v0.93: registering with nfnetlink.
 [    3.623551 ] ip _tables: (C) 2000-2006 Netfilter Core Team
 [    3.623673 ] Initializing XFRM netlink socket
 [    3.628155 ] NET: Registered protocol family 10
 [    3.629331 ] ip6 _tables: (C) 2000-2006 Netfilter Core Team
 [    3.629608 ] sit: IPv6 over IPv4 tunneling driver
 [    3.631471 ] NET: Registered protocol family 17
 [    3.631596 ] Key type dns _resolver registered
 [    3.633160 ] Using IPI No-Shortcut mode
 [    3.634441 ] registered taskstats version 1
 [    3.637184 ]   Magic number: 4:6:0
 [    3.863597 ] console  [netcon0 ] enabled
 [    3.867317 ] netconsole: network logging started
 [    3.871941 ] Switched to clocksource tsc
 [    3.872297 ] ALSA device list:
 [    3.872300 ]   No soundcards found.
 [    3.883075 ] Freeing unused kernel memory: 600K (c19e4000 - c1a7a000)
 [    3.889884 ] Write protecting the kernel text: 7208k
 [    3.895200 ] Write protecting the kernel read-only data: 2204k
Starting logging: OK
Starting mdev ...
 [    3.996795 ] sh (842) used greatest stack depth: 6900 bytes left
 [    4.019437 ] cfg80211: Calling CRDA to update world regulatory domain
 [    4.032645 ] mdev (844) used greatest stack depth: 6892 bytes left
 [    4.099253 ] mdev (839) used greatest stack depth: 6876 bytes left
Starting watchdog ...
Initializing random number generator ...
[    4.119398 ] random: dd
urandom read with 0 bits of entropy available
done.
Starting network ...
 [    4.141680 ] ip (859) used greatest stack depth: 6384 bytes left
Welcome to Buildroot
buildroot login:
```
### CPU Info
```
# cat /proc/cpuinfo
processor      : 0
vendor_id      : GenuineIntel
cpu family      : 6
model          : 74
model name      : Genuine Intel(R) CPU   4000  @  500MHz
stepping        : 8
microcode      : 0x81f
cpu MHz        : 499.200
cache size      : 1024 KB
physical id    : 0
siblings        : 2
core id        : 0
cpu cores      : 2
apicid          : 0
initial apicid  : 0
fdiv_bug        : no
f00f_bug        : no
coma_bug        : no
fpu            : yes
fpu_exception  : yes
cpuid level    : 11
wp              : yes
flags          : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge
mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe nx
rdtscp lm constant_tsc arch_perfmon pebs bts xtopology nonstop_tsc
aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx est tm2 ssse3 cx16
xtpr pdcm sse4_1 sse4_2 movbe popcnt tsc_deadline_timer aes rdrand
lahf_lm 3dnowprefetch ida arat epb dtherm tpr_shadow vnmi flexpriority
ept vpid tsc_adjust smep erms
bugs            :
bogomips        : 998.40
clflush size    : 64
cache_alignment : 64
address sizes  : 36 bits physical, 48 bits virtual
power management:

processor      : 1
vendor_id      : GenuineIntel
cpu family      : 6
model          : 74
model name      : Genuine Intel(R) CPU   4000  @  500MHz
stepping        : 8
microcode      : 0x81f
cpu MHz        : 499.200
cache size      : 1024 KB
physical id    : 0
siblings        : 2
core id        : 1
cpu cores      : 2
apicid          : 2
initial apicid  : 2
fdiv_bug        : no
f00f_bug        : no
coma_bug        : no
fpu            : yes
fpu_exception  : yes
cpuid level    : 11
wp              : yes
flags          : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge
mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe nx
rdtscp lm constant_tsc arch_perfmon pebs bts xtopology nonstop_tsc
aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx est tm2 ssse3 cx16
xtpr pdcm sse4_1 sse4_2 movbe popcnt tsc_deadline_timer aes rdrand
lahf_lm 3dnowprefetch ida arat epb dtherm tpr_shadow vnmi flexpriority
ept vpid tsc_adjust smep erms
bugs            :
bogomips        : 998.40
clflush size    : 64
cache_alignment : 64
address sizes  : 36 bits physical, 48 bits virtual
```
 

### PCI Bus

Note, Intel MID platforms have only few PCI devices, i.e. PCI root
bridge, GFX, IPU. Everything else are located in so called PCI Shim,
which is just a slice of memory. One may say the rest of the devices are
not PCI, but mimic PCI programming interface (with some mistakes like
using IRQ line 0 for no interrupt assigned).

```
# lspci
00:00.0 Host bridge: Intel Corporation Device 1170 (rev 01)
00:01.0 SD Host controller: Intel Corporation Merrifield SD/SDIO/eMMC Controller (rev 01)
00:01.2 SD Host controller: Intel Corporation Merrifield SD/SDIO/eMMC Controller (rev 01)
00:01.3 SD Host controller: Intel Corporation Merrifield SD/SDIO/eMMC Controller (rev 01)
00:02.0 Display controller: Intel Corporation Device 1182 (rev 01)
00:04.0 Serial controller: Intel Corporation Merrifield Serial IO HSUART Controller (rev 01)
00:04.1 Serial controller: Intel Corporation Merrifield Serial IO HSUART Controller (rev 01)
00:04.2 Serial controller: Intel Corporation Merrifield Serial IO HSUART Controller (rev 01)
00:04.3 Serial controller: Intel Corporation Merrifield Serial IO HSUART Controller (rev 01)
00:05.0 Serial controller: Intel Corporation Merrifield Serial IO HSUART DMA Controller (rev 01)
00:06.0 System peripheral: Intel Corporation Device 1193 (rev 01)
00:06.1 System peripheral: Intel Corporation Device 1193 (rev 01)
00:07.0 System peripheral: Intel Corporation Merrifield Serial IO SPI Controller (rev 01)
00:07.1 System peripheral: Intel Corporation Merrifield Serial IO SPI Controller (rev 01)
00:07.2 System peripheral: Intel Corporation Merrifield Serial IO SPI Controller (rev 01)
00:08.0 Communication controller: Intel Corporation Merrifield Serial IO I2C Controller (rev 01)
00:08.1 Communication controller: Intel Corporation Merrifield Serial IO I2C Controller (rev 01)
00:08.2 Communication controller: Intel Corporation Merrifield Serial IO I2C Controller (rev 01)
00:08.3 Communication controller: Intel Corporation Merrifield Serial IO I2C Controller (rev 01)
00:09.0 Communication controller: Intel Corporation Merrifield Serial IO I2C Controller (rev 01)
00:09.1 Communication controller: Intel Corporation Merrifield Serial IO I2C Controller (rev 01)
00:09.2 Communication controller: Intel Corporation Merrifield Serial IO I2C Controller (rev 01)
00:0a.0 Communication controller: Intel Corporation Device 1197 (rev 01)
00:0b.0 Encryption controller: Intel Corporation Device 1198 (rev 01)
00:0c.0 System peripheral: Intel Corporation Merrifield GPIO Controller (rev 01)
00:0d.0 Multimedia audio controller: Intel Corporation Device 119a (rev 01)
00:0e.0 System peripheral: Intel Corporation Device 119b (rev 01)
00:11.0 USB controller: Intel Corporation Merrifield USB Device Controller (OTG) (rev 01)
00:12.0 Signal processing controller: Intel Corporation Device 119f (rev 01)
00:13.0 Co-processor: Intel Corporation Merrifield SCU IPC (rev 01)
00:14.0 Co-processor: Intel Corporation Merrifield Power Management Unit (rev 01)
00:15.0 System peripheral: Intel Corporation Merrifield Serial IO DMA Controller (rev 01)
00:16.0 Co-processor: Intel Corporation Device 11a3 (rev 01)
00:16.1 Co-processor: Intel Corporation Device 11a4 (rev 01)
00:17.0 System peripheral: Intel Corporation Merrifield Serial IO PWM Controller (rev 01)
00:18.0 Display controller: Intel Corporation Device 11a6 (rev 01)
```

### What is working and what doesn't

As of today Linux next repository has a support of I2C, GPIO expanders
(Arduino version), SPI, pinctrl, PWRMU (yoohoo!), and even more. We have
PWRMU enabled, which means that South Complex devices will be powered
automatically when needed.

Starving for volunteers:

  * [ACPI for Edison](acpi)
    (U-Boot support)

Current work in progress covers:

  * [ACPI for Edison](acpi)
    (U-Boot and kernel support)

Low priority:

  * Supporting full FIFO size in SPI host (now 16 bytes are used out of 32)
  * Enabling I2S (LPE Audio)
  * ~~MCU / Zephyr (if even possible)~~ It's not possible AFAIK without Intel
    involvement. For now considering just forward port of MCU
    application loader
  * P-Unit (to power off unused IPs in North Complex, such as GPU, Camera)

Rather never will be done (anyone who would like to do this?):

  * ~~Pin control support for I2C family (enabling I2C6 mode)~~ Sven Schwermer
    [did
    realize](https://svenschwermer.de/2017/10/14/intel-edison-i2c6-with-vanilla-linux.html)
    what's missed to make it working based on the patches I wrote to
    pinctrl-merrifield.c driver.

[What is done in experimental
[branch](https://github.com/andy-shev/linux/tree/eds)
(from time to time I submit the patches upstream):

  * Various bug fixes

What is in upstream (in order of appearance):

  * HS UART and HSU DMA (v4.1)
  * x86_64 build (v4.5)
  * I2C (v4.8)
  * Pinctrl and GPIO drivers are in upstream (v4.8)
  * SPI in PIO mode (v4.8)
  * Wi-Fi is working, at least it connects to open networks where I'm able to
    ping and get some data from server (v4.9)
  * SD card detection and SD card interface are enabled (v4.9)
  * intel_idle (v4.10)
  * FBTFT bug fixes (v4.11-rc1)
  * GPDMA support for SPI (v4.11-rc1)
  * Power Button support (v4.11-rc3)
  * PWM support (v4.11-rc1)
  * RTC support (v4.11-rc1)
  * Bluetooth, requires testing and debugging existing bugs (v4.12-rc1)
  * Enabling TI ADS7951 A/DC connected to SPI on Edison/Arduino (v4.14-rc1). Note,
    it requires ACPI-enabled platform.

Thread about progress on Intel Communities site: [Newer Kernel on
Edison (OpenWrt)](https://communities.intel.com/thread/75472).


## Conclusion

This shows that it is possible to run vanilla Linux kernel on the
[Edison](https://edison-fw.github.io/meta-intel-edison/).
It might not support all the peripherals that are available on the
[Edison](https://edison-fw.github.io/meta-intel-edison/)
(eMMC, i2c, i2s, spi etc.), but it is possible to boot and obtain a
serial console showing the kernel output, and that is a pretty darn good
start of porting the remaining drivers.
