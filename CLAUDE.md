# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MiSTer FPGA core implementing the Sega Saturn console and ST-V (Sega Titan Video) arcade platform. Targets the Cyclone V FPGA (5CSEBA6U23I7) on the MiSTer DE10-Nano board. Status: WIP/Beta.

Hardware requirement: 128 MB primary SDRAM module; dual SDRAM recommended.

## Build System

Uses **Intel Quartus Prime 17.0.2 Standard Edition**. There is no Makefile; synthesis is driven by Quartus project files.

### Project Variants

| Project File | Macros | Pin TCL | Description |
|---|---|---|---|
| `Saturn.qsf` | — | `sys_analog.tcl` | Main Saturn core (single SDRAM) |
| `Saturn_DS.qsf` | — | `sys_dual_sdram.tcl` | Saturn with dual SDRAM |
| `Saturn_Q13.qsf` | `DEBUG=1` | `sys_dual_sdram.tcl` | Alternate FPGA variant |
| `STV.qsf` | `STV_BUILD=1` | `sys_analog.tcl` | ST-V arcade board |
| `STV_DS.qsf` | `STV_BUILD=1` | `sys_dual_sdram.tcl` | ST-V with dual SDRAM |

All variants also define `MISTER_DISABLE_ALSA=1` (except Q13). The `STV_BUILD` macro enables arcade-specific I/O and cartridge handling throughout the RTL. Dual-SDRAM variants use `sys/sys_dual_sdram.tcl` for pin assignments instead of `sys/sys_analog.tcl`.

### Building

Open the desired `.qpf` in Quartus and run full compilation, or from command line:
```
quartus_sh --flow compile Saturn
```
Output goes to `output_files/`. The build produces an `.rbf` file for MiSTer.

### Cleaning

Run `clean.bat` to remove all Quartus intermediate files (db, incremental_db, output_files, simulation directories, build_id.v, etc.).

### Important Build Rules

- **Do NOT add files via the Quartus IDE** — it will corrupt `Saturn.qsf`. Add HDL files manually to `files.qip`.
- `files.qip` is the central file list; it references sub-`.qip` files for each major subsystem (`rtl/FX68K/fx68k.qip`, `rtl/SH/SH7604/SH7604.qip`, `rtl/SH/SH7034/SH7034.qip`, `rtl/ADSP_21xx/ADSP_2181.qip`, `rtl/Saturn/Saturn.qip`).
- `sys/sys.tcl` is sourced by all QSF files for base pin assignments. A second TCL (`sys_analog.tcl` or `sys_dual_sdram.tcl`) is sourced depending on the variant.
- `sys/build_id.tcl` auto-generates `build_id.v` with timestamps during compilation.
- `Saturn.sdc` provides timing constraints (PLL clock derivation, multicycle paths for HQ2x scaler and `ascal`).

## Architecture

### Module Hierarchy

```
sys_top (sys/sys_top.v)              — MiSTer platform hardware abstraction
  └── emu (Saturn.sv)                — MiSTer API wrapper (HPS I/O, video, audio, OSD)
      └── Saturn (rtl/Saturn/Saturn.sv) — Saturn SoC top-level
          ├── 2x SH7604 (SH-2 CPUs)    — rtl/SH/SH7604/ (master + slave @ 28.6 MHz)
          ├── FX68K (68000 CPU)         — rtl/FX68K/ (sound processor)
          ├── SCSP                      — rtl/Saturn/SCSP/ (sound synthesis)
          ├── ADSP-2181                 — rtl/ADSP_21xx/ (audio DSP)
          ├── VDP1                      — rtl/Saturn/VDP1/ (sprite/polygon processor)
          ├── VDP2                      — rtl/Saturn/VDP2/ (background/scroll planes)
          ├── SCU                       — rtl/Saturn/SCU/ (DMA controller + DSP)
          ├── SMPC                      — rtl/Saturn/SMPC.sv (system manager, I/O)
          ├── CD                        — rtl/Saturn/CD/ (CD-ROM via SH7034 + YGR019)
          └── CART                      — rtl/Saturn/CART.sv (cartridge interface)
```

### Key Subsystem Details

