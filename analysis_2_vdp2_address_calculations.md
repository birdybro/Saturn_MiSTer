# Optimization 2: VDP2 Address Calculation Time-Multiplexing

**Subsystem:** VDP2 (Background/Scroll Processor)
**Primary Files:** `rtl/Saturn/VDP2/VDP2_pkg.sv` (function definitions), `rtl/Saturn/VDP2/VDP2.sv` (instantiation)
**Estimated Savings:** 400-600 ALMs
**Risk:** Medium — must preserve display pipeline timing

---

## Problem Statement

The VDP2 computes 12 VRAM address calculations in parallel every cycle across 7 combinational functions. Each function contains multiple wide multiplexers, adders, and case statements. The address logic totals an estimated 1,400-1,600 LUTs. Since not all 12 addresses are needed simultaneously (VRAM has only 4 ports), time-multiplexing or hardware sharing could significantly reduce this.

---

## Current Implementation

### Address Functions Overview

Seven address calculation functions are defined in `VDP2_pkg.sv`:

| Function | Lines | Inputs | Output | Est. LUTs | Purpose |
|----------|-------|--------|--------|-----------|---------|
| `NxPNAddr` | 1760-1784 | 11 params, 55 bits | 19 bits | 120-140 | Normal background plane name address |
| `NxCHAddr` | 1786-1828 | 11 params + PN[4], 180+ bits | 19 bits | 280-320 | Normal background character address (NBG0/1) |
| `NxCHAddr2` | 1830-1870 | 10 params + PN[2], 120+ bits | 19 bits | 240-270 | Normal background character address (NBG2/3) |
| `NxBMAddr` | 1872-1894 | ~8 params | 19 bits | ~100 | Normal background bitmap address |
| `RxPNAddr` | 2093-2113 | 8 params, 130+ bits | 19 bits | 100-120 | Rotation background plane name address |
| `RxCHAddr` | 2115-2138 | 5 params, 60+ bits | 19 bits | 140-170 | Rotation background character address |
| `RxBMAddr` | 2140-2159 | ~6 params | 19 bits | ~80 | Rotation background bitmap address |

### Instantiation at VDP2.sv Lines 952-968

All 12 calculations execute simultaneously in a single `always_comb` block:

```systemverilog
// Plane Name addresses (6 parallel)
NxPN_ADDR[0] = NxPNAddr(NBG_PN_CNT[0], VA_PIPE[0].NxX[0], VA_PIPE[0].NxY[0], ...);  // line 952
NxPN_ADDR[1] = NxPNAddr(NBG_PN_CNT[1], VA_PIPE[0].NxX[1], VA_PIPE[0].NxY[1], ...);  // line 953
NxPN_ADDR[2] = NxPNAddr(2'b00, VA_PIPE[0].NxX[2], VA_PIPE[0].NxY[2], ...);           // line 954
NxPN_ADDR[3] = NxPNAddr(2'b00, VA_PIPE[0].NxX[3], VA_PIPE[0].NxY[3], ...);           // line 955
RxPN_ADDR[0] = RxPNAddr(VA_PIPE[0].RxX[PN_RP][11:0], VA_PIPE[0].RxY[PN_RP][11:0], ...); // line 956
RxPN_ADDR[1] = RxPNAddr(VA_PIPE[0].RxX[1][11:0], VA_PIPE[0].RxY[1][11:0], ...);      // line 957

// Character addresses (6 parallel, some conditional on bitmap mode)
NxCH_ADDR[0] = !NSxREG[0].BMEN ? NxCHAddr(...) : NxBMAddr(...);  // line 959-960
NxCH_ADDR[1] = !NSxREG[1].BMEN ? NxCHAddr(...) : NxBMAddr(...);  // line 961-962
NxCH_ADDR[2] = NxCHAddr2(...);                                    // line 963
NxCH_ADDR[3] = NxCHAddr2(...);                                    // line 964
RxCH_ADDR[0] = !RSxREG[0].BMEN ? RxCHAddr(...) : RxBMAddr(...);  // line 966-967
RxCH_ADDR[1] = RxCHAddr(...);                                     // line 968
```

