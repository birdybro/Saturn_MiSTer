# Saturn MiSTer FPGA Logic Utilization Analysis

**Date:** 2026-04-01
**Target:** Cyclone V 5CSEBA6U23I7 (41,910 ALMs / 166,036 LEs)
**Current Utilization:** 98-99%

---

## Executive Summary

The Saturn core is critically close to the Cyclone V capacity ceiling. This analysis identifies optimization opportunities across all major subsystems, organized by estimated impact and implementation risk. The realistic savings range is **4,000-8,000 ALMs (10-19%)**, which would bring utilization down to the 80-89% range.

The largest opportunities are:
1. SH-2 instruction decode refactoring (duplicated across 2 CPUs)
2. Conditional compilation cleanup (removing unused feature paths)
3. SCSP channel logic serialization
4. VDP2 address calculation time-multiplexing
5. FX68K decode PLA conversion to block RAM

---

## 1. SH-2 CPU Subsystem (2 instances — savings are doubled)

The dual SH-2 architecture is the single largest consumer of logic. Every optimization here yields 2x savings.

### 1.1 Instruction Decode Function — CRITICAL

**File:** `rtl/SH/core/SH_pkg.sv`, lines 180-1288

The `Decode()` function is 1,108 lines of nested combinational case statements (82 case/endcase pairs) covering the full SH-2 instruction set. This is instantiated once per CPU with no sharing.

**Opportunities:**
- Restructure into hierarchical decode: first decode `IR[15:12]` (16 groups), then sub-decode within each group. Currently all levels are flattened.
- Remove `VER==0` (SH-1) decode paths if SH-1 compatibility is unnecessary (lines 83-84 show version-gated logic).
- Consider ROM-based decode: a 64K-entry lookup table would use ~2 M10K blocks but save hundreds of ALMs per instance.

**Estimated savings:** 500-1,500 ALMs (x2 = 1,000-3,000 total)

### 1.2 Duplicate Wait Signal Logic

**File:** `rtl/Saturn/Saturn.sv`, lines 507-508

```systemverilog
assign MSHWAIT_N = CWAIT_N & (MEM_WAIT_N | (MSHCS3_N & DRAMCE_N & ROMCE_N & SRAMCE_N));
assign SSHWAIT_N = CWAIT_N & (MEM_WAIT_N | (MSHCS3_N & DRAMCE_N & ROMCE_N & SRAMCE_N));
```

Identical logic computed twice. Both use master CPU chip selects (`MSHCS3_N`), suggesting this may also be a functional bug (slave should use its own chip selects).

**Estimated savings:** 50-75 ALMs

### 1.3 Clock Divider Duplication

**File:** `rtl/SH/SH7604/SH7604.sv`, lines 369-421

13 clock enable signals (`CLK2_CE` through `CLK8192_CE`) are generated independently in each SH7604 instance with identical divider logic. These could be generated once at the top level and shared.

**Estimated savings:** 200-300 ALMs

### 1.4 Cache LRU Logic

**File:** `rtl/SH/SH7604/CACHE.sv`, lines 74-108

The `WayFromLRU()` function uses a 6-way `casez` and `LRUSelect()` uses nested ternary operators. Both could be replaced with small ROM lookup tables (block RAM).

**Estimated savings:** 100-200 ALMs

### 1.5 Interrupt Priority Encoder

**File:** `rtl/SH/SH7604/INTC.sv`, lines 135-152

15 sequential if-else conditions for interrupt priority evaluation, all combinational with no early termination. A parameterized priority encoder or ROM lookup would be smaller.

**Estimated savings:** 150-250 ALMs

### 1.6 Disabled Feature Ports

**File:** `rtl/Saturn/Saturn.sv`, lines 342-478

Both SH7604 instances tie off many ports (UBC, SCI, external bus, DMA requests, UART). If the SH7604 module doesn't fully optimize these away, a trimmed variant without UBC/SCI/serial would save logic.

**Estimated savings:** 150-200 ALMs per instance (300-400 total)

---

