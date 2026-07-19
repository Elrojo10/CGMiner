# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

cgminer is a multi-threaded, multi-pool FPGA and ASIC Bitcoin miner written in C (version 4.11.1, GPLv3). This is a fork of ckolivas/cgminer. It is a single large C program — almost all sources live in the repository root; there is no `src/` directory.

## Building

Autotools-based build. There is no test suite and no linter; verification is "it compiles and runs".

```sh
./autogen.sh          # only needed when building from git (regenerates configure)
CFLAGS="-O2 -Wall -march=native" ./configure <options>
make
```

Key points:

- **All hardware drivers are disabled by default.** You must pass at least one `--enable-*` flag to configure for the binary to support any device, e.g. `--enable-icarus`, `--enable-avalon`, `--enable-avalon8`, `--enable-bitfury`, `--enable-bflsc`, `--enable-ants1`/`--enable-ants2` (Antminer), `--enable-knc`, `--enable-sp10`/`--enable-sp30` (Spondoolies), etc. Run `./configure --help` for the full list.
- Mandatory deps: pkg-config, libtool (plus autoconf/automake when building from git). Optional: libcurl (GBT support; use `--disable-libcurl` if absent), ncurses (TUI; `--without-curses` to skip), libusb-1.0 + libudev (USB device support). Bundled copies of jansson (`compat/jansson-2.9`) and uthash are used when system versions are unavailable (`--with-system-jansson` to override).
- Ubuntu deps: `sudo apt-get install build-essential autoconf automake libtool pkg-config libcurl4-openssl-dev libudev-dev libusb-1.0-0-dev libncurses5-dev`
- Windows builds are cross-compiled with mxe (32-bit, `--host=i686-pc-mingw32`); see README and windows-build.txt.
- The binary runs from the build directory; `make install` is optional.

Run: `cgminer -o stratum+tcp://pool:port -u username -p password` (see README for pool/proxy variants).

## Architecture

### Core files

- `cgminer.c` — the entire mining engine: `main()`, option parsing (using ccan/opt), pool management and failover, stratum/GBT/getwork work generation, the work queues, the watchdog/statistics threads, and the per-device mining thread loops. ~10k lines; most cross-cutting changes land here.
- `miner.h` — the central header shared by everything: `struct cgpu_info` (a device instance), `struct device_drv` (the driver vtable), `struct pool`, `struct work`, global externs, and API declarations. Every driver includes it.
- `api.c` — the JSON RPC API served on port 4028 (see API-README). Drivers expose stats through `get_api_stats`/`get_api_debug` callbacks.
- `util.c` — networking, stratum protocol implementation, time helpers.
- `usbutils.c` — the libusb abstraction layer for all USB devices, including the device table (`usb_find_devices`) that maps USB VID/PID to drivers.
- `logging.c`, `sha2.c`, `noncedup.c`, `klist.c` — logging, SHA-256d hashing, duplicate-nonce detection, list helpers.
- `ccan/`, `compat/`, `lib/`, `m4/` — vendored ccan modules (opt, etc.), bundled jansson, gnulib shims, autoconf macros. Don't hand-edit generated/vendored code.

### Driver model

Every piece of mining hardware is a driver implementing the `struct device_drv` vtable defined in `miner.h` (~line 320):

- `drv_detect` enumerates hardware (usually via `usbutils.c` for USB devices, or SPI/I2C contexts — `spi-context.c`, `i2c-context.c` — for SoC/board miners).
- Work is executed through one of two loops chosen in `cgminer.c`: `scanhash` (device handles one work item at a time, driven by `hash_sole_work`) or `scanwork` + `queue_full` (device manages its own queue, driven by `hash_queued_work`/`hash_driver_work`).
- `flush_work`/`update_work` notify the driver of block changes and new stratum templates.

Drivers live in `driver-*.c` files (e.g. `driver-avalon8.c`, `driver-icarus.c`, `driver-bitfury.c`); families with extra support code use prefixes (`bf16-*` for Bitfury16, `A1-*` for Bitmine A1, `dm_*`/`dragonmint_t1.*` for DragonMint T1, `knc-*` for KnC).

### Adding or modifying a driver

Wiring a driver touches three places:

1. `driver-<name>.c` (+ header) implementing `device_drv`.
2. `configure.ac` — add an `--enable-<name>` option and AM_CONDITIONAL.
3. `Makefile.am` — add the sources under the matching `if HAS_<NAME>` block.

USB devices additionally need an entry in the device table in `usbutils.c`.

## Documentation conventions

User-facing behavior changes should be reflected in the matching doc: README (general options/building), ASIC-README (ASIC device options), FPGA-README (FPGA devices), API-README (RPC API commands). New command-line options are defined in `cgminer.c`'s opt table and documented in these files, and `example.conf` shows config-file syntax (config keys are the long option names without `--`).
