# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

YAPicoprobe is a CMSIS-DAP compatible USB debug probe firmware for Raspberry Pi Pico (RP2040) and Pico 2 (RP2350). It turns a Pico into a SWD debug probe for ARM Cortex-M targets. Written in C with PIO assembly, running on FreeRTOS.

## Build System

Requires `PICO_SDK_PATH` environment variable pointing to the Pico SDK. Uses CMake with Ninja generator. FreeRTOS and CMSIS_DAP are git submodules.

### Initial setup
```bash
git submodule update --init --recursive
```

### Configure and build (using CMake presets)
```bash
cmake --preset pico-debug       # RP2040 debug build -> _build/pico-debug/
cmake --preset pico2-release    # RP2350 release build -> _build/pico2-release/
cmake --build --preset pico-debug-build
```

Available presets: `pico-debug`, `pico-release`, `pico2-debug`, `pico2-release`.

### Key CMake options
Features are toggled via `OPT_*` cache variables in CMakeLists.txt:
- `OPT_CMSIS_DAPV1/V2` - DAP protocol versions
- `OPT_TARGET_UART` - Target UART CDC bridge
- `OPT_PROBE_DEBUG_OUT` - Debug output destination (CDC/RTT/UART/"")
- `OPT_MSC` - Mass Storage for drag-n-drop flashing
- `OPT_SIGROK` - Logic analyzer support
- `OPT_NET` - Networking via NCM/ECM/RNDIS
- `OPT_MCU_OVERCLOCK_MHZ` - CPU frequency (default 168, empty = 120MHz)
- `PICO_BOARD` - Target board: `pico`, `pico_w`, `pico_debug_probe`, `pico2`

## Architecture

### Source layout (`src/`)
- `main.c` - Entry point, creates FreeRTOS tasks for USB, DAP, UART, etc.
- `probe.c` / `probe.pio` - SWD bit-banging via RP2040/RP2350 PIO hardware
- `sw_dp_pio.c` - SWD data phase implementation using PIO
- `cmsis-dap/` - CMSIS-DAP protocol server (v1/v2)
- `cdc/` - USB CDC interfaces: debug output (`cdc_debug`), target UART bridge (`cdc_uart`), SystemView (`cdc_sysview`)
- `msc/` - USB Mass Storage for UF2 drag-n-drop flashing
- `daplink-pico/` - Target flash algorithms (RP2040, RP2350, nRF52)
- `pico-sigrok/` - Logic analyzer / oscilloscope via CDC
- `net/` - USB network (NCM/ECM/RNDIS) to lwIP bridge
- `lib/` - Vendored libraries: DAPLink, SEGGER RTT, minIni
- `usb_descriptors.c` - USB device/interface descriptors (composite device)

### Key headers (`include/`)
- `picoprobe_config.h` - Central probe configuration and pin definitions
- `DAP_config.h` - CMSIS-DAP hardware abstraction
- `tusb_config.h` - TinyUSB configuration
- `FreeRTOSConfig.h` - RTOS tuning
- `boards/*.h` - Per-board pin assignments (pico.h, pico_w.h, pico2.h, pico_debug_probe.h)

### Design patterns
- **Compile-time feature selection**: All optional features use `OPT_*` preprocessor guards and CMake `option()` directives. Conditional compilation is pervasive.
- **PIO for SWD timing**: SWD signaling uses RP2040/RP2350 PIO state machines (`probe.pio`) for precise timing, not software bit-banging.
- **FreeRTOS concurrency**: USB stack, DAP server, UART bridge, and debug output run as separate FreeRTOS tasks.
- **Multi-board support**: Board-specific pin configs live in `include/boards/` and are selected via `TARGET_BOARD_*` defines derived from `PICO_BOARD`.
- **Runtime configuration**: Probe settings stored in flash via minIni, accessible through the debug CDC interface.
- **Tool auto-detection**: The DAP server detects connecting tools (OpenOCD, pyOCD, probe-rs) and adjusts packet parameters.

## No test framework

There is no unit test suite. Validation is done via CMSIS-DAP compliance tests in the `CMSIS_DAP/Firmware/Validation/` submodule and manual hardware testing.
