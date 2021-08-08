# Automate build process for Precursor gateware & firmware images (for flashing)

#HW_IMG=xous-core/target/riscv32imac-unknown-xous-elf/release/xous.img
#LOADER=xous-core/target/riscv32imac-unknown-xous-elf/release/loader.bin
HW_IMG=xous-core/target/riscv32imac-unknown-none-elf/release/xous.img
LOADER=xous-core/target/riscv32imac-unknown-none-elf/release/loader.bin
XC7S_GATEWARE=xous-core/precursors/soc_csr.bin
UP5K_GATEWARE=betrusted-ec/target/riscv32i-unknown-none-elf/release/bt-ec.bin
UP5K_CSR=betrusted-ec/build/csr.csv
UP5K_SW=betrusted-ec/target/riscv32i-unknown-none-elf/release/sw
OBJDUMP_SW=riscv64-unknown-elf-objdump -h $(UP5K_SW)
SED_SW_TEXT_SIZE=sed -n 's/.*.text \+\([^ ]\+\).*/\1/p'
TEXT_SIZE_PY3=print('EC sw .text size: 0x$${HEX_SIZE} =', 0x$$HEX_SIZE, 'bytes')

help:
	@echo Sorry, this is a work in progress.
	@echo Read Makefile for details on available build targets.

# Build all the binaries and gather them into precursors/
bin: precursors $(HW_IMG) $(LOADER) $(XC7S_GATEWARE) $(UP5K_GATEWARE)
	cp $(HW_IMG) $(LOADER) $(XC7S_GATEWARE) $(UP5K_GATEWARE) $(UP5K_CSR) precursors/
	@HEX_SIZE=`$(OBJDUMP_SW)|$(SED_SW_TEXT_SIZE)`; python3 -c "$(TEXT_SIZE_PY3)"

# Build Xous firmware and loader, reusing other binaries
.PHONY: fast-soc
fast-soc:
	cd xous-core; cargo xtask hw-image precursors/soc.svd
	cp $(HW_IMG) $(LOADER) $(XC7S_GATEWARE) $(UP5K_GATEWARE) $(UP5K_CSR) precursors/

# Build EC firmware and reuse other binaries (assumes you did a make bin earlier)
.PHONY: fast-ec
fast-ec:
	cd betrusted-ec; cargo xtask hw-image
	cp $(HW_IMG) $(LOADER) $(XC7S_GATEWARE) $(UP5K_GATEWARE) $(UP5K_CSR) precursors/
	@HEX_SIZE=`$(OBJDUMP_SW)|$(SED_SW_TEXT_SIZE)`; python3 -c "$(TEXT_SIZE_PY3)"

# Build EC image
$(UP5K_GATEWARE): betrusted-ec icestorm-tools ubuntu-packages rust-tools
	cd betrusted-ec; python3 ./betrusted_ec.py --revision=pvt --seed=1
	cd betrusted-ec; cargo xtask copy-precursors
	cd betrusted-ec; cargo xtask hw-image

# Build Xous image: loader + OS + application
$(HW_IMG): xous-core rust-tools
	cd xous-core; cargo xtask hw-image precursors/soc.svd

# Make a directory to hold the build artifacts
precursors:
	mkdir -p precursors

# Clone xous-core repo and get submodules (currently there aren't any)
xous-core:
	git clone https://github.com/betrusted-io/xous-core.git
	cd xous-core; git submodule init; git submodule update

# Clone betrusted-ec repo and get submodules
betrusted-ec:
	git clone https://github.com/betrusted-io/betrusted-ec.git
	cd betrusted-ec; git submodule init; git submodule update

# This target is meant to fail if you need to install Ubuntu (or Debian-ish distro) packages.
# - Having gcc-riscv64-unknown-elf installed will confuse cargo xtask hw-image badly
ubuntu-packages:
	@echo checking .deb package dependencies...
	dpkg -l libssl-dev
	if dpkg -l gcc-riscv64-unknown-elf; then false; else true; fi

# This build target is meant to fail if you don't have all the necessary iCE40 FPGA tools installed.
# See github.com/betrusted-io/betrusted-ec/README.adoc for install instructions.
icestorm-tools:
	python3 --version
	yosys --version
	nextpnr-ice40 --version

# This build target is meant to fail if you don't have all the necessary rust tools installed.
# If that happens, see https://www.rust-lang.org/learn/get-started, then do whatever of this
# stuff you haven't done already:
# 1. `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
# 2. Answer the prompts to finish installing rustup
# 3. Make sure your PATH is set correctly
# 4. `rustup target add riscv32imac-unknown-none-elf`
# 5. `rustup target add riscv32i-unknown-none-elf`
RV32IMAC=riscv32imac-unknown-none-elf
RV32I=riscv32i-unknown-none-elf
rust-tools:
	rustup --version
	cargo --version
	@echo Checking for rust RV32 cross compiler targets...
	rustup target list | grep $(RV32IMAC) | grep installed
	rustup target list | grep $(RV32I) | grep installed

.PHONY: help rust-tools icestorm-tools ubuntu-packages