## 2. VDP2 (Background/Scroll Processor)

### 2.1 Parallel Address Calculations — HIGH PRIORITY

**File:** `rtl/Saturn/VDP2/VDP2.sv`, lines 945-983

12 address calculations run in parallel every cycle:

```
NxPN_ADDR[0..3]  — 4 pattern name address calculations
NxCH_ADDR[0..3]  — 4 character address calculations
RxPN_ADDR[0..1]  — 2 rotation pattern name calculations
RxCH_ADDR[0..1]  — 2 rotation character calculations
```

Each calculation (defined in `VDP2_pkg.sv` lines 1760-1870) contains multiple wide multiplexers, XOR operations, and 19-bit addition chains (~180 LUTs each).

**Opportunity:** Time-multiplex: serialize 4 NBG address calculations across 4 cycles. The display pipeline can tolerate 2-4 cycle latency.

**Estimated savings:** 400-600 ALMs

### 2.2 Color Calculation Pipeline

**File:** `rtl/Saturn/VDP2/VDP2_pkg.sv`, lines 2376-2414

`ColorCalc()` and `ExtColorCalc()` perform three parallel 8-bit multiplications (`CA * RA + CB * RB`) without pipelining, instantiated for every display pixel.

**Opportunities:**
- Pipeline the multiply-accumulate stages (insert registers)
- Time-multiplex across R/G/B channels (process one per cycle)
- Use Cyclone V DSP blocks instead of LUT-based multipliers

**Estimated savings:** 200-300 ALMs

### 2.3 Rotation Matrix Multiplication

**File:** `rtl/Saturn/VDP2/VDP2_pkg.sv`, lines 2087-2091

`MultRC()` performs full 32x32-bit signed multiplication, extracting bits [47:16]. The actual data precision is lower than 32 bits; reducing to 24x24 would halve the multiplier cost.

**Estimated savings:** 100-200 ALMs

### 2.4 VRAM Access Dispatch

**File:** `rtl/Saturn/VDP2/VDP2.sv`, lines 1004-1100

8 nested if-else conditions for VRAM fetch arbitration (LS, LW, RPA, RCTA, RCTB, BACK, LN). Could use a priority encoder instead.

**Estimated savings:** 75-150 ALMs

### 2.5 Layer Priority Mixing

**File:** `rtl/Saturn/VDP2/VDP2.sv`, lines 845-883

4 identical priority selection blocks (A0, A1, B0, B1), each with 4 cascading if-statements. Could be a single parameterized function called 4 times.

**Estimated savings:** 50-100 ALMs

---

## 3. VDP1 (Sprite/Polygon Processor)

### 3.1 Gouraud Shading Color Interpolation

**File:** `rtl/Saturn/VDP1/VDP1_pkg.sv`, lines 346-391

`GouraudAdd()` performs saturating arithmetic on 5-bit color channels (3x for RGB). `ColorCalc()` below it is an 8-way case statement creating an 80+ LUT multiplexer.

**Opportunities:**
- Use DSP blocks for saturating add
- Reduce 8-way case to 3-bit encoded independent muxes

**Estimated savings:** 100-150 ALMs

### 3.2 Pixel Pipeline Register Width

**File:** `rtl/Saturn/VDP1/VDP1.sv`, lines 1602-1768

The draw pipeline maintains ~120 bits of state per stage (DRAW_X, DRAW_Y, DRAW_GHCOLOR as full ColorFP_t, DRAW_PAT, error terms). The `ColorFP_t` structure could use 8-bit fixed point instead of the full representation, saving 24 bits of latches.

**Estimated savings:** 75-100 ALMs

### 3.3 Framebuffer Address Decode

**File:** `rtl/vdp1_fb.sv`

Three separate memory regions each have independent address logic and write enable decode (duplicated 3x). A unified address decoder with address-based output selection would be smaller.

**Estimated savings:** 50-75 ALMs

---

## 4. SCSP (Sound Synthesizer — 32 Channels)

### 4.1 Per-Channel Register Arrays — HIGH PRIORITY

