# Optimization 5: FX68K Decode PLA to Block RAM Conversion

**Subsystem:** FX68K (Motorola 68000 CPU — sound processor)
**Primary File:** `rtl/FX68K/uaddrPla.sv` (2,195 lines)
**Secondary Files:** `rtl/FX68K/fx68k.sv`, `rtl/FX68K/fx68kAlu.sv`
**Estimated Savings:** 300-400 ALMs (PLA only), up to 500 ALMs (including ALU decode)
**Risk:** Low — the PLA is a pure combinational lookup function with no sequential elements

---

## Problem Statement

The FX68K Motorola 68000 core uses a 2,195-line combinational PLA (`uaddrPla.sv`) to decode 16-bit opcodes into microcode entry addresses. This PLA contains 126 case/casez statements organized across 7 `always_comb` blocks, consuming an estimated 800-1,200 LUTs. Since the PLA is a pure function (no registered outputs, no state), it is an ideal candidate for conversion to a block RAM lookup table. The Cyclone V has abundant M10K blocks (553 available), and this conversion would trade 1-2 blocks for hundreds of ALMs.

---

## Current Implementation

### Module Interface

**File:** `rtl/FX68K/uaddrPla.sv`
**Module:** `pla_lined` (lines 44-2195)

| Port | Direction | Width | Purpose |
|------|-----------|-------|---------|
| `movEa` | input | 4 | Effective address mode for MOVE instructions |
| `col` | input | 4 | Column selector (0-11, microcode phase) |
| `opcode` | input | 16 | Full 16-bit M68K instruction |
| `lineBmap` | input | 16 | One-hot encode of `opcode[15:12]` |
| `palIll` | output | 1 | Illegal instruction flag |
| `plaA1` | output | 10 | Microcode address output A1 |
| `plaA2` | output | 10 | Microcode address output A2 |
| `plaA3` | output | 10 | Microcode address output A3 |

**Total output width: 31 bits** (3 × 10-bit addresses + 1 illegal flag)

### PLA Structure

The PLA is organized as 7 `always_comb` blocks:

| Block | Lines | Purpose | Cases |
|-------|-------|---------|-------|
| 1 | 70-105 | Simple line decode (lines 6, 7, A, F) | 2 |
| 2 | 86-141 | Line E (shifts) conditional | 4 |
| 3 | 144-168 | Line 4 special ops (TRAP, LINK, UNLK, etc.) | 2 |
| 4 | 178-547 | **Line 0 primary decode** | 44 |
| 5 | 548-899 | **Lines 1-D primary decode** | 72 |
| 6 | 900-2195 | Helper functions, onehotEncoder | 2 |
| **Total** | | | **126** |

### Decode Pattern

The PLA maps `{opcode, col, movEa}` to `{plaA1, plaA2, plaA3, palIll}`. The decode is organized by instruction line (`opcode[15:12]`), then sub-decoded by various opcode bit fields.

**Example output (lines 184-190):**
```
col=2: arA1='h2B9, arA23='h006, scA3='h299
col=3: arA1='h2B9, arA23='h21C, scA3='h299
col=4: arA1='h2B9, arA23='h103, scA3='h299
```

Each (opcode pattern, column) combination maps to three 10-bit microcode entry addresses plus an illegal flag. The actual number of unique output combinations is far less than the theoretical maximum — many opcodes share the same microcode routines.

### Sequential Elements

**None.** The PLA is 100% combinational:
- No `always_ff` blocks
- No registered outputs
- Zero-latency interface
- All outputs are direct functions of inputs

This is confirmed by inspection of all 2,195 lines — every block uses `always_comb` or continuous `assign`.

---

## Existing ROM Usage in FX68K

The FX68K already uses block RAM for microcode and nanocode storage:

### microrom (fx68k.sv lines 2454-2462)

