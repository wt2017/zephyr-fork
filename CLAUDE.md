# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Zephyr RTOS project - a scalable real-time operating system supporting multiple hardware architectures (ARM, x86, RISC-V, ARC, Xtensa, MIPS, SPARC), optimized for resource-constrained embedded devices.

## Build Commands

Zephyr uses `west` (Zephyr's meta-tool) for building and project management:

```bash
# Build a sample application for a specific board
west build -b <board> samples/hello_world

# Build with pristine (clean) build directory
west build -b <board> --pristine

# Build with additional CMake arguments
west build -b <board> -- -DCONFIG_DEBUG_OPTIMIZATIONS=y

# Flash to device
west flash

# List available boards
west boards
```

## Test Commands

Zephyr uses **Twister** as its test runner:

```bash
# Run all tests
./scripts/twister --all

# Run tests for specific platform
./scripts/twister --platform qemu_cortex_m3

# Run specific test suite
./scripts/twister -T tests/kernel/semaphore

# Run with verbose output
./scripts/twister -v -T <test_path>

# Run tests matching tags
./scripts/twister --all --tag kernel

# Run slow tests (disabled by default)
./scripts/twister --all --enable-slow
```

## Code Style

- **C Code**: Follow Linux kernel coding style with Zephyr-specific modifications:
  - Tabs are 8 characters
  - Line length max 100 columns
  - Use snake_case for code and variables
  - Braces required for all `if`, `else`, `for`, `while`, `do`, `switch` blocks
  - Use C89-style comments `/* */`, not `//`
  - Use `clang-format` with `.clang-format` configuration

- **Code checking**: Run `./scripts/checkpatch.pl` on patches

## Architecture Overview

```
zephyr/
├── arch/           # CPU architecture support (arm, riscv, x86, arc, xtensa, etc.)
├── kernel/         # Core RTOS kernel (scheduler, threads, IPC, memory management)
├── drivers/        # Device drivers (gpio, uart, spi, i2c, usb, etc.)
├── subsys/         # Subsystems (bluetooth, networking, logging, shell, fs, etc.)
├── include/zephyr/ # Public API headers
├── lib/            # Library code
├── samples/        # Example applications
├── tests/          # Test suites using ztest framework
├── boards/         # Board definitions and devicetree files
├── soc/            # System-on-Chip specific code
├── dts/            # Devicetree bindings and common definitions
├── cmake/          # CMake modules and toolchain definitions
├── scripts/        # Build scripts, twister, code generators
└── modules/        # External modules (hal, crypto, etc.) - managed by west
```

## Key Concepts

### Device Model
- All devices use `struct device` with unified API
- Devices are defined via `DEVICE_DT_INST_DEFINE()` or `DEVICE_DEFINE()` macros
- Initialization occurs in priority levels: `PRE_KERNEL_1`, `PRE_KERNEL_2`, `POST_KERNEL`, `APPLICATION`
- Device configuration via **Kconfig** (`.config`) and **Devicetree** (`.dts`)

### Initialization Levels
```
EARLY → PRE_KERNEL_1 → PRE_KERNEL_2 → POST_KERNEL → APPLICATION
```

### Testing with Ztest
```c
#include <zephyr/ztest.h>

ZTEST_SUITE(my_suite, NULL, NULL, NULL, NULL, NULL);

ZTEST(my_suite, test_example)
{
    zassert_equal(1, 1, "Basic math failed");
}
```

Test files require:
- `CMakeLists.txt` with `find_package(Zephyr REQUIRED)`
- `prj.conf` for test configuration
- `testcase.yaml` for twister metadata

### Devicetree
- Hardware description in `.dts` files
- Bindings in `dts/bindings/`
- Access via `DT_*` macros from `<zephyr/devicetree.h>`

### Kconfig
- Configuration system in `Kconfig*` files
- Options set via `prj.conf`, menuconfig, or CMake arguments

## West Workspace

Zephyr uses west for multi-repository management. The workspace structure:
```
zephyrproject/
├── .west/          # West configuration
├── zephyr/         # Main repository (this repo)
└── modules/        # External modules (hal_*, mbedtls, etc.)
```

Key west commands:
```bash
west update        # Update all projects to manifest revisions
west diff          # Show local changes across all projects
west forall -c <cmd>  # Run command in all projects
```

## Environment Setup

```bash
# Source environment (Linux/macOS)
source zephyr-env.sh

# Source environment (Windows)
zephyr-env.cmd
```

This sets `ZEPHYR_BASE` and adds scripts to PATH.

## Commit Guidelines

- Use DCO (Developer Certificate of Origin): commits require `Signed-off-by:` line
- Follow conventional commit format
- Run `./scripts/checkpatch.pl` before submitting
- Reference issues in commit messages when applicable