**File:** `rtl/Saturn/SCSP/SCSP.sv`, lines 235-244

32-element arrays like `SCR0_KB[32]` and `SCR0_KB_OLD[32]` are stored as flip-flops. Since the SCSP processes one slot per cycle serially, these should be in block RAM with single-port access.

**Estimated savings:** 400-500 ALMs

### 4.2 Envelope Generator Duplication

**File:** `rtl/Saturn/SCSP/SCSP.sv`, lines 731-800

Attack/decay volume calculations with arithmetic shifters are computed per-channel. Since processing is serial (one slot per cycle, confirmed at line 691-695), the envelope hardware can be fully time-multiplexed.

**Estimated savings:** 250-350 ALMs

### 4.3 Dual LFO per Channel

**File:** `rtl/Saturn/SCSP/SCSP.sv`, lines 330-343; `SCSP_pkg.sv`, lines 465-495

Two LFOs per channel (ALFO, PLFO) × 32 channels = 64 oscillators. A single shared LFO with slot-indexed RAM would suffice.

**Estimated savings:** 200-300 ALMs

### 4.4 Volume Calculation Multiplier

**File:** `rtl/Saturn/SCSP/SCSP_pkg.sv`, line 569

`VolCalc()` uses a 16x8 signed multiplication. If instantiated per-channel (rather than shared), this wastes DSP blocks or LUT multipliers.

**Estimated savings:** 100-150 ALMs

---

## 5. FX68K (Motorola 68000)

### 5.1 Microcode Address Decode PLA — HIGH PRIORITY

**File:** `rtl/FX68K/uaddrPla.sv`

The opcode-to-microaddress PLA is implemented as large combinational logic. A 64K-word × 17-bit ROM would use ~2-3 M10K blocks (of 553 available) and eliminate hundreds of ALMs of combinational decode.

**Estimated savings:** 300-400 ALMs

### 5.2 ALU Decode Consolidation

**File:** `rtl/FX68K/fx68kAlu.sv`, lines 532-844

Three separate lookup functions (`aluGetOp`, `rowDecoder`, `ccrTable`) could be consolidated into a single ROM per opcode class.

**Estimated savings:** 100-150 ALMs

---

## 6. SCU (System Control Unit)

### 6.1 DSP Multiplier Sharing

**File:** `rtl/Saturn/SCU/DSP.sv`, line 190

32x32-bit signed multiplier running continuously. Could potentially be time-shared with SCSP or VDP2 multipliers if bus scheduling permits.

**Estimated savings:** 50-100 ALMs (or 1-2 DSP blocks)

### 6.2 DMA State Machine Encoding

**File:** `rtl/Saturn/SCU/SCU.sv`, lines 322-344

20-state DMA FSM uses binary encoding (5 bits). One-hot encoding may improve timing but not area; the real opportunity is simplifying the 3-channel arbitration logic (lines 366-379).

**Estimated savings:** 75-100 ALMs

---

## 7. Memory Subsystem

### 7.1 SDRAM Controller Duplication — HIGH PRIORITY

**Files:** `rtl/sdram1.sv` (450 lines), `rtl/sdram2.sv` (371 lines)

These share extensive duplicated logic: initialization parameters, init FSM, control constants, command encoding, and address multiplexing. A single parameterized module would eliminate ~30% of the combined logic.

**Estimated savings:** 250-350 ALMs

### 7.2 DDRAM Cache Instances

**File:** `rtl/ddram.sv`, lines 126-196

9-10 separate cache instances (RAMH, RAML, VDP1VRAM, VDP1FB, CDRAM, CDBUF, CART, RAX, BIOS, BSRAM), each with independent address registers, dirty flags, and data holding registers. Parameterizing into a single cache module would reduce duplication. Additionally, RAMH uses 4-way LRU (expensive); reducing to 2-way would save further.

**Estimated savings:** 200-400 ALMs

---

## 8. Top-Level / MiSTer Wrapper

### 8.1 Conditional Compilation Bloat — HIGH PRIORITY