```systemverilog
module uRom(input clk, input [UADDR_WIDTH-1:0] microAddr,
            output logic [UROM_WIDTH-1:0] microOutput);
    reg [UROM_WIDTH-1:0] uRam[UROM_DEPTH];
    initial $readmemb("microrom.mem", uRam);
    always_ff @(posedge clk) microOutput <= uRam[microAddr];
endmodule
```

- **Size:** 1,024 × 17 bits = 17,408 bits (~2 M10K blocks)
- **File:** `rtl/FX68K/microrom.mem` (1,024 lines of 17-bit binary)

### nanorom (fx68k.sv lines 2465-2473)

```systemverilog
module nanoRom(input clk, input [NADDR_WIDTH-1:0] nanoAddr,
              output logic [NANO_WIDTH-1:0] nanoOutput);
    reg [NANO_WIDTH-1:0] nRam[NANO_DEPTH];
    initial $readmemb("nanorom.mem", nRam);
    always_ff @(posedge clk) nanoOutput <= nRam[nanoAddr];
endmodule
```

- **Size:** 336 × 68 bits = 22,848 bits (~3 M10K blocks)
- **File:** `rtl/FX68K/nanorom.mem` (336 lines of 68-bit binary)

### microToNanoAddr (fx68k.sv lines 2476-2637)

This module maps 10-bit micro-addresses to 9-bit nano-addresses. It's implemented as combinational logic (~162 lines of case statements). This is another ROM conversion candidate but lower priority than the PLA.

**Current M10K usage by FX68K: ~5 blocks.** Adding 1-2 more for the PLA ROM is feasible.

---

## Microcode Execution Pipeline

Understanding the pipeline is critical for determining ROM latency requirements:

```
T1: Opcode fetch → IR (instruction register)
T2: uaddrDecode active (combinational PLA) → a1, a2, a3 available
T3: Conditional bits latched → address selection
T4: Final microAddr latched → uRom (microcode ROM) accessed
```

**Key insight:** The PLA output (a1, a2, a3) is consumed on T2 but the microcode ROM read happens on T4. There are 2 cycles of slack between PLA output and microcode consumption.

### uaddrDecode Wrapper (fx68k.sv line 342)

```systemverilog
uaddrDecode uaddrDecode(.opcode(Ir), .a1, .a2, .a3,
                        .isPriv, .isIllegal, .isLineA, .isLineF,
                        .lineBmap());
```

The `uaddrDecode` module (lines 1787-1869) wraps `pla_lined` and adds privilege checking and line A/F detection. The wrapper is small combinational logic (~80 lines).

---

## ROM Conversion Design

### Address Space Analysis

The PLA inputs are:
- `opcode[15:0]` — 16 bits (65,536 possible values)
- `col[3:0]` — 4 bits (12 used: 0-11)
- `movEa[3:0]` — 4 bits (only relevant for MOVE instructions, line 1-3)

Naively, this is 24 bits = 16M entries — far too large.

However, not all input bits are independent:
- `opcode[15:12]` selects the instruction "line" (16 groups)
- Within each line, only specific bit fields matter (typically `opcode[11:6]` or `opcode[8:6]` + `opcode[5:3]`)
- `movEa` is only used for lines 1-3 (MOVE instructions)

### Approach A: Compressed Two-Level ROM (Recommended)

**Level 1 — Opcode Classifier (64K × 8-bit ROM):**
- Input: `opcode[15:0]` (16 bits)
- Output: 8-bit instruction class ID (0-255)
- Size: 65,536 × 8 = 524,288 bits = ~57 M10K blocks

**This is too large.** We need a more compressed approach.

### Approach B: Hierarchical ROM with Line Partitioning

Partition by `opcode[15:12]` (line), then use per-line sub-ROMs:

**Per-line ROM sizes:**

