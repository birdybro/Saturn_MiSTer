# Optimization 1: SH-2 Instruction Decode Refactoring

**Subsystem:** SH-2 CPU (dual instances — Master + Slave)
**Primary File:** `rtl/SH/core/SH_pkg.sv`, lines 180-1288
**Consumer:** `rtl/SH/core/SH_core.sv`, lines 282-285
**Estimated Savings:** 1,000-3,000 ALMs (depends on approach)
**Risk:** Medium — requires validation against sh2test, scutest, and commercial titles

---

## Problem Statement

The SH-2 instruction decoder is a 1,108-line purely combinational function (`Decode()`) containing 41 nested case statements across 3 levels of hierarchy. It produces a 79-bit packed struct (`DecInstr_t`) from a 16-bit instruction register. This function is instantiated **four times** in the Saturn core (twice per CPU, two CPUs), making it the single largest source of combinational logic in the design.

---

## Current Implementation

### The Decode Function

**Location:** `rtl/SH/core/SH_pkg.sv`, lines 180-1288
**Signature:** `function DecInstr_t Decode(bit [15:0] IR, bit [2:0] STATE, bit BC, bit VER)`

The function takes four inputs:
- `IR[15:0]` — the instruction register (primary decode input)
- `STATE[2:0]` — multi-cycle instruction state counter (0-7)
- `BC` — branch condition (true/false, from T flag evaluation)
- `VER` — CPU version (0 = SH-1, 1 = SH-2; always 1 in Saturn)

### Output Structure — DecInstr_t (79 bits)

Defined at `SH_pkg.sv` lines 135-155:

| Field | Type | Bits | Purpose |
|-------|------|------|---------|
| `DP` | `Datapath_t` | 11 | Register source select, PC mode, bypass controls |
| `IMMT` | `IMMType_t` | 3 | Immediate type selector |
| `ALU` | `ALU_t` | 12 | ALU source A/B, operation, condition, compare |
| `MEM` | `Mem_t` | 8 | Memory address source, write data source, size, R/W |
| `RA` | `Reg_t` | 7 | Register A: number (5), read, write |
| `RB` | `Reg_t` | 7 | Register B: number (5), read, write |
| `R0R` | bit | 1 | R0 read flag |
| `PCW` | bit | 1 | PC write |
| `TAS` | bit | 1 | Test-and-set |
| `SLP` | bit | 1 | Sleep |
| `IBI` | bit | 1 | Interrupt block inhibit |
| `IACP` | bit | 1 | Interrupt accept |
| `VECR` | bit | 1 | Vector read |
| `ILI` | bit | 1 | Illegal instruction |
| `CTRL` | `Control_t` | 6 | Control write, source, source select |
| `MAC` | `MAC_t` | 8 | MAC operation select (multiply-accumulate) |
| `BR` | `Branch_t` | 6 | Branch: immediate, type, delay, condition valid, subroutine |
| `LST` | bit[2:0] | 3 | Last state for multi-cycle instructions |
| **Total** | | **79** | |

### Case Statement Structure

The top-level decode is organized by `IR[15:12]` (16 instruction groups):

| IR[15:12] | Lines | Instruction Family | States |
|-----------|-------|--------------------|--------|
| 0x0 | 191-423 | Register/Control: STC, BSRF, BRAF, MUL.L, RTS, SLEEP, RTE, MAC.L | 0-4 |
| 0x1 | 425-432 | MOV.L store (displacement) | 0 |
| 0x2 | 434-499 | ALU register: MOV, TST, AND, XOR, OR | 0 |
| 0x3 | 501-557 | Arithmetic: CMP variants, SUB, ADD | 0 |
| 0x4 | 559-807 | Shift/Stack/MAC: SHLL, STS.L, LDC.L, TAS.B, MAC.W | 0-3 |
| 0x5 | 809-816 | MOV.L load (displacement) | 0 |
| 0x6 | 818-875 | Data transfer: MOV, SWAP, NEG, EXTU | 0 |
| 0x7 | 877-882 | ADD immediate | 0 |
| 0x8 | 884-954 | R0 displacement: MOV R0, BT, BF, BT/S, BF/S | 0-1 |
| 0x9 | 956-965 | MOV.W PC-relative | 0-1 |
| 0xA | 967-975 | BRA | 0-1 |
| 0xB | 977-985 | BSR | 0-1 |
| 0xC | 987-1124 | GBR/Exception: MOV GBR, TRAPA, TST.B | 0-2 |
| 0xD | 1126-1130 | MOV.L PC-relative | 0-1 |
| 0xE | 1132-1136 | MOV immediate | 0 |
| 0xF | 1138-1282 | Exceptions: Reset, Interrupt, Illegal | 0-7 |

