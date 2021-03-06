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

This is more complicated because the EC debug serial port pins are shared with
keyboard scanning pins (subject to gateware build-time settings). And, the
UART, when enabled, is tunneled through a wishbone-bridge. If you want to try
the EC serial monitor, first you should read this:
https://github.com/betrusted-io/betrusted-ec/blob/main/README.adoc#debugging-notes
-- really, please read it.

### Initial Setup

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


### Test wishbone-tool connectivity

This will check that you can talk to the EC gateware over the Pi GPIO serial
port using wishbone-tool to check the EC's git revision register.

1. On your build machine, edit `betrusted-ec/betrusted_ec.py` to enable the
   wishbone-bridge on the EC's UART pins: First, find the code at around line
   560 with:
   ```
   debugonly = False

   if debugonly:
       self.submodules.uart_bridge = UARTWishboneBridge(...
   ```
   Then change `debugonly = False` to `debugonly = True`. This change will turn
   off the keyboard scan muxing on the serial port pins. The tradeoff is that
   you get reliable serial connectivity, but using the power-on key combo will
   not work to wake from sleep mode until you revert the change and re-build the
   EC gateware.

2. On your build machine, run a `make bin` to rebuild the binaries.

3. On your dev workstation, copy the new binaries to your Pi with scp.

4. On your Pi, flash the new EC gateware:
   ```
   cd ~/code/betrusted-scripts
   ./config_up5k.sh
   ```
   To reliably make a wishbone-bridge connection, you must **start**
   **wishbone-tool in the window from 50ms to 2000ms after an EC reset.**
   Otherwise, wishbone-tool will generate many lines like
   ```
   ERROR [wishbone_bridge::bridges::uart] serial port was closed: peek IoError
   ```
   Sleeping for 0.1 seconds between an EC reset and starting wishbone-tool seems
   to work reliably.

5. On your Pi, find the wishbone bus address of the EC's `gitrev` register in
   `precursors/csr.csv` (copied from `betrusted-ec/build/csr.csv`):
   ```
   $ cd ~/code/precursors
   $ grep gitrev csr.csv
   csr_register,git_gitrev,0xe0003000,1,ro
   ```
   In this example, the address is `0xe0003000`.

6. On your Pi, switch the Debug Hat serial mux to `up5k` (the EC), reset the EC,
   and use `wishbone-tool` to peek at the address for the git `gitrev` register:
   ```
   $ cd ~/code/betrusted-scripts
   $ ./uart_up5k.sh
   $ ./reset-ec.sh && sleep 0.1 && RUST_LOG=debug wishbone-tool --uart /dev/serial0 -b 115200 0xe0003000
   INFO [wishbone_bridge::bridges::uart] Re-opened serial device /dev/serial0
   DEBUG [wishbone_bridge::bridges::uart] Peeking @ e0003000
   DEBUG [wishbone_bridge::bridges::uart] PEEK @ e0003000 = f635beec
   Value at e0003000: f635beec
   DEBUG [wishbone_bridge::bridges::uart] strong count: 2  weak count: 0
   DEBUG [wishbone_tool] Exited MemoryAccess thread
   DEBUG [wishbone_bridge::bridges::uart] strong count: 1  weak count: 0
   DEBUG [wishbone_bridge::bridges::uart] serial_connect_thread requested exit
   ```
   In the example above, the git revision is `f635beec`.


### Tunnel EC UART through wishbone-tool

1. Make sure you can successfully complete the procedure above up to the point
   of reading the `gitrev` value. If that works, you should be able to run
   wishbone-tool without `RUST_LOG=debug` and assume that it will work.

   If you start the serial bridge when there is a serial connectivity problem,
   wishbone-tool can get into an error loop where it spends its time trying to
   make a serial connection, to the point of neglecting to check the keyboard
   input. In that case, pressing Ctrl-c will not stop wishbone-tool.

   If that happens, you need to make a second SSH connection to the Pi, find the
   process id for wishbone-tool with `ps x | grep wishbone` then do a
   `kill $PID` (substituting the correct process id) break the wishbone-tool
   process out of its infinite error loop. For example:
   ```
   $ ps x | grep wishbone-tool
   2139 pts/0    Sl+    0:00 wishbone-tool --serial /dev/serial0 -s terminal --csr-csv ../precursors/csr.csv
   2407 pts/1    S+     0:00 grep --color=auto wishbone-tool
   $ kill 2139
   ```

2. On your build machine, edit `betrusted-ec/sw/Cargo.toml` to uncomment the
   `debug_uart` default feature. The `[features]` section should have a line
   like this:
   ```
   default = ["debug_uart"]
   ```
   Also, make sure that `debugonly = True` is set in `betrusted_ec.py`.

3. Re-build your binaries, copy them to your Pi, and flash the EC binary with
   `betrusted-scripts/config_up5k.sh`. Use the same procedures described above
   in the section about testing wishbone-tool connectivity.

4. On your Pi, reset the EC and start the wishbone-tool serial terminal bridge:
   ```
   cd ~/code/betrusted-scripts
   ./uart_up5k.sh
   ./reset-ec.sh && sleep 0.1 && wishbone-tool --serial /dev/serial0 -s terminal --csr-csv ../precursors/csr.csv
   ```
   To stop the wishbone-tool serial terminal, press Ctrl-c.

You should see serial debug messages similar to this:
```
$ ./reset-ec.sh && sleep 0.06 && wishbone-tool --uart /dev/serial0 -b 115200 -s terminal --csr-csv ../precursors/csr.csv
Hello world!
before nommu
i2c
time_init
i2c_init
charger
chg_set_safety
gg_start
chg_set_autoparams
chg_start
BtUsbCc:new
tusb320lai_rev: 00000006
usb_cc
gyro_init
backlight
charger.update_regs
spi_standby
debug_uart selected: watchdog is not enabled
main loop
initializing wifi!
enabling interrupt: mask 1 channel 5
wf200 started!
Wifi ready
```
