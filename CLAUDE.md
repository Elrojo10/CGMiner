# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

cgminer 4.11.1 — a multi-threaded, multi-pool ASIC/FPGA Bitcoin (SHA256) miner written in C, licensed GPLv3. This repository (Elrojo10/cgminer) is a fork of ckolivas/cgminer. The whole application builds into a single `cgminer` binary.

Reference docs in the repo root: `README` (usage + build), `ASIC-README`, `FPGA-README`, and `API-README` (RPC API protocol and command list).

## Build Commands

Autotools-based. There is no test suite and no linter — verifying a change means a clean compile (watch for new `-Wall` warnings) and, where possible, running the binary.

```sh
./autogen.sh                                          # only when building from git / after changing configure.ac or Makefile.am
CFLAGS="-O2 -Wall -march=native" ./configure <options>
make                                                  # produces ./cgminer in the build directory
```

Important: **every device driver is disabled by default.** Pass at least one `--enable-*` flag to configure (e.g. `--enable-icarus`, `--enable-avalon8`, `--enable-bitfury`) or the resulting binary supports no hardware. The full list of driver flags is in `README` (search "CGMiner specific configuration options") or `./configure --help`. Other notable flags: `--disable-libcurl`, `--without-curses`, `--with-system-jansson`.

Ubuntu dependencies:

```sh
sudo apt-get install build-essential autoconf automake libtool pkg-config \
    libcurl4-openssl-dev libudev-dev libusb-1.0-0-dev libncurses5-dev
```

Bundled copies of jansson (`compat/jansson-2.9/`) and uthash (`uthash.h`) are used when system versions are absent. Windows builds are cross-compiled with MXE (see `README`); `windows-build.txt` is outdated.

The project version lives in `configure.ac` (`v_maj`/`v_min`/`v_mic` m4 defines at the top).

## Architecture

Flat, single-directory C codebase producing one binary. The key pieces:

- `cgminer.c` — the core: `main()`, option parsing, pool management (failover / round-robin / rotate / load-balance / balance strategies), work generation (stratum, GBT, getwork), the watchdog thread that restarts sick devices, hotplug, statistics, and the curses TUI. Most global state lives here.
- `miner.h` — the central header everything includes. Defines the core structs: `struct device_drv` (the driver vtable, ~line 320), `struct cgpu_info` (one device instance), `struct pool`, `struct work`, and the locking wrappers.
- `api.c` — the RPC API (TCP port 4028, plain-text or JSON requests). Protocol and commands documented in `API-README`. Per-driver stats surface through the `get_api_stats` driver hook.
- `util.c` — stratum socket handling, hex/string helpers, time helpers.
- `usbutils.c` — the USB abstraction over libusb used by all USB drivers, including device claiming and hotplug support.
- `logging.c`, `sha2.c`, `klist.c`, `noncedup.c` — support code.
- `lib/` (gnulib shims), `compat/jansson-2.9/`, `ccan/` — vendored libraries built as static archives; each is an automake subdirectory.

### Driver model

Each hardware family is a `driver-*.c` file implementing a `struct device_drv`: `drv_detect()` finds devices, then the hash loop is either `scanhash` (device processes one work item at a time) or the queued model (`scanwork` + `queue_full`) for devices with their own work queues. Other hooks include `flush_work` (block change), `update_work` (new stratum template), `get_api_stats`, `get_statline_before`, and `set_device`.

Drivers are registered through X-macros in `miner.h` (~line 240): `FPGA_PARSE_COMMANDS` / `ASIC_PARSE_COMMANDS` expand to generate the `drv_driver` enum and the extern `*_drv` declarations. Adding a driver means touching four places:

1. `miner.h` — add a `DRIVER_ADD_COMMAND(name)` entry to the appropriate X-macro list
2. `configure.ac` — add the `--enable-*` option and its automake conditional
3. `Makefile.am` — add the sources under that conditional
4. `driver-<name>.c` — define `struct device_drv <name>_drv`

Driver code is guarded by `#ifdef USE_<NAME>` (defined by configure). Consequence: when editing shared code (`cgminer.c`, `miner.h`, `api.c`, `usbutils.c`), most driver-specific blocks are not compiled in your configuration — enable the relevant drivers when testing changes that touch them.

### Threading and locking

cgminer is heavily multi-threaded: one or more mining threads per device, per-pool stratum receive/send threads, plus API, watchdog, and curses input/log threads. Shared state is protected by the wrappers in `miner.h` — `mutex_lock`/`mutex_unlock`, `cg_rlock`/`cg_wlock`/`cg_ilock` (a custom `cglock_t` read/intermediate/write lock) — which record file/function/line for lock debugging. Use these wrappers, never raw pthread calls.