**Total parallel combinational logic: ~1,400-1,600 LUTs**

---

## Detailed Function Analysis

### NxPNAddr (Normal Plane Name Address)

**Lines 1760-1784** (25 lines)

Key internal operations:
1. **Zoom mask computation** (line 1768): `ZM_MASK = {NxZMQT, NxZMQT|NxZMHF}` — 2-bit logic
2. **X offset addition** (line 1769): `OFFX = NxOFFX + {NxPN_CNT & ZM_MASK, 3'b000}` — 11-bit adder
3. **4:1 map address mux** (lines 1771-1775): `case(NxPLSZ)` selects `map_addr` from combinations of `NxMP`, `NxMPn[index]`, offset bits. The `NxMPn` indexing (`NxMPn[{NxOFFY[9], OFFX[9]}]`) creates deep mux trees (6 levels for 4 map pointers × varying bit combinations).
4. **4:1 final address mux** (lines 1776-1780): `case({NxPNB, NxCHSZ})` selects the 19-bit output from various slices of `map_addr`, `NxOFFY`, and `OFFX`.

### NxCHAddr (Normal Character Address — NBG0/1)

**Lines 1786-1828** (43 lines) — the most complex function

Key internal operations:
1. **PN selection mux** (lines 1798-1802): Selects from `NxPN[4]` array based on `NxCH_CNT` and zoom mask — 4:1 mux on 32-bit PN structures
2. **Offset calculation** (lines 1804-1808): OFFX adjusted for color depth mode — `case(NxCHCN)` with 5 branches
3. **Flip XOR operations** (lines 1810-1812): `x_offs = OFFX[3:0] ^ {4{PN.HF}}`, `y_offs = NxOFFY[3:0] ^ {4{PN.VF}}`, `ch_cnt = NxCH_CNT[2:0] ^ {3{PN.HF}}` — 3 parallel XOR operations for horizontal/vertical flip
4. **Cell offset mux** (lines 1814-1817): `case(NxCHSZ)` — 2:1 mux for 8x8 vs 16x16 character size
5. **5:1 final address mux** (lines 1818-1825): `case(NxCHCN)` with 5 color depth modes, each performing `CHRN + offset` with different shift amounts — the dominant LUT consumer (~120 LUTs)

### NxCHAddr2 (Normal Character Address — NBG2/3)

**Lines 1830-1870** (41 lines)

Structurally identical to NxCHAddr but simplified:
- Takes `NxPN[2]` instead of `NxPN[4]` (2:1 mux instead of 4:1)
- Same flip XOR, cell offset, and color depth logic
- **~240-270 LUTs** (slightly less than NxCHAddr due to smaller PN array)

### RxPNAddr (Rotation Plane Name Address)

**Lines 2093-2113** (21 lines)

Simpler than NxPNAddr — no zoom logic:
1. **4:1 map address mux** (lines 2099-2104): `case(RxPLSZ)` — similar to NxPNAddr but indexes into `RxMPn[16]` (16 map pointers vs 4), using higher coordinate bits (`RxOFFY[11:10]`, `RxOFFX[11:10]`)
2. **4:1 final address mux** (lines 2105-2110): `case({RxPNB, RxCHSZ})` — same structure as NxPNAddr

### RxCHAddr (Rotation Character Address)

**Lines 2115-2138** (24 lines)

Simpler than NxCHAddr — takes single PN instead of array:
1. **Flip XOR** (lines 2121-2122): Same pattern as NxCHAddr
2. **Cell offset mux** (lines 2124-2127): Same `case(RxCHSZ)`
3. **4:1 final address** (lines 2128-2135): `case(RxCHCN)` with 4 modes (vs 5 in NxCHAddr)

---

## VRAM Access Pipeline Architecture

### VA_PIPE Structure

Defined at `VDP2_pkg.sv` lines 1634-1681. The pipeline has 5 stages (indices [0] through [4]):

