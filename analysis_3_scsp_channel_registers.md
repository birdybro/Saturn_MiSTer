# Optimization 3: SCSP Channel Register Storage

**Subsystem:** SCSP (Sound Synthesizer — 32 Channels)
**Primary Files:** `rtl/Saturn/SCSP/SCSP.sv`, `rtl/Saturn/SCSP/SCSP_pkg.sv`
**Estimated Savings:** 400-500 ALMs
**Risk:** Low — the SCSP already uses block RAM for most per-slot state

---

## Problem Statement

The SCSP sound synthesizer processes 32 channels (slots) sequentially through a 7-stage pipeline. The initial logic utilization analysis identified per-channel register arrays stored as flip-flops as a conversion target. After detailed investigation, the SCSP is already well-optimized — nearly all per-slot state uses block RAM. However, two small flip-flop arrays remain, and more importantly, the arithmetic pipeline contains optimization opportunities in the envelope generator and LFO hardware that were obscured by the initial register-focused analysis.

---

## Current Storage Architecture

### Already in Block RAM (Well-Optimized)

The SCSP uses 6 dedicated RAM modules plus 12 slot control register RAMs, all implemented as Altera `altdpram`/`altsyncram` (block RAM):

| RAM Module | Instance Line | Width | Depth | Total Bits | Purpose |
|------------|---------------|-------|-------|------------|---------|
| `SCSP_KEY_RAM` | 354 | 2 | 32 | 64 | Key On/Off pulse state |
| `SCSP_LFO_RAM` | 358 | 18 | 32 | 576 | LFO divider + data per slot |
| `SCSP_SO_RAM` | 580 | 19 | 32 | 608 | Sample offset + loop state |
| `SCSP_PHASE_RAM` | 584 | 14 | 32 | 448 | Phase fractional accumulator |
| `SCSP_EVOL_RAM` | 876 | 12 | 32 | 384 | Envelope state + volume |
| `SCSP_STACK_RAM` (×2) | 2062-2063 | 16 | 32 | 1,024 | Direct output mixing (dual-buffered) |
| `SCSP_RAM_8X2` (×12) | 2012-2056 | 16 | 32 | 6,144 | SCR0-SCR8, SA, LSA, LEA registers |

**Total block RAM storage: 9,248 bits across 20 RAM instances**

All RAM modules are defined within `SCSP.sv` (lines 2156-2799) as parameterized wrappers around Altera memory primitives. Each provides single-port or dual-port access indexed by the 5-bit `SLOT` counter.

### Remaining Flip-Flop Arrays

Only two 32-element arrays remain as flip-flops:

| Array | Line | Width | Total Bits | Purpose |
|-------|------|-------|------------|---------|
| `SCR0_KB[32]` | 235 | 1 | 32 | Key On/Off bit (from SCR0 register write) |
| `SCR0_KB_OLD[32]` | 244 | 1 | 32 | Previous KB state (edge detection) |

**Total flip-flop storage: 64 bits**

These are accessed one slot at a time:
- `SCR0_KB[SLOT]` written at line 284-285 when the CPU writes the SCR0 register
- `SCR0_KB_OLD[SLOT]` written at line 298, read at lines 292, 295 for edge detection

---

## Pipeline Architecture

The SCSP processes one slot per 8 clock cycles through 7 operation stages:

```
CYCLE_NUM[2:0] cycles 0-7 per slot (lines 192-195)
CYCLE0_CE fires on even CYCLE_NUM (line 202)
CYCLE1_CE fires on odd CYCLE_NUM (line 203)
SLOT1_CE  fires when CYCLE_NUM[2:1]==2'b11 AND CYCLE1_CE (line 205)

Total: 32 slots × 8 cycles = 256 cycles per sample period
```

### Pipeline Stages

| Stage | Lines | Function | RAM Read | RAM Write |
|-------|-------|----------|----------|-----------|
| **OP1** | 233-358 | PLFO, Phase Generator, Key On/Off detection | LFO_RAM, KEY_RAM | KEY_RAM, LFO_RAM |
| **OP2** | 359-585 | Modulation read, Address/Phase calc | SO_RAM, PHASE_FRAC_RAM, SCR regs | SO_RAM, PHASE_FRAC_RAM |
| **OP3** | 586-647 | Sample fetch address generation | — | Memory read request |
| **OP4** | 649-877 | Interpolation, Envelope Generator, ALFO | EVOL_RAM, LFO_RAM | EVOL_RAM |
| **OP5** | 878-896 | Level calculation (EG + ALFO) | — | — |
| **OP6** | 898-927 | Total Level addition | SCR3 | — |
| **OP7** | 929-1043 | Volume calc, panning, direct output | SCR7, SCR8 | STACK_RAM |

