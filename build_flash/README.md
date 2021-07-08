# Build & Flash Precursor EC and SoC firmware

## Dev LAN

1. Wifi or ethernet

2. Build machine running Ubuntu 20 LTS

3. Raspberry Pi is configured with Debug Hat and Debug cable to Precursor and
   running latest Raspberry Pi OS

4. Dev workstation with ssh logins to build machine and Pi is used for controlling builds,
   staging "precursors" binaries, flashing, and debug monitor over serial (via ssh to Pi)


## Build

Use the [Makefile](Makefile) in this directory to orchestrate building of `betrusted-ec`
and `xous-core`.

If you want to use forks instead of the betrusted-io upstreams, do your own clones before
running `make`. The makefile will clone betrusted-ec and xous-core if they don't already
exist. But, if the directories are present, the makefile will leave them alone.


## Staging binaries (precursors)

1. Set up the Pi with a `code/` directory containing a clone of
   `betrusted-scripts`, a clone of `fomu-flash`, and a `precursors/` directory

2. From Dev workstation, set up a scratch directory and run these commands
   (perhaps as scripts)

   ```
   #!/bin/sh
   BUILD_BOX=ci@buildbox.local
   PI=pi@raspberrypi.local
   BUILD_DIR=wherever/you/put/it
   scp "$BUILD_BOX:$BUILD_DIR/precursors/*" .
   scp bt-ec.bin loader.bin soc_csr.bin xous.img csr.csv $PI:code/precursors/
   ```


## Flashing binaries

1. **Carefully** consider your power supply to the Precursor handset.
   Precursor's power system is designed to have a battery present. Booting from
   USB or debug cable power alone is possible, but doing so is not stable or
   recommended. Do not attempt to flash without a battery connected.

   If you have the battery configured with a disconnect switch, the handset
   will come up with the charge controller in ship mode once you connect the
   battery. To wake from ship mode, you need to apply external power to the
   USB-C port or the power pins on the debug FPC cable (see `vbus.sh` script).

   **Applying power from USB-C and the debug cable at the same time is not
   recommended!** You can use USB and the debug cable at the same time, but
   make sure that debug cable vbus power is off. Also, if you apply power from
   USB-C while the debug cable is connected, it can back-power the Raspberry
   Pi. So, you need to be careful about voltage differential between the USB-C
   power and your Raspberry Pi power supply.

   Recommended configurations:
   - Debug cable to Pi and leave USB-C disconnected
   - Leave debug cable disconnected and safely stowed, then use USB-C
   - Debug cable to Pi and USB-C to the same Pi (assuming vbus == vusb)

2. Once your handset and Pi are powered up, SSH to the PI and run these
   commands:
   ```
   cd code/betrusted-scripts
   provision-xous.sh
   config_up5k.sh
   ```


## Xous Serial Monitor

You can see log messages from Xous and the EC through the serial port mux
on the Pi Debug Hat. When the mux is configured for the Xous serial port,
it is also possible to inject keyboard input by typing in your serial
terminal emulator.

1. Use `sudo raspi-config` to configure your Raspberry Pi to use the GPIO
   serial pins as a general purpose serial port (not a login console).
   See https://www.raspberrypi.org/documentation/configuration/uart.md

3. SSH to the Pi

4. For a Xous serial monitor (mux set for XC7S UART), do this:
   ```
   cd ~/code/betrusted-scripts
   ./uart_fpga.sh
   screen /dev/serial0 115200
   ```
   To exit screen, type Ctrl-a k y

   To view the scrollback buffer (enter copy mode), type Ctrl-a ESC, then use
   vi nav keys. To exit copy mode, type ESC again.


## EC Serial Monitor

This is more complicated because the EC debug serial port can be tunneled through
wishbone-bridge. If you want to try the EC serial monitor, first you need to read
this: https://github.com/betrusted-io/betrusted-ec/blob/main/README.adoc#debugging-notes
-- really, please read it.

1. Make sure you can successfully use the Xous Serial Monitor procedure.

2. Build wishbone-tool for Raspberry Pi using the upstream source at
   litex/wishbone-utils rather than fomu-toolchain. SSH to the Pi. If you haven't
   installed rust, do so:
   ```
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   # Update your PATH (or you can log out an log back in)
   source ~/.cargo/env
   ```
   Then clone wishbone-utils and build wishbone-tool. This may take about 10
   minutes depending on which Raspberry Pi you have, what the SD card is like,
   and so on:
   ```
   cd ~/code
   git clone https://github.com/litex-hub/wishbone-utils.git
   cd wishbone-utils/wishbone-tool/
   cargo build --release
   cp target/release/wishbone-tool ~/.cargo/bin
   ```
   You can read more context about wishbone-tool at:
   - https://github.com/betrusted-io/betrusted-ec/blob/main/README.adoc
   - https://wishbone-utils.readthedocs.io/en/latest/wishbone-tool/
   - https://github.com/litex-hub/wishbone-utils

3. On your build machine, rebuild `bc-ec.bin` after editing
   `betrusted-ec/betrusted_ec.py` to enable the wishbone bus crossover UART.
   [TODO: how to enable the UART properly???]

4. Copy `bc-ec.bin` from your build machine to your Raspberry Pi.

5. Flash the EC firmware and start the wishbone-tool serial terminal bridge:
   ```
   cd ~/code/betrusted-scripts
   ./uart_up5k.sh && RUST_LOG=debug wishbone-tool --serial /dev/serial0 -s terminal --csr-csv ../precursors/csr.csv
   ```
   To exit the wishbone-tool serial monitor, assuming it connected okay, you
   can type Ctrl-c. If that works, you can remove the `RUST_LOG=debug`.

   If Ctrl-c doesn't work and you see lots of debug messages about serial port
   timeouts, wishbone-tool is attempting and failing to connect to the UART.
   This probably means you need to re-build the EC gateware (`bt-ec.bin`) with
   a different configuration (crossover UART is probably not enabled).

   If the crossover UART didn't connect, you won't be able to use Ctrl-c to
   kill wishbone-tool. In that case, make another terminal tab, SSH to another
   shell on your Pi, and do this to kill the wishbone-tool process:
   ```
   $ ps x | grep wishbone-tool
   2139 pts/0    Sl+    0:00 wishbone-tool --serial /dev/serial0 -s terminal --csr-csv ../precursors/csr.csv
   2407 pts/1    S+     0:00 grep --color=auto wishbone-tool
   $ kill 2139
   ```
