# pre_docs

Unofficial developer documentation for Precursor.

## Contents

1. [Overview](#overview)
2. [Glossary of Precursor Project Jargon](#glossary-of-precursor-project-jargon)
3. [Hardware Setup](#hardware-setup)
   1. [Battery and FPC Handling Precautions](#-battery-and-fpc-handling-precautions-)
   2. [Parts List](#parts-list)
   3. [Recommended Tools](#recommended-tools)
   4. [Battery](#battery)
   5. [Raspberry Pi + Debug HAT](#raspberry-pi--debug-hat)
   6. [JTAG Cable](#jtag-cable)
4. [Software Setup](#software-setup)
   1. [SD Card with Raspberry Pi OS](#sd-card-with-raspberry-pi-os)
   2. [Install JTAG Packages](#install-jtag-packages)
   3. [Get Validation Firmware](#get-validation-firmware)
   4. [Get Firmware Programming Scripts](#get-firmware-programming-scripts)
5. [Flashing Firmware](#flashing-firmware)


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

### Where can I learn more?

Information about Precursor is spread around a bit.

These are some good starting points:
- https://precursor.dev
- https://github.com/betrusted-io/betrusted-wiki/wiki
- https://github.com/betrusted-io

## Glossary of Precursor Project Jargon

To understand the code and documentation in the [Precursor repositories][repos]
on GitHub, it helps to be familiar with the following terms:

- **EC**, short for Embedded Controller, refers to the combination of the iCE40
  FPGA and its gateware configured with a Litex VexRISC-V "[minimal]" RV32I
  core and an SPI controller from [PicoRV32]. EC denotes the place where
  firmware for Precursor's untrusted domain runs. The EC manages battery
  charging and the wifi radio, among other things.
  See [Guided Tour of the Precursor Motherboard][EC].

  [minimal]: https://github.com/betrusted-io/betrusted-ec/blob/d167b75b7127a930e7602ed8d7bf40dec200662f/betrusted_ec.py#L790-L792
  [PicoRV32]: https://github.com/cliffordwolf/picorv32

- **SoC** refers to the combination of the Spartan 7 FPGA and its gateware
  configured with a VexRISC-V RV32IMAC core and crypto accelerators. SoC denotes
  the place where firmware for Precursor's trusted domain runs. The SoC manages
  keyboard input and display output, and it runs OS and application code. See
  [What is a System-on-Chip (SoC), and Why Do We Care if They are Open Source?][SoC]

- **FPC** is short for Flexible Printed Circuit. Precursor uses three flexible
  printed circuits inside the case as cables to connect components. Two FPCs
  connect the display and backlight to the mainboard. Another FPC, sometimes
  referred to as "the FPC", connects the mainboard to GPIO breakout pads,
  battery pads (some mainboard revisions use other battery connectors),
  vibration motor, and speaker. The JTAG debug cable is also an FPC.

- **EVT**, **DVT**, and **PVT** are short for Engineering Validation Test,
  Design Validation Test, and Production Validation Test. EVT, DVT, and PVT are
  often used in the engineering and manufacturing of consumer electronics
  products to denote iterative phases of prototyping. "Test" refers to making a
  design, producing a small batch of prototypes according to that design,
  evaluating the prototypes, then using what was learned to create an improved
  design revision.

  The important thing to understand when looking at Precursor design files on
  GitHub, is that files labeled EVT or DVT are for early design revisions. The
  final, or nearly final, revisions will be labeled PVT. Changes can still
  happen in the PVT phase for reasons like correcting problems with
  manufacturing yield. Photographs of PVT prototypes may not match the final
  PVT design revision that ships to campaign backers. In particular, the final
  production run will have a custom battery pack.

[EC]: https://www.bunniestudios.com/blog/?p=5942
[SoC]: https://www.bunniestudios.com/blog/?p=5971
[repos]: https://github.com/betrusted-io


## Hardware Setup

### 🔥 Battery and FPC Handling Precautions 🔥

**WARNING**: When handling partially unplugged FPC cable assemblies, be careful
not to short power rail connections on the exposed traces.

1. Some prototype revisions use FPC solder pads for the battery connection. To
   disconnect those batteries, you need to unplug the FPC from the mainboard.
   When you insert or remove the FPC, be sure to keep it pointed straight
   relative to the FPC connector. Orienting the FPC diagonally in the connector
   could short adjacent traces.

2. When an FPC with an attached battery is disconnected from the mainboard,
   **cover the exposed FPC traces with insulating tape.**

3. You will need to remove the keyboard PCB for access to the mainboard FPC
   connector latches. It should be fine to remove or install the keyboard while
   the battery is still connected, just be careful not to short anything on the
   mainboard while handling the keyboard.

4. The JTAG debug cable and USB-C port both connect to power rails on the
   mainboard. When the JTAG cable is connected to the Debug HAT, it is safe to
   also connect USB-C, because the Debug HAT will prevent USB-C power from
   reaching the Raspberry Pi. **But**, if you have the Debug HAT end of the
   JTAG cable disconnected, **cover the exposed FPC traces with insulating
   tape.**

### Parts List

Parts and equipment you will need to complete the hardware setup procedure:
- Raspberry Pi 3B, 3B+, or 3A+ (other versions may work, but are not supported)
- Good quality micro SD card
  (see https://www.raspberrypi.org/documentation/installation/sd-cards.md)
- Good quality USB power supply
  (see https://www.raspberrypi.org/documentation/hardware/raspberrypi/power/README.md)
- USB keyboard
- Monitor and HDMI cable (see https://www.raspberrypi.org/documentation/setup/)
- M2.5 x 11mm standoffs
- Insulating tape with clean-removal adhesive (Kapton, Scotch 35, or similar)
- Precursor Debug HAT with JTAG cable
- Precursor

### Recommended Tools

To follow the procedures described below, you will need:
- ESD protection gear (wrist strap and work mat)
- Screwdriver with T3 Torx bit
- Spudger
- Computer with micro SD port
  (see https://www.raspberrypi.org/documentation/installation/installing-images/README.md)

### Battery

The production run will come with an installed battery (source: "What's in the
Box?" section of campaign page). For pre-production prototypes, it may be
necessary to obtain a battery locally and install it yourself.

For details on the thickness of the battery compartment (3.5 mm plus a small
cushion to accommodate normal battery pack swelling), see
[Precursor’s Mechanical Design](https://www.bunniestudios.com/blog/?p=6023).

### Raspberry Pi + Debug HAT

1. Prepare your ESD protected work area
2. Attach two M2.5 x 11mm standoffs to the Debug HAT on the side of the board
   opposite the 40-pin connector
3. Connect the Debug HAT to your Raspberry Pi's GPIO connector

### JTAG Cable

**WARNING**: If you haven't already, please read the
[Battery and FPC Handling Precautions](#-battery-and-fpc-handling-precautions-)
section above.

*TODO: How to handle battery disconnect while gaining access to mainboard?*
- *Is it even possible?*
- *Is the upper FPC connector blocked by the keyboard overlay?*
- *What about the final production mainboard battery connector?*

To attach the Precursor end of the JTAG cable:
1. Remove Precursor Bezel (six Torx T3 screws)
2. Remove keyboard overlay
3. Gently pry the keyboard PCB straight up and set it aside (*TODO: screws?*)
4. Use spudger to open the latch on the JTAG connector (just inside the bottom
   edge of the case, to right of headset jack)
5. Cover the row of gold contacts on one end of the JTAG cable with insulating
   tape (to protect against ESD glitches or shorting power rails)
6. Orient the JTAG cable with the printed side on top and the exposed gold
   contacts on the bottom
7. Insert the end of the JTAG cable with the exposed contacts through the slot
   next to the headset jack and into the JTAG connector on the mainboard, making
   sure the cable is fully seated
8. Close the latch on the JTAG connector
9. Replace the keyboard PCB, making sure the connector in the center-left
   section of the PCB aligns with its matching connector on the mainboard
10. Replace the keyboard overlay
11. Replace the front bezel (six T3 screws)

To attach the Raspberry Pi end of the JTAG cable:
1. Make sure the Raspberry Pi power supply is disconnected
2. Use spudger to open the latch of the JTAG connector on the Debug Hat
3. Remove insulating tape from the JTAG cable (avoid shorting exposed contacts)
4. Insert JTAG cable (gold contacts facing down) into the JTAG connector
5. Close the JTAG connector latch


## Software Setup

### SD Card with Raspberry Pi OS

First, please be sure to follow the Raspberry Pi Foundation's advice on
sourcing a suitable [SD card][pi_sd] and [power supply][pi_ps]. Poor quality SD
cards and inadequate power supplies commonly cause Raspberry Pi problems.

[pi_sd]: https://www.raspberrypi.org/documentation/installation/sd-cards.md
[pi_ps]: https://www.raspberrypi.org/documentation/hardware/raspberrypi/power/README.md

1. Download a "Raspberry Pi OS **Lite**" operating system SD card image from
   https://www.raspberrypi.org/software/operating-systems/
2. Next to the Download button, use the SHA256 link to get the file integrity
   hash for the zip file you just downloaded
3. Verify the zip file against the hash, and re-download if there's a problem.
4. Flash the OS image onto your micro SD card following the instructions at
   https://www.raspberrypi.org/documentation/installation/installing-images/README.md
5. Insert the micro SD card into your Raspberry Pi, being careful not to
   unplug or damage the JTAG debug cable
6. Connect your HDMI monitor and USB keyboard to the Pi
7. Connect power to the Pi
8. Log in (u:pi, p:raspberry)
9. Set a proper password with `passwd`
   (see https://www.raspberrypi.org/documentation/linux/usage/users.md)
10. Run `sudo raspi-config` (*TODO: more specific details of menu navigation*)
    - Localization Options: locale, country code, timezone
    - hostname
    - wifi (see https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)
    - ssh
11. *(optional)* Add ssh public keys to `~/.ssh/authorized_keys`
12. Run `sudo apt update` to refresh the package index
13. Run `sudo apt upgrade` to install updates
14. Reboot
15. ... ? (*TODO*)

Alternate setup method: headless setup without keyboard or monitor:
- see https://www.raspberrypi.org/documentation/configuration/wireless/headless.md
### Install JTAG Packages

See https://github.com/im-tomu/fomu-flash and https://github.com/betrusted-io/jtag-trace

### Get Validation Firmware

See https://github.com/betrusted-io/betrusted-soc

### Get Firmware Programming Scripts

See https://github.com/betrusted-io/betrusted-scripts
- Main scripts are `provision-fw.sh`, `provision-xous.sh`, `config_up5k.sh`,
  and `wfx_firmware.sh`
- Scripts depend on `jtag-tools`, `fomu-flash`, and `wishbone-tool`
- betrusted-scripts repo gets the `jtag-tools` dependency by including
  [betrusted-io/jtag-trace](https://github.com/betrusted-io/jtag-trace)
  as a submodule (needs recursive clone)


## Flashing Firmware



## License

Dual licensed under the terms of [Apache 2.0](LICENSE-APACHE) or
[MIT](LICENSE-MIT), at your option.