Each stage passes its slot number forward through the pipeline (`OP1.SLOT` → `OP2.SLOT` → ... → `OP7.SLOT`), ensuring RAM accesses are always single-slot indexed.

---

## Optimization Opportunities

### Opportunity A: Convert SCR0_KB/SCR0_KB_OLD to Block RAM

**Current:** 64 flip-flops for two 32×1 arrays
**Proposed:** Merge into a single 32×2 block RAM instance (or extend KEY_RAM from 2 to 4 bits)

The existing `SCSP_KEY_RAM` (line 354) is a 32×2 block RAM storing `{KOFF, KON}` per slot. Extending it to 32×4 to include `{KB, KB_OLD, KOFF, KON}` would eliminate the flip-flop arrays with zero additional M10K usage (the existing RAM block has unused capacity).

**Implementation:**
```systemverilog
// Current KEY_RAM: 32 × 2 bits
// Proposed: 32 × 4 bits (still fits in same M10K block)
// Add KB and KB_OLD to the KEY_RAM data word
```

The access pattern is compatible: both `SCR0_KB` and `SCR0_KB_OLD` are read and written one slot at a time during the OP1 stage.

**Complication:** `SCR0_KB[SLOT]` is written asynchronously from the CPU bus (line 284-285) when the CPU writes the SCR0 register, while KEY_RAM is written synchronously from the pipeline. This creates a dual-write-port requirement. Solutions:
1. Use a true dual-port RAM (both already available via `altsyncram`)
2. Buffer the CPU write and merge during the slot's processing cycle
3. Keep `SCR0_KB` as flip-flops (only 32 FFs — may not be worth the complexity)

**Savings:** 32-64 flip-flops (~15-30 ALMs)
**Risk:** Low, but savings are small
**Effort:** Low-Medium

### Opportunity B: Envelope Generator Arithmetic Optimization

**Lines 731-800** (OP4 stage)

The envelope generator computes attack and decay volume updates:

```systemverilog
// Line 786
ATTACK_VOL_CALC = {1'b0, OP4_EVOL} + (ENV_STEP ? $signed($signed(~{1'b0,OP4_EVOL}) >>> SRAC[4:0]) : 11'd0);
// Line 787
DECAY_VOL_CALC = {1'b0, OP4_EVOL} + (ENV_STEP ? $signed(10'd16 >>> SRAC[4:0]) : 11'd0);
```

These arithmetic shifters (`>>>` by a 5-bit variable amount) synthesize to barrel shifters consuming significant LUT resources. Each barrel shifter on an 11-bit value requires ~55 LUTs.

**Optimization:** The `SRAC[4:0]` value comes from `EffRateCalc()` (`SCSP_pkg.sv` line 525), which produces rates from a limited set. If the actual rate values used map to a small set of shift amounts, a lookup table (case statement with fixed shifts) would be smaller than a full barrel shifter.

Alternatively, since only one envelope calculation is active per slot per cycle, the attack and decay paths can share a single barrel shifter with a mode select mux.

**Savings:** ~50-80 ALMs
**Risk:** Low — arithmetic refactor
**Effort:** Low

### Opportunity C: VolCalc Multiplier Optimization

**`SCSP_pkg.sv` line 569:**

```systemverilog
function bit signed [15:0] VolCalc(bit signed [15:0] WAVE, bit [9:0] LEVEL);
    bit [22:0] MULT;
    MULT = $signed(WAVE) * ({2'b01, ~LEVEL[5:0]});  // 16×8 signed multiply
    RES = $signed($signed(MULT[22:7]) >>> LEVEL[9:6]);
```

This performs a 16×8 signed multiplication followed by a variable-width arithmetic right shift. The multiplier can target a Cyclone V DSP block instead of LUT-based multiplication.

**Current synthesis behavior:** Quartus should automatically infer DSP block usage for this multiply, but the combination with the variable shift may prevent inference. Explicit DSP targeting via `(* multstyle = "dsp" *)` attribute or manual DSP instantiation would ensure this.

**Savings:** 80-120 ALMs (if currently in LUTs) or 0 (if already in DSP)
**Risk:** Low
**Effort:** Low — add synthesis attribute or check Fitter report

### Opportunity D: LFO Hardware Sharing

**Lines 330-343** (OP1 stage)

Each slot has two LFOs: ALFO (amplitude) and PLFO (pitch), computed at `SCSP_pkg.sv` lines 465-495.

