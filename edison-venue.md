# Dell Venue 7 3740 HSPA+

The Dell Venue 7 is a 7" 800x1280 Tablet with 16GB EMMC / 1G LPDDR3 and LTE
radio with Intel Merrifield dual-core 1.6GHz.

Dell provides images to root, unlock and unbrick as well as source files for the system image [here](https://opensource.dell.com/releases/Venue_8_3840_Merrifield/developer-edition/).

## Making Edison images run on Venue 7 3740

This document contain my notes on getting the Venue 7 to run Linux.

Keep in mind that the approach taken here is backwards: Instead of
carefully analyzing the hardware and the Dell provided sources, build
ACPI tables and drivers into U-Boot and Linux, then iron out the bugs,
here I attempt to start at the (wrong) end.

Knowing that the SOC is similar to Edison's, quickly get Edison's U-Boot, kernel and rootfs running, fixing what is preventing us from booting only. This way we can get the tools to analyze the hardware further and we have a platform to port drivers one by one.

In the following I run Kubuntu 21.10, which provides `fastboot` and `adb` and `xfstk-dldr-solo` built from here: [https://github.com/edison-fw/xFSTK ](https://github.com/edison-fw/xFSTK). This means I use none of the Dell provided tools from the FlashTool directory and needed to write equivalent bash scripts.

### Provisioning Venue 7 for U-Boot

First the Venue 7 must be provisioned with a new firmware before U-Boot can be installed. The Dell manual [OSS_A195.pdf](https://opensource.dell.com/releases/Venue_8_3840_Merrifield/developer-edition/Doc/OSS_A195.pdf) explains what needs to be done:
 1. Bring the Android firmware to a known state (same as unbrick)
 2. Rooting and bootloader unlocking
 3. Flash Android kernel image

During step 2 the partitioning of the EMMC is done. Here we add some modifications to make space for U-Boot's environment (env) partitions. Adding these partitions does not prevent Android from running.

During step 3 we flash U-Boot instead of Android. In principle this should run U-Boot instead of Android and leave Fastboot operational. As it appears, trying to boot into Android will fail and fall back to Recovery while trying to boot into Fastboot will boot into U-Boot. So each time we need Fastboot we need to use Recovery to perform step 2 again. This is a little boring so another reason to get Linux up as quickly as possible.

Currently, inside U-Boot there are things that need to be entered through the console. One way would be to get network access into U-Boot (netconsole). In my case I chose to break the Venue slightly and attached a TTL-USB serial converter following any Shevchenko's instructions [here](https://andy-shev.dreamwidth.org/152335.html).

### 1. Flash A195 image / Unbrick 
 1. Unzip YTD802A519500-2014-07-16-16.tgz.
 2. Create a script `unbrick.sh`.

        #!/bin/bash
        
        xfstk-dldr-solo --gpflags 0x80000007 --osimage droidboot.img.POS.bin --fwdnx dnx_fwr_PRQ.bin --fwimage for_product_ifwi_PRQ.bin --osdnx dnx_osr_PRQ.bin
        
 3. Create a script `P802_flash_device_lte.sh`.

        #!/bin/bash
        
        fastboot oem start_partitioning
        fastboot flash /tmp/partition.tbl partition.tbl
        fastboot oem partition /tmp/partition.tbl
        
        fastboot erase system
        fastboot erase cache
        fastboot erase data
        fastboot erase logs
        fastboot erase config
        
        fastboot oem stop_partitioning
        
        fastboot flash radio radio_firmware.bin
        fastboot flash boot boot.img
        fastboot flash fastboot droidboot.img
        fastboot flash recovery recovery.img
        fastboot flash system system.img
        fastboot flash /tmp/7160_conf_1.tlv 7160_conf_1.tlv
        fastboot oem nvm apply /tmp/7160_conf_1.tlv
        fastboot flash splashscreen_7 oemlogo_7.bin
        fastboot flash splashscreen_8 oemlogo_8.bin
        fastboot flash dnx dnx_fwr_PRQ.bin
        fastboot flash dnx dnx_osr_PRQ.bin
        fastboot flash ifwi for_product_ifwi_PRQ.bin
        
        read -p "Press any key to continue ..."
        
        fastboot reboot
        
 4. Modify `partition.tbl`.

        partition_table=gpt
        create -z /dev/block/mmcblk0
        create /dev/block/mmcblk0
        boot -p /dev/block/mmcblk0
        reload /dev/block/mmcblk0
        add -b 34 -s 524288 -t data -u 80868086-8086-8086-8086-000000000000 -l reserved -T 0 -P 0 /dev/block/mmcblk0
        add -b 524322 -s 65536 -t data -u 80868086-8086-8086-8086-000000000001 -l panic -T 0 -P 0 /dev/block/mmcblk0
        add -b 589858 -s 262144 -t data -u 80868086-8086-8086-8086-000000000002 -l factory -T 0 -P 0 /dev/block/mmcblk0
        add -b 852002 -s 262144 -t data -u 80868086-8086-8086-8086-000000000003 -l misc -T 0 -P 0 /dev/block/mmcblk0
        add -b 1114146 -s 258048 -t data -u 80868086-8086-8086-8086-000000000004 -l config -T 0 -P 0 /dev/block/mmcblk0
        add -b 1372194 -s 2048 -t data -u 80868086-8086-8086-8086-000000000005 -l u-boot-env0 -T 0 -P 0 /dev/block/mmcblk0
        add -b 1374242 -s 2048 -t data -u 80868086-8086-8086-8086-000000000006 -l u-boot-env1 -T 0 -P 0 /dev/block/mmcblk0
        add -b 1376290 -s 3145728 -t data -u 80868086-8086-8086-8086-000000000007 -l cache -T 0 -P 0 /dev/block/mmcblk0
        add -b 4522018 -s 49152 -t data -u 80868086-8086-8086-8086-000000000008 -l logs -T 0 -P 0 /dev/block/mmcblk0
        add -b 4571170 -s 3145728 -t data -u 80868086-8086-8086-8086-000000000009 -l system -T 0 -P 0 /dev/block/mmcblk0
        add -b 7716898 -s 23052254 -t data -u 80868086-8086-8086-8086-000000000010 -l data -T 0 -P 0 /dev/block/mmcblk0
        reload /dev/block/mmcblk0
               
    This will create 2 partitions for the U-Boot env, in addition to those needed by android.

 5. Follow section **A. Flash A195 image to the tablet**.
 6. Create `enable_fastboot.sh`.

        #!/bin/bash
        
        xfstk-dldr-solo --gpflags 0x80000007 --fwdnx  fwr_dnx_PRQ_ww27_001.bin --fwimage IFWI_MERR_PRQ_UOS_TH2_YT2_ww27_001.bin

    in a directory holding also the files:

        fwr_dnx_PRQ_ww27_001.bin
        IFWI_MERR_PRQ_UOS_TH2_YT2_ww27_001.bin

 7. Follow section **B. Rooting and bootloader unlocking process using OTA package - 1. enable Fastboot mode** by running `enable_fastboot.sh`
 8. Skip copying to the ROM “A195/OTA/YTD802A519600-
f-2014-07-16-16_OSS.zip" to flash.
    Instead go directly to Recovery.

<img src="images/recovery.jpg" alt="Recovery screen" width="200"/>

 9. Choose **sideload from adb**.

    Then from the directory holding the ROM:

        adb sideload YTD802A519600-f-2014-07-16-16_OSS.zip

 10. You now have a booting rooted tablet with space to hold U-Boot environment and you can use `adb` get a remote shell.

        adb shell

    Or push files to a r/w file system

        adb push a-file /cache

### Creating U-Boot and Linux kernel
In the following I assume you have Yocto setup to build an Intel Edison image from [here](https://github.com/edison-fw/meta-intel-edison).

Yocto build U-Boot, U-Boot Env, Linux kernel and a `rootfs.`

 1. Patching U-Boot (u-boot-common_2021.10)

    SRC_URI += "file://no-wdt.cfg"
    SRC_URI += "file://venue_env.cfg"

    To turn of the watchdog as it appears SCU command is not supported, "no-wdt.cfg" with contents:

        # CONFIG_CMD_WDT is not set
        CONFIG_WATCHDOG_AUTOSTART=y
        CONFIG_WDT=y
        # CONFIG_WDT_ASPEED is not set
        # CONFIG_WDT_AST2600 is not set
        # CONFIG_WDT_AT91 is not set
        # CONFIG_WDT_CDNS is not set
        # CONFIG_WDT_CORTINA is not set
        # CONFIG_WDT_ORION is not set
        # CONFIG_WDT_SBSA is not set
        # CONFIG_WDT_SP805 is not set
        # CONFIG_WDT_STM32MP is not set
        # CONFIG_XILINX_TB_WATCHDOG is not set
        # CONFIG_WDT_TANGIER is not set
        
    And to tell U-Boot where to find env on the EMMC, "venue_env.cfg":

        CONFIG_ENV_OFFSET=0x29E04400
        CONFIG_ENV_OFFSET_REDUND=0x29F04400

 2. Patching kernel (linux-yocto_5.16.bb)

        SRC_URI_append = " file://venue7.cfg"

    To turn of the watchdog as it appears SCU command is not supported, "venue7.cfg" with contents:

        # CONFIG_WATCHDOG is not set

 3. Modify **env**:

    In `btrfs.env` to prevent U-Boot from re-partitioning on first boot change:
        -do_flash_os_done=0
        +do_flash_os_done=1
        
    Same for `edison.env`:
        -do_partition_done=0
        +do_partition_done=1
        
    To be able to access env from Linux using `fw_printenv` change `fw_env.config`:

        -# On Edison, the u-boot environments are located on partitions 2 and 4 and both have a size of 64kB
        -/dev/mmcblk0p2		0x0000		0x10000
        -/dev/mmcblk0p4		0x0000		0x10000
        +# On Dell Venue 7, the u-boot environments are located on partitions 6 and 7 and both have a size of 64kB
        +/dev/mmcblk0p6		0x0000		0x10000
        +/dev/mmcblk0p7		0x0000		0x10000

 4. Build image as normal (`make image` followed by `make postbuild`)

    It is probably a good idea (I did) to mount the resulting
    `edison-image-edison.btrfs` in `tmp/deploy/images/edison/`
    and change `/etc/fstab`

        -/dev/disk/by-partlabel/home   /boot       btrfs		nofail,noatime,subvol=/@boot     1   1
        -/dev/disk/by-partlabel/home   /lib/modules    btrfs	nofail,x-systemd.required-by=systemd-modules-load.service,LazyUnmount=true,noatime,subvol=/@modules  1       1
        -/dev/disk/by-partlabel/home   /home       auto    nofail,x-systemd.automount,noatime,subvol=/@home  1       1
        +/dev/disk/by-partlabel/data   /boot       btrfs		nofail,noatime,subvol=/@boot     1   1
        +/dev/disk/by-partlabel/data   /lib/modules    btrfs	nofail,x-systemd.required-by=systemd-modules-load.service,LazyUnmount=true,noatime,subvol=/@modules  1       1
        +/dev/disk/by-partlabel/data   /home       auto    nofail,x-systemd.automount,noatime,subvol=/@home  1       1

### Send the needed kernel, `initrd` and `rootfs`

Use `adb push` to send to `/factory`:
  * edison-btrfs.bin
  * bzImage
  * core-image-minimal-initramfs-edison.cpio.gz (rename this one to initrd)

Then use `adb shell` to go into the tablet and `mv` `bzImage` and `initrd` into a directory `/boot`.

Then `cp` `edison-btrfs.bin` to the both env partitions:

        cp edison-btrfs.bin /dev/disk/.../u-boot-env0
        cp edison-btrfs.bin /dev/disk/.../u-boot-env1

It is good to note that `u-boot-env*` and `factory` will survive restoring Android.

### Overwrite `data` partition
Use `adb push` to send to `/cache`:
  * edison-image-edison.btrfs

Then use `adb shell` to go into the tablet and `umount /data`.

Then `cp` `edison-image-edison.btrfs` to the data partition:

        cp edison-btrfs.bin /dev/disk/.../data

### Overwrite Android following "C. Build the kernel image from the kernel sources and flash kernel image - 2. Flash boot.img and droidboot.img"

But not exactly. We will not be creating an Android boot.img, but instead will write the Edison U-Boot OSIP image using this script `flash_u-boot.sh`.

        #!/bin/bash
        
        fastboot flash boot u-boot-edison-2021.10-r0.img
        fastboot flash fastboot droidboot.img
        
        read -p "Press any key to continue ..."
        
        fastboot reboot

### Boot into U-Boot

Simply rebooting will take you to the Recovery screen. This is good to know as from here you can sideload the Android image again (see 9.) above) and have a restored, rooted Android again.

Instead, push the buttons to go to Fastboot. Unexpectedly (possibly to incorrect OSIP format?) this will take you into U-Boot.

Using the serial console, you can modify env, to load kernel and initrd from partition 3 (factory) and modify kernel command line to find `rootfs` on partition 11. Run `saveenv`.

Then, finally, run `run do_boot` and Linux will start.

### Going back to Android.

It is good to note that `u-boot-env*` and `factory` will survive restoring Android.

Go into Recovery and  choose **sideload from adb**. Then from the directory holding the ROM:

        adb sideload YTD802A519600-f-2014-07-16-16_OSS.zip

## What works in Linux.

Not so much, the ACPI tables don't match the hardware, some drivers are
missing. Bluetooth, WiFi are not working. The SD card is not accessible.
But fortunately USB is working in device mode, so Ethernet over USB (CDC),
is working, sound over USB, etc. as usually on Intel Edison.

To make networking work, see [Gadget](https://edison-fw.github.io/meta-intel-edison/4.2-networking.html#gadget).

### PCI devices
```
00:00.0 Host bridge: Intel Corporation Device 1170 (rev 01)
00:01.0 SD Host controller: Intel Corporation Merrifield SD/SDIO/eMMC Controller (rev 01)
00:01.2 SD Host controller: Intel Corporation Merrifield SD/SDIO/eMMC Controller (rev 01)
00:01.3 SD Host controller: Intel Corporation Merrifield SD/SDIO/eMMC Controller (rev 01)
00:02.0 Display controller: Intel Corporation Device 1181 (rev 01)
00:03.0 Multimedia controller: Intel Corporation Device 1179 (rev 01)
00:04.0 Serial controller: Intel Corporation Merrifield Serial IO HSUART Controller (rev 01)
00:04.1 Serial controller: Intel Corporation Merrifield Serial IO HSUART Controller (rev 01)
00:04.2 Serial controller: Intel Corporation Merrifield Serial IO HSUART Controller (rev 01)
00:04.3 Serial controller: Intel Corporation Merrifield Serial IO HSUART Controller (rev 01)
00:05.0 Serial controller: Intel Corporation Merrifield Serial IO HSUART DMA Controller (rev 01)
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
00:10.0 USB controller: Intel Corporation Device 119d (rev 01)
00:11.0 USB controller: Intel Corporation Merrifield USB Device Controller (OTG) (rev 01)
00:12.0 Signal processing controller: Intel Corporation Device 119f (rev 01)
00:13.0 Co-processor: Intel Corporation Merrifield SCU IPC (rev 01)
00:14.0 Co-processor: Intel Corporation Merrifield Power Management Unit (rev 01)
00:15.0 System peripheral: Intel Corporation Merrifield Serial IO DMA Controller (rev 01)
00:17.0 System peripheral: Intel Corporation Merrifield Serial IO PWM Controller (rev 01)
00:18.0 Display controller: Intel Corporation Device 11a6 (rev 01)
```
### Linux kernel modules (loaded at boot time)
```
Module                  Size  Used by
iptable_nat            16384  0
snd_sof_nocodec        16384  0
spi_pxa2xx_platform    32768  0
dw_dmac                16384  0
usb_f_uac2             36864  2
u_audio                28672  1 usb_f_uac2
usb_f_mass_storage     57344  2
usb_f_eem              20480  2
u_ether                28672  1 usb_f_eem
usb_f_serial           20480  2
u_serial               28672  1 usb_f_serial
libcomposite           65536  17 usb_f_serial,u_audio,usb_f_eem,usb_f_mass_storage,usb_f_uac2
pwm_lpss_pci           16384  1
pwm_lpss               16384  1 pwm_lpss_pci
intel_mrfld_pwrbtn     20480  0
dw_dmac_pci            16384  4
intel_mrfld_adc        16384  0
dw_dmac_core           32768  2 dw_dmac_pci,dw_dmac
snd_sof_pci_intel_tng    20480  0
snd_sof_pci            20480  1 snd_sof_pci_intel_tng
snd_sof_intel_atom     20480  1 snd_sof_pci_intel_tng
snd_sof               131072  4 snd_sof_nocodec,snd_sof_intel_atom,snd_sof_pci_intel_tng,snd_sof_pci
snd_sof_xtensa_dsp     16384  1 snd_sof_pci_intel_tng
snd_soc_acpi           16384  1 snd_sof_intel_atom
spi_pxa2xx_pci         16384  0
ti_ads7950             40960  0
hci_uart               53248  0
industrialio_triggered_buffer    16384  1 ti_ads7950
btbcm                  16384  1 hci_uart
kfifo_buf              16384  1 industrialio_triggered_buffer
leds_gpio              16384  0
tun                    53248  0
ledtrig_timer          16384  0
ledtrig_heartbeat      16384  0
mmc_block              49152  3
extcon_intel_mrfld     16384  0
sdhci_pci              69632  0
cqhci                  32768  1 sdhci_pci
sdhci                  73728  1 sdhci_pci
led_class              20480  2 leds_gpio,sdhci
mmc_core              163840  4 sdhci,cqhci,mmc_block,sdhci_pci
intel_soc_pmic_mrfld    16384  0
btrfs                1511424  1
libcrc32c              16384  1 btrfs
xor                    24576  1 btrfs
zlib_deflate           28672  1 btrfs
raid6_pq              118784  1 btrfs
zstd_compress         299008  1 btrfs
```
### Disk (eMMC) partitions
```
GNU Parted 3.4
Using /dev/mmcblk0
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Model: MMC HAG2e (sd/mmc)
Disk /dev/mmcblk0: 15.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name         Flags
 1      17.4kB  268MB   268MB                reserved     msftdata
 2      268MB   302MB   33.6MB               panic        msftdata
 3      302MB   436MB   134MB   ext4         factory      msftdata
 4      436MB   570MB   134MB                misc         msftdata
 5      570MB   703MB   132MB   ext4         config       msftdata
 6      703MB   704MB   1049kB               u-boot-env0  msftdata
 7      704MB   705MB   1049kB               u-boot-env1  msftdata
 8      705MB   2315MB  1611MB  ext4         cache        msftdata
 9      2315MB  2340MB  25.2MB  ext4         logs         msftdata
10      2340MB  3951MB  1611MB  ext4         system       msftdata
11      3951MB  15.8GB  11.8GB  btrfs        data         msftdata
```
### dmesg
```
Linux version 5.16.0-rc8-edison-acpi-standard (oe-user@oe-host) (x86_64-poky-linux-gcc (GCC) 10.2.0, GNU ld (GNU Binutils) 2.36.1.20210209) #1 SMP PREEMPT Sun Jan 2 22:23:25 UTC 2022
Command line: quiet root=/dev/mmcblk0p11 rootflags=compress=lzo rootfstype=btrfs console=ttyS2,115200n8 earlyprintk=ttyS2,115200n8,keep loglevel=4 systemd.unit=multi-user.target
x86/fpu: x87 FPU will use FXSAVE
signal: max sigframe size: 1440
BIOS-provided physical RAM map:
BIOS-e820: [mem 0x0000000000000000-0x0000000000097fff] usable
BIOS-e820: [mem 0x0000000000100000-0x0000000003ffffff] usable
BIOS-e820: [mem 0x0000000004000000-0x0000000005ffffff] reserved
BIOS-e820: [mem 0x0000000006000000-0x000000003f4fffff] usable
BIOS-e820: [mem 0x000000003f500000-0x000000003fffffff] reserved
BIOS-e820: [mem 0x00000000fec00000-0x00000000fec00fff] reserved
BIOS-e820: [mem 0x00000000fec04000-0x00000000fec07fff] reserved
BIOS-e820: [mem 0x00000000fee00000-0x00000000fee00fff] reserved
BIOS-e820: [mem 0x00000000ff000000-0x00000000ffffffff] reserved
printk: console [earlyser0] enabled
NX (Execute Disable) protection: active
SMBIOS 2.6 present.
DMI: Intel Corporation Merrifield/Silver Ridge, BIOS 509 2014.07.16:17.35.04
tsc: Detected 1066.667 MHz processor
e820: update [mem 0x00000000-0x00000fff] usable ==> reserved
e820: remove [mem 0x000a0000-0x000fffff] usable
last_pfn = 0x3f500 max_arch_pfn = 0x400000000
x86/PAT: Configuration [0-7]: WB  WC  UC- UC  WB  WP  UC- WT  
e820: update [mem 0x04000000-0x05ffffff] usable ==> reserved
RAMDISK: [mem 0x06000000-0x077fffff]
ACPI: Early table checksum verification disabled
ACPI: RSDP 0x00000000000E4500 000024 (v02 U-BOOT)
ACPI: XSDT 0x00000000000E45E0 00004C (v01 U-BOOT U-BOOTBL 20211004 INTL 00000000)
ACPI: FACP 0x00000000000E54D0 0000F4 (v06 U-BOOT U-BOOTBL 20211004 INTL 00000000)
ACPI: DSDT 0x00000000000E4780 000B41 (v02 U-BOOT U-BOOTBL 00010000 INTL 20210105)
ACPI: MCFG 0x00000000000E55D0 00003C (v01 U-BOOT U-BOOTBL 20211004 INTL 00000000)
ACPI: APIC 0x00000000000E5610 000048 (v02 U-BOOT U-BOOTBL 20211004 INTL 00000000)
ACPI: CSRT 0x00000000000E5660 000058 (v00 U-BOOT U-BOOTBL 20211004 INTL 00000000)
ACPI: SPCR 0x00000000000E56C0 000050 (v02 U-BOOT U-BOOTBL 20211004 INTL 00000000)
ACPI: Reserving FACP table memory at [mem 0xe54d0-0xe55c3]
ACPI: Reserving DSDT table memory at [mem 0xe4780-0xe52c0]
ACPI: Reserving MCFG table memory at [mem 0xe55d0-0xe560b]
ACPI: Reserving APIC table memory at [mem 0xe5610-0xe5657]
ACPI: Reserving CSRT table memory at [mem 0xe5660-0xe56b7]
ACPI: Reserving SPCR table memory at [mem 0xe56c0-0xe570f]
No NUMA configuration found
Faking a node at [mem 0x0000000000000000-0x000000003f4fffff]
NODE_DATA(0) allocated [mem 0x3f4fc000-0x3f4fffff]
Zone ranges:
  DMA      [mem 0x0000000000001000-0x0000000000ffffff]
  DMA32    [mem 0x0000000001000000-0x000000003f4fffff]
  Normal   empty
Movable zone start for each node
Early memory node ranges
  node   0: [mem 0x0000000000001000-0x0000000000097fff]
  node   0: [mem 0x0000000000100000-0x0000000003ffffff]
  node   0: [mem 0x0000000006000000-0x000000003f4fffff]
Initmem setup node 0 [mem 0x0000000000001000-0x000000003f4fffff]
On node 0, zone DMA: 1 pages in unavailable ranges
On node 0, zone DMA: 104 pages in unavailable ranges
On node 0, zone DMA32: 8192 pages in unavailable ranges
On node 0, zone DMA32: 2816 pages in unavailable ranges
IOAPIC[0]: apic_id 2, version 32, address 0xfec00000, GSI 0-54
ACPI: Using ACPI (MADT) for SMP configuration information
ACPI: SPCR: console: uart,mmio,0xff010180
TSC deadline timer available
smpboot: Allowing 2 CPUs, 0 hotplug CPUs
PM: hibernation: Registered nosave memory: [mem 0x00000000-0x00000fff]
PM: hibernation: Registered nosave memory: [mem 0x00098000-0x000fffff]
PM: hibernation: Registered nosave memory: [mem 0x04000000-0x05ffffff]
[mem 0x40000000-0xfebfffff] available for PCI devices
clocksource: refined-jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1910969940391419 ns
setup_percpu: NR_CPUS:64 nr_cpumask_bits:64 nr_cpu_ids:2 nr_node_ids:1
percpu: Embedded 53 pages/cpu s178904 r8192 d29992 u1048576
pcpu-alloc: s178904 r8192 d29992 u1048576 alloc=1*2097152
pcpu-alloc: [0] 0 1 
Fallback order for Node 0: 0 
Built 1 zonelists, mobility grouping on.  Total pages: 246828
Policy zone: DMA32
Kernel command line: quiet root=/dev/mmcblk0p11 rootflags=compress=lzo rootfstype=btrfs console=ttyS2,115200n8 earlyprintk=ttyS2,115200n8,keep loglevel=4 systemd.unit=multi-user.target
Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
Inode-cache hash table entries: 65536 (order: 7, 524288 bytes, linear)
mem auto-init: stack:off, heap alloc:off, heap free:off
Memory: 923344K/1004124K available (16398K kernel code, 2933K rwdata, 4156K rodata, 1304K init, 2888K bss, 80520K reserved, 0K cma-reserved)
SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
Kernel/User page tables isolation: enabled
Dynamic Preempt: voluntary
rcu: Preemptible hierarchical RCU implementation.
rcu: 	RCU event tracing is enabled.
rcu: 	RCU restricting CPUs from NR_CPUS=64 to nr_cpu_ids=2.
	Trampoline variant of Tasks RCU enabled.
	Tracing variant of Tasks RCU enabled.
rcu: RCU calculated value of scheduler-enlistment delay is 100 jiffies.
rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
NR_IRQS: 4352, nr_irqs: 512, preallocated irqs: 0
random: get_random_bytes called from start_kernel+0x49b/0x65f with crng_init=0
Console: colour dummy device 80x25
printk: console [ttyS2] enabled
ACPI: Core revision 20210930
APIC: Switch to symmetric I/O mode setup
clocksource: tsc-early: mask: 0xffffffffffffffff max_cycles: 0xf6018f8508, max_idle_ns: 440795250352 ns
Calibrating delay loop (skipped), value calculated using timer frequency.. 2133.33 BogoMIPS (lpj=1066667)
pid_max: default: 32768 minimum: 301
LSM: Security Framework initializing
SELinux:  Initializing.
Mount-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
Mountpoint-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
CPU0: Thermal monitoring enabled (TM1)
process: using mwait in idle threads
Last level iTLB entries: 4KB 48, 2MB 0, 4MB 0
Last level dTLB entries: 4KB 128, 2MB 16, 4MB 16, 1GB 0
Spectre V1 : Mitigation: usercopy/swapgs barriers and __user pointer sanitization
Spectre V2 : Mitigation: Full generic retpoline
Spectre V2 : Spectre v2 / SpectreRSB mitigation: Filling RSB on context switch
MDS: Vulnerable: Clear CPU buffers attempted, no microcode
Freeing SMP alternatives memory: 44K
smpboot: Estimated ratio of average max frequency by base frequency (times 1024): 1536
smpboot: CPU0: Genuine Intel(R) CPU   4000  @ 1.06GHz (family: 0x6, model: 0x4a, stepping: 0x8)
Performance Events: PEBS fmt2+, 8-deep LBR, Silvermont events, 8-deep LBR, full-width counters, Intel PMU driver.
... version:                3
... bit width:              40
... generic registers:      2
... value mask:             000000ffffffffff
... max period:             0000007fffffffff
... fixed-purpose events:   3
... event mask:             0000000700000003
rcu: Hierarchical SRCU implementation.
smp: Bringing up secondary CPUs ...
x86: Booting SMP configuration:
.... node  #0, CPUs:      #1
smp: Brought up 1 node, 2 CPUs
smpboot: Max logical packages: 1
smpboot: Total of 2 processors activated (4266.66 BogoMIPS)
devtmpfs: initialized
DMA-API: preallocated 65536 debug entries
DMA-API: debugging enabled by kernel config
clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1911260446275000 ns
futex hash table entries: 512 (order: 3, 32768 bytes, linear)
pinctrl core: initialized pinctrl subsystem
regulator-dummy: no parameters, enabled
PM: RTC time: 21:30:52, date: 2022-01-11
NET: Registered PF_NETLINK/PF_ROUTE protocol family
audit: initializing netlink subsys (disabled)
audit: type=2000 audit(1641936652.018:1): state=initialized audit_enabled=0 res=1
thermal_sys: Registered thermal governor 'step_wise'
thermal_sys: Registered thermal governor 'user_space'
cpuidle: using governor menu
ACPI FADT declares the system doesn't support PCIe ASPM, so disable it
PCI: MMCONFIG for domain 0000 [bus 00-00] at [mem 0x3f500000-0x3f5fffff] (base 0x3f500000)
PCI: MMCONFIG at [mem 0x3f500000-0x3f5fffff] reserved in E820
Intel MID platform detected, using MID PCI ops
PCI: Using configuration type 1 for base access
ENERGY_PERF_BIAS: Set to 'normal', was 'performance'
kprobes: kprobe jump-optimization is enabled. All kprobes are optimized if possible.
HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
ACPI: Added _OSI(Module Device)
ACPI: Added _OSI(Processor Device)
ACPI: Added _OSI(3.0 _SCP Extensions)
ACPI: Added _OSI(Processor Aggregator Device)
ACPI: Added _OSI(Linux-Dell-Video)
ACPI: Added _OSI(Linux-Lenovo-NV-HDMI-Audio)
ACPI: Added _OSI(Linux-HPI-Hybrid-Graphics)
ACPI: 1 ACPI AML tables successfully acquired and loaded
ACPI: Interpreter enabled
ACPI: PM: (supports S0)
ACPI: Using IOAPIC for interrupt routing
PCI: Using host bridge windows from ACPI; if necessary, use "pci=nocrs" and report a bug
ACPI: PCI Root Bridge [PCI0] (domain 0000 [bus 00-ff])
acpi PNP0A08:00: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI HPX-Type3]
acpi PNP0A08:00: _OSC: OS now controls [PME PCIeCapability LTR]
acpi PNP0A08:00: FADT indicates ASPM is unsupported, using BIOS configuration
acpi PNP0A08:00: [Firmware Info]: MMCONFIG for domain 0000 [bus 00-00] only partially covers this bridge
PCI host bridge to bus 0000:00
pci_bus 0000:00: root bus resource [io  0x0000-0x0cf7 window]
pci_bus 0000:00: root bus resource [io  0x0d00-0xffff window]
pci_bus 0000:00: root bus resource [mem 0x000ddcc0-0x000ddccf window]
pci_bus 0000:00: root bus resource [mem 0x04819000-0x04898fff window]
pci_bus 0000:00: root bus resource [mem 0x04919000-0x04920fff window]
pci_bus 0000:00: root bus resource [mem 0x05e00000-0x05ffffff window]
pci_bus 0000:00: root bus resource [mem 0x80000000-0xffffffff window]
pci_bus 0000:00: root bus resource [bus 00-ff]
pci 0000:00:00.0: [8086:1170] type 00 class 0x060000
pci 0000:00:01.0: [8086:1190] type 00 class 0x080501
pci 0000:00:01.0: reg 0x10: [mem 0xff3fc000-0xff3fc0ff]
pci 0000:00:01.0: PME# supported from D0 D3hot
pci 0000:00:01.2: [8086:1190] type 00 class 0x080501
pci 0000:00:01.2: reg 0x10: [mem 0xff3fa000-0xff3fa0ff]
pci 0000:00:01.2: PME# supported from D0 D3hot
pci 0000:00:01.3: [8086:1190] type 00 class 0x080501
pci 0000:00:01.3: reg 0x10: [mem 0xff3fb000-0xff3fb0ff]
pci 0000:00:01.3: PME# supported from D0 D3hot
pci 0000:00:02.0: [8086:1181] type 00 class 0x038000
pci 0000:00:02.0: reg 0x10: [mem 0xc0000000-0xc1ffffff]
pci 0000:00:02.0: reg 0x18: [mem 0x80000000-0x8fffffff]
pci 0000:00:02.0: reg 0x20: [io  0x7ff8-0x7fff]
pci 0000:00:03.0: [8086:1179] type 00 class 0x048000
pci 0000:00:03.0: reg 0x10: [mem 0xc2000000-0xc23fffff]
pci 0000:00:04.0: [8086:1191] type 00 class 0x070002
pci 0000:00:04.0: reg 0x10: [mem 0xff010000-0xff01007f]
pci 0000:00:04.0: PME# supported from D0 D3hot
pci 0000:00:04.1: [8086:1191] type 00 class 0x070002
pci 0000:00:04.1: reg 0x10: [mem 0xff010080-0xff0100ff]
pci 0000:00:04.1: PME# supported from D0 D3hot
pci 0000:00:04.2: [8086:1191] type 00 class 0x070002
pci 0000:00:04.2: reg 0x10: [mem 0xff010100-0xff01017f]
pci 0000:00:04.2: PME# supported from D0 D3hot
pci 0000:00:04.3: [8086:1191] type 00 class 0x070002
pci 0000:00:04.3: reg 0x10: [mem 0xff010180-0xff0101ff]
pci 0000:00:04.3: PME# supported from D0 D3hot
pci 0000:00:05.0: [8086:1192] type 00 class 0x070002
pci 0000:00:05.0: reg 0x10: [mem 0xff010400-0xff0107ff]
pci 0000:00:05.0: PME# supported from D0 D3hot
pci 0000:00:07.0: [8086:1194] type 00 class 0x088000
pci 0000:00:07.0: reg 0x10: [mem 0xff188000-0xff188fff]
pci 0000:00:07.0: PME# supported from D0 D3hot
pci 0000:00:07.1: [8086:1194] type 00 class 0x088000
pci 0000:00:07.1: reg 0x10: [mem 0xff189000-0xff189fff]
pci 0000:00:07.1: PME# supported from D0 D3hot
pci 0000:00:07.2: [8086:1194] type 00 class 0x088000
pci 0000:00:07.2: reg 0x10: [mem 0xff18a000-0xff18afff]
pci 0000:00:07.2: PME# supported from D0 D3hot
pci 0000:00:08.0: [8086:1195] type 00 class 0x078000
pci 0000:00:08.0: reg 0x10: [mem 0xff18b000-0xff18bfff]
pci 0000:00:08.0: PME# supported from D0 D3hot
pci 0000:00:08.1: [8086:1195] type 00 class 0x078000
pci 0000:00:08.1: reg 0x10: [mem 0xff18c000-0xff18cfff]
pci 0000:00:08.1: PME# supported from D0 D3hot
pci 0000:00:08.2: [8086:1195] type 00 class 0x078000
pci 0000:00:08.2: reg 0x10: [mem 0xff18d000-0xff18dfff]
pci 0000:00:08.2: PME# supported from D0 D3hot
pci 0000:00:08.3: [8086:1195] type 00 class 0x078000
pci 0000:00:08.3: reg 0x10: [mem 0xff18e000-0xff18efff]
pci 0000:00:08.3: PME# supported from D0 D3hot
pci 0000:00:09.0: [8086:1196] type 00 class 0x078000
pci 0000:00:09.0: reg 0x10: [mem 0xff18f000-0xff18ffff]
pci 0000:00:09.0: PME# supported from D0 D3hot
pci 0000:00:09.1: [8086:1196] type 00 class 0x078000
pci 0000:00:09.1: reg 0x10: [mem 0xff190000-0xff190fff]
pci 0000:00:09.1: PME# supported from D0 D3hot
pci 0000:00:09.2: [8086:1196] type 00 class 0x078000
pci 0000:00:09.2: reg 0x10: [mem 0xff191000-0xff191fff]
pci 0000:00:09.2: PME# supported from D0 D3hot
pci 0000:00:0a.0: [8086:1197] type 00 class 0x078000
pci 0000:00:0a.0: reg 0x10: [mem 0xff3f8000-0xff3f8fff]
pci 0000:00:0a.0: PME# supported from D0 D3hot
pci 0000:00:0b.0: [8086:1198] type 00 class 0x108000
pci 0000:00:0b.0: reg 0x10: [mem 0xf9038000-0xf903ffff]
pci 0000:00:0b.0: PME# supported from D0 D3hot
pci 0000:00:0c.0: [8086:1199] type 00 class 0x088000
pci 0000:00:0c.0: reg 0x10: [mem 0xff008000-0xff008fff]
pci 0000:00:0c.0: reg 0x14: [mem 0x000dfa30-0x000dfa3f]
pci 0000:00:0c.0: PME# supported from D0 D3hot
pci 0000:00:0d.0: [8086:119a] type 00 class 0x040100
pci 0000:00:0d.0: reg 0x10: [mem 0x05e00000-0x05ffffff]
pci 0000:00:0d.0: reg 0x14: [mem 0xff340000-0xff343fff]
pci 0000:00:0d.0: reg 0x18: [mem 0xff344000-0xff344fff]
pci 0000:00:0d.0: reg 0x1c: [mem 0xff2c0000-0xff2dffff]
pci 0000:00:0d.0: reg 0x20: [mem 0xff300000-0xff33ffff]
pci 0000:00:0d.0: PME# supported from D0 D3hot
pci 0000:00:0e.0: [8086:119b] type 00 class 0x088000
pci 0000:00:0e.0: reg 0x10: [mem 0xff298000-0xff29bfff]
pci 0000:00:0e.0: reg 0x14: [mem 0xff2a2000-0xff2a2fff]
pci 0000:00:0e.0: PME# supported from D0 D3hot
pci 0000:00:10.0: [8086:119d] type 00 class 0x0c0320
pci 0000:00:10.0: reg 0x10: [mem 0xf9060000-0xf907ffff]
pci 0000:00:10.0: PME# supported from D0 D3hot
pci 0000:00:11.0: [8086:119e] type 00 class 0x0c0320
pci 0000:00:11.0: reg 0x10: [mem 0xf9100000-0xf911ffff]
pci 0000:00:11.0: PME# supported from D0 D3hot
pci 0000:00:12.0: [8086:119f] type 00 class 0x118000
pci 0000:00:12.0: reg 0x10: [mem 0xf9009000-0xf9009fff]
pci 0000:00:12.0: reg 0x14: [mem 0xf90a0000-0xf90affff]
pci 0000:00:12.0: reg 0x18: [mem 0xfa000000-0xfaffffff]
pci 0000:00:12.0: PME# supported from D0 D3hot
pci 0000:00:13.0: [8086:11a0] type 00 class 0x0b4000
pci 0000:00:13.0: reg 0x10: [mem 0xff009000-0xff009fff]
pci 0000:00:13.0: PME# supported from D0 D3hot
pci 0000:00:14.0: [8086:11a1] type 00 class 0x0b4000
pci 0000:00:14.0: reg 0x10: [mem 0xff00b000-0xff00bfff]
pci 0000:00:14.0: PME# supported from D0 D3hot
pci 0000:00:15.0: [8086:11a2] type 00 class 0x088000
pci 0000:00:15.0: reg 0x10: [mem 0xff192000-0xff192fff]
pci 0000:00:15.0: PME# supported from D0 D3hot
pci 0000:00:17.0: [8086:11a5] type 00 class 0x088000
pci 0000:00:17.0: reg 0x10: [mem 0xff013400-0xff0137ff]
pci 0000:00:17.0: PME# supported from D0 D3hot
pci 0000:00:18.0: [8086:11a6] type 00 class 0x038000
pci 0000:00:18.0: PME# supported from D0 D3hot
iommu: Default domain type: Translated 
iommu: DMA domain TLB invalidation policy: lazy mode 
vgaarb: loaded
SCSI subsystem initialized
libata version 3.00 loaded.
ACPI: bus type USB registered
usbcore: registered new interface driver usbfs
usbcore: registered new interface driver hub
usbcore: registered new device driver usb
pps_core: LinuxPPS API ver. 1 registered
pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
PTP clock support registered
Advanced Linux Sound Architecture Driver Initialized.
Bluetooth: Core ver 2.22
NET: Registered PF_BLUETOOTH protocol family
Bluetooth: HCI device and connection manager initialized
Bluetooth: HCI socket layer initialized
Bluetooth: L2CAP socket layer initialized
Bluetooth: SCO socket layer initialized
NetLabel: Initializing
NetLabel:  domain hash size = 128
NetLabel:  protocols = UNLABELED CIPSOv4 CALIPSO
NetLabel:  unlabeled traffic allowed by default
PCI: Probing PCI hardware
PCI: root bus 00: using default resources
PCI: Probing PCI hardware (bus 00)
PCI: pci_cache_line_size set to 64 bytes
pci 0000:00:0c.0: can't claim BAR 1 [mem 0x000dfa30-0x000dfa37]: no compatible bridge window
pci 0000:00:0c.0: BAR 1 [mem size 0x00000008] is immovable
e820: reserve RAM buffer [mem 0x00098000-0x0009ffff]
e820: reserve RAM buffer [mem 0x3f500000-0x3fffffff]
clocksource: Switched to clocksource tsc-early
VFS: Disk quotas dquot_6.6.0
VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
pnp: PnP ACPI init
system 00:01: [mem 0x3f500000-0x3f5fffff] has been reserved
pnp: PnP ACPI: found 2 devices
NET: Registered PF_INET protocol family
IP idents hash table entries: 16384 (order: 5, 131072 bytes, linear)
tcp_listen_portaddr_hash hash table entries: 512 (order: 1, 8192 bytes, linear)
TCP established hash table entries: 8192 (order: 4, 65536 bytes, linear)
TCP bind hash table entries: 8192 (order: 5, 131072 bytes, linear)
TCP: Hash tables configured (established 8192 bind 8192)
UDP hash table entries: 512 (order: 2, 16384 bytes, linear)
UDP-Lite hash table entries: 512 (order: 2, 16384 bytes, linear)
NET: Registered PF_UNIX/PF_LOCAL protocol family
RPC: Registered named UNIX socket transport module.
RPC: Registered udp transport module.
RPC: Registered tcp transport module.
RPC: Registered tcp NFSv4.1 backchannel transport module.
pci_bus 0000:00: resource 4 [io  0x0000-0x0cf7 window]
pci_bus 0000:00: resource 5 [io  0x0d00-0xffff window]
pci_bus 0000:00: resource 6 [mem 0x000ddcc0-0x000ddccf window]
pci_bus 0000:00: resource 7 [mem 0x04819000-0x04898fff window]
pci_bus 0000:00: resource 8 [mem 0x04919000-0x04920fff window]
pci_bus 0000:00: resource 9 [mem 0x05e00000-0x05ffffff window]
pci_bus 0000:00: resource 10 [mem 0x80000000-0xffffffff window]
PCI: CLS 0 bytes, default 64
clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0xf6018f8508, max_idle_ns: 440795250352 ns
clocksource: Switched to clocksource tsc
clocksource: Nonstop clocksource tsc should not supply suspend/resume interfaces
Unpacking initramfs...
Initialise system trusted keyrings
workingset: timestamp_bits=56 max_order=18 bucket_order=0
NFS: Registering the id_resolver key type
Key type id_resolver registered
Key type id_legacy registered
Key type asymmetric registered
Asymmetric key parser 'x509' registered
Block layer SCSI generic (bsg) driver version 0.4 loaded (major 249)
io scheduler mq-deadline registered
io scheduler kyber registered
gpio-merrifield 0000:00:0c.0: can't enable device: BAR 1 [mem size 0x00000008] not assigned
gpio-merrifield: probe of 0000:00:0c.0 failed with error -22
hsu_dma_pci 0000:00:05.0: Found HSU DMA, 12 channels
Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
8250_mid: probe of 0000:00:04.0 failed with error -16
8250_mid 0000:00:04.1: GPIO lookup for consumer rs485-term
8250_mid 0000:00:04.1: using ACPI for GPIO lookup
acpi device:0d: GPIO: looking up rs485-term-gpios
acpi device:0d: GPIO: looking up rs485-term-gpio
8250_mid 0000:00:04.1: using lookup tables for GPIO lookup
8250_mid 0000:00:04.1: No GPIO consumer rs485-term found
0000:00:04.1: ttyS0 at MMIO 0xff010080 (irq = 10, base_baud = 1843200) is a TI16750
serial serial0: tty port ttyS0 registered
8250_mid 0000:00:04.2: GPIO lookup for consumer rs485-term
8250_mid 0000:00:04.2: using lookup tables for GPIO lookup
8250_mid 0000:00:04.2: No GPIO consumer rs485-term found
0000:00:04.2: ttyS1 at MMIO 0xff010100 (irq = 11, base_baud = 1843200) is a TI16750
printk: console [ttyS2] disabled
8250_mid 0000:00:04.3: GPIO lookup for consumer rs485-term
8250_mid 0000:00:04.3: using lookup tables for GPIO lookup
8250_mid 0000:00:04.3: No GPIO consumer rs485-term found
0000:00:04.3: ttyS2 at MMIO 0xff010180 (irq = 12, base_baud = 1843200) is a TI16750
printk: console [ttyS2] enabled
Non-volatile memory driver v1.3
Linux agpgart interface v0.103
ACPI: bus type drm_connector registered
loop: module loaded
mdio_bus fixed-0: GPIO lookup for consumer reset
mdio_bus fixed-0: using lookup tables for GPIO lookup
mdio_bus fixed-0: No GPIO consumer reset found
libphy: Fixed MDIO Bus: probed
e100: Intel(R) PRO/100 Network Driver
e100: Copyright(c) 1999-2006 Intel Corporation
e1000: Intel(R) PRO/1000 Network Driver
e1000: Copyright (c) 1999-2006 Intel Corporation.
e1000e: Intel(R) PRO/1000 Network Driver
e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
igb: Intel(R) Gigabit Ethernet Network Driver
igb: Copyright (c) 2007-2014 Intel Corporation.
ixgbe: Intel(R) 10 Gigabit PCI Express Network Driver
ixgbe: Copyright (c) 1999-2016 Intel Corporation.
sky2: driver version 1.30
usbcore: registered new interface driver r8152
usbcore: registered new interface driver asix
usbcore: registered new interface driver ax88179_178a
usbcore: registered new interface driver cdc_ether
usbcore: registered new interface driver net1080
usbcore: registered new interface driver cdc_subset
usbcore: registered new interface driver zaurus
usbcore: registered new interface driver MOSCHIP usb-ethernet driver
usbcore: registered new interface driver cdc_ncm
Initramfs unpacking failed: invalid magic at start of compressed archive
Freeing initrd memory: 24576K
modprobe (41) used greatest stack depth: 14704 bytes left
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer reset
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up reset-gpios
acpi device:08: GPIO: looking up reset-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer reset found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer cs
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up cs-gpios
acpi device:08: GPIO: looking up cs-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer cs found
ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
ehci-pci: EHCI PCI platform driver
ehci-pci 0000:00:10.0: EHCI Host Controller
ehci-pci 0000:00:10.0: new USB bus registered, assigned bus number 1
ehci-pci 0000:00:10.0: irq 14, io mem 0xf9060000
ehci-pci 0000:00:10.0: USB a.d started, EHCI 1.10
usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 5.16
usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
usb usb1: Product: EHCI Host Controller
usb usb1: Manufacturer: Linux 5.16.0-rc8-edison-acpi-standard ehci_hcd
usb usb1: SerialNumber: 0000:00:10.0
hub 1-0:1.0: USB hub found
hub 1-0:1.0: 2 ports detected
ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
ohci-pci: OHCI PCI platform driver
uhci_hcd: USB Universal Host Controller Interface driver
usbcore: registered new interface driver usblp
usbcore: registered new interface driver usb-storage
usbcore: registered new interface driver pl2303
usbserial: USB Serial support registered for pl2303
rtc_cmos 00:00: registered as rtc0
rtc_cmos 00:00: GPIO lookup for consumer wp
rtc_cmos 00:00: using ACPI for GPIO lookup
acpi PNP0B00:00: GPIO: looking up wp-gpios
acpi PNP0B00:00: GPIO: looking up wp-gpio
rtc_cmos 00:00: using lookup tables for GPIO lookup
rtc_cmos 00:00: No GPIO consumer wp found
rtc_cmos 00:00: no alarms, 242 bytes nvram
i2c-designware-pci 0000:00:08.0: GPIO lookup for consumer scl
i2c-designware-pci 0000:00:08.0: using ACPI for GPIO lookup
acpi device:05: GPIO: looking up scl-gpios
acpi device:05: GPIO: looking up scl-gpio
i2c-designware-pci 0000:00:08.0: using lookup tables for GPIO lookup
i2c-designware-pci 0000:00:08.0: No GPIO consumer scl found
i2c-designware-pci 0000:00:08.1: GPIO lookup for consumer scl
i2c-designware-pci 0000:00:08.1: using lookup tables for GPIO lookup
i2c-designware-pci 0000:00:08.1: No GPIO consumer scl found
i2c-designware-pci 0000:00:08.2: GPIO lookup for consumer scl
i2c-designware-pci 0000:00:08.2: using lookup tables for GPIO lookup
i2c-designware-pci 0000:00:08.2: No GPIO consumer scl found
i2c-designware-pci 0000:00:08.3: GPIO lookup for consumer scl
i2c-designware-pci 0000:00:08.3: using lookup tables for GPIO lookup
i2c-designware-pci 0000:00:08.3: No GPIO consumer scl found
i2c-designware-pci 0000:00:09.0: GPIO lookup for consumer scl
i2c-designware-pci 0000:00:09.0: using lookup tables for GPIO lookup
i2c-designware-pci 0000:00:09.0: No GPIO consumer scl found
i2c-designware-pci 0000:00:09.1: GPIO lookup for consumer scl
i2c-designware-pci 0000:00:09.1: using ACPI for GPIO lookup
acpi device:06: GPIO: looking up scl-gpios
acpi device:06: GPIO: looking up scl-gpio
i2c-designware-pci 0000:00:09.1: using lookup tables for GPIO lookup
i2c-designware-pci 0000:00:09.1: No GPIO consumer scl found
i2c-designware-pci 0000:00:09.2: GPIO lookup for consumer scl
i2c-designware-pci 0000:00:09.2: using lookup tables for GPIO lookup
i2c-designware-pci 0000:00:09.2: No GPIO consumer scl found
device-mapper: ioctl: 4.45.0-ioctl (2021-03-22) initialised: dm-devel@redhat.com
intel_pstate: CPU model not supported
ledtrig-cpu: registered to indicate activity on CPUs
hid: raw HID events driver (C) Jiri Kosina
usbcore: registered new interface driver usbhid
usbhid: USB HID core driver
Initializing XFRM netlink socket
NET: Registered PF_INET6 protocol family
Segment Routing with IPv6
In-situ OAM (IOAM) with IPv6
sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
NET: Registered PF_PACKET protocol family
Key type dns_resolver registered
microcode: sig=0x406a8, pf=0x1, revision=0x81b
microcode: Microcode Update Driver: v2.2.
IPI shorthand broadcast: enabled
sched_clock: Marking stable (2843900874, 934312)->(2859130215, -14295029)
registered taskstats version 1
Loading compiled-in X.509 certificates
modprobe (70) used greatest stack depth: 14624 bytes left
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer reset
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up reset-gpios
acpi device:08: GPIO: looking up reset-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer reset found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer cs
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up cs-gpios
acpi device:08: GPIO: looking up cs-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer cs found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer reset
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up reset-gpios
acpi device:08: GPIO: looking up reset-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer reset found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer cs
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up cs-gpios
acpi device:08: GPIO: looking up cs-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer cs found
PM:   Magic number: 6:501:550
clockevents clockevent1: hash matches
printk: console [netcon0] enabled
netconsole: network logging started
cfg80211: Loading compiled-in X.509 certificates for regulatory database
cfg80211: Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
platform regulatory.0: Direct firmware load for regulatory.db failed with error -2
cfg80211: failed to load regulatory.db
ALSA device list:
  No soundcards found.
8250_mid 0000:00:04.3: forbid DMA for kernel console
Freeing unused kernel image (initmem) memory: 1304K
Write protecting the kernel read-only data: 24576k
Freeing unused kernel image (text/rodata gap) memory: 2032K
Freeing unused kernel image (rodata/data gap) memory: 1988K
Run /init as init process
  with arguments:
    /init
  with environment:
    HOME=/
    TERM=linux
raid6: sse2x4   gen()  1284 MB/s
raid6: sse2x4   xor()   590 MB/s
raid6: sse2x2   gen()  1250 MB/s
raid6: sse2x2   xor()   707 MB/s
raid6: sse2x1   gen()   933 MB/s
raid6: sse2x1   xor()   507 MB/s
raid6: using algorithm sse2x4 gen() 1284 MB/s
raid6: .... xor() 590 MB/s, rmw enabled
raid6: using ssse3x2 recovery algorithm
xor: measuring software checksum speed
   prefetch64-sse  :  2052 MB/sec
   generic_sse     :  1818 MB/sec
xor: using function: prefetch64-sse (2052 MB/sec)
Btrfs loaded, crc32c=crc32c-generic, zoned=no, fsverity=no
modprobe (77) used greatest stack depth: 14336 bytes left
sdhci: Secure Digital Host Controller Interface driver
sdhci: Copyright(c) Pierre Ossman
sdhci-pci 0000:00:01.0: SDHCI controller found [8086:1190] (rev 1)
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer reset
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up reset-gpios
acpi device:08: GPIO: looking up reset-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer reset found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer cs
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up cs-gpios
acpi device:08: GPIO: looking up cs-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer cs found
mmc0: SDHCI controller on PCI [0000:00:01.0] using ADMA
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
sdhci-pci 0000:00:01.3: SDHCI controller found [8086:1190] (rev 1)
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer reset
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up reset-gpios
acpi device:08: GPIO: looking up reset-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer reset found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer cs
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up cs-gpios
acpi device:08: GPIO: looking up cs-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer cs found
mmc1: SDHCI controller on PCI [0000:00:01.3] using ADMA
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer reset
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up reset-gpios
acpi device:08: GPIO: looking up reset-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer reset found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer cs
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up cs-gpios
acpi device:08: GPIO: looking up cs-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer cs found
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
mmc0: new DDR MMC card at address 0001
ACPI: Host-directed Dynamic ACPI Table Load:
ACPI: SSDT 0xFFFF9288488B8000 00106D (v05        LEDS-DS2 00000001 INTL 20210105)
acpi-tables-loa (124) used greatest stack depth: 14272 bytes left
acpi INT3491:00: GPIO: looking up 0 in _CRS
acpi INT3491:01: GPIO: looking up 0 in _CRS
pca953x i2c-INT3491:01: GPIO lookup for consumer reset
pca953x i2c-INT3491:01: using ACPI for GPIO lookup
acpi INT3491:01: GPIO: looking up reset-gpios
acpi INT3491:01: GPIO: looking up reset-gpio
pca953x i2c-INT3491:01: using lookup tables for GPIO lookup
pca953x i2c-INT3491:01: No GPIO consumer reset found
pca953x i2c-INT3491:01: supply vcc not found, using dummy regulator
pca953x i2c-INT3491:01: using no AI
pca953x i2c-INT3491:01: failed writing register
pca953x: probe of i2c-INT3491:01 failed with error -121
acpi INT3491:02: GPIO: looking up 0 in _CRS
pca953x i2c-INT3491:02: GPIO lookup for consumer reset
pca953x i2c-INT3491:02: using ACPI for GPIO lookup
acpi INT3491:02: GPIO: looking up reset-gpios
acpi INT3491:02: GPIO: looking up reset-gpio
pca953x i2c-INT3491:02: using lookup tables for GPIO lookup
pca953x i2c-INT3491:02: No GPIO consumer reset found
pca953x i2c-INT3491:02: supply vcc not found, using dummy regulator
pca953x i2c-INT3491:02: using no AI
pca953x i2c-INT3491:02: failed writing register
pca953x: probe of i2c-INT3491:02 failed with error -121
acpi INT3491:03: GPIO: looking up 0 in _CRS
pca953x i2c-INT3491:03: GPIO lookup for consumer reset
pca953x i2c-INT3491:03: using ACPI for GPIO lookup
acpi INT3491:03: GPIO: looking up reset-gpios
acpi INT3491:03: GPIO: looking up reset-gpio
pca953x i2c-INT3491:03: using lookup tables for GPIO lookup
pca953x i2c-INT3491:03: No GPIO consumer reset found
pca953x i2c-INT3491:03: supply vcc not found, using dummy regulator
pca953x i2c-INT3491:03: using no AI
pca953x i2c-INT3491:03: failed writing register
pca953x: probe of i2c-INT3491:03 failed with error -121
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer reset
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up reset-gpios
acpi device:08: GPIO: looking up reset-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer reset found
tusb1210 dwc3.0.auto.ulpi: GPIO lookup for consumer cs
tusb1210 dwc3.0.auto.ulpi: using ACPI for GPIO lookup
acpi device:08: GPIO: looking up cs-gpios
acpi device:08: GPIO: looking up cs-gpio
tusb1210 dwc3.0.auto.ulpi: using lookup tables for GPIO lookup
tusb1210 dwc3.0.auto.ulpi: No GPIO consumer cs found
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
acpi INT3491:00: GPIO: looking up 0 in _CRS
mmcblk0: mmc0:0001 HAG2e\x04 14.7 GiB 
 mmcblk0: p1 p2 p3 p4 p5 p6 p7 p8 p9 p10 p11
mmcblk0boot0: mmc0:0001 HAG2e\x04 4.00 MiB 
mmcblk0boot1: mmc0:0001 HAG2e\x04 4.00 MiB 
mmcblk0rpmb: mmc0:0001 HAG2e\x04 4.00 MiB, chardev (246:0)
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
acpi INT3491:00: GPIO: looking up 0 in _CRS
random: fast init done
BTRFS: device fsid 6378889a-a216-490f-9811-252716c4b862 devid 1 transid 151 /dev/mmcblk0p11 scanned by udevd (92)
mount.sh (252) used greatest stack depth: 14232 bytes left
EXT4-fs (mmcblk0p9): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
mount (287) used greatest stack depth: 13808 bytes left
BTRFS info (device mmcblk0p11): disk space caching is enabled
BTRFS info (device mmcblk0p11): has skinny extents
EXT4-fs (mmcblk0p5): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
BTRFS info (device mmcblk0p11): enabling ssd optimizations
mount (323) used greatest stack depth: 13632 bytes left
EXT4-fs (mmcblk0p8): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
EXT4-fs (mmcblk0p3): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
EXT4-fs (mmcblk0p10): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
udevd (95) used greatest stack depth: 13568 bytes left
udevd (98) used greatest stack depth: 13488 bytes left
umount (432) used greatest stack depth: 13320 bytes left
BTRFS info (device mmcblk0p11): use lzo compression, level 0
BTRFS info (device mmcblk0p11): disk space caching is enabled
BTRFS info (device mmcblk0p11): has skinny extents
BTRFS info (device mmcblk0p11): enabling ssd optimizations
systemd[1]: systemd 247.6+ running in system mode. (+PAM -AUDIT -SELINUX +IMA -APPARMOR -SMACK +SYSVINIT +UTMP -LIBCRYPTSETUP -GCRYPT -GNUTLS -ACL +XZ -LZ4 -ZSTD -SECCOMP +BLKID -ELFUTILS +KMOD -IDN2 -IDN -PCRE2 default-hierarchy=hybrid)
systemd[1]: Detected architecture x86-64.
systemd[1]: Set hostname to <venue>.
systemd-debug-g (468) used greatest stack depth: 13144 bytes left
systemd-sysv-generator[476]: SysV service '/etc/init.d/xl2tpd' lacks a native systemd unit file. Automatically generating a unit file for compatibility. Please update package to include a native systemd unit file, in order to make it more safe and robust.
systemd[1]: Queued start job for default target Multi-User System.
systemd[1]: Created slice system-getty.slice.
systemd[1]: Created slice system-modprobe.slice.
systemd[1]: Created slice system-serial\x2dgetty.slice.
systemd[1]: Created slice User and Session Slice.
systemd[1]: Started Dispatch Password Requests to Console Directory Watch.
systemd[1]: Started Forward Password Requests to Wall Directory Watch.
systemd[1]: Reached target Host and Network Name Lookups.
systemd[1]: Reached target Paths.
systemd[1]: Reached target Remote File Systems.
systemd[1]: Reached target Slices.
systemd[1]: Reached target Swap.
systemd[1]: Listening on Syslog Socket.
systemd[1]: Listening on initctl Compatibility Named Pipe.
systemd[1]: Listening on Journal Audit Socket.
systemd[1]: Listening on Journal Socket (/dev/log).
systemd[1]: Listening on Journal Socket.
systemd[1]: Listening on Network Service Netlink Socket.
systemd[1]: Listening on udev Control Socket.
systemd[1]: Listening on udev Kernel Socket.
systemd[1]: Listening on User Database Manager Socket.
systemd[1]: Mounting Huge Pages File System...
systemd[1]: Mounting POSIX Message Queue File System...
systemd[1]: Mounting Kernel Debug File System...
systemd[1]: Mounting Kernel Trace File System...
systemd[1]: Mounting Temporary Directory (/tmp)...
systemd[1]: Starting Create list of static device nodes for the current kernel...
systemd[1]: Starting Load Kernel Module configfs...
systemd[1]: Starting Load Kernel Module drm...
systemd[1]: Starting Load Kernel Module fuse...
systemd[1]: Starting Journal Service...
systemd[1]: Starting Load Kernel Modules...
systemd[1]: Starting Remount Root and Kernel File Systems...
systemd[1]: Starting Coldplug All udev Devices...
systemd[1]: Mounted Huge Pages File System.
systemd[1]: Mounted POSIX Message Queue File System.
systemd[1]: Mounted Kernel Debug File System.
systemd[1]: Mounted Kernel Trace File System.
systemd[1]: Mounted Temporary Directory (/tmp).
systemd[1]: Finished Create list of static device nodes for the current kernel.
systemd[1]: modprobe@configfs.service: Succeeded.
systemd[1]: Finished Load Kernel Module configfs.
systemd[1]: modprobe@drm.service: Succeeded.
systemd[1]: Finished Load Kernel Module drm.
systemd[1]: modprobe@fuse.service: Succeeded.
systemd[1]: Finished Load Kernel Module fuse.
systemd[1]: Finished Remount Root and Kernel File Systems.
systemd[1]: Condition check resulted in FUSE Control File System being skipped.
tun: Universal TUN/TAP device driver, 1.6
systemd[1]: Mounting Kernel Configuration File System...
systemd[1]: Condition check resulted in Rebuild Hardware Database being skipped.
systemd[1]: Condition check resulted in Platform Persistent Storage Archival being skipped.
systemd[1]: Condition check resulted in Create System Users being skipped.
systemd[1]: Starting Create Static Device Nodes in /dev...
systemd[1]: Finished Load Kernel Modules.
systemd[1]: Mounted Kernel Configuration File System.
systemd[1]: Starting Apply Kernel Variables...
systemd[1]: Finished Create Static Device Nodes in /dev.
systemd[1]: Reached target Local File Systems (Pre).
systemd[1]: Set up automount factory.automount.
systemd[1]: Mounting /var/volatile...
audit: type=1334 audit(1641936662.346:2): prog-id=5 op=LOAD
audit: type=1334 audit(1641936662.347:3): prog-id=6 op=LOAD
systemd[1]: Starting Rule-based Manager for Device Events and Files...
systemd[1]: Finished Apply Kernel Variables.
systemd[1]: Mounted /var/volatile.
systemd[1]: Condition check resulted in Bind mount volatile /var/cache being skipped.
systemd[1]: Condition check resulted in Bind mount volatile /var/lib being skipped.
systemd[1]: Starting Load/Save Random Seed...
systemd[1]: Condition check resulted in Bind mount volatile /var/spool being skipped.
systemd[1]: Condition check resulted in Bind mount volatile /srv being skipped.
systemd[1]: Reached target Local File Systems.
systemd[1]: Condition check resulted in Rebuild Dynamic Linker Cache being skipped.
systemd[496]: systemd-udevd.service: ProtectHostname=yes is configured, but the kernel does not support UTS namespaces, ignoring namespace setup.
systemd[1]: Started Rule-based Manager for Device Events and Files.
systemd[1]: Started Journal Service.
systemd-journald[487]: Received client request to flush runtime journal.
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
acpi INT3491:00: GPIO: looking up 0 in _CRS
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
Bluetooth: HCI UART driver ver 2.3
Bluetooth: HCI UART protocol H4 registered
acpi INT3491:00: GPIO: looking up 0 in _CRS
Bluetooth: HCI UART protocol Broadcom registered
hci_uart_bcm serial0-0: GPIO lookup for consumer device-wakeup
hci_uart_bcm serial0-0: using ACPI for GPIO lookup
acpi BCM2E95:00: GPIO: looking up device-wakeup-gpios
acpi BCM2E95:00: GPIO: _DSD returned BCM2E95:00 1 0 0
hci_uart_bcm serial0-0: No GPIO consumer device-wakeup found
resource sanity check: requesting [mem 0xff200000-0xff3fffff], which spans more than 0000:00:0e.0 [mem 0xff298000-0xff29bfff]
caller devm_ioremap+0x45/0x80 mapping multiple BARs
pmd_set_huge: Cannot satisfy [mem 0x05e00000-0x06000000] with a huge-page mapping due to MTRR override.
sof-audio-pci-intel-tng 0000:00:0d.0: warning: No matching ASoC machine driver found
sof-audio-pci-intel-tng 0000:00:0d.0: Using nocodec machine driver
sof-audio-pci-intel-tng 0000:00:0d.0: Firmware info: version 1:8:0-9e7a8
sof-audio-pci-intel-tng 0000:00:0d.0: Firmware: ABI 3:18:1 Kernel ABI 3:18:0
sof-audio-pci-intel-tng 0000:00:0d.0: unknown sof_ext_man header type 3 size 0x30
random: crng init done
dw_dmac_pci 0000:00:15.0: DesignWare DMA Controller, 8 channels
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
acpi INT3491:00: GPIO: looking up 0 in _CRS
hci_uart_bcm serial0-0: GPIO lookup for consumer device-wakeup
hci_uart_bcm serial0-0: using ACPI for GPIO lookup
acpi BCM2E95:00: GPIO: looking up device-wakeup-gpios
acpi BCM2E95:00: GPIO: _DSD returned BCM2E95:00 1 0 0
hci_uart_bcm serial0-0: No GPIO consumer device-wakeup found
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
acpi INT3491:00: GPIO: looking up 0 in _CRS
hci_uart_bcm serial0-0: GPIO lookup for consumer device-wakeup
hci_uart_bcm serial0-0: using ACPI for GPIO lookup
acpi BCM2E95:00: GPIO: looking up device-wakeup-gpios
acpi BCM2E95:00: GPIO: _DSD returned BCM2E95:00 1 0 0
hci_uart_bcm serial0-0: No GPIO consumer device-wakeup found
sof-audio-pci-intel-tng 0000:00:0d.0: Firmware info: version 1:8:0-9e7a8
sof-audio-pci-intel-tng 0000:00:0d.0: Firmware: ABI 3:18:1 Kernel ABI 3:18:0
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
acpi INT3491:00: GPIO: looking up 0 in _CRS
hci_uart_bcm serial0-0: GPIO lookup for consumer device-wakeup
hci_uart_bcm serial0-0: using ACPI for GPIO lookup
acpi BCM2E95:00: GPIO: looking up device-wakeup-gpios
acpi BCM2E95:00: GPIO: _DSD returned BCM2E95:00 1 0 0
hci_uart_bcm serial0-0: No GPIO consumer device-wakeup found
input: mrfld_bcove_pwrbtn as /devices/pci0000:00/0000:00:13.0/INTC100E:00/mrfld_bcove_pwrbtn/input/input0
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
BUG: unable to handle page fault for address: ffffa322400ea000
#PF: supervisor read access in kernel mode
#PF: error_code(0x0000) - not-present page
PGD 1000067 P4D 1000067 PUD 10fe067 PMD 10ff067 PTE 0
Oops: 0000 [#1] PREEMPT SMP PTI
CPU: 1 PID: 507 Comm: systemd-udevd Not tainted 5.16.0-rc8-edison-acpi-standard #1
Hardware name: Intel Corporation Merrifield/Silver Ridge, BIOS 509 2014.07.16:17.35.04
RIP: 0010:pwm_lpss_probe+0xd3/0x110 [pwm_lpss]
Code: 83 c3 01 39 58 08 76 b1 48 63 c3 48 8d 14 40 48 8d 14 90 49 8b 44 24 38 48 8d 04 d0 48 8b 50 18 8b 40 10 c1 e0 0a 48 03 42 40 <8b> 00 85 c0 79 cb be 05 00 00 00 48 89 ef e8 ba 52 b0 f9 eb bc 0f
RSP: 0000:ffffa322403a3c30 EFLAGS: 00010286
RAX: ffffa322400ea000 RBX: 0000000000000003 RCX: 0000000000000000
RDX: ffff928846f8c7a8 RSI: 0000000000000286 RDI: 00000000ffffffff
RBP: ffff928841f7d8c8 R08: ffffffffbaef00e6 R09: ffff928848726b00
R10: 0000000000000000 R11: ffffffffbb408d60 R12: ffff928846f8c7a8
R13: ffff928841f7dbb0 R14: 000000000000000f R15: 0000000000000000
FS:  00007f701a5c77c0(0000) GS:ffff92887e300000(0000) knlGS:0000000000000000
CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
CR2: ffffa322400ea000 CR3: 0000000002c90000 CR4: 00000000001006e0
Call Trace:
 <TASK>
 pwm_lpss_probe_pci+0x2d/0x60 [pwm_lpss_pci]
 local_pci_probe+0x3d/0x70
 ? pci_match_device+0xd2/0x120
 pci_device_probe+0xa9/0x160
 really_probe+0x1f0/0x3f0
 __driver_probe_device+0xf9/0x170
 driver_probe_device+0x19/0x90
 __driver_attach+0xbb/0x1c0
 ? __device_attach_driver+0xd0/0xd0
 ? __device_attach_driver+0xd0/0xd0
 bus_for_each_dev+0x73/0xb0
 bus_add_driver+0x144/0x1e0
 driver_register+0x86/0xd0
 ? 0xffffffffc0275000
 do_one_initcall+0x3f/0x1e0
 ? kmem_cache_alloc_trace+0x3a/0x1b0
 do_init_module+0x56/0x240
 __do_sys_finit_module+0xae/0x110
 do_syscall_64+0x3b/0x90
 entry_SYSCALL_64_after_hwframe+0x44/0xae
RIP: 0033:0x7f701a6de0ad
Code: a5 0c 00 0f 05 eb a9 66 0f 1f 44 00 00 f3 0f 1e fa 48 89 f8 48 89 f7 48 89 d6 48 89 ca 4d 89 c2 4d 89 c8 4c 8b 4c 24 08 0f 05 <48> 3d 01 f0 ff ff 73 01 c3 48 8b 0d 9b 5d 0c 00 f7 d8 64 89 01 48
RSP: 002b:00007ffd8c1d0c88 EFLAGS: 00000246 ORIG_RAX: 0000000000000139
RAX: ffffffffffffffda RBX: 00005576eb5164b0 RCX: 00007f701a6de0ad
RDX: 0000000000000000 RSI: 00007f701a849283 RDI: 000000000000000f
RBP: 0000000000020000 R08: 0000000000000000 R09: 0000000000000000
R10: 000000000000000f R11: 0000000000000246 R12: 00007f701a849283
R13: 0000000000000000 R14: 0000000000000007 R15: 00005576eb5164b0
 </TASK>
Modules linked in: pwm_lpss_pci(+) pwm_lpss intel_mrfld_pwrbtn dw_dmac_pci intel_mrfld_adc dw_dmac_core snd_sof_pci_intel_tng snd_sof_pci snd_sof_intel_atom snd_sof snd_sof_xtensa_dsp snd_soc_acpi spi_pxa2xx_pci ti_ads7950 hci_uart industrialio_triggered_buffer btbcm kfifo_buf leds_gpio tun ledtrig_timer ledtrig_heartbeat mmc_block extcon_intel_mrfld sdhci_pci cqhci sdhci led_class mmc_core intel_soc_pmic_mrfld btrfs libcrc32c xor zlib_deflate raid6_pq zstd_compress
CR2: ffffa322400ea000
---[ end trace f8c01322edca4a4c ]---
RIP: 0010:pwm_lpss_probe+0xd3/0x110 [pwm_lpss]
Code: 83 c3 01 39 58 08 76 b1 48 63 c3 48 8d 14 40 48 8d 14 90 49 8b 44 24 38 48 8d 04 d0 48 8b 50 18 8b 40 10 c1 e0 0a 48 03 42 40 <8b> 00 85 c0 79 cb be 05 00 00 00 48 89 ef e8 ba 52 b0 f9 eb bc 0f
RSP: 0000:ffffa322403a3c30 EFLAGS: 00010286
RAX: ffffa322400ea000 RBX: 0000000000000003 RCX: 0000000000000000
RDX: ffff928846f8c7a8 RSI: 0000000000000286 RDI: 00000000ffffffff
RBP: ffff928841f7d8c8 R08: ffffffffbaef00e6 R09: ffff928848726b00
R10: 0000000000000000 R11: ffffffffbb408d60 R12: ffff928846f8c7a8
R13: ffff928841f7dbb0 R14: 000000000000000f R15: 0000000000000000
FS:  00007f701a5c77c0(0000) GS:ffff92887e300000(0000) knlGS:0000000000000000
CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
CR2: ffffa322400ea000 CR3: 0000000002c90000 CR4: 00000000001006e0
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
acpi INT3491:00: GPIO: looking up 0 in _CRS
hci_uart_bcm serial0-0: GPIO lookup for consumer device-wakeup
hci_uart_bcm serial0-0: using ACPI for GPIO lookup
acpi BCM2E95:00: GPIO: looking up device-wakeup-gpios
acpi BCM2E95:00: GPIO: _DSD returned BCM2E95:00 1 0 0
hci_uart_bcm serial0-0: No GPIO consumer device-wakeup found
using random self ethernet address
using random host ethernet address
Mass Storage Function, version: 2009/09/11
LUN: removable file: (no medium)
usb0: HOST MAC aa:bb:cc:dd:ee:f2
usb0: MAC aa:bb:cc:dd:ee:f1
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
spi_master spi5: GPIO lookup for consumer cs
spi_master spi5: using ACPI for GPIO lookup
acpi device:04: GPIO: looking up cs-gpios
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:04: GPIO: _DSD returned device:04 0 0 0
spi_master spi5: No GPIO consumer cs found
pxa2xx-spi pxa2xx-spi.5: problem registering SPI controller
sof-audio-pci-intel-tng 0000:00:0d.0: Topology: ABI 3:18:1 Kernel ABI 3:18:0
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
sof-nocodec sof-nocodec: ASoC: Parent card not yet available, widget card binding deferred
acpi INT3491:00: GPIO: looking up 0 in _CRS
hci_uart_bcm serial0-0: GPIO lookup for consumer device-wakeup
hci_uart_bcm serial0-0: using ACPI for GPIO lookup
acpi BCM2E95:00: GPIO: looking up device-wakeup-gpios
spi_master spi5: GPIO lookup for consumer cs
spi_master spi5: using ACPI for GPIO lookup
acpi device:04: GPIO: looking up cs-gpios
acpi BCM2E95:00: GPIO: _DSD returned BCM2E95:00 1 0 0
acpi device:04: GPIO: _DSD returned device:04 0 0 0
hci_uart_bcm serial0-0: No GPIO consumer device-wakeup found
spi_master spi5: No GPIO consumer cs found
pxa2xx-spi pxa2xx-spi.5: problem registering SPI controller
sdhci-pci 0000:00:01.2: SDHCI controller found [8086:1190] (rev 1)
sdhci-pci 0000:00:01.2: GPIO lookup for consumer cd
sdhci-pci 0000:00:01.2: using ACPI for GPIO lookup
acpi device:00: GPIO: looking up cd-gpios
acpi device:00: GPIO: _DSD returned device:00 0 0 0
sdhci-pci 0000:00:01.2: No GPIO consumer cd found
acpi INT3491:00: GPIO: looking up 0 in _CRS
hci_uart_bcm serial0-0: GPIO lookup for consumer device-wakeup
hci_uart_bcm serial0-0: using ACPI for GPIO lookup
acpi BCM2E95:00: GPIO: looking up device-wakeup-gpios
spi_master spi5: GPIO lookup for consumer cs
spi_master spi5: using ACPI for GPIO lookup
acpi device:04: GPIO: looking up cs-gpios
acpi device:04: GPIO: _DSD returned device:04 0 0 0
acpi BCM2E95:00: GPIO: _DSD returned BCM2E95:00 1 0 0
spi_master spi5: No GPIO consumer cs found
hci_uart_bcm serial0-0: No GPIO consumer device-wakeup found
pxa2xx-spi pxa2xx-spi.5: problem registering SPI controller
audit: type=1334 audit(1641936666.278:4): prog-id=7 op=LOAD
audit: type=1334 audit(1641936666.285:5): prog-id=8 op=LOAD
audit: type=1334 audit(1641936666.693:6): prog-id=9 op=LOAD
audit: type=1334 audit(1641936666.709:7): prog-id=10 op=LOAD
audit: type=1334 audit(1641936667.472:8): prog-id=11 op=LOAD
audit: type=1334 audit(1641936667.487:9): prog-id=12 op=LOAD
audit: type=1325 audit(1641936667.784:10): table=filter family=2 entries=0 op=xt_register pid=586 subj=kernel comm="connmand"
audit: type=1300 audit(1641936667.784:10): arch=c000003e syscall=55 success=yes exit=0 a0=9 a1=0 a2=40 a3=557629aff230 items=0 ppid=1 pid=586 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="connmand" exe="/usr/sbin/connmand" subj=kernel key=(null)
audit: type=1327 audit(1641936667.784:10): proctitle=2F7573722F7362696E2F636F6E6E6D616E64002D6E
audit: type=1325 audit(1641936667.785:11): table=filter family=2 entries=4 op=xt_replace pid=586 subj=kernel comm="connmand"
audit: type=1300 audit(1641936667.785:11): arch=c000003e syscall=54 success=yes exit=0 a0=9 a1=0 a2=40 a3=557629b00500 items=0 ppid=1 pid=586 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="connmand" exe="/usr/sbin/connmand" subj=kernel key=(null)
audit: type=1327 audit(1641936667.785:11): proctitle=2F7573722F7362696E2F636F6E6E6D616E64002D6E
audit: type=1325 audit(1641936667.908:12): table=mangle family=2 entries=0 op=xt_register pid=586 subj=kernel comm="connmand"
audit: type=1300 audit(1641936667.908:12): arch=c000003e syscall=55 success=yes exit=0 a0=9 a1=0 a2=40 a3=557629b00920 items=0 ppid=1 pid=586 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="connmand" exe="/usr/sbin/connmand" subj=kernel key=(null)
systemd-udevd (501) used greatest stack depth: 12096 bytes left
kauditd_printk_skb: 23 callbacks suppressed
audit: type=1334 audit(1641936699.733:23): prog-id=0 op=UNLOAD
audit: type=1334 audit(1641936699.733:24): prog-id=0 op=UNLOAD
EXT4-fs (mmcblk0p3): mounted filesystem with ordered data mode. Opts: discard. Quota mode: none.
EXT4-fs (mmcblk0p9): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
EXT4-fs (mmcblk0p8): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
```