**File:** `Saturn.sv`, various locations

Multiple `ifdef` paths create parallel logic that Quartus may not fully optimize:

| Macro | Impact | Location |
|-------|--------|----------|
| `STV_BUILD` | STV I/O, cart, crypto chips | Lines 24, 93, 127, 536, 1136 |
| `MISTER_DUAL_SDRAM` | Full duplicate SDRAM paths | Lines 1444-1609 |
| `SATURN_FULL_FB` | 512x256 vs 352x256 FB | Lines 1640-1807 |
| `DEBUG` | Disables lightgun, crop | Various |

For a single target build (e.g., Saturn single-SDRAM), ensuring only one path is compiled saves significant logic from the unused ifdef branches.

**Estimated savings:** 300-500 ALMs (for a specific single-config build)

### 8.2 Lightgun Modules (When Unused)

**File:** `Saturn.sv`, lines 1204-1331

Two complete lightgun instances with crosshair overlay logic. If not needed, hardcoding `lg_p*_ena = 0` allows Quartus to optimize away the dependent logic chains.

**Estimated savings:** 200-250 ALMs

### 8.3 Open Bus Latch Logic

**File:** `rtl/Saturn/Saturn.sv`, lines 483-504

32-bit latch with per-byte selectable updates and redundant condition (`&MSHDQM_N_OLD & MSHDQM_N_OLD` tests the same value twice). Simplifying the update logic and fixing the redundant condition saves a small amount.

**Estimated savings:** 75-125 ALMs

### 8.4 CDO Bus Multiplexer Chain

**File:** `rtl/Saturn/Saturn.sv`, lines 511-517

4-way multiplexer on 32-bit bus combining multiple chip selects with OR logic. Refactoring to priority-encoded mux reduces gate depth.

**Estimated savings:** 100-150 ALMs

---

## 9. ADSP-2181 (Audio DSP)

### 9.1 Serial Port Duplication

**File:** `rtl/ADSP_21xx/ADSP_2181.sv`, lines 337-542

SPORT0 and SPORT1 are ~200 lines each of nearly identical logic (shift registers, clock dividers, bit counters). Extracting into a parameterized `SPORT_TRANSCEIVER` module would allow Quartus to better optimize shared structures.

**Estimated savings:** 100-150 ALMs

---

## Summary Table

| # | Subsystem | Opportunity | Est. ALM Savings | Priority | Risk |
|---|-----------|-------------|-----------------|----------|------|
| 1.1 | SH-2 (x2) | Instruction decode refactor/ROM | 1,000-3,000 | **CRITICAL** | Medium |
| 4.1 | SCSP | Channel registers to block RAM | 400-500 | **HIGH** | Low |
| 2.1 | VDP2 | Time-multiplex address calcs | 400-600 | **HIGH** | Medium |
| 5.1 | FX68K | Decode PLA to block RAM | 300-400 | **HIGH** | Low |
| 8.1 | Wrapper | Single-config build cleanup | 300-500 | **HIGH** | Low |
| 1.6 | SH-2 (x2) | Trim unused peripherals | 300-400 | HIGH | Low |
| 4.2 | SCSP | Serialize envelope generators | 250-350 | HIGH | Medium |
| 7.1 | SDRAM | Parameterize controllers | 250-350 | HIGH | Low |
| 8.2 | Wrapper | Remove lightgun (if unused) | 200-250 | MEDIUM | Low |
| 2.2 | VDP2 | Pipeline color calcs | 200-300 | MEDIUM | Medium |
| 4.3 | SCSP | Share LFO hardware | 200-300 | MEDIUM | Medium |
| 7.2 | DDRAM | Parameterize cache instances | 200-400 | MEDIUM | Medium |
| 1.3 | SH-2 | Share clock dividers | 200-300 | MEDIUM | Low |
| 1.5 | SH-2 | Priority encoder for INTC | 150-250 | MEDIUM | Low |
| 1.4 | SH-2 | Cache LRU to ROM | 100-200 | MEDIUM | Low |
| 2.3 | VDP2 | Reduce MultRC precision | 100-200 | MEDIUM | Medium |
| 3.1 | VDP1 | Gouraud shading simplify | 100-150 | MEDIUM | Medium |
| 5.2 | FX68K | ALU decode consolidation | 100-150 | MEDIUM | Low |
| 4.4 | SCSP | Share VolCalc multiplier | 100-150 | LOW | Low |
| 9.1 | ADSP | Parameterize SPORTs | 100-150 | LOW | Low |
| 8.3 | Top | Open bus latch simplify | 75-125 | LOW | Low |
| 8.4 | Top | CDO mux refactor | 100-150 | LOW | Low |
| 2.4 | VDP2 | VRAM dispatch priority enc. | 75-150 | LOW | Low |
| 6.2 | SCU | DMA FSM simplify | 75-100 | LOW | Low |
| 3.2 | VDP1 | Compress pipeline registers | 75-100 | LOW | Low |
| 1.2 | SH-2 | Fix duplicate wait logic | 50-75 | LOW | Low |
| 2.5 | VDP2 | Parameterize priority mixing | 50-100 | LOW | Low |
| 3.3 | VDP1 | Unify FB address decode | 50-75 | LOW | Low |
| 6.1 | SCU | Share DSP multiplier | 50-100 | LOW | High |

