# U-Boot

This page is about upgrading U-Boot on Intel Edison. The goal is to get Intel Edison supported by U-Boot out-of-the-box. We are anticipating Intel Edison support in **v2017.09** out-of-the-box.


**UPDATE**

So, U-Boot **v2017.09** is out. It supports Intel Edison out of the box. Thus, this page is kinda in a freeze state from now on. Use my Github page to report an issue, if any, or send any of your concerns directly to upstream.

The task I'm working on now is to bring [ACPI](acpi) support for Edison board. Minimum support is available starting from **v2017.11-rc2**.

## New version

I would like to announce I'm able to build and flash latest U-Boot (**v2016.11**) on Edison which allows me to boot x86_64 kernel directly! So, the links:

  * [Using a vanilla Linux kernel with Intel Edison](vanilla)
  * [U-Boot Git tree](https://github.com/andy-shev/u-boot/tree/edison)
  * [Linux kernel Git tree](https://github.com/andy-shev/linux/tree/eds)

I have established tagging in repository. All working cases are tagged with **edison** prefix. As of today (**Sep 19**) U-Boot version bumped to **v2017.09** (stable). Development version is based on **v2017.09**.

To build just run couple of simple commands:
```
 $ make edison_defconfig
 $ make -j16
```
Don't pay attention on non-standard configuration option error. You will have **u-boot.bin** compiled anyway. **Solved (Jan 21).**

First you need to prepare an image to be suitable for DFU. There are few options available. You have to choose one suitable for your case. Most likely it will be **Option 2**.

### Option 1

Installed **edison-v2017.03** (or later) or development version **v2017.03-rc1** (or later). Installing **edison-v2017.03** (or later) or development version **v2017.03-rc1** (or later).

```
  nothing special is required, there are patches that fixes alignment
```

### Option 2

Installed **v2014.04** or **edison-v2016.11**, or **edison-v2017.01**. Installing **edison-v2017.03** (or later) or
development version **v2017.03-rc1** (or later).

```
  truncate -s %4096 u-boot.bin]
```

### Option 3

Installed **v2014.04** or **edison-v2016.11**, or **edison-v2017.01**. Installing **edison-v2016.11**, or **edison-v2017.01**.

```
  $ dd if=u-boot.bin of=u-boot-4k.bin bs=4k seek=1 && truncate -s %4096 u-boot-4k.bin && mv u-boot-4k.bin u-boot.bin
```

When you get a **u-boot.bin** run the following command to get the image you flash with DFU:


```
 $ dfu-util -v -d 8087:0a99 --alt u-boot0 -D u-boot.bin
```

There is [a script](https://gist.github.com/andy-shev/2c388310f2773ead647d9c1a3f1c813f) which allows to create a suitable U-Boot image for xFSTK and DFU including preset environment.

### known-bugs
  * To enable DFU don't forget to change **mmc** to **raw** in the corresponding U-Boot environment variables. See [Issue #3](https://github.com/01org/edison-u-boot/issues/3) as well.
  * ~~GPT table when written by gpt command will make device unresponsive (recovery via xfstk), see [Issue #1](https://github.com/andy-shev/u-boot/issues/1) as well.~~ Solved (**Jan 31**).
  * ~~DFU timeout patch is absent, so if you use default bootcmd setting you have to press Ctrl+C manually each time it boots.~~ In v2020.07.
  * ~~Unclean build (last stage failed due to custom configuration options).~~ Got clean build for U-Boot today (**Jan 21**). Removed few patches from that BSP hell.
  * ~~Upstreaming patches. 11 patches has been sent upstream, 4 of them had already been applied (**Feb 1**4). It's only 10 patches left in the queue, where couple of them rather will be squashed (**Mar 1**). 5 (five) patches left in a queue (**May 10**).~~ Everything is in upstream in **v2017.09-rc1**.

Feel free to use my GitHub page for pull requests (with bug fixes), bug reports, improvements, etc.

```
U-Boot 2017.03-rc2-00019-gb986ef9a09 (Feb 17 2017 - 17:59:07 +0300)

CPU: x86_64, vendor Intel, device 406a8h
DRAM:  980.6 MiB
MMC:   mmc@ff3fc000: 0, mmc@ff3fa000: 1
In:    serial@ff010180
Out:   serial@ff010180
Err:   serial@ff010180
Net:   Net Initialization Skipped
No ethernet found.
Hit any key to stop autoboot:  0
=>
=> pci enum
=> dm tree
 [Class       Probed   Name]{.T2}
----------------------------------------
 [root        [ + ]    root_driver]{.T2}
 [timer       [ + ]    |-- tsc-timer]{.T2}
 [pci         [ + ]    |-- pci]{.T2}
 [pci_generic [   ]    |   |-- pci_0:0.0]{.T2}
 [pci_generic [   ]    |   `-- pci_0:2.0]{.T2}
 [serial      [ + ]    |-- serial@ff010180]{.T2}
 [mmc         [ + ]    |-- mmc@ff3fc000]{.T2}
 [blk         [ + ]    |   `-- mmc@ff3fc000.blk]{.T2}
 [mmc         [ + ]    |-- mmc@ff3fa000]{.T2}
 [blk         [   ]    |   `-- mmc@ff3fa000.blk]{.T2}
 [misc        [ + ]    |-- power@ff00b000]{.T2}
 [simple_bus  [ + ]    `-- cpus]{.T2}
 [cpu         [ + ]        |-- cpu@0]{.T2}
 [cpu         [ + ]        `-- cpu@1]{.T2}
=> cpu list
  [0: cpu@0              Intel(R) Atom(TM) CPU  U1000  @  500MHz]{.T2}
  [1: cpu@1              Intel(R) Atom(TM) CPU  U1000  @  500MHz]{.T2}
=>
```

### Step-by-step example of updating stock version

After boot I have checked the version of U-Boot:

```
U-Boot 2014.04 (Jun 06 2016 - 14:40:07)
```

Interrupted to get U-Boot shell and run:

```
boot > run do_force_flash_os
Saving Environment to MMC...
Writing to redundant MMC(0)... done
GADGET DRIVER: usb_dnl_dfu
```

Open a new terminal and run on the host:

```
 $ make clean && make edison_defconfig && make -j16
 $ truncate -s %4096 u-boot.bin
 $ dfu-util -v -d 8087:0a99 --alt u-boot0 -D u-boot.bin
```

Again in U-Boot shell:

```
#
DFU complete CRC32: 0xf340088e
DOWNLOAD ... OK
Ctrl+C to exit ...
 [boot > reset]{.T2}
resetting ...

******************************
PSH KERNEL VERSION: b0182b2b
                [WR: 20104000]{.T2}
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

U-Boot 2017.03-rc3-00012-ge6566cddc9 (Mar 01 2017 - 21:01:06 +0200)

CPU: x86_64, vendor Intel, device 406a8h
DRAM:  980.6 MiB
MMC:   mmc@ff3fc000: 0, mmc@ff3fa000: 1
In:    serial@ff010180
Out:   serial@ff010180
Err:   serial@ff010180
Net:   Net Initialization Skipped
No ethernet found.
Hit any key to stop autoboot:  0
=>
```
