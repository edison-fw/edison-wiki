# Linux 4.0.x on Intel Edison
Linux 4.1 have been officially released, so rather than continuing down along this path, focus have been moved to [vanilla Linux kernel](vanilla)

As described on [this page](vanilla) it is possible to boot the Intel® [Edison](https://edison-fw.github.io/meta-intel-edison/) using a vanilla Linux kernel rather than the factory kernel used in the Yocto builds. Unfortunately that kernel is a release candidate. Ideally, the [Edison](https://edison-fw.github.io/meta-intel-edison/) should boot using the latest stable kernel, which at the time of writing is the 4.0 series (I think 4.0.3, but who's counting).

Ionic and OpenWrt are constantly being updated to use the latest kernels, so this page focuses on the changed/patches necessary to get a 4.0 kernel running on the [Edison](https://edison-fw.github.io/meta-intel-edison/).

##Serial Console

In order to get the serial console working, the High Speed UART DMA features from the 4.1 kernel is necessary. The
drivers/dma/hsu builds fine with the 4.0 kernel, so this is pretty straight forward. 

Patch is available [here](https://github.com/bright-things/ionic/blob/edison/target/linux/edison/patches-4.0/010-hsu.patch)

The result is:
```
******************************
PSH KERNEL VERSION: b0182b2b
                WR: 20104000
******************************

SCU IPC: 0x800000d0  0xfffce92c

PSH miaHOB version: TNG.B0.VVBD.0000000c

microkernel built 11:24:08 Feb  5 2015

******* PSH loader *******
PCM page cache size = 192 KB
Cache Constraint = 0 Pages
Arming IPC driver ..
Adding page store pool ..
PagestoreAddr(IMR Start Address) = 0x04899000
pageStoreSize(IMR Size)          = 0x00080000

*** Ready to receive application ***

U-Boot 2014.04 (Jun 04 2015 - 12:37:07)

       Watchdog enabled
DRAM:  980.6 MiB
MMC:   tangier_sdhci: 0
In:    serial
Out:   serial
Err:   serial
Hit any key to stop autoboot:  0
boot > run bootcmd_edsboot
reading vmlinuz.efi
5096112 bytes read in 131 ms (37.1 MiB/s)
reading initrd
                                                                       

1645487 bytes read in 55 ms (28.5 MiB/s)
Valid Boot Flag
Setup Size = 0x00003c00
Magic signature found
Using boot protocol version 2.0d
Linux kernel version 4.0.4 (lth@ncpws04) #1 SMP Mon Jun 15 19:21:05
MYT 2015
Building boot_params at 0x00090000
Loading bzImage at address 00100000 (5080752 bytes)
Magic signature found
Initial RAM disk at linear address 0x00800000, size 8388608 bytes
Kernel command line: "console=tty1 console=ttyS2,115200n8
root=/dev/ram0 rw initrd=0x800000,8M"

Starting kernel ...

[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Linux version 4.0.4 (lth@ncpws04) (gcc version 4.8.3
(OpenWrt/Linaro GCC 4.8-2014.04 r45695) ) #1 SMP Mon Jun 15 19:21:05
MYT 2015
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem
0x0000000000000000-0x0000000000097fff] usable
[    0.000000] BIOS-e820: [mem
0x0000000000100000-0x0000000003ffffff] usable
[    0.000000] BIOS-e820: [mem
0x0000000004000000-0x0000000005ffffff] reserved
[    0.000000] BIOS-e820: [mem
0x0000000006000000-0x000000003f4fffff] usable
[    0.000000] BIOS-e820: [mem
0x000000003f500000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem
0x00000000fec00000-0x00000000fec00fff] reserved
[    0.000000] BIOS-e820: [mem
0x00000000fec04000-0x00000000fec07fff] reserved
[    0.000000] BIOS-e820: [mem
0x00000000fee00000-0x00000000fee00fff] reserved
[    0.000000] BIOS-e820: [mem
0x00000000ff000000-0x00000000ffffffff] reserved
[    0.000000] Notice: NX (Execute Disable) protection cannot be enabled: non-PAE kernel!
[    0.000000] SMBIOS 2.6 present.
[    0.000000] e820: last_pfn = 0x3f500 max_arch_pfn = 0x100000
[    0.000000] PAT configuration [0-7]: WB  WC  UC- UC  WB  WC  UC-
UC
[    0.000000] Scanning 1 areas for low memory corruption
[    0.000000] init_memory_mapping: [mem 0x00000000-0x000fffff]
[    0.000000] init_memory_mapping: [mem 0x37000000-0x373fffff]
[    0.000000] init_memory_mapping: [mem 0x00100000-0x03ffffff]
[    0.000000] init_memory_mapping: [mem 0x06000000-0x36ffffff]
[    0.000000] init_memory_mapping: [mem 0x37400000-0x377fdfff]
[    0.000000] RAMDISK: [mem 0x00800000-0x00ffffff]
[    0.000000] ACPI: Early table checksum verification disabled
[    0.000000] ACPI BIOS Error (bug): A valid RSDP was not found (20150204/tbxfroot-243)
[    0.000000] 125MB HIGHMEM available.
[    0.000000] 887MB LOWMEM available.
[    0.000000]   mapped low ram: 0 - 377fe000
[    0.000000]   low ram: 0 - 377fe000
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000000001000-0x0000000000ffffff]
[    0.000000]   Normal   [mem 0x0000000001000000-0x00000000377fdfff]
[    0.000000]   HighMem  [mem 0x00000000377fe000-0x000000003f4fffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000001000-0x0000000000097fff]
[    0.000000]   node   0: [mem 0x0000000000100000-0x0000000003ffffff]
[    0.000000]   node   0: [mem 0x0000000006000000-0x000000003f4fffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000001000-0x000000003f4fffff]
[    0.000000] Using APIC driver default
[    0.000000] SFI: Simple Firmware Interface v0.81 http://simplefirmware.org
[    0.000000] SFI: SYST E31F0, 0060 (v1  INTEL INTELFDK)
[    0.000000] SFI: CPUS E3296, 0020 (v1  INTEL INTELFDK)
[    0.000000] SFI: FREQ E32C2, 0030 (v1  INTEL INTELFDK)
[    0.000000] SFI: MMAP E32FE, 01A4 (v1  INTEL INTELFDK)
[    0.000000] SFI: XSDT E34B0, 002C (v1  INTEL INTELFDK)
[    0.000000] SFI: APIC E353E, 0020 (v1  INTEL INTELFDK)
[    0.000000] SFI: WAKE E356A, 0020 (v2  INTEL INTELFDK)
[    0.000000] SFI: DEVS E359E, 047D (v1  INTEL INTELFDK)
[    0.000000] SFI: GPIO E3A27, 0964 (v1  INTEL INTELFDK)
[    0.000000] SFI: OEMB E4397, 0060 (v5 UMGFDK CFGINFO!)
[    0.000000] SFI: registering lapic[0]
[    0.000000] SFI: registering lapic[2]
[    0.000000] IOAPIC[0]: apic_id 0, version 32, address 0xfec00000, GSI 0-54
[    0.000000] smpboot: Allowing 2 CPUs, 0 hotplug CPUs
[    0.000000] PM: Registered nosave memory: [mem 0x00000000-0x00000fff]
[    0.000000] PM: Registered nosave memory: [mem 0x00098000-0x000fffff]
[    0.000000] PM: Registered nosave memory: [mem 0x04000000-0x05ffffff]
[    0.000000] e820: [mem 0x40000000-0xfebfffff] available for PCI devices
[    0.000000] setup_percpu: NR_CPUS:8 nr_cpumask_bits:8 nr_cpu_ids:2 nr_node_ids:1
[    0.000000] PERCPU: Embedded 16 pages/cpu @f6dec000 s35916 r0 d29620 u65536
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 248811
[    0.000000] Kernel command line: console=tty1 console=ttyS2,115200n8 root=/dev/ram0 rw initrd=0x800000,8M
[    0.000000] PID hash table entries: 4096 (order: 2, 16384 bytes)
[    0.000000] Dentry cache hash table entries: 131072 (order: 7, 524288 bytes)
[    0.000000] Inode-cache hash table entries: 65536 (order: 6, 262144 bytes)
[    0.000000] Initializing CPU#0
[    0.000000] Initializing HighMem for node 0 (000377fe:0003f500)
[    0.000000] Initializing Movable for node 0 (00000000:00000000)
[    0.000000] Memory: 974876K/1004124K available (6401K kernel code, 341K rwdata, 1988K rodata, 500K init, 560K bss, 29248K reserved, 0K cma-reserved, 128008K highmem)
[    0.000000] virtual kernel memory layout:
[    0.000000]     fixmap  : 0xfff15000 - 0xfffff000   ( 936 kB)
[    0.000000]     pkmap   : 0xff800000 - 0xffc00000   (4096 kB)
[    0.000000]     vmalloc : 0xf7ffe000 - 0xff7fe000   ( 120 MB)
[    0.000000]     lowmem  : 0xc0000000 - 0xf77fe000   ( 887 MB)
[    0.000000]       .init : 0xc188a000 - 0xc1907000   ( 500 kB)
[    0.000000]       .data : 0xc1640910 - 0xc18886c0   (2335 kB)
[    0.000000]       .text : 0xc1000000 - 0xc1640910   (6402 kB)
[    0.000000] Checking if this processor honours the WP bit even in supervisor mode...Ok.
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] Hierarchical RCU implementation.
[    0.000000]  Additional per-CPU info printed with stalls.
[    0.000000]  RCU restricting CPUs from NR_CPUS=8 to nr_cpu_ids=2.
[    0.000000] RCU: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS:2304 nr_irqs:512 0
[    0.000000] Console: colour dummy device 80x25
[    0.000000] console [tty1] enabled
[    0.000000] console [ttyS2] enabled
[    0.000000] tsc: Detected 499.200 MHz processor
[    0.000007] Calibrating delay loop (skipped), value calculated using timer frequency.. 998.40 BogoMIPS (lpj=499200)
[    0.000315] pid_max: default: 32768 minimum: 301
[    0.000545] Security Framework initialized
[    0.000682] SELinux:  Initializing.
[    0.000863] Mount-cache hash table entries: 2048 (order: 1, 8192 bytes)
[    0.001061] Mountpoint-cache hash table entries: 2048 (order: 1, 8192 bytes)
[    0.002088] Initializing cgroup subsys blkio
[    0.002236] Initializing cgroup subsys memory
[    0.002401] Initializing cgroup subsys devices
[    0.002545] Initializing cgroup subsys freezer
[    0.002687] Initializing cgroup subsys net_cls
[    0.002890] CPU: Physical Processor ID: 0
[    0.003019] CPU: Processor Core ID: 0
[    0.003141] ENERGY_PERF_BIAS: Set to 'normal', was 'performance'
[    0.003313] ENERGY_PERF_BIAS: View and update with x86_energy_perf_policy(8)
[    0.019103] mce: CPU supports 6 MCE banks
[    0.019246] CPU0: Thermal monitoring enabled (TM1)
[    0.019394] process: using mwait in idle threads
[    0.019546] Last level iTLB entries: 4KB 48, 2MB 0, 4MB 0
[    0.019706] Last level dTLB entries: 4KB 128, 2MB 16, 4MB 16, 1GB 0
[    0.020582] Freeing SMP alternatives memory: 28K (c1907000 - c190e000)
[    0.020788] SFI: MCFG E34F6, 003C (v1  INTEL INTELFDK)
[    0.021188] Enabling APIC mode:  Flat.  Using 1 I/O APICs
[    0.021399] smpboot: CPU0: Genuine Intel(R) CPU   4000  @  500MHz (fam: 06, model: 4a, stepping: 08)
[    0.021743] Performance Events: PEBS fmt2+, generic architected perfmon, full-width counters, Intel PMU driver.
[    0.022076] ... version:                3
[    0.022200] ... bit width:              40
[    0.022326] ... generic registers:      2
[    0.022485] ... value mask:             000000ffffffffff
[    0.022644] ... max period:             000000ffffffffff
[    0.022799] ... fixed-purpose events:   3
[    0.022922] ... event mask:             0000000700000003
[    0.025026] x86: Booting SMP configuration:
[    0.025159] .... node  #0, CPUs:      #1
[    0.036325] Initializing CPU#1
[    0.052188] Skipped synchronization checks as TSC is reliable.
[    0.052474] x86: Booted up 1 node, 2 CPUs
[    0.052606] smpboot: Total of 2 processors activated (1996.80 BogoMIPS)
[    0.054163] devtmpfs: initialized
[    0.056182] kworker/u4:0 (17) used greatest stack depth: 7468 bytes left
[    0.056600] SFI: SFI sysfs interfaces init success
[    0.057354] RTC time:  1:42:00, date: 06/16/15
[    0.058358] NET: Registered protocol family 16
[    0.059633] kworker/u4:1 (26) used greatest stack depth: 7228 bytes left
[    0.066637] cpuidle: using governor ladder
[    0.072665] cpuidle: using governor menu
[    0.073987] PCI: MMCONFIG for domain 0000 [bus 00-00] at [mem 0x3f500000-0x3f5fffff] (base 0x3f500000)
[    0.074266] PCI: MMCONFIG at [mem 0x3f500000-0x3f5fffff] reserved in E820
[    0.074459] PCI: Using MMCONFIG for extended config space
[    0.074617] PCI: Using configuration type 1 for base access
[    0.078356] kworker/u4:0 (68) used greatest stack depth: 7224 bytes left
[    0.079075] kworker/u4:1 (76) used greatest stack depth: 7040 bytes left
[    0.110540] ACPI: Interpreter disabled.
[    0.111066] vgaarb: loaded
[    0.111729] SCSI subsystem initialized
[    0.112281] usbcore: registered new interface driver usbfs
[    0.112701] usbcore: registered new interface driver hub
[    0.112957] usbcore: registered new device driver usb
[    0.113358] pps_core: LinuxPPS API ver. 1 registered
[    0.113530] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.113871] PTP clock support registered
[    0.114835] Advanced Linux Sound Architecture Driver Initialized.
[    0.115030] Intel MID platform detected, using MID PCI ops
[    0.115191] PCI: Probing PCI hardware
[    0.115523] PCI host bridge to bus 0000:00
[    0.115700] pci_bus 0000:00: root bus resource [io  0x0000-0xffff]
[    0.115890] pci_bus 0000:00: root bus resource [mem 0x00000000-0xffffffff]
[    0.116092] pci_bus 0000:00: No busn resource found for root bus, will use [bus 00-ff]
[    0.135905] Bluetooth: Core ver 2.20
[    0.136119] NET: Registered protocol family 31
[    0.136260] Bluetooth: HCI device and connection manager initialized
[    0.136451] Bluetooth: HCI socket layer initialized
[    0.136604] Bluetooth: L2CAP socket layer initialized
[    0.136805] Bluetooth: SCO socket layer initialized
[    0.137512] cfg80211: Calling CRDA to update world regulatory domain
[    0.137955] NetLabel: Initializing
[    0.138090] NetLabel:  domain hash size = 128
[    0.138223] NetLabel:  protocols = UNLABELED CIPSOv4
[    0.138435] NetLabel:  unlabeled traffic allowed by default
[    0.139461] Switched to clocksource refined-jiffies
[    0.140003] pnp: PnP ACPI: disabled
[    0.157553] NET: Registered protocol family 2
[    0.158718] TCP established hash table entries: 8192 (order: 3, 32768 bytes)
[    0.159021] TCP bind hash table entries: 8192 (order: 4, 65536 bytes)
[    0.159358] TCP: Hash tables configured (established 8192 bind 8192)
[    0.159630] TCP: reno registered
[    0.159750] UDP hash table entries: 512 (order: 2, 16384 bytes)
[    0.159956] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes)
[    0.160349] NET: Registered protocol family 1
[    0.160971] RPC: Registered named UNIX socket transport module.
[    0.161154] RPC: Registered udp transport module.
[    0.161297] RPC: Registered tcp transport module.
[    0.161438] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.164449] Unpacking initramfs...
[    1.874441] Initramfs unpacking failed: junk in compressed archive
[    1.881992] Freeing initrd memory: 8192K (c0800000 - c1000000)
[    1.883564] microcode: CPU0 sig=0x406a8, pf=0x1, revision=0x81f
[    1.883803] microcode: CPU1 sig=0x406a8, pf=0x1, revision=0x81f
[    1.884247] microcode: Microcode Update Driver: v2.00 <tigran@aivazian.fsnet.co.uk>, Peter Oruba
[    1.885693] Scanning for low memory corruption every 60 seconds
[    1.888023] futex hash table entries: 512 (order: 3, 32768 bytes)
[    1.888291] audit: initializing netlink subsys (disabled)
[    1.888522] audit: type=2000 audit(1434418921.665:1): initialized
[    1.889769] HugeTLB registered 4 MB page size, pre-allocated 0 pages
[    1.902877] VFS: Disk quotas dquot_6.5.2
[    1.903304] VFS: Dquot-cache hash table entries: 1024 (order 0, 4096 bytes)
[    1.907065] NFS: Registering the id_resolver key type
[    1.907312] Key type id_resolver registered
[    1.907445] Key type id_legacy registered
[    1.911435] bounce: pool size: 64 pages
[    1.911579] io scheduler noop registered
[    1.911714] io scheduler deadline registered
[    1.912334] io scheduler cfq registered (default)
[    1.914645] pci_hotplug: PCI Hot Plug PCI Core version: 0.5
[    1.917000] hsu_dma_pci 0000:00:05.0: Found HSU DMA, 12 channels
[    1.917809] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    1.919827] serial: probe of 0000:00:04.0 failed with error -16
[    1.920463] 0000:00:04.1: ttyS0 at MMIO 0xff010080 (irq = 28, base_baud = 1843200) is a TI16750
[    1.921482] 0000:00:04.2: ttyS1 at MMIO 0xff010100 (irq = 29, base_baud = 1843200) is a TI16750
[    1.922442] console [ttyS2] disabled
[    1.922611] 0000:00:04.3: ttyS2 at MMIO 0xff010180 (irq = 54, base_baud = 1843200) is a TI16750
[    3.093353] console [ttyS2] enabled
[    3.098135] Non-volatile memory driver v1.3
[    3.102982] Linux agpgart interface v0.103
[    3.107597] [drm] Initialized drm 1.1.0 20060810
[    3.118824] loop: module loaded
[    3.123262] e100: Intel(R) PRO/100 Network Driver, 3.5.24-k2-NAPI
[    3.129404] e100: Copyright(c) 1999-2006 Intel Corporation
[    3.135075] e1000: Intel(R) PRO/1000 Network Driver - version 7.3.21-k8-NAPI
[    3.142165] e1000: Copyright (c) 1999-2006 Intel Corporation.
[    3.148091] e1000e: Intel(R) PRO/1000 Network Driver - 2.3.2-k
[    3.153967] e1000e: Copyright(c) 1999 - 2014 Intel Corporation.
[    3.160081] sky2: driver version 1.30
[    3.565056] Switched to clocksource tsc
[    3.569521] xhci-hcd xhci-hcd.1.auto: xHCI Host Controller
[    3.575340] xhci-hcd xhci-hcd.1.auto: new USB bus registered, assigned bus number 1
[    3.583237] xhci-hcd xhci-hcd.1.auto: hcc params 0x0220f06c hci version 0x100 quirks 0x00010010
[    3.592062] xhci-hcd xhci-hcd.1.auto: irq 34, io mem 0xf9100000
[    3.598228] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002
[    3.605073] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    3.612383] usb usb1: Product: xHCI Host Controller
[    3.617307] usb usb1: Manufacturer: Linux 4.0.4 xhci-hcd
[    3.622659] usb usb1: SerialNumber: xhci-hcd.1.auto
[    3.628340] hub 1-0:1.0: USB hub found
[    3.632184] hub 1-0:1.0: 1 port detected
[    3.636608] xhci-hcd xhci-hcd.1.auto: xHCI Host Controller
[    3.642399] xhci-hcd xhci-hcd.1.auto: new USB bus registered, assigned bus number 2
[    3.650348] usb usb2: New USB device found, idVendor=1d6b, idProduct=0003
[    3.657192] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    3.664479] usb usb2: Product: xHCI Host Controller
[    3.669396] usb usb2: Manufacturer: Linux 4.0.4 xhci-hcd
[    3.674748] usb usb2: SerialNumber: xhci-hcd.1.auto
[    3.680377] hub 2-0:1.0: USB hub found
[    3.684218] hub 2-0:1.0: 1 port detected
[    3.688709] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    3.695299] ehci-pci: EHCI PCI platform driver
[    3.699905] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    3.706161] ohci-pci: OHCI PCI platform driver
[    3.710757] uhci_hcd: USB Universal Host Controller Interface driver
[    3.717415] usbcore: registered new interface driver usblp
[    3.723113] usbcore: registered new interface driver usb-storage
[    3.729480] mousedev: PS/2 mouse device common for all mice
[    3.736857] intel_mid_wdt intel_mid_wdt: Intel MID watchdog device probed
[    3.744313] device-mapper: ioctl: 4.30.0-ioctl (2014-12-22) initialised: dm-devel@redhat.com
[    3.752870] Bluetooth: HCI UART driver ver 2.2
[    3.757359] Bluetooth: HCI H4 protocol initialized
[    3.762215] Driver 'mmcblk' needs updating - please use bus_type methods
[    3.769026] sdhci: Secure Digital Host Controller Interface driver
[    3.775250] sdhci: Copyright(c) Pierre Ossman
[    3.779718] sdhci-pci 0000:00:01.0: SDHCI controller found [8086:1190] (rev 1)
[    3.787031] sdhci-pci: probe of 0000:00:01.0 failed with error -16
[    3.793470] sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
[    3.800966] sdhci-pci 0000:00:01.3: SDHCI controller found [8086:1190] (rev 1)
[    3.808662] hidraw: raw HID events driver (C) Jiri Kosina
[    3.815010] usbcore: registered new interface driver usbhid
[    3.820640] usbhid: USB HID core driver
[    3.826347] Netfilter messages via NETLINK v0.30.
[    3.831303] nf_conntrack version 0.5.0 (15360 buckets, 61440 max)
[    3.838161] ctnetlink v0.93: registering with nfnetlink.
[    3.844215] ip_tables: (C) 2000-2006 Netfilter Core Team
[    3.849694] TCP: cubic registered
[    3.853060] Initializing XFRM netlink socket
[    3.858455] NET: Registered protocol family 10
[    3.864119] ip6_tables: (C) 2000-2006 Netfilter Core Team
[    3.869825] sit: IPv6 over IPv4 tunneling driver
[    3.875500] NET: Registered protocol family 17
[    3.880144] Bluetooth: RFCOMM socket layer initialized
[    3.885366] Bluetooth: RFCOMM ver 1.11
[    3.889217] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[    3.894585] Bluetooth: BNEP socket layer initialized
[    3.899601] Bluetooth: HIDP (Human Interface Emulation) ver 1.2
[    3.905567] Bluetooth: HIDP socket layer initialized
[    3.910685] Key type dns_resolver registered
[    3.916254] Using IPI No-Shortcut mode
[    3.921126] registered taskstats version 1
[    3.926988]   Magic number: 3:987:710
[    3.930819] console [netcon0] enabled
[    3.934525] netconsole: network logging started
[    3.939495] ALSA device list:
[    3.942525]   No soundcards found.
[    3.946839] Freeing unused kernel memory: 500K (c188a000 - c1907000)
[    3.953569] Write protecting the kernel text: 6404k
[    3.958703] Write protecting the kernel read-only data: 1992k
[    3.973190] mount (783) used greatest stack depth: 6676 bytes left
Starting logging: OK
Starting mdev...
Starting watchdog...
Initializing random number generator... [    4.183372] random: dd
urandom read with 0 bits of entropy available
done.
Starting network...
[    4.206809] ip (817) used greatest stack depth: 6136 bytes left
```

Welcome to Buildroot

buildroot login:

## Power Management

To be added

## Flash Storage

To be added

## I2C

To be added

## SPI

To be added

## Sound/I2S

To be added

## Quark_RT_Processor

To be added
