# Optimization 4: Conditional Compilation Cleanup

**Subsystem:** Top-level wrapper (`Saturn.sv`) and core (`rtl/Saturn/Saturn.sv`)
**Estimated Savings:** 300-500 ALMs (for a focused single-config build strategy)
**Risk:** Low — Quartus already eliminates dead ifdef paths; this is about confirming that and finding residual waste

---

## Problem Statement

The Saturn core uses 5 conditional compilation macros across ~80 `ifdef`/`ifndef` blocks in the two main files. While Quartus should fully eliminate inactive paths during synthesis, the complexity of nested conditionals creates opportunities for:
1. Residual logic from signals declared outside ifdefs but only used inside them
2. Suboptimal synthesis due to large multiplexers that select between ifdef-gated paths
3. Confusion about which features are active in a given build

This analysis maps every conditional block, confirms Quartus elimination behavior, and identifies cases where the ifdef structure itself causes unnecessary logic.

---

## Build Variant Macro Configuration

From QSF analysis:

| Build | Defined Macros | Pin TCL | Active Features |
|-------|---------------|---------|-----------------|
| **Saturn.qsf** | `MISTER_DISABLE_ALSA` | `sys_analog.tcl` | CD drive, backup SRAM, lightgun, single SDRAM |
| **Saturn_DS.qsf** | `MISTER_DISABLE_ALSA` | `sys_dual_sdram.tcl` | CD drive, backup SRAM, lightgun, dual SDRAM |
| **Saturn_Q13.qsf** | `DEBUG` | `sys_dual_sdram.tcl` | Debug keyboard, no lightgun UI, no CoFi filter |
| **STV.qsf** | `MISTER_DISABLE_ALSA`, `STV_BUILD` | `sys_analog.tcl` | Arcade I/O, EEPROM, protection chips, no CD |
| **STV_DS.qsf** | `MISTER_DISABLE_ALSA`, `STV_BUILD` | `sys_dual_sdram.tcl` | Arcade I/O + dual SDRAM |