Nesting depth reaches 3 levels: `IR[15:12]` → `IR[3:0]`/`IR[7:4]` → `STATE[2:0]`.

### Multi-Cycle Instructions

18 instruction families use `case(STATE)` for multi-cycle sequencing. The maximum state value is 7 (interrupt exception, 8 total cycles). Multi-cycle state logic accounts for approximately 500 lines (45% of the function).

### VER-Gated Instructions (SH-2 Extensions)

Six instruction families are gated by `VER==1` (lines 214, 245, 397, 521, 721, 932):
- BSRF, BRAF (branch to far subroutine)
- MUL.L (multiply long)
- MAC.L (multiply-accumulate long)
- DMULU.L, DMULS.L (double multiply)
- DT (decrement and test)
- BT/S, BF/S (delayed conditional branch)

Since both CPUs in Saturn use `VER=1`, these paths are always active. However, when `VER=0`, these decode to illegal instruction — the synthesis tool should optimize away the VER=0 paths, but this should be verified.

---

## Why Four Instantiations?

### SH_core.sv (lines 282-285)

```systemverilog
assign ID_DECI_RAW = Decode(DEC_IR, STATE, 1'b0, VER);      // line 282
// ...
assign ID_DECI     = Decode(DEC_IR, STATE, BR_COND_DEC, VER); // line 285
```

Both calls decode the **same IR and STATE**. The only difference is the `BC` (branch condition) parameter:
- `ID_DECI_RAW` uses `BC=0` (unconditional decode)
- `ID_DECI` uses `BC=BR_COND_DEC` (actual branch condition from T flag)

For approximately 85% of instructions (all non-branch), `BC` is unused and both decoders produce **identical outputs**. Only BT, BF, BT/S, and BF/S instructions use the BC parameter.

Since there are 2 CPUs (MSH and SSH, instantiated at `Saturn.sv` lines 342 and 412), the total is:

```
2 CPUs × 2 Decode calls/CPU = 4 Decode() instances
4 × 79 output bits = 316 bits of decode logic
```

### Pipeline Context

The decode result is clocked into a pipeline register at `SH_core.sv` line 330:
```systemverilog
PIPE.EX.DI <= ID_DECI;
```

There is a register boundary between decode and execute, so adding one cycle of decode latency (for ROM access) would require pipeline adjustment but is architecturally feasible.

---

## Optimization Approaches

### Approach A: Eliminate Dual Decode (40-50% savings per CPU)

**Concept:** Compute a single decode for all instructions, then patch only the branch-affected fields based on BC.

The BC parameter only affects the `BR.BCV` (branch condition valid) and `BR.BD` (branch delay) fields in the output. A single full decode plus a small BC-dependent correction mux would eliminate one entire 79-bit decoder per CPU.

**Implementation:**
```systemverilog
// Single decode
wire DecInstr_t deci_base = Decode(DEC_IR, STATE, 1'b0, VER);

// Patch branch fields only when BC matters (BT, BF, BT/S, BF/S)
wire DecInstr_t deci_final;
assign deci_final = deci_base;
assign deci_final.BR.BCV = is_conditional_branch ? BR_COND_DEC : deci_base.BR.BCV;
```

**Savings:** ~500-750 ALMs per CPU (1,000-1,500 total)
**Risk:** Low — straightforward refactor, easily verified
**Effort:** Low

### Approach B: Hierarchical Decode Restructure

**Concept:** Break the monolithic 1,108-line function into a two-stage hierarchy:

1. **Stage 1:** Decode `IR[15:12]` to select instruction group (16-way mux, ~50 ALMs)
2. **Stage 2:** Per-group sub-decoders that only examine relevant IR bits