| Line (IR[15:12]) | Relevant Bits | Entries | × 12 cols | Total |
|-------------------|---------------|---------|-----------|-------|
| 0x0 | IR[11:0] | 4,096 | × 12 | 49,152 |
| 0x1-0x3 | IR[11:6] + movEa | 64 × 16 | × 12 | 12,288 |
| 0x4 | IR[11:0] | 4,096 | × 12 | 49,152 |
| 0x5-0x6 | IR[11:6] | 64 | × 12 | 768 |
| 0x7 | IR[11:8] | 16 | × 12 | 192 |
| 0x8-0x9 | IR[11:6] | 64 | × 12 | 768 |
| 0xA-0xB | (none) | 1 | × 12 | 12 |
| 0xC | IR[11:6] | 64 | × 12 | 768 |
| 0xD | IR[11:6] | 64 | × 12 | 768 |
| 0xE | IR[11:6] | 64 | × 12 | 768 |
| 0xF | — | — | — | — |

This is still complex. A simpler approach:

### Approach C: Direct ROM with Bit-Field Compression (Recommended)

**Observation:** For any given line (`opcode[15:12]`), the PLA uses at most 10 additional opcode bits. Combined with `col[3:0]` (4 bits), the effective address is:

```
rom_addr = {opcode[15:12], col[3:0], opcode[11:6]}  // 14 bits
```

**ROM dimensions:** 16,384 × 32 bits = 524,288 bits = ~57 M10K blocks.

Still too large for a single ROM. But we can split:

**Split by line (16 small ROMs):**
- Each line ROM: `{col[3:0], opcode[11:6]}` = 10 bits = 1,024 entries × 32 bits
- Total: 16 × 1,024 × 32 = 524,288 bits — same total, but each ROM fits in 1 M10K block
- `opcode[15:12]` selects which ROM's output is used (16:1 mux on 32-bit bus)

**This uses 16 M10K blocks** — feasible but expensive (3% of total).

### Approach D: Hybrid ROM + Logic (Best Trade-off)

**Concept:** Keep `opcode[15:12]` as a combinational first-stage decode (16-way case), but replace the per-line sub-decodes with small ROMs.

The largest sub-decodes are lines 0x0 and 0x4 (each ~180 lines of case statements). Replacing just these two with ROMs saves the majority of LUTs while using only 2 M10K blocks.

**Per-line complexity analysis:**

| Line | Case Lines | Est. LUTs | ROM Candidate? |
|------|-----------|-----------|----------------|
| 0x0 | 370 | 250-350 | **YES** — largest |
| 0x1-0x3 | 60 | 40-60 | Maybe (MOVE is simple) |
| 0x4 | 250 | 180-250 | **YES** — second largest |
| 0x5 | 20 | 15-25 | No (too small) |
| 0x6 | 60 | 40-60 | Maybe |
| 0x7 | 8 | 5-10 | No |
| 0x8 | 70 | 50-70 | Maybe |
| 0xC | 140 | 100-140 | Yes |
| 0xE | 60 | 40-60 | Maybe |
| Others | <20 each | <15 each | No |

**Recommended targets:** Lines 0x0, 0x4, and 0xC (710 case lines → ~430-740 LUTs)

**ROM sizing for these 3 lines:**
- Line 0x0: `{col[3:0], opcode[11:0]}` — but opcode[11:0] is sparse. Using `{col[3:0], opcode[8:3]}` = 10 bits = 1K × 32 = 1 M10K
- Line 0x4: `{col[3:0], opcode[11:3]}` = 13 bits = 8K × 32 = 2 M10K
- Line 0xC: `{col[3:0], opcode[7:3]}` = 9 bits = 512 × 32 = 1 M10K

**Total: 4 M10K blocks for 430-740 LUT savings** (~215-370 ALMs)

---

## ALU Decode Optimization

### aluGetOp (fx68kAlu.sv lines 531-620)

Maps `{row[15:0], col[2:0], isCorf}` → 5-bit ALU operation code.