**Key observations:**
- `MISTER_FB` and `MISTER_FB_PALETTE` are **never defined** in any QSF — dead code
- `SATURN_FULL_FB` is **never defined** — the 352×256 framebuffer path is always active
- `MISTER_DUAL_SDRAM` is **not explicitly defined** in any QSF (it's defined via `sys_dual_sdram.tcl` being sourced, which sets up the SDRAM2 pins — but the Verilog macro must be set separately)

---

## Complete ifdef Block Map

### Saturn.sv (Root-level MiSTer Wrapper)

#### STV_BUILD Blocks (32 blocks)

| Lines | Type | Size | Content |
|-------|------|------|---------|
| 282-296 | ifndef/else | 14 | Saturn menu config vs STV BIOS select |
| 298-301 | ifndef | 4 | Backup RAM mount/save options |
| 319-324 | ifdef | 6 | STV CRT H/V offset controls |
| 327-345 | ifndef/else | 17 | Input device menu (Saturn) vs empty (STV) |
| 396-402 | ifndef/else | 7 | PS2 keyboard LED signals |
| 475-479 | ifndef/else | 5 | OSD menu visibility masks |
| 485-501 | ifndef/else | 17 | CD/boot download detection vs STV mode config |
| 503-508 | ifndef/else | 6 | Cart type from status vs STV mode hardcoded |
| 637-687 | ifndef/else | 41+9 | Region detection + joystick wiring |
| 714-716 | ifdef | 2 | STVIO_CS_N wire declaration |
| 894-896 | ifdef | 2 | Saturn module STVIO port |
| 998-1007 | ifdef | 10 | Saturn module RAX ports |
| 1041-1047 | ifndef/else | 7 | Audio output selection |
| **1049-1156** | **ifdef** | **108** | **STVIO module + EEPROM state machine** |
| **1160-1331** | **ifndef/else** | **205** | **Lightgun modules (2 players) + crosshair** |
| 1458-1466 | ifndef | 9 | CD RAM buffer allocation |
| 1493-1499 | ifndef | 7 | CD buffer read/write |
| 1516-1530 | ifdef | 15 | STV EEPROM + RAX ROM allocation |
| 1539-1549 | ifndef/else | 11 | Backup SRAM routing |
| 1611-1637 | ifdef/else + nested | 27 | Memory multiplexer (3-way nested STV + dual SDRAM) |
| **1810-1960** | **ifndef** | **151** | **Backup SRAM save/load system** |
| 2087-2100 | ifndef/else | 14 | Lightgun crosshair color blend |
| 2126-2146 | ifdef/else | 21 | CRT offset resync (STV) vs passthrough |

**STV_BUILD toggles ~700 lines** between Saturn and STV configurations.

#### MISTER_DUAL_SDRAM Blocks (6 blocks)

| Lines | Type | Size | Content |
|-------|------|------|---------|
| 141-153 | ifdef | 13 | SDRAM2 interface port declarations |
| 349-351 | ifndef | 3 | Timing preset menu (single SDRAM only) |
| 866-870 | ifdef/else | 5 | RAMH_SLOW parameter (0 for dual, 1 for single) |
| 1444-1454 | ifdef/else | 11 | RAMH bus control routing |
| **1582-1609** | **ifdef** | **28** | **sdram2 controller instantiation** |
| 1611-1637 | nested | — | Memory mux (nested within STV_BUILD blocks) |

**MISTER_DUAL_SDRAM toggles ~60 lines** plus the sdram2 module instantiation.

#### DEBUG Blocks (5 blocks)

| Lines | Type | Size | Content |
|-------|------|------|---------|
| 244-248 | ifndef/else | 5 | 216p crop enable |
| 1502-1512 | ifdef/else | 11 | Cart memory bus (DEBUG zeros it out) |
| **1696-1807** | **ifndef** | **113** | **External VDP1 FB paging controller** |
| **2038-2085** | **ifndef/else** | **48** | **Horizontal crop + CoFi color blend** |
| **2157-2202** | **ifdef** | **46** | **Keyboard debug interface** |

**DEBUG toggles ~223 lines** of video processing and debug interface.

#### MISTER_FB / SATURN_FULL_FB Blocks (4 blocks)

| Lines | Type | Size | Content |
|-------|------|------|---------|
| 57-84 | ifdef + nested | 28 | Framebuffer I/O ports (NEVER DEFINED — dead) |
| **1641-1691** | **ifdef/else** | **51** | **VDP1 FB RAM: 512×256 vs 352×256** |
| 1696-1808 | nested ifndef | 113 | External FB paging (only when !SATURN_FULL_FB) |

### rtl/Saturn/Saturn.sv (Core)

| Lines | Type | Macro | Size | Content |
|-------|------|-------|------|---------|
| 23-26 | ifdef | STV_BUILD | 2 | STVIO_CS_N output port |
| **93-111** | **ifndef** | **STV_BUILD** | **18** | **CD drive interface ports** |
| 113-119 | ifndef/else | STV_BUILD | 6 | Cart mode vs STV protection chip modes |
| 127-136 | ifdef | STV_BUILD | 9 | RAX memory/audio outputs |
| 138-140 | ifdef | STV_BUILD | 1 | STV switch input |
| 173-194 | ifdef | DEBUG | 22 | VDP1 debug signals + breakpoint FSM |
| 511-517 | ifdef | STV_BUILD | 7 | STVIO_DO in CDO bus mux |
| 535-544 | ifndef/else | STV_BUILD | 10 | CD RAM path vs direct addressing |
| 687-689 | ifdef | STV_BUILD | 3 | STVIO_CS_N address decode |
| 884-897 | ifndef/else | STV_BUILD | 14 | SCSP/SCPU reset logic |
| **994-1131** | **ifndef** | **STV_BUILD** | **138** | **CD subsystem: SH7034 + YGR019** |
| 1136-1189 | ifndef/else | STV_BUILD | 54 | CART vs STV_CART module instantiation |
| 797-801 | ifdef | DEBUG | 5 | VDP1 debug output ports |
| 946-948 | ifdef | STV_BUILD | 3 | SCSP STV switch input |

---

## Quartus Elimination Analysis

### Confirmed: Fully Eliminated Paths

Quartus RTL synthesis evaluates `ifdef` conditions during Verilog preprocessing (before synthesis). Inactive paths are **removed from the netlist entirely** — they generate zero logic gates, zero registers, and zero routing resources.

This means:
- **Saturn build:** STV_BUILD blocks (STVIO, EEPROM, 5838/5881 crypto, STV_CART, RAX) produce zero logic
- **STV build:** CD subsystem (SH7034, YGR019, hps2cdd), backup SRAM, lightgun produce zero logic
- **Non-DEBUG build:** Debug keyboard, VDP1 breakpoint FSM produce zero logic
- **Non-DUAL build:** sdram2 controller, SDRAM2 ports produce zero logic

**No residual logic from inactive ifdef paths.**

### Potential Issue: Nested Conditional Multiplexers

The memory data path at lines 1611-1637 has a complex 3-way nesting:

```systemverilog
`ifdef MISTER_DUAL_SDRAM
    `ifdef STV_BUILD
        assign MEM_DI = !RAMH_CS_N ? sdr2_do :
                         !STVIO_CS_N ? STVIO_DO :     // STV-specific
                         ... : raml_do;
    `else
        assign MEM_DI = !RAMH_CS_N ? sdr2_do : ... : raml_do;
    `endif
`else
    `ifdef STV_BUILD
        assign MEM_DI = !RAMH_CS_N ? ramh_do :
                         !STVIO_CS_N ? STVIO_DO :     // STV-specific
                         ... : raml_do;
    `else
        assign MEM_DI = !RAMH_CS_N ? ramh_do : ... : raml_do;
    `endif
`endif
```

For a standard Saturn build (neither STV nor DUAL), this resolves to:
```systemverilog
assign MEM_DI = !RAMH_CS_N ? ramh_do : ... : raml_do;
```

This is clean — no wasted logic. But the nested structure makes the code harder to maintain and increases the risk of bugs when modifying one path without updating the parallel paths.

---

## Actual Optimization Opportunities

Since Quartus eliminates dead ifdef paths, the real savings come from logic that is **active but unnecessary** in specific builds.

### Opportunity A: Lightgun Modules (200-250 ALMs)

**Lines 1160-1331** (Saturn.sv) — `ifndef STV_BUILD`

Two complete lightgun modules are instantiated for Player 1 and Player 2 in all Saturn builds, even though lightgun support is only relevant when the user selects it via the OSD menu (`status[18:15]==4'd1`).

**Current:** Both modules are always synthesized. The `lg_p1_ena`/`lg_p2_ena` signals gate the crosshair drawing at runtime, but the lightgun coordinate tracking logic, sync detection, and crosshair pixel generators remain in the netlist.

**Optimization options:**
1. **Runtime gating (minimal savings):** The synthesis tool may not optimize away gated logic if it can't prove the enable signals are constant.
2. **Compile-time disable:** Add a `LIGHTGUN_ENABLE` macro (default on) that can be set to 0 for builds that don't need lightgun support. This would save ~200-250 ALMs.
3. **Lazy instantiation:** Restructure the lightgun modules to share a single coordinate tracker with a time-multiplexed output (Player 1 on even frames, Player 2 on odd). This halves the logic while maintaining both-player support.

**Recommended:** Option 2 — add compile-time disable for space-constrained builds.

### Opportunity B: External VDP1 Framebuffer Paging Controller (100-150 ALMs)

**Lines 1696-1808** (Saturn.sv) — `ifndef SATURN_FULL_FB` and `ifndef DEBUG`

This 113-line state machine manages SDRAM access for VDP1 framebuffer writes that exceed the 352×256 on-chip buffer. It includes:
- 4-state arbiter (lines 1704-1796)
- Pending write tracking
- Priority handling for read/write conflicts
- Page address calculation

**When `SATURN_FULL_FB` is defined:** This controller is eliminated and replaced by the larger on-chip buffer (512×256). However, `SATURN_FULL_FB` is never defined in any current QSF.

**Trade-off:**
- Defining `SATURN_FULL_FB` eliminates the paging controller (~100-150 ALMs of logic) but uses more block RAM for the larger framebuffer (estimated 2-3 additional M10K blocks).
- If M10K blocks are available, this is a net logic savings.

**Recommended:** Evaluate block RAM utilization. If M10K is available, define `SATURN_FULL_FB` to trade block RAM for ALMs.

### Opportunity C: Backup SRAM Save/Load System (100-150 ALMs)

**Lines 1810-1960** (Saturn.sv) — `ifndef STV_BUILD`

The 151-line SD card save/load state machine is always present in Saturn builds. If backup SRAM is not needed (e.g., for testing or specific game compatibility), this could be made conditional.

**Recommended:** Low priority — this is core functionality that most users expect. Only consider for extreme space pressure.

### Opportunity D: CoFi Color Filter (50-75 ALMs)

**Lines 2038-2085** (Saturn.sv) — `ifndef DEBUG`

The Composite Filtering (CoFi) module blends adjacent pixels to simulate composite video color bleeding. This is a visual enhancement that some users prefer to disable.

**Current:** Always present in non-DEBUG builds.

**Optimization:** Make CoFi conditional with a `COFI_ENABLE` macro, or check if the runtime enable signal (from OSD status) allows Quartus to optimize away the datapath when disabled.

**Savings:** 50-75 ALMs
**Recommended:** Low priority unless other savings are insufficient.

### Opportunity E: Verify Macro Definitions Match Intent

**Finding:** `MISTER_DUAL_SDRAM` is referenced in 6 ifdef blocks in Saturn.sv, but it's not explicitly set as a `VERILOG_MACRO` in `Saturn_DS.qsf`. It may be implicitly defined by `sys/sys_dual_sdram.tcl`, or it may need to be added. If it's not properly defined, the dual-SDRAM code paths may not activate correctly in DS builds.

**Action:** Verify that `sys_dual_sdram.tcl` defines the `MISTER_DUAL_SDRAM` Verilog macro, or add it explicitly to `Saturn_DS.qsf` and `STV_DS.qsf`.

```tcl
# In Saturn_DS.qsf, confirm or add:
set_global_assignment -name VERILOG_MACRO "MISTER_DUAL_SDRAM=1"
```

---

## Summary of Active Optimizations

| Opportunity | Savings | Approach | Risk | Effort |
|-------------|---------|----------|------|--------|
| A: Lightgun compile-time disable | 200-250 ALMs | New `LIGHTGUN_ENABLE` macro | Low | Low |
| B: Full FB mode (trade RAM for ALMs) | 100-150 ALMs | Define `SATURN_FULL_FB` | Low | Low |
| C: Backup SRAM conditional | 100-150 ALMs | New macro (not recommended) | Medium | Low |
| D: CoFi conditional | 50-75 ALMs | New macro or verify runtime opt | Low | Low |
| E: Verify DUAL_SDRAM macro | 0 (correctness) | Check TCL/QSF | N/A | Low |
| **Total (A+B+D)** | **350-475 ALMs** | | | |

---

## Broader Recommendation: Build Configuration Matrix

Rather than ad-hoc macros, formalize a build configuration system:

| Config | Macros | Target Use |
|--------|--------|------------|
| **Saturn-Full** | (none extra) | Maximum compatibility, all features |
| **Saturn-Lean** | `LIGHTGUN_DISABLE`, `SATURN_FULL_FB` | Maximum logic savings |
| **Saturn-DS** | `MISTER_DUAL_SDRAM` | Dual SDRAM |
| **STV** | `STV_BUILD` | Arcade |
| **STV-DS** | `STV_BUILD`, `MISTER_DUAL_SDRAM` | Arcade + dual SDRAM |
| **Debug** | `DEBUG` | Development/testing |

The "Saturn-Lean" configuration would save 350-475 ALMs over "Saturn-Full" by disabling lightgun and enabling the larger on-chip framebuffer. This could be the difference between fitting and not fitting when other optimizations are applied.