Currently all 41 case statements are flattened, forcing the synthesis tool to find sharing opportunities across unrelated instruction groups. A hierarchical structure gives the tool better optimization boundaries.

**Savings:** 200-500 ALMs per instance (difficult to estimate without synthesis)
**Risk:** Low — same logic, different structure
**Effort:** Medium

### Approach C: ROM-Based Decode

**Concept:** Replace the combinational decode with a block RAM lookup table.

**ROM dimensions:**
- Address: `{IR[15:0], STATE[2:0], BC}` = 20 bits → 1M entries (too large)
- Compressed: `{IR[15:12], IR[11:0] compressed, STATE[2:0]}` with secondary table

A practical approach uses a two-level ROM:
1. **Primary ROM:** `IR[15:0]` → 8-bit instruction class + compressed fields (64K × 8 = 512 Kbit, ~56 M10K blocks)
2. **Secondary ROM:** `{class, STATE, BC}` → 79-bit DecInstr_t (2K × 79 = 158 Kbit, ~17 M10K blocks)

**Total block RAM:** ~73 M10K blocks (of 553 available = 13%)

This is feasible but expensive in block RAM. A more practical variant:

**Single-level compressed ROM:**
- Many instructions share the same decode output for STATE=0
- Only ~200-300 unique decode outputs exist across all IR values
- An 8K × 79-bit ROM (using {IR[15:12], IR[11:6], STATE} as address) would cover most cases with a small combinational fallback for irregular encodings

**Savings:** 600-1,200 ALMs per instance (2,400-4,800 total for 4 instances)
**Risk:** Medium — requires exhaustive verification, ROM initialization generation
**Effort:** High — need to generate ROM contents from current logic, verify all 65536 opcodes × 8 states

### Approach D: Trim Unused Peripherals in SH7604

**Concept:** The SH7604 wrapper (`rtl/SH/SH7604/SH7604.sv`) includes peripheral modules that are disabled via port tie-offs in `Saturn.sv` (lines 342-478):

```systemverilog
.UBC_DISABLE(1), .SCI_DISABLE(1), .BUS_SIZE_BYTE_DISABLE(1)
.DREQ0(1'b1), .DREQ1(1'b1)   // DMA requests disabled
.FTCI(1'b1)                    // Free-running timer input tied
.RXD(1'b1), .TXD()             // UART not connected
.SCKO(), .SCKI(1'b1)           // SPI not connected
```

If the synthesis tool doesn't fully optimize these away, creating a trimmed SH7604 variant without UBC, SCI, and unused DMA/timer logic would save area.

**Savings:** 150-200 ALMs per CPU (300-400 total)
**Risk:** Low — removing unused features
**Effort:** Medium — requires refactoring SH7604 module

---

## Recommended Implementation Order

| Step | Approach | Savings | Effort | Risk |
|------|----------|---------|--------|------|
| 1 | A: Eliminate dual decode | 1,000-1,500 ALMs | Low | Low |
| 2 | D: Trim SH7604 peripherals | 300-400 ALMs | Medium | Low |
| 3 | B: Hierarchical restructure | 400-1,000 ALMs | Medium | Low |
| 4 | C: ROM-based decode | 1,000-2,000 ALMs additional | High | Medium |

**Phase 1 alone (steps 1-2)** yields 1,300-1,900 ALMs with low risk.
**All phases** could yield 2,700-4,900 ALMs but require significant verification effort.

---

## Verification Strategy

1. **Functional equivalence:** Generate a testbench that sweeps all 65,536 opcodes × 8 states × 2 BC values and compares original vs. modified decode outputs bit-for-bit.
2. **Hardware validation:** Run sh2test and scutest ROMs on MiSTer hardware — these exercise SH-2 instruction decode and exception handling.
3. **Commercial game regression:** Test titles known to stress SH-2 edge cases (e.g., Burning Rangers for dual-CPU synchronization, Panzer Dragoon Saga for interrupt-heavy workloads).

---

## Critical Path Impact

The current decode has a combinational depth of 6-7 LUT levels for non-branch instructions and 8-9 levels for the branch condition feedback path (`IR → Decode #1 → BR_COND_DEC → Decode #2`). Eliminating the dual decode (Approach A) removes the longest critical path entirely, potentially improving Fmax by 5-10%.