**Conservative total (high-confidence items):** ~4,000-6,000 ALMs (10-14%)
**Optimistic total (all items):** ~6,000-9,500 ALMs (14-23%)

---

## Recommended Action Plan

### Phase 1 — Low-Risk, High-Impact (target: 2,000-3,500 ALMs)

1. **Single-config build cleanup** (8.1): Ensure only one set of `ifdef` paths is active for the target build. Verify Quartus is actually eliminating dead code from inactive paths.
2. **SCSP registers to block RAM** (4.1): Move 32-element flip-flop arrays to M10K block RAM with slot-indexed access. The serial processing model already supports this.
3. **FX68K decode PLA to ROM** (5.1): Replace `uaddrPla.sv` combinational logic with a precomputed M10K lookup table.
4. **SDRAM controller parameterization** (7.1): Merge `sdram1.sv` and `sdram2.sv` into a single module with configuration parameters.
5. **Share SH-2 clock dividers** (1.3): Generate clock enables once in `Saturn.sv`, pass to both CPU instances.

### Phase 2 — Medium-Risk, High-Impact (target: 1,500-3,000 ALMs)

6. **SH-2 instruction decode refactor** (1.1): Restructure `Decode()` into hierarchical decode or partial ROM. This is the single highest-value change but requires careful verification against test ROMs.
7. **VDP2 address time-multiplexing** (2.1): Serialize 4 NBG address calculations across clock cycles. Requires confirming the display pipeline can absorb the latency.
8. **SCSP envelope/LFO serialization** (4.2, 4.3): Share envelope and LFO hardware across channels using slot-indexed RAM.

### Phase 3 — Polish (target: 500-1,500 ALMs)

9. Remaining items from the table above, prioritized by savings-to-effort ratio.

---

## Notes

- **Block RAM budget:** Cyclone V 5CSEBA6U23I7 has 553 M10K blocks (5.7 Mbit). Converting logic to ROM-based lookup tables trades abundant block RAM for scarce ALMs. Verify block RAM utilization before committing to ROM-based approaches.
- **DSP block budget:** 112 variable-precision DSP blocks available. Currently underutilized — multiplier-heavy operations (VDP2 rotation, SCSP volume, SCU DSP) should target these instead of LUT-based multiplication.
- **Timing impact:** Some optimizations (time-multiplexing, pipelining) add latency. Verify against Saturn test ROMs (sh2test, scutest) and commercial titles after each change.
- **Measurement:** Run `quartus_sh --flow compile` and check the Fitter report (`output_files/Saturn.fit.summary`) for actual ALM counts before and after each optimization. The estimates in this document are based on code analysis, not synthesis results.