**Input:** 20 bits (16-bit one-hot row + 3-bit col + 1-bit isCorf)
**Output:** 5 bits
**Complexity:** 8 case statements, ~40 unique outputs
**LUT estimate:** 50-80 LUTs

**ROM conversion:** 
- The row is one-hot encoded (16 bits), so effective address is `{row_encoded[3:0], col[2:0], isCorf}` = 8 bits
- ROM: 256 × 5 bits = 1,280 bits (fits in 1 M10K with room to spare)
- **Savings:** 40-60 LUTs (~20-30 ALMs)

### rowDecoder (fx68kAlu.sv lines 629-760)

Maps `ird[15:0]` → 16-bit one-hot row + 3 flags.

**Input:** 16 bits
**Output:** 19 bits
**Complexity:** 12 case statements with nested conditions, ~50 decode branches
**LUT estimate:** 150-200 LUTs

**ROM conversion:**
- Full 16-bit address → 64K × 19 bits = too large
- Most decode only uses `ird[15:12]`, `ird[11:9]`, `ird[8:6]` = 10 bits
- ROM: 1,024 × 19 bits = 19,456 bits = ~2 M10K blocks
- **Savings:** 100-150 LUTs (~50-75 ALMs)
- **Complication:** Some cases test `ird[5:3]` and `ird[2:0]`, making compressed addressing harder

**Recommendation:** Keep as combinational logic unless ALM pressure is extreme. The decode has good structure for LUT mapping.

### ccrTable (fx68kAlu.sv lines 762-844)

Maps `{col[2:0], row[15:0], finish}` → 5-bit CCR update mask.

**Input:** 20 bits
**Output:** 5 bits
**Complexity:** 6 case statements, ~30 unique outputs
**LUT estimate:** 50-80 LUTs

**ROM conversion:** Same approach as aluGetOp — encode row to 4 bits, combine with col and finish.
- ROM: 256 × 5 bits = 1,280 bits
- **Savings:** 40-60 LUTs (~20-30 ALMs)

### microToNanoAddr (fx68k.sv lines 2476-2637)

Maps 10-bit micro-address → 9-bit nano-address.

**Input:** 10 bits
**Output:** 9 bits
**Complexity:** ~162 lines of case statements
**LUT estimate:** 80-120 LUTs

**ROM conversion:**
- ROM: 1,024 × 9 bits = 9,216 bits = 1 M10K block
- **Savings:** 60-100 LUTs (~30-50 ALMs)

---

## Implementation Plan

### Phase 1: PLA ROM Conversion (Highest ROI)

**Step 1: Generate ROM Contents**

Write a script (Python or similar) that:
1. Instantiates the current `pla_lined` logic as a simulation model
2. Sweeps all valid `{opcode, col, movEa}` combinations
3. Records the output `{plaA1, plaA2, plaA3, palIll}` for each
4. Produces `.mif` (Memory Initialization File) for Quartus

```python
# Pseudocode for ROM generation
for opcode in range(65536):
    for col in range(12):
        for movEa in range(16):
            result = simulate_pla(opcode, col, movEa)
            rom[compute_address(opcode, col, movEa)] = result
```

**Step 2: Create ROM Module**

```systemverilog
module pla_rom(
    input        clk,
    input [15:0] opcode,
    input  [3:0] col,
    input  [3:0] movEa,
    output       palIll,
    output [9:0] plaA1,
    output [9:0] plaA2,
    output [9:0] plaA3
);
    // Address computation (compressed)
    wire [13:0] rom_addr = {opcode[15:12], col[3:0], opcode[11:6]};
    
    reg [30:0] rom_data;
    reg [30:0] rom [0:16383];
    
    initial $readmemb("pla_rom.mem", rom);
    
    always_ff @(posedge clk)
        rom_data <= rom[rom_addr];
    
    assign {palIll, plaA3, plaA2, plaA1} = rom_data;
endmodule
```

