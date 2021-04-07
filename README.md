# pre_docs

Unofficial developer documentation for Precursor.

## Contents

1. [Overview](#overview)
2. [Hardware Setup](#hardware-setup)
   1. [Parts List](#parts-list)
   2. [Recommended Tools](#recommended-tools)
   3. [Battery](#battery)
   4. [Raspberry Pi + Debug HAT](#raspberry-pi--debug-hat)
   5. [JTAG Cable](#jtag-cable)
3. [Software Setup](#software-setup)
   1. [SD Card with Raspberry Pi OS](#sd-card-with-raspberry-pi-os)
   2. [Install JTAG Packages](#install-jtag-packages)
   3. [Get Validation Firmware](#get-validation-firmware)
   4. [Get Firmware Programming Scripts](#get-firmware-programming-scripts)
4. [Flashing Firmware](#flashing-firmware)


## Overview

This unofficial documentation is for developers who want to experiment with
modifying gateware or firmware for Precursor using its JTAG debug port.

### Why might you want to modify Precursor?

Precursor is for developers who want the security benefits of being able to
audit and modify low-level design features that, in other system architectures,
would be etched in silicon, buried in epoxy, and hidden behind NDAs.

Mobile devices commonly use tightly integrated System on Chip (SoC) modules to
reduce the cost, size, and power consumption of incorporating many peripherals
and CPU cores into a design. One disadvantage of SoC-based designs is that you
cannot modify the silicon to fix security flaws. Also, writing firmware is
often complicated by the inclusion of proprietary technology with documentation
restricted behind a Non-Disclosure Agreement (NDA).

Precursor takes a different approach by using Field Programmable Gate Array
(FPGA) chips instead of SoC modules. Implementing peripherals and CPU cores
with programmable gates inside an FPGA provides increased transparency into
details of the design. Also, with FPGAs, CPU design flaws can be corrected by
updating the FPGA with new gateware.

### How is gateware different from firmware?

Gateware is code that can configure an FPGA to act like a CPU core. Firmware is
code that can run on top of such a CPU core.

With an FPGA, low-level features can be changed in ways that would otherwise
require replacing etched silicon by physically swapping a chip for one with a
different design. When an FPGA powers up, it loads gateware to configure its
behavior. Changing the gateware is like etching new silicon and swapping the
chip, but with far less hassle and expense.

### What's the JTAG debug port about?

The JTAG interface allows for reprogramming Precursor's gateware and firmware
without any dependency on a bootloader or OS update service. JTAG is good for
changing low-level features.

Typical mobile or embedded products come with a bootloader or update service
that can overwrite some of the device's flash memory to install firmware
updates. But, how does the bootloader get programmed into flash to begin with?
In Precursor's case, JTAG can solve that problem, among others.

### Why does Precursor have more than one FPGA?

Short answer: Spliting features between two FPGAs helps to save power and
provide physical separation of device features into trusted and untrusted
security domains.

Medium answer: The goal is to use a low-power Lattice iCE40UP5K in the
untrusted domain to listen for incoming traffic and filter out irrelevant
packets. If successful, that will allow the power hungry XC7S50 (Xilinx Spartan
7) to spend most of its time in a deep low power mode. The trusted domain needs
the Spartan 7 to provide enough capacity for compute intensive encrypted chat
protocols. The Spartan 7 also has electrical control over the keyboard and
display to keep them isolated from the radio module.

Long answer: Read the description and updates at https://precursor.dev

### Can't the JTAG port bypass Precursor's security?

The production run will ship with a packet of epoxy.

### Where can I learn more?

Information about Precursor is spread around a bit.

These are some good starting points:
- https://precursor.dev
- https://github.com/betrusted-io/betrusted-wiki/wiki
- https://github.com/betrusted-io


## Hardware Setup

### Parts List

Parts and equipment you will need to complete the hardware setup procedure:
- Raspberry Pi 3B, 3B+, or 3A+ (other versions may work, but are not supported)
- Good quality SD card (see https://www.raspberrypi.org/documentation/installation/sd-cards.md)
- M2.5 x 11mm standoffs
- Good quality 2.5A USB power supply for the Raspberry Pi (don't use a cheap phone charger)
- Precursor Debug HAT with JTAG cable
- Precursor

### Recommended Tools

To follow the procedures described below, you will need:
- ESD protection gear (wrist strap and work mat)
- Screwdriver with T3 Torx bit
- Spudger

### Battery

The production run will come with an installed battery (source: "What's in the
Box?" section of campaign page). For pre-production prototypes, it may be
necessary to obtain a battery locally and install it yourself.

For details on the thickness of the battery compartment (3.5 mm plus a small
cushion to accommodate normal battery pack swelling), see
[Precursor’s Mechanical Design](https://www.bunniestudios.com/blog/?p=6023).

### Raspberry Pi + Debug HAT

Assemble Raspberry Pi and Debug HAT JTAG interface

### JTAG Cable

Connect Precursor and Debug HAT with JTAG cable


## Software Setup

### SD Card with Raspberry Pi OS

See https://www.raspberrypi.org/software/

### Install JTAG Packages

See https://github.com/im-tomu/fomu-flash

### Get Validation Firmware

See https://github.com/betrusted-io/betrusted-soc

### Get Firmware Programming Scripts

See https://github.com/betrusted-io/betrusted-scripts


## Flashing Firmware