- **SH-2 CPUs**: Shared execution core in `rtl/SH/core/` (`SH_core.sv`, `SH_pkg.sv`). Instruction decoding and pipeline types are defined in `SH_pkg.sv` (~1460 lines). The SH7604 wrapper (`rtl/SH/SH7604/`) adds peripherals (cache, bus controller, DMA, interrupts, divider, timers). SH7034 (`rtl/SH/SH7034/`) is a separate wrapper used for CD-ROM control.
- **VDP1** (rtl/Saturn/VDP1/): Sprite and polygon rendering engine. Framebuffer managed via `rtl/vdp1_fb.sv`.
- **VDP2** (rtl/Saturn/VDP2/): Multi-layer background/scroll processor. Largest single subsystem (~4260 lines).
- **SCSP** (rtl/Saturn/SCSP/): Yamaha sound synthesizer with 32 channels (~3140 lines).
- **SCU** (rtl/Saturn/SCU/): System Control Unit handling bus arbitration, DMA, and a dedicated DSP (`DSP.sv`) (~3050 lines).
- **FX68K** (rtl/FX68K/): Motorola 68000 core driving the sound subsystem. Uses microcode ROMs (`microrom.mem`, `nanorom.mem`). Self-contained with its own `.qip`.
- **ST-V extensions**: `rtl/Saturn/STV/` adds arcade-specific I/O, cartridge handling, and protection chips (5838, 5881). Conditionally compiled via `STV_BUILD` macro.

### Memory Architecture

- **SDRAM**: `rtl/sdram1.sv` (primary 128MB), `rtl/sdram2.sv` (secondary for dual-SDRAM configs)
- **DDR3**: `rtl/ddram.sv` interfaces with HPS DDR3 for CD image data and large transfers
- **Block RAM**: `rtl/bram.vhd` and `rtl/mlab.vhd` provide on-chip memory primitives
- **Firmware ROMs**: `.mif` files in `rtl/` contain BIOS and microcode (cdb105.mif, cdb106.mif, sh7034.mif)

### HDL Languages

Mix of SystemVerilog (`.sv`, primary), Verilog (`.v`), and VHDL (`.vhd`). New code should use SystemVerilog.

### Clock Architecture

- **Input**: 50 MHz `CLK_50M` from MiSTer board.
- **PLL** generates `clk_sys` (main system clock, ~114 MHz) and `clk_ram` (SDRAM clock). The PLL is dynamically reconfigured based on PAL/NTSC region and `SMPC_DOTSEL` (dot clock select).
- **Clock enables**: `SYS_CE_F` and `SYS_CE_R` gate logic at the correct subsystem rates.

### MiSTer Integration Layer

- `Saturn.sv` (root): The `emu` module — MiSTer API wrapper. Handles HPS I/O, PLL configuration, video output, audio mixing, OSD status bits, and ioctl transfers (BIOS, cartridge, save RAM). The 128-bit `status` register encodes all OSD-configurable options (region, cart type, video modes, input devices, etc.).
- `sys/` directory: Shared MiSTer framework (video scalers, audio output, HPS I/O, OSD, SD card). Generally not modified per-core. Key interface is `hps_io.sv` for ARM-FPGA communication.

### Input/Peripheral Handling

- `rtl/hps2pad.sv` — translates HPS gamepad data to Saturn controller signals
- `rtl/hps2cdd.sv` — HPS to CD-ROM data interface
- `rtl/lightgun.sv`, `rtl/ps2keyboard.sv`, `rtl/ps2mouse.sv` — peripheral emulation

## Testing

No formal testbench infrastructure. Testing is done on hardware with Saturn test ROMs (sh2test, scutest) and commercial games. Verilator linting has been applied for code quality (modules comply with Verilator linter).

## Design Documentation

Spreadsheets in the source tree provide register maps and protocol details:
- `rtl/Saturn/B-Bus.xlsx` — B-Bus protocol
- `rtl/Saturn/VDP2/VDP2.xlsx` — VDP2 registers
- `rtl/Saturn/SCU/SCU DSP.xlsx` — SCU DSP instructions
- `rtl/SH/SH opcodes.xlsx` — SH-2 instruction set
- `rtl/sdr.xlsx` — SDRAM timing
