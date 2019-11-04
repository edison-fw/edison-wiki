# acpi

ACPI is an advanced mechanism to do a power management and provide a full featured platform description. It's quite flexible while it has a set of strong rules. More information about ACPI you may find on [the official site](http://www.uefi.org/acpi/specs)

### Rationale

Intel had spent a lot into IoT and mobile markets which ends up with
x86-based smartphones, tablets, and few IoT platforms. The one of IoT
platform, i.e. Intel Edison, inherited some stuff which had been
designed for smartphones. One of the items is so called SFI tables which
supposed to be a quite simplified substitution of ACPI for mobile
devices. While it works more or less fine on smartphones, it almost
useless otherwise, especially on IoT DIY environment where user needs
flexibility over, for example, time to boot.

ACPI is well supported framework, especially now when ARM64 officially
runs ACPI. Moving towards ACPI allows to unleash a power of flexibility
as it's done for Intel Galileo or Joule platforms [here](https://github.com/westeri/meta-acpi/).

### Prerequisites
  * U-Boot support]
    * Should be **v2018.01** or newer
  * Linux kernel
    * Should be v5.??
  * ACPICA
    * Should be **20170531** or newer

### Progress
#### U-Boot support

   * Should have the following key patches (they might require others to be applied smoothly):
      * [256df1e1c6](http://git.denx.de/?p=u-boot.git;a=commitdiff;h=256df1e1c6664e94926affe9318faa8258c18582)             edison: Bring minimal ACPI support to the board.
      * [39665beed6](http://git.denx.de/?p=u-boot.git;a=commitdiff;h=39665beed6f741fd8ae6ebf23d07fbd546a0d55e)             tangier: Enable ACPI support for Intel Tangier.
      * [1602d215b5](http://git.denx.de/?p=u-boot.git;a=commitdiff;h=1602d215b595b9cf4be460101f5c8892623fe3a0)  tangier: Use official ACPI HID for FLIS IP.
      * [d08953e045](http://git.denx.de/?p=u-boot.git;a=commitdiff;h=d08953e04596a83076b0f55e7c20b2c2c472e793) tangier: Use actual GPIO hardware numbers.
      * [5d8c4ebd95](http://git.denx.de/?p=u-boot.git;a=commitdiff;h=5d8c4ebd95e23a606a40a73920b8d7d096a91d37)
            tangier: Add Bluetooth to ACPI table. (optional)
   * `CONFIG_GENERATE_ACPI_TABLE=y` in *edison_defconfig* (maybe we will create another *defconfig* for that).

#### Linux kernel
  * Should have the following key patches (they might require others to be applied smoothly):
  * Generic core
    *  [Make IRQ allocation a bit more flexible](https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git/commit/?id=5b395e2be6c4621962889cb28ba2d6d1be42e39d).
    *  [Move PCI initialization to arch_init()](https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git/commit/?id=a912a7584ec39647fb032c1001eb69746f27b1d3).
    *  [Add special handling for ACPI HW reduced platforms](https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git/commit/?id=50beba07a0e42ebd4454adc97515a2a2a969645b).
    *  ???
  * ACPI / boot
    * [Use INVALID_ACPI_IRQ instead of 0 for acpi_sci_override_gsi](https://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm.git/commit/?id=4565c4f60569).
    * [Don't setup SCI on HW-reduced platforms](https://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm.git/commit/?id=7c7bcfeae2d8)
  * Pin control
    * ACPI glue layer to support ACPI r6.2 for pin control resources
    * [Introduce ACPI device table](https://git.kernel.org/pub/scm/linux/kernel/git/linusw/linux-pinctrl.git/commit/?id=dabd4bc6de2b)
    * [Add support of ACPI enabled platforms](https://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm.git/commit/?id=dd1dbf94d282)
    * ???
  * SDIO / Wi-Fi
    * [Fix up power if device has ACPI companion](https://git.kernel.org/pub/scm/linux/kernel/git/ulfh/mmc.git/commit/?id=0e39220ed6be)
    * [Advertise 2.0v supply on SDIO host controller](https://git.kernel.org/pub/scm/linux/kernel/git/ulfh/mmc.git/commit/?id=2a609abe71ca)
  * USB OTG
    * [Add support for Merrifield Basin Cove PMIC](https://git.kernel.org/pub/scm/linux/kernel/git/lee/mfd.git/commit/?id=b9a801dfa591)
  * A/DC ADS7951 (Edison/Arduino)
    * [Add OF device ID table](https://git.kernel.org/pub/scm/linux/kernel/git/jic23/iio.git/commit/?id=fb4b6f92d19)
    * [Allow to use on ACPI platforms](https://git.kernel.org/pub/scm/linux/kernel/git/jic23/iio.git/commit/?id=8cfa26a77cf)
    * ???

#### Yocto
  * Currently all the prerequisites here have been incorporated into [unofficial meta-intel-edison layer](https://github.com/htot/meta-intel-edison/tree/pyro64-acpi).
    * This layer builds a 64 bit linux 4.15.0 with ACPI patches applied
    * U-Boot v2018.01 with ACPI patches applied
    * Uses [meta-acpi layer expanded with edison acpi SSDT example tables](https://github.com/htot/meta-acpi/tree/eds)
    * Provides Yocto Pyro (experimental Rocko version available)
    * ACPI functionality is enabled by adding "acpi" to DISTRO_FEATURES 
      in poky-edison.conf.
    * Users can select to flash the system image using flashall or Flash
      Tool Lite or selectively update u-boot, install new kernel
      with initramfs into the former OTA partition and run the
      rootfs from either SD card or USB without breaking the
      factory image
    * (Note: running older kernels including factory from ACPI enabled
      U-Boot requires setting acpi=off on the kernel command
      line)

#### ACPI tables
  * ~~An example of Edison/Arduino board support in ACPI: [File:Edison-arduino-asl.zip](https://edison.internet-share.com/wiki/File:Edison-arduino-asl.zip)~~
        The archive is a bit outdated. Most recent examples are available in
        [meta-acpi](https://github.com/westeri/meta-acpi/tree/master/recipes-bsp/acpi-tables/samples/edison)

### Known issues

We need volunteers!

#### U-Boot support
  * ~~Requires a patch to enable SD card~~ (in **v2019.10-rc2**)
  * Requires a patch to enable timeout on DFU (makes slightly more convenient
    with the original environment)
  * There is no crystal clear split between SoC and platform with regard to ACPI
    tables (will we need this at all?)

#### Linux kernel
  * ~~SDIO requires a quirk to allow 1.8v on the bus (Wi-Fi)]~~
  * DMA burst size should be 8. ~~Can we supply this information via
    _DSD] Maintainer wants no new DT bindings.~~ SPI gets it based on PCI ID.
  * Not confirmed. ~~There is a silicon(?) bug in SPI/DMA for
    speeds < 800 kHz (see original code), there is also patch to
    make another threshold, i.e. > 500 bytes when frequency is > 4 MHz.~~
    Also see [Fix FIFO size for Intel Merrifield](https://git.kernel.org/pub/scm/linux/kernel/git/vkoul/slave-dma.git/commit/?id=ffe843b18211)
  * I2C bus 6 requires non-standard pin control on the OS level. ~~Can we
        utilize U-Boot for enabling it? Can MCU on the other hand send
        commands to SCU to switch pin modes? I think we may
        consider to use Operation Region to set up I2C6 initial mode as
        well (needs to be investigated).~~ U-Boot sets up correct
        mode via Device Tree. Either U-Boot or ACPI approach makes no
        additional code needed to Linux kernel, although it might be
        required in case there is MCU routine which is using I2C bus 6
        pins.
  * First GPIO I2C expander fails to probe from time to time (root cause?).
        Linux kernel got the driver refactored very much.
  * ~~RTC is platform legacy device (should be PNP0B02)]~~ Enabled via
        ACPI.
  * A/DC driver (ti-ads7950) is going to support hard coded 5V reference (see
        patch list above).
  * Basin Cove [PMIC](https://edison.internet-share.com/wiki/PMIC)[
        support is ~~absent~~ on its way to the upstream.

#### ACPI tables
  * _DEP method is supported only for Operation Regions in Linux kernel
        (to check ACPI specification) 
  * Misc [GPIO controller](https://edison.internet-share.com/wiki/ACPI/GPIO_controller)
        enumeration issue

### TODO

### U-Boot support
  * Intel Tangier support in [U-Boot](https://edison.internet-share.com/wiki/U-Boot)
        (in **v2017.09**)
  * ACPI support code for Intel Tangier (in **v2017.11-rc2**)
  * Offical ACPI ID for FLIS (in **v2018.01-rc3**)
  * Correct numbering scheme for GPIO pins (in **v2018.01**)

#### Linux kernel has to be patched a bit to support these modifications
  * USB OTG support via [PMIC](https://edison.internet-share.com/wiki/PMIC)
        with help from ACPI
  * [PMIC](https://edison.internet-share.com/wiki/PMIC)
     * OperationRegion support
  * ACPI glue layer to support pin configuration and pin muxing
  * HW reduced platforms are not handled properly in the kernel (partially
        done, see patch list above)
  * Intel Edison platform code should cope with ACPI enabled firmware (U-Boot)
  * Bug fixes related to this activity

#### Yocto [meta-acpi layer](https://github.com/westeri/meta-acpi/) to be extended
  * As meta-acpi is the layer providing ACPI support to Galileo, Joule and
        Minnowboard planning has started to move the ACPI pieces from
        meta-intel-edison here.

#### ACPI tables
  * DSDT
     * USB Do we need anything special here?
     * GFX
     * Disable (_STA to return 0) unused devices?
     * ~~RTC as a PNP device~~ Done! (~~kernel needs to be patched?~~
            Seems fine as is
     * SDHCI (SD card detection) has hard coded stuff in the kernel, we
            want to make least intrusion, so, leave it as is for now ->
            no ACPI entry
     * Edison/Arduino board description for meta-acpi layer
     * Edison/Mini break out example??? USB OTG works only on this official
            board.
     * [PMIC](https://edison.internet-share.com/wiki/PMIC)
            description, official ACPI ID INTC100E (in
            **2019.04-rc2**)
        * DPTF thermal zones table
        * LPAT calibration table for temperature sensor (needs
                [PMIC](https://edison.internet-share.com/wiki/PMIC)
                A/DC support I guess)
     * ???
  * HPET
     * HPET is present in the (internal) documentation, is it enabled? Can
            we use it?

### Unsorted

Experimental output of `dmesg | grep -i`
```
[    0.000000] Command line: console=tty1 console=ttyS2,115200n8 rootfstype=ramfs rw ignore_loglevel apic=debug acpi.debug_layer=0xffff056d acpi.debug_level=0x0000200f
[    0.000000] ACPI: SSDT ACPI table found in initrd [kernel/firmware/acpi/arduino-all.aml][0x5c4]
[    0.000000] ACPI: SSDT ACPI table found in initrd [kernel/firmware/acpi/spidev.aml][0xa6]
[    0.000000] modified: [mem 0x000000003f4ff000-0x000000003f4ff669] ACPI data
[    0.000000] ACPI: Early table checksum verification disabled
[    0.000000] ACPI: RSDP 0x00000000000E4500 000024 (v02 U-BOOT)
[    0.000000] ACPI: XSDT 0x00000000000E45E0 00003C (v01 U-BOOT U-BOOTBL 20170725 INTL 00000000)
[    0.000000] ACPI: FACP 0x00000000000E4CE0 0000F4 (v06 U-BOOT U-BOOTBL 20170725 INTL 00000000)
[    0.000000] ACPI: DSDT 0x00000000000E4780 000460 (v02 U-BOOT U-BOOTBL 00010000 INTL 20170303)
[    0.000000] ACPI: APIC 0x00000000000E4DE0 000048 (v04 U-BOOT U-BOOTBL 20170725 INTL 00000000)
[    0.000000] ACPI: MCFG 0x00000000000E4E30 00003C (v01 U-BOOT U-BOOTBL 20170725 INTL 00000000)
[    0.000000] ACPI: Table Upgrade: install [SSDT-      - ARDUINO]
[    0.000000] ACPI: SSDT 0x000000003F4FF000 0005C4 (v05  ARDUINO  00000001 INTL 20170303)
[    0.000000] ACPI: Table Upgrade: install [SSDT-      -  SPIDEV]
[    0.000000] ACPI: SSDT 0x000000003F4FF5C4 0000A6 (v05        SPIDEV   00000001 INTL 20170303)
[    0.000000] ACPI: Local APIC address 0xfee00000
[    0.000000] ACPI: no legacy devices present
[    0.000000] ACPI: Local APIC address 0xfee00000
[    0.000000] ACPI: LAPIC (acpi_id[0x00] lapic_id[0x00] enabled)
[    0.000000] ACPI: LAPIC (acpi_id[0x02] lapic_id[0x02] enabled)
[    0.000000] ACPI: IOAPIC (id[0x02] address[0xfec00000] gsi_base[0])
[    0.000000] Using ACPI (MADT) for SMP configuration information
[    0.000000] Kernel command line: console=tty1 console=ttyS2,115200n8 rootfstype=ramfs rw ignore_loglevel apic=debug acpi.debug_layer=0xffff056d acpi.debug_level=0x0000200f
[    0.000493] ACPI: Core revision 20170629
[    0.001000] ACPI: 3 ACPI AML tables successfully acquired and loaded
[    0.078249] ACPI FADT declares the system doesn't support PCIe ASPM, so disable it
[    0.078267] ACPI: bus type PCI registered
[    0.128630] ACPI: Added _OSI(Module Device)
[    0.128774] ACPI: Added _OSI(Processor Device)
[    0.128910] ACPI: Added _OSI(3.0 _SCP Extensions)
[    0.129050] ACPI: Added _OSI(Processor Aggregator Device)
[    0.129396] ACPI: Executed 2 blocks of module-level executable AML code
[    0.134394] ACPI: Interpreter enabled
[    0.134572] ACPI: (supports S0)
[    0.134679] ACPI: Using IOAPIC for interrupt routing
[    0.134938] PCI: Using host bridge windows from ACPI; if necessary, use "pci=nocrs" and report a bug
[    0.184018]       bus-0135 bus_get_status        : Device [ACPI] status [0000000f]
[    0.186317] ACPI: PCI Root Bridge [PCI0] (domain 0000 [bus 00-ff])
[    0.186780] acpi PNP0A08:00: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI]
[    0.187645] acpi PNP0A08:00: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability]
[    0.187885] acpi PNP0A08:00: FADT indicates ASPM is unsupported, using BIOS configuration
[    0.189539] acpi PNP0A08:00: [Firmware Info]: MMCONFIG for domain 0000 [bus 00-00] only partially covers this bridge
[    0.276498] ACPI: bus type USB registered
[    0.357736] pnp: PnP ACPI init
[    0.358903] pnp: PnP ACPI: found 0 devices

```
Experimental output of Rocko image (provides libgpiod). This may be
useful to find the correct pin also on Pyro.
```
Poky (Yocto Project Reference Distro) 2.4.1 edison ttyS2
edison login: root
root@edison:~# uname -a
Linux edison 4.15.0-edison-acpi-standard #2 SMP Tue Feb 13 22:01:53 CET
2018 x86_64 x86_64 x86_64 GNU/Linux
root@edison:~# gpio
gpiodetect  gpiofind    gpioget     gpioinfo    gpiomon     gpioset

root@edison:~# gpiodetect
gpiochip4 [INT3491:03] (16 lines)
gpiochip3 [INT3491:02] (16 lines)
gpiochip2 [INT3491:01] (16 lines)
gpiochip1 [INT3491:00] (16 lines)
gpiochip0 [0000:00:0c.0] (192 lines)

root@edison:~# gpiofind "TRI_STATE_ALL"
gpiochip1 14
root@edison:~# cat
/sys/class/i2c-adapter/i2c-1/i2c-INT3491:00/gpio/gpiochip496/base 496
```

So on Pyro you can access "TRI_STATE_ALL" by exporting (496 + 14) = 510

```
root@edison:~# gpioinfo
gpiochip4 - 16 lines:
        [line   0:  "MUX33_DIR" "uart1-rx-oe" output active-high [kernel]]
        [line   1:  "MUX31_DIR" "uart1-tx-oe" output active-high [kernel]]
        [line   2:  "MUX29_DIR"       unused   input  active-high ]
        [line   3:  "MUX27_DIR"       unused   input  active-high ]
        [line   4:  "MUX24_DIR"       unused   input  active-high ]
        [line   5:  "MUX21_DIR"       unused   input  active-high ]
        [line   6:  "MUX19_DIR"       unused   input  active-high ]
        [line   7:  "MUX32_DIR"       unused   input  active-high ]
        [line   8:  "MUX30_DIR"       unused   input  active-high ]
        [line   9:  "MUX28_DIR"       unused   input  active-high ]
        [line  10:  "MUX26_DIR" "ssp5-fs1-oe" output active-high [kernel]]
        [line  11:  "MUX23_DIR" "ssp5-txd-oe" output active-high [kernel]]
        [line  12:  "MUX20_DIR" "ssp5-rxd-oe" output active-high [kernel]]
        [line  13:  "MUX18_DIR" "ssp5-clk-oe" output active-high [kernel]]
        [line  14:  "MUX22_SEL" "ssp5-txd-mux" output active-high [kernel]]
        [line  15:  "MUX25_SEL" "ssp5-fs1-mux" output active-high [kernel]]
gpiochip3 - 16 lines:
        [line   0:  "MUX14_DIR"       unused   input  active-high ]
        [line   1:  "MUX12_DIR"       unused   input  active-high ]
        [line   2:  "MUX10_DIR"       unused   input  active-high ]
        [line   3:   "MUX8_DIR"       unused   input  active-high ]
        [line   4:   "MUX6_DIR"       unused   input  active-high ]
        [line   5:   "MUX4_DIR"       unused   input  active-high ]
        [line   6:  "U16_IO0.6"       unused   input  active-high ]
        [line   7:  "U16_IO0.7"       unused   input  active-high ]
        [line   8: "SPI_FS_SEL" "ssp5-fs1-mux" output active-high [kernel]]
        [line   9: "SPI_TXD_SEL" "ssp5-txd-mux" output active-high [kernel]]
        [line  10: "SPI_RXD_SEL" "ssp5-rxd-mux" output active-high [kernel]]
        [line  11: "SPI_CLK_SEL" "ssp5-clk-mux" output active-high [kernel]]
        [line  12:  "U16_IO1.4"       unused   input  active-high ]
        [line  13:  "U16_IO1.5"       unused   input  active-high ]
        [line  14:  "U16_IO1.6"       unused   input  active-high ]
        [line  15:  "U16_IO1.7"       unused   input  active-high ]
gpiochip2 - 16 lines:
        [line   0: "DIG0_PU_PD" "uart1-rx-pu" input active-high [kernel]]
        [line   1: "DIG1_PU_PD" "uart1-tx-pu" input active-high [kernel]]
        [line   2: "DIG2_PU_PD"       unused   input  active-high ]
        [line   3: "DIG3_PU_PD"       unused   input  active-high ]
        [line   4: "DIG4_PU_PD"       unused   input  active-high ]
        [line   5: "DIG5_PU_PD"       unused   input  active-high ]
        [line   6: "DIG6_PU_PD"       unused   input  active-high ]
        [line   7: "DIG7_PU_PD"       unused   input  active-high ]
        [line   8: "DIG8_PU_PD"       unused   input  active-high ]
        [line   9: "DIG9_PU_PD"       unused   input  active-high ]
        [line  10: "DIG10_PU_PD" "ssp5-fs1-pu" input active-high [kernel]]
        [line  11: "DIG11_PU_PD" "ssp5-txd-pu" input active-high [kernel]]
        [line  12: "DIG12_PU_PD" "ssp5-rxd-pu" input active-high [kernel]]
        [line  13: "DIG13_PU_PD" "ssp5-clk-pu" input active-high [kernel]]
        [line  14:  "U39_IO1.6"       unused   input  active-high ]
        [line  15:  "U39_IO1.7"       unused   input  active-high ]
gpiochip1 - 16 lines:
        [line   0:  "MUX15_SEL"       unused   input  active-high ]
        [line   1:  "MUX13_SEL"       unused   input  active-high ]
        [line   2:  "MUX11_SEL"       unused   input  active-high ]
        [line   3:   "MUX9_SEL"       unused   input  active-high ]
        [line   4:   "MUX7_SEL"       unused   input  active-high ]
        [line   5:   "MUX5_SEL"       unused   input  active-high ]
        [line   6:  "U17_IO0.6"       unused   input  active-high ]
        [line   7: "SHLD_RESET0" unused input active-high ]
        [line   8:   "A0_PU_PD"       unused   input  active-high ]
        [line   9:   "A1_PU_PD"       unused   input  active-high ]
        [line  10:   "A2_PU_PD"       unused   input  active-high ]
        [line  11:   "A3_PU_PD"       unused   input  active-high ]
        [line  12:   "A4_PU_PD"       unused   input  active-high ]
        [line  13:   "A5_PU_PD"       unused   input  active-high ]
        [line  14: "TRI_STATE_ALL" unused output active-high ]
        [line  15: "SHLD_RESET1" unused input active-high ]
gpiochip0 - 192 lines:
        [line   0:      unnamed       unused   input  active-high ]
        [line   1:      unnamed       unused   input  active-high ]
        [line   2:      unnamed       unused  output  active-high ]
        [line   3:      unnamed       unused   input  active-high ]
        [line   4:      unnamed       unused  output  active-high ]
        [line   5:      unnamed       unused  output  active-high ]
        [line   6:      unnamed       unused  output  active-high ]
        [line   7:      unnamed       unused   input  active-high ]
        [line   8:      unnamed       unused   input  active-high ]
        [line   9:      unnamed       unused  output  active-high ]
        [line  10:      unnamed       unused  output  active-high ]
        [line  11:      unnamed       unused  output  active-high ]
        [line  12:      unnamed       unused  output  active-high ]
        [line  13:      unnamed       unused  output  active-high ]
        [line  14:      unnamed       unused   input  active-high ]
        [line  15:      unnamed  "interrupt"   input  active-high
[kernel]]
        [line  16:      unnamed       unused   input  active-high ]
        [line  17:      unnamed       unused   input  active-high ]
        [line  18:      unnamed       unused   input  active-high ]
        [line  19:      unnamed       unused   input  active-high ]
        [line  20:      unnamed       unused   input  active-high ]
        [line  21:      unnamed       unused   input  active-high ]
        [line  22:      unnamed       unused   input  active-high ]
        [line  23:      unnamed       unused   input  active-high ]
        [line  24:      unnamed       unused   input  active-high ]
        [line  25:      unnamed       unused   input  active-high ]
        [line  26:      unnamed       unused   input  active-high ]
        [line  27:      unnamed       unused   input  active-high ]
        [line  28:      unnamed       unused   input  active-high ]
        [line  29:      unnamed       unused   input  active-high ]
        [line  30:      unnamed       unused   input  active-high ]
        [line  31:      unnamed       unused   input  active-high ]
        [line  32:      unnamed       unused   input  active-high ]
        [line  33:      unnamed       unused   input  active-high ]
        [line  34:      unnamed       unused   input  active-high ]
        [line  35:      unnamed       unused   input  active-high ]
        [line  36:      unnamed       unused   input  active-high ]
        [line  37:      unnamed       unused   input  active-high ]
        [line  38:      unnamed       unused   input  active-high ]
        [line  39:      unnamed       unused   input  active-high ]
        [line  40:      unnamed       unused  output  active-high ]
        [line  41:      unnamed       unused   input  active-high ]
        [line  42:      unnamed       unused   input  active-high ]
        [line  43:      unnamed       unused   input  active-high ]
        [line  44:      unnamed       unused   input  active-high ]
        [line  45:      unnamed       unused   input  active-high ]
        [line  46:      unnamed       unused   input  active-high ]
        [line  47:      unnamed       unused   input  active-high ]
        [line  48:      unnamed       unused   input  active-high ]
        [line  49:      unnamed       unused   input  active-high ]
        [line  50:      unnamed       unused  output  active-high ]
        [line  51:      unnamed       unused   input  active-high ]
        [line  52:      unnamed       unused   input  active-high ]
        [line  53:      unnamed       unused   input  active-high ]
        [line  54:      unnamed       unused  output  active-high ]
        [line  55:      unnamed       unused   input  active-high ]
        [line  56:      unnamed       unused   input  active-high ]
        [line  57:      unnamed       unused   input  active-high ]
        [line  58:      unnamed       unused   input  active-high ]
        [line  59:      unnamed       unused   input  active-high ]
        [line  60:      unnamed       unused   input  active-high ]
        [line  61:      unnamed       unused   input  active-high ]
        [line  62:      unnamed       unused   input  active-high ]
        [line  63:      unnamed       unused   input  active-high ]
        [line  64:      unnamed       unused   input  active-high ]
        [line  65:      unnamed       unused   input  active-high ]
        [line  66:      unnamed       unused   input  active-high ]
        [line  67:      unnamed       unused   input  active-high ]
        [line  68:      unnamed       unused   input  active-high ]
        [line  69:      unnamed       unused   input  active-high ]
        [line  70:      unnamed       unused   input  active-high ]
        [line  71:      unnamed   "shutdown"  output  active-high
[kernel]]
        [line  72:      unnamed       unused   input  active-high ]
        [line  73:      unnamed       unused   input  active-high ]
        [line  74:      unnamed       unused   input  active-high ]
        [line  75:      unnamed       unused   input  active-high ]
        [line  76:      unnamed       unused   input  active-high ]
        [line  77:      unnamed      "sd_cd"   input  active-high
[kernel]]
        [line  78:      unnamed       unused   input  active-high ]
        [line  79:      unnamed       unused   input  active-high ]
        [line  80:      unnamed       unused   input  active-high ]
        [line  81:      unnamed       unused   input  active-high ]
        [line  82:      unnamed       unused   input  active-high ]
        [line  83:      unnamed       unused   input  active-high ]
        [line  84:      unnamed       unused   input  active-high ]
        [line  85:      unnamed       unused   input  active-high ]
        [line  86:      unnamed       unused   input  active-high ]
        [line  87:      unnamed       unused   input  active-high ]
        [line  88:      unnamed       unused   input  active-high ]
        [line  89:      unnamed       unused   input  active-high ]
        [line  90:      unnamed       unused   input  active-high ]
        [line  91:      unnamed       unused   input  active-high ]
        [line  92:      unnamed       unused   input  active-high ]
        [line  93:      unnamed       unused   input  active-high ]
        [line  94:      unnamed       unused   input  active-high ]
        [line  95:      unnamed       unused   input  active-high ]
        [line  96:      unnamed "ACPI:OpRegion" output active-high
[kernel]]
        [line  97:      unnamed       unused   input  active-high ]
        [line  98:      unnamed       unused   input  active-high ]
        [line  99:      unnamed       unused   input  active-high ]
        [line 100:      unnamed       unused   input  active-high ]
        [line 101:      unnamed       unused   input  active-high ]
        [line 102:      unnamed       unused   input  active-high ]
        [line 103:      unnamed       unused   input  active-high ]
        [line 104:      unnamed       unused   input  active-high ]
        [line 105:      unnamed       unused   input  active-high ]
        [line 106:      unnamed       unused   input  active-high ]
        [line 107:      unnamed       unused   input  active-high ]
        [line 108:      unnamed       unused   input  active-high ]
        [line 109:      unnamed       unused   input  active-high ]
        [line 110:      unnamed         "cs"  output  active-high
[kernel]]
        [line 111:      unnamed         "cs"  output  active-high
[kernel]]
        [line 112:      unnamed         "cs"  output  active-high
[kernel]]
        [line 113:      unnamed         "cs"  output  active-high
[kernel]]
        [line 114:      unnamed       unused   input  active-high ]
        [line 115:      unnamed       unused   input  active-high ]
        [line 116:      unnamed       unused   input  active-high ]
        [line 117:      unnamed       unused   input  active-high ]
        [line 118:      unnamed       unused   input  active-high ]
        [line 119:      unnamed       unused   input  active-high ]
        [line 120:      unnamed       unused   input  active-high ]
        [line 121:      unnamed       unused   input  active-high ]
        [line 122:      unnamed       unused   input  active-high ]
        [line 123:      unnamed       unused   input  active-high ]
        [line 124:      unnamed       unused   input  active-high ]
        [line 125:      unnamed       unused   input  active-high ]
        [line 126:      unnamed       unused   input  active-high ]
        [line 127:      unnamed       unused   input  active-high ]
        [line 128:      unnamed       unused   input  active-high ]
        [line 129:      unnamed       unused   input  active-high ]
        [line 130:      unnamed       unused   input  active-high ]
        [line 131:      unnamed       unused   input  active-high ]
        [line 132:      unnamed       unused   input  active-high ]
        [line 133:      unnamed       unused   input  active-high ]
        [line 134:      unnamed       unused   input  active-high ]
        [line 135:      unnamed       unused   input  active-high ]
        [line 136:      unnamed       unused   input  active-high ]
        [line 137:      unnamed       unused   input  active-high ]
        [line 138:      unnamed       unused   input  active-high ]
        [line 139:      unnamed       unused   input  active-high ]
        [line 140:      unnamed       unused   input  active-high ]
        [line 141:      unnamed       unused   input  active-high ]
        [line 142:      unnamed       unused   input  active-high ]
        [line 143:      unnamed       unused   input  active-high ]
        [line 144:      unnamed       unused   input  active-high ]
        [line 145:      unnamed       unused   input  active-high ]
        [line 146:      unnamed       unused   input  active-high ]
        [line 147:      unnamed       unused   input  active-high ]
        [line 148:      unnamed       unused   input  active-high ]
        [line 149:      unnamed       unused   input  active-high ]
        [line 150:      unnamed       unused   input  active-high ]
        [line 151:      unnamed       unused   input  active-high ]
        [line 152:      unnamed       unused   input  active-high ]
        [line 153:      unnamed       unused   input  active-high ]
        [line 154:      unnamed       unused   input  active-high ]
        [line 155:      unnamed       unused   input  active-high ]
        [line 156:      unnamed       unused   input  active-high ]
        [line 157:      unnamed       unused   input  active-high ]
        [line 158:      unnamed       unused   input  active-high ]
        [line 159:      unnamed       unused   input  active-high ]
        [line 160:      unnamed       unused   input  active-high ]
        [line 161:      unnamed       unused   input  active-high ]
        [line 162:      unnamed       unused   input  active-high ]
        [line 163:      unnamed       unused   input  active-high ]
        [line 164:      unnamed       unused   input  active-high ]
        [line 165:      unnamed       unused  output  active-high ]
        [line 166:      unnamed       unused   input  active-high ]
        [line 167:      unnamed       unused   input  active-high ]
        [line 168:      unnamed       unused   input  active-high ]
        [line 169:      unnamed       unused   input  active-high ]
        [line 170:      unnamed       unused   input  active-high ]
        [line 171:      unnamed       unused   input  active-high ]
        [line 172:      unnamed       unused   input  active-high ]
        [line 173:      unnamed       unused  output  active-high ]
        [line 174:      unnamed       unused   input  active-high ]
        [line 175:      unnamed       unused  output  active-high ]
        [line 176:      unnamed       unused  output  active-high ]
        [line 177:      unnamed       unused  output  active-high ]
        [line 178:      unnamed       unused   input  active-high ]
        [line 179:      unnamed       unused  output  active-high ]
        [line 180:      unnamed       unused   input  active-high ]
        [line 181:      unnamed       unused  output  active-high ]
        [line 182:      unnamed       unused  output  active-high ]
        [line 183:      unnamed       unused   input  active-high ]
        [line 184:      unnamed "device-wakeup" output active-high [kernel]]
        [line 185:      unnamed "host-wakeup" input active-high [kernel]]
        [line 186:      unnamed       unused   input  active-high ]
        [line 187:      unnamed       unused   input  active-high ]
        [line 188:      unnamed       unused   input  active-high ]
        [line 189:      unnamed       unused  output  active-high ]
        [line 190:      unnamed       unused  output  active-high ]
        [line 191:      unnamed       unused   input  active-high ]
```