```systemverilog
// LFO frequency divider update (line 330-343)
if (!OP1_LFO_DIV) begin
    NEW_LFO_DIV = LFOFreqDiv(OP1_SCR6.LFOF);
    NEW_LFO_DATA = OP1_LFO_DATA + 8'd1;
end else begin
    NEW_LFO_DIV = OP1_LFO_DIV - 10'd1;
    NEW_LFO_DATA = OP1_LFO_DATA;
end
```

The LFO state (divider counter + waveform data) is already stored in `SCSP_LFO_RAM` (32×18 block RAM, line 358). The ALFO and PLFO calculations (`ALFOCalc` line 465, `PLFOCalc` line 481) share the same `LFO_DATA` input but produce different outputs based on waveform and depth parameters.

**Current:** Both ALFO and PLFO are computed in the same cycle (OP1 for PLFO, OP4 for ALFO).

**Optimization:** Since ALFO and PLFO use the same `LFO_DATA` value, their waveform generation logic (triangle, sawtooth, square, noise — implemented as case statements) could share a single waveform generator with a time-multiplexed output. The generator would compute the waveform once and scale it differently for amplitude vs. pitch modulation.

**Savings:** ~50-80 ALMs
**Risk:** Medium — timing coupling between OP1 and OP4
**Effort:** Medium

### Opportunity E: DSP Effect Processor Optimization

**Lines 1044-2010** contain the SCSP's built-in DSP effect processor:

| RAM | Instance Lines | Width | Depth | Bits |
|-----|----------------|-------|-------|------|
| MPRO_RAM | ~3062 | 64 | 128 | 8,192 |
| TEMP_RAM | — | 24 | 128 | 3,072 |
| MEMS_RAM | — | 24 | 32 | 768 |
| MIXS_RAM | — | 20 | 16 | 320 (×2 dual-buffered) |
| COEF_RAM | — | 13 | 64 | 832 |
| MADRS_RAM | — | 16 | 32 | 512 |
| EFREG_RAM | — | 16 | 16 | 256 |

The DSP executes 128 microprogram steps per sample period, processing reverb/chorus/delay effects. The microprogram ALU includes a 24×13 multiplier (`SCSP_pkg.sv`).

**Optimization:** Ensure the 24×13 multiplier targets a DSP block. If the effect processing is not needed for certain games, a compile-time disable option could save the entire DSP (~200-300 ALMs), but this would break sound effects in games that use it.

**Savings:** 50-100 ALMs (DSP block targeting) or 200-300 ALMs (full disable, with functionality loss)
**Risk:** High for full disable; Low for DSP block targeting
**Effort:** Low for DSP targeting

---

## Revised Savings Estimate

The initial estimate of 400-500 ALMs was based on the assumption that significant per-channel state was in flip-flops. The actual flip-flop storage is only 64 bits. The revised breakdown:

| Optimization | Savings | Risk | Effort |
|-------------|---------|------|--------|
| A: SCR0_KB/KB_OLD to RAM | 15-30 ALMs | Low | Low-Medium |
| B: Envelope barrel shifter sharing | 50-80 ALMs | Low | Low |
| C: VolCalc DSP block targeting | 0-120 ALMs | Low | Low |
| D: LFO waveform sharing | 50-80 ALMs | Medium | Medium |
| E: DSP multiplier targeting | 50-100 ALMs | Low | Low |
| **Total** | **165-410 ALMs** | | |

The savings are lower than initially estimated because the SCSP is already well-architected for FPGA — the serial slot processing with block RAM storage is exactly the right pattern. The remaining opportunities are in arithmetic optimization and DSP block inference rather than storage conversion.

---

## Key Finding

**The SCSP is a model of good FPGA design for this core.** Its architecture — serial slot processing through a pipelined datapath with block RAM for all per-slot state — is precisely what other subsystems (particularly the SH-2 decode and VDP2 address calculations) should aspire to. The 20 block RAM instances are well-justified and leave the remaining logic budget focused on the arithmetic pipeline.

The most impactful optimization here is ensuring Quartus places the multipliers (VolCalc 16×8 and DSP 24×13) into hardware DSP blocks rather than consuming LUTs. This should be verified by examining the Quartus Fitter report:

```
quartus_sh --flow compile Saturn
# Then check: output_files/Saturn.fit.summary
# Look for: "DSP block 18-bit elements" utilization
```

If DSP blocks are underutilized while multipliers are in LUTs, adding synthesis attributes is a zero-risk, high-reward fix.