```
VA_PIPE[0]  ──► Combinational: coordinate calculation (lines 784-795)
VA_PIPE[1]  ──► Registered (1 cycle delayed)
VA_PIPE[2]  ──► Registered (2 cycles delayed)
VA_PIPE[3]  ──► Registered (3 cycles delayed)
VA_PIPE[4]  ──► Registered (4 cycles delayed)
```

Pipeline advancement at lines 806-809:
```systemverilog
VA_PIPE[1] <= VA_PIPE[0];
VA_PIPE[2] <= VA_PIPE[1];
VA_PIPE[3] <= VA_PIPE[2];
VA_PIPE[4] <= VA_PIPE[3];
```

**Key timing split:**
- **PN address calculations** use `VA_PIPE[0]` coordinates (fast path, combinational)
- **CH address calculations** use `VA_PIPE[4]` coordinates (slow path, 4-cycle delay)

This means PN and CH calculations are temporally independent — they operate on different pixel coordinates at any given cycle.

### PN → CH Data Dependency

The PN and CH calculations are not directly chained in a single cycle:

1. **Cycle T:** `NxPN_ADDR` computed from `VA_PIPE[0]` → drives VRAM read
2. **Cycle T+1 to T+3:** VRAM returns plane name data → stored in `PN_PIPE`
3. **Cycle T+4:** `NxCH_ADDR` computed from `VA_PIPE[4]` coordinates + `PN_PIPE[1]` data

The `PN_PIPE` structure (`VDP2.sv` lines 2271-2274) stores fetched plane name data through its own 5-stage pipeline, aligned so that CH calculations receive the PN data matching their (delayed) coordinates.

### VRAM Port Architecture

The VDP2 has 4 VRAM ports (lines 1134-1239):
- **VRAMA0, VRAMA1** — Bank A (2 independent address ports)
- **VRAMB0, VRAMB1** — Bank B (2 independent address ports)

Each port can service one of: NBG0-3 PN/CH, RBG0-1 PN/CH per cycle. Scheduling is determined by `VA_PIPE[0]` flag bits (`NxA0PN[4]`, `NxA1PN[4]`, `NxB0PN[4]`, etc.).

**This means at most 4 of the 12 address calculations are actually used per cycle.** The remaining 8 are computed but discarded.

---

## Why Full Serialization Is Constrained

Despite only 4 addresses being used per cycle, full serialization faces constraints:

1. **VA_PIPE[0] coordinates are available for only 1 cycle** — the pipeline shifts every clock
2. **The VRAM scheduler (lines 1134-1239) expects all 12 addresses simultaneously** to select among them based on the current fetch window
3. **Adding pipeline stages would require redesigning the VA_PIPE → VRAM → PN_PIPE → CH data flow**, which is tightly coupled

---

## Optimization Approaches

### Approach A: Share NxCHAddr Hardware Between NBG0-3 (Recommended)

**Concept:** NxCHAddr and NxCHAddr2 have ~70% structural overlap. The 5:1 color depth mux (lines 1818-1825 and 1860-1867) and flip XOR logic (lines 1810-1812 and 1852-1854) are identical. Create a single parameterized function that handles both the 4-PN and 2-PN variants.

**Current:**
- NxCHAddr (NBG0/1): ~300 LUTs × 2 = 600 LUTs
- NxCHAddr2 (NBG2/3): ~255 LUTs × 2 = 510 LUTs
- Total: 1,110 LUTs

**Optimized:** Single `NxCHAddrUnified` with PN-count parameter:
- Shared color depth mux: 120 LUTs (shared, not duplicated)
- Shared flip XOR: 30 LUTs (shared)
- PN selection: 50 LUTs (4:1) + 20 LUTs (2:1) = 70 LUTs
- Per-instance unique logic: ~100 LUTs × 4 = 400 LUTs
- Shared infrastructure: ~150 LUTs
- Total: ~620 LUTs

**Savings:** ~490 LUTs (~250 ALMs)
**Risk:** Low — same logic, consolidated
**Effort:** Low-Medium

