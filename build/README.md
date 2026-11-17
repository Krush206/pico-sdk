# Compiling and Linking an RP2040 Program

This directory contains the two bare commands used to build an RP2040 program
against the Raspberry Pi Pico SDK, using the ARM cross compiler.

Set `PICO_SDK_PATH` to match your installation before running the commands below:

```sh
export PICO_SDK_PATH=/path/to/pico-sdk  # e.g. /opt/pico-sdk-2.3.1
```

Make sure the ARM cross compiler tools (`arm-none-eabi-gcc`, `arm-none-eabi-ld`,
`arm-none-eabi-objdump`, `arm-none-eabi-size`) are on your `PATH`.

The flow is two steps:

1. **Compile** each `.c`/`.S` source file into an object file (`.o`).
2. **Link** the object files together with the SDK's second-stage bootloader
   (`.boot2`) into an ELF executable.

---

## Step 1 — Compile

```sh
arm-none-eabi-gcc \
  -mcpu=cortex-m0plus -mthumb -mfloat-abi=soft -ftls-model=local-exec -O2 -g -c test.c
```

| Flag | Meaning |
|------|---------|
| `-mcpu=cortex-m0plus` | Target the RP2040's Cortex-M0+ core |
| `-mthumb` | Use Thumb instruction set |
| `-mfloat-abi=soft` | Software floating point (no FPU on Cortex-M0+) |
| `-ftls-model=local-exec` | Thread-local storage model for embedded |
| `-O2` | Optimisation level 2 |
| `-g` | Emit debug information |
| `-c` | Compile only, do not link (produce `test.o`) |

Run this once per source file:

```sh
arm-none-eabi-gcc -mcpu=cortex-m0plus -mthumb -mfloat-abi=soft -ftls-model=local-exec -O2 -g -c foo.c
arm-none-eabi-gcc -mcpu=cortex-m0plus -mthumb -mfloat-abi=soft -ftls-model=local-exec -O2 -g -c bar.c
```

---

## Step 2 — Link

Run from the build directory:

```sh
arm-none-eabi-ld \
  -L${PICO_SDK_PATH}/src/rp2040/pico_platform/script_include \
  -L${PICO_SDK_PATH}/src/rp2040/pico_platform \
  -L${PICO_SDK_PATH}/src/rp2_common/pico_standard_link/script_include \
  -L$PWD/src/rp2_common/pico_standard_link \
  -T${PICO_SDK_PATH}/src/rp2040/pico_platform/script_include/default_locations.ld \
  -T${PICO_SDK_PATH}/src/rp2040/pico_platform/memmap_default.ld \
  -e 0x10000000 \
  -o test.elf \
  src/rp2040/boot_stage2/CMakeFiles/bs2_default_library.dir/bs2_default_padded_checksummed.S.o \
  test.o
```

| Flag / argument | Meaning |
|-----------------|---------|
| `-L <dir>` | Add a directory to the linker script include search path (used by `INCLUDE` directives inside `.ld` files) |
| `-T <file>` | Use this linker script to control memory layout and section placement |
| `-e 0x10000000` | Set the ELF entry point to the start of flash (`0x10000000`) |
| `-o test.elf` | Output ELF executable |
| `bs2_default_padded_checksummed.S.o` | The SDK's second-stage bootloader (`.boot2`), exactly 256 bytes, checksummed |
| `test.o` | Your compiled application object file |

### Important: `.boot2` must come first

The RP2040 ROM bootloader expects the second-stage bootloader (`.boot2`) to be the
very first section in the image, at flash offset `0x10000000`. That is why:

1. The boot_stage2 object is listed **before** your object files on the `ld` command line.
2. The linker include paths put the **rp2040** `script_include` directory **before**
   **rp2_common**, so the SDK picks the `sections_default_text.incl` variant that
   emits `.boot2` ahead of `.text`.

If `.text` is placed before `.boot2`, the bootrom will reject the image.

### Linker scripts used

* `default_locations.ld` — defines memory origin/length defaults (RAM, flash, scratch, XIP, …)
* `memmap_default.ld` — pulls in `pico_flash_region.ld` (the flash memory region) and the default section layout

---

## Inspecting the result

The result is `test.elf`. Inspect it with:

```sh
arm-none-eabi-objdump -h test.elf
arm-none-eabi-size test.elf
```

You should see `.boot2` at `0x10000000` with size `0x100` (256 bytes), followed by `.text`.

---

## Producing a UF2

To flash via drag-and-drop, convert the ELF to a UF2 with `picotool` or `elf2uf2`:

```sh
picotool load -v test.elf
picotool reboot
```

or produce `test.uf2` and copy it onto the board while holding BOOTSEL.