**Step 3: Integrate into uaddrDecode**

Replace `pla_lined` instantiation with `pla_rom`. Add 1-cycle latency compensation in the microcode pipeline (the 2-cycle slack between T2 and T4 accommodates this).

**Important:** The ROM output is now registered (1 cycle latency), while the original PLA was combinational. This changes the pipeline timing — the `uaddrDecode` output arrives 1 cycle later. Review the microcode sequencer in `fx68k.sv` to confirm this is absorbed by the T2→T4 slack.

### Phase 2: ALU Decode ROM (Lower Priority)

Convert `aluGetOp` and `ccrTable` to a shared 256×10-bit ROM:

```systemverilog
module alu_decode_rom(
    input        clk,
    input  [3:0] row_enc,  // one-hot row encoded to 4 bits
    input  [2:0] col,
    input        isCorf,
    input        finish,
    output [4:0] aluOp,
    output [4:0] ccrMask
);
    wire [7:0] rom_addr = {row_enc, col, isCorf};
    reg [9:0] rom_data;
    reg [9:0] rom [0:255];
    initial $readmemb("alu_decode.mem", rom);
    always_ff @(posedge clk) rom_data <= rom[rom_addr];
    assign {ccrMask, aluOp} = rom_data;
endmodule
```

---

## Savings Summary

| Target | Current LUTs | ROM Size | M10K Blocks | ALM Savings |
|--------|-------------|----------|-------------|-------------|
| pla_lined (Approach D: 3 lines) | 430-740 | 3 ROMs | 4 | 215-370 |
| pla_lined (Approach C: full) | 800-1,200 | 16K×32 | 16 | 400-600 |
| aluGetOp | 50-80 | 256×5 | <1 | 20-30 |
| ccrTable | 50-80 | 256×5 | <1 | 20-30 |
| microToNanoAddr | 80-120 | 1K×9 | 1 | 30-50 |
| rowDecoder | 150-200 | 1K×19 | 2 | 50-75 |
| **Total (recommended)** | **610-1,020** | | **5-6** | **285-480** |
| **Total (aggressive)** | **1,130-1,680** | | **20** | **520-780** |

**Recommended path:** Approach D (hybrid) for the PLA + aluGetOp + ccrTable + microToNanoAddr.
Uses 5-6 M10K blocks, saves 285-480 ALMs.

---

## Verification Strategy

1. **Exhaustive ROM verification:** Simulate all 65,536 opcodes × 12 columns × 16 movEa values through both the original PLA and the ROM. Compare outputs bit-for-bit. This is a one-time verification step that can run in seconds.

2. **Pipeline timing:** Verify that the 1-cycle ROM latency doesn't break the microcode sequencer. Check T2→T3 transition for cases where the microcode address is used immediately.

3. **FX68K test suite:** Run the Motorola 68000 instruction test suite (if available) or test with Saturn games that exercise the sound processor heavily:
   - Radiant Silvergun (complex SCSP + 68K sound driver)
   - NiGHTS into Dreams (heavy sound processing)
   - Any game with CD-DA + sound effects simultaneously

4. **Corner cases:** Verify illegal instruction detection (`palIll`) for all undefined opcodes. The ROM must produce the same illegal flag as the combinational PLA for every input.

---

## Risk Assessment

**Why this is low risk:**
- The PLA is a pure function — same inputs always produce same outputs
- ROM conversion preserves this property exactly
- The existing microrom and nanorom are already ROM-based, proving the FX68K architecture supports ROM latency
- The FX68K is a mature, well-tested third-party core (Jorge Cwik's fx68k)
- The ROM contents can be verified exhaustively before synthesis

**Potential pitfall:**
- If the PLA output is used combinationally in the same cycle it's computed (before being registered), adding ROM latency would break timing. This must be verified by tracing the `a1`/`a2`/`a3` signals through `fx68k.sv` to confirm they're only consumed after being registered.