### Approach B: Merge PN Address Functions

**Concept:** NxPNAddr and RxPNAddr share the same structure (plane size mux → final address mux). The key differences are:
- Nx has zoom adjustment; Rx does not
- Rx uses 12-bit coordinates; Nx uses 11-bit
- Rx indexes into 16 map pointers; Nx into 4

A single `PNAddrUnified` function with a mode parameter could share the map address mux and final address mux.

**Current:** ~130 LUTs × 4 (Nx) + ~110 LUTs × 2 (Rx) = 740 LUTs
**Optimized:** ~120 LUTs shared + ~50 LUTs × 6 instances = 420 LUTs

**Savings:** ~320 LUTs (~160 ALMs)
**Risk:** Low
**Effort:** Low-Medium

### Approach C: Color Depth Case-to-Shift Conversion

**Concept:** The 5:1 `case(NxCHCN)` in NxCHAddr (lines 1818-1825) selects different shift amounts for the character number base address. The cases are:

| NxCHCN | Color Depth | Shift |
|--------|------------|-------|
| 3'b000 | 16 colors (4bpp) | CHRN << 5 |
| 3'b001 | 256 colors (8bpp) | CHRN << 6 |
| 3'b010 | 2048 colors (11bpp) | CHRN << 7 |
| 3'b011 | 32768 colors (15bpp) | CHRN << 7 |
| 3'b100 | 16M colors (24bpp) | CHRN << 8 |

This can be replaced with a barrel shifter: `CHRN << (NxCHCN + 5)` (with saturation at 8). A 3-bit controlled shift on a 16-bit value is cheaper than a 5:1 × 19-bit multiplexer.

**Savings:** ~50-80 LUTs per NxCHAddr instance × 4 = 200-320 LUTs (~100-160 ALMs)
**Risk:** Low — arithmetic equivalence is straightforward to verify
**Effort:** Low

### Approach D: Register Address Outputs

**Concept:** Currently all 12 address outputs are purely combinational, feeding directly into the VRAM port assignment mux (lines 1134-1239). Adding a register stage between address calculation and VRAM assignment would:
1. Break the combinational critical path
2. Allow the synthesis tool more freedom in resource sharing
3. Enable retiming optimizations

The cost is 12 × 19 = 228 register bits (trivial) and 1 cycle additional latency. The VA_PIPE would need to be extended from 5 to 6 stages to compensate.

**Savings:** Indirect — better synthesis optimization potential, estimated 100-200 ALMs
**Risk:** Medium — requires adjusting the entire display pipeline by 1 cycle
**Effort:** Medium

---

## Recommended Implementation Order

| Step | Approach | Savings | Effort | Risk |
|------|----------|---------|--------|------|
| 1 | C: Case-to-shift conversion | 100-160 ALMs | Low | Low |
| 2 | A: Merge NxCHAddr/NxCHAddr2 | ~250 ALMs | Low-Medium | Low |
| 3 | B: Merge PN address functions | ~160 ALMs | Low-Medium | Low |
| 4 | D: Register address outputs | 100-200 ALMs | Medium | Medium |

**Steps 1-3** yield 510-570 ALMs with low risk and moderate effort.
**Step 4** provides additional savings but requires pipeline timing adjustment.

---

## Verification Strategy

1. **Scroll test patterns:** Saturn VDP2 test ROMs that exercise all 4 NBG layers + RBG with various scroll, zoom, and color depth settings.
2. **Bitmap mode:** Verify NxBMAddr/RxBMAddr conditional paths still work when BMEN is set.
3. **Rotation backgrounds:** Test rotation with various coefficient table settings — these stress RxPNAddr/RxCHAddr with dynamically changing coordinates.
4. **Commercial games:** Games known to use complex VDP2 configurations:
   - Radiant Silvergun (heavy NBG + RBG usage)
   - Panzer Dragoon Saga (rotation backgrounds)
   - Soukyugurentai (multiple scroll layers with zoom)
