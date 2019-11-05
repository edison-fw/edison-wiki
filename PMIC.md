# PMIC

## ADC

Basin Cove PMIC contains a 10-bits ADC with 9 channels.

### The environment

What each input measures.

| Channel | Name in original driver      | Measured property | Apparent range | Description |
|:------------|:------------|:-----------|:------------|:------------|
| 0 | VBAT | Voltage | 0 (0V ?) to 0x3ff (4.5V ?)| Certainly not (directly)  Li-Ion battery voltage, as this does not belong to the Edison board but to a base board. Probably the V_BAT_BKU P pin (short-dura tion battery/cap acitor for power source change). |
| 1 | BATID | Resistance | 0..0x3ff ? | ?|
| 2 | IBAT | Current | 0 (most negative current ?)..0x3ff (most positive current ?) 0x1fe to 0x200 on an accessory-less base board | Maybe V_BAT_BKUP charge/discharge current ?  |
| 3 | PMICTEMP | Temperature | 0..0x3ff ? | Internal PMIC sensor ?|
| 4 | BATTEMP0 | Temperature | 0..0x3ff ? | ? |
| 5 | BATTEMP1 | Temperature | 0..0x3ff ? | ? |
| 6 | SYSTEMP0 | Temperature | 0..0x3ff ? | Connected to something. (what ?)|
| 7 | SYSTEMP1 | Temperature | 0..0x3ff ? | Unconnected |
| 8 | SYSTEMP2 | Temperature | 0..0x3ff ? | Unconnected |

### The chip

How the ADC interfaces with the CPU


 | Address | Name | Description | b7 | b6 | b5 | b4 | b3 | b2 | b1| b0 |
 |:--------|:-----|:------------|:--:|:--:|:--:|:--:|:--:|:--:|:-:|:--:|
 | 0x06 | ADCIRQ    | (defined but unused in original driver) |  ?| ? | ? | ? | ? | ? | ? | ? |
 | 0x11 | MADCIRQ   | IRQ mask ? | ? | ? | battery current/voltage ?| battery (what ?)| system temperature | battery temperature | ? | ? |
 | 0x0c | MIRQLVL1  | irq enable ? | ? | ? | ? | ADC (what ?) | ? | ? | ? | ? |
 | 0xdd | ADC1CNTL  | (defined but unused in original driver) | ? | ? | ? | ? | ? | ? | ? | ? |
 | 0xdc | GPADCREQ  | main ADC control register ? Writing to this register triggers a measure. | ? | ? | set by CPU to enable bank 3 measures (battery voltage and current) | set by CPU to enable bank 2 measures (battery id ?) | set by CPU to enable bank 1 measures (pmic and system 0, 1 and 2 temperatures) | set by CPU to enable bank 0 measures (battery temperatures 0 and 1) | set by CPU to enable IRQ | set by chip when adc busy |
 | 0xe9 | CH0 (MSB) | VBAT     | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xea | CH0 (LSB) | VBAT     | X | X | X | X | X | X | X | X |
 | 0xeb | CH1 (MSB) | BATID    | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xec | CH1 (LSB) | BATID    | X | X | X | X | X | X | X | X |
 | 0xed | CH2 (MSB) | IBAT     | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xee | CH2 (LSB) | IBAT     | X | X | X | X | X | X | X | X |
 | 0xcc | CH3 (MSB) | PMICTEMP | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xcd | CH3 (LSB) | PMICTEMP | X | X | X | X | X | X | X | X |
 | 0xc8 | CH4 (MSB) | BATTEMP0 | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xc9 | CH4 (LSB) | BATTEMP0 | X | X | X | X | X | X | X | X |
 | 0xca | CH5 (MSB) | BATTEMP1 | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xcb | CH5 (LSB) | BATTEMP1 | X | X | X | X | X | X | X | X |
 | 0xc2 | CH6 (MSB) | SYSTEMP0 | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xc3 | CH6 (LSB) | SYSTEMP0 | X | X | X | X | X | X | X | X |
 | 0xc4 | CH7 (MSB) | SYSTEMP1 | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xc5 | CH7 (LSB) | SYSTEMP1 | X | X | X | X | X | X | X | X |
 | 0xc6 | CH8 (MSB) | SYSTEMP2 | 0 | 0 | 0 | 0 | 0 | 0 | X | X |
 | 0xc7 | CH8 (LSB) | SYSTEMP2 | X | X | X | X | X | X | X | X |

## Driver

Since most of the Intel Atom SoCs have PMIC driver based on ACPI
description we need to develop similar for Basin Cove.

### State of affairs

[U-Boot](u-boot-update)
has a separate edison-acpi branch in my GitHub tree to
provide additional AML code to support some devices. One of them is
PMIC. Locally in my kernel tree (I'm going to publish the changes
sooner or later) I have done the following drivers: 

 * Common MFD driver for Basin Cove PMIC
 * Power button
 * GPADC under IIO subsystem

The work is still in progress and volunteers are needed to test and help
with development!

Currently I'm doing

 * The extcon driver to provide a USB cable status (dual role and charger type)
