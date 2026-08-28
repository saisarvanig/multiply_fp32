# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design currently targets:
- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- The design behaves as: z = a*b 
- z, a and b are single precision 32-bit IEEE-754 numbers

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in | 1 | Clock |
| `rst`   | in | 1 | Async reset (posedge) |
| `valid` | in | 1 | **1-cycle start pulse**; accepted only when not busy |
| `a`     |  in | 32 | Operand A (FP32 bits) |
| `b`     | in | 32 | Operand B (FP32 bits) |
| `z`         | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |

### Handshake contract
- When `busy==0`, a high `valid` on a rising edge **starts** an operation:
  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
  - The FSM begins at `counter = 1`.
- While `busy==1`, new `valid` pulses are **ignored**.
- When the operation completes:
  - `z` is updated,
  - `out_valid` pulses high for 1 clock cycle,
  - `busy` is cleared.

---

## Latency and Throughput

### Latency
- Fixed latency of **7 stages**.
- In this implementation the operation begins at stage `counter=1` and completes at `counter=7`.
- `out_valid` asserts on the cycle where stage 7 packing finishes.

The start edge is the rising clock edge where `valid=1` is sampled while
`busy=0`. This is cycle N.

The required timing is:

- Cycle N: accept `valid`, register `a` and `b`, and start the operation.
- Cycle N+1: Stage 1 — Unpack.
- Cycle N+2: Stage 2 — Special classification + denormal setup.
- Cycle N+3: Stage 3 — Input normalization.
- Cycle N+4: Stage 4 — Multiply core.
- Cycle N+5: Stage 5 — Extract mantissa and rounding bits.
- Cycle N+6: Stage 6 — Normalize and round.
- Cycle N+7: Stage 7 — Pack; update `z` and assert `out_valid`.

Therefore, if an operation is accepted on rising edge N, `out_valid` must
assert on rising edge N+7. `out_valid` must remain high for exactly one
clock cycle.
### Throughput
- **Not pipelined** (single-issue).
- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

---

## Internal Data Model (IEEE-754 binary32)
For each operand:
- `sign` = bit 31
- `exp`  = bits 30:23 (biased exponent)
- `mant` = bits 22:0 (fraction)

Internal signals:
- `a_s, b_s, z_s`: sign bits
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE

---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- For normal operation:
  - If exponent is nonzero => sets implicit leading 1: `a_m[23] = 1`.
  - If exponent is zero (subnormal) => forces exponent to -126 (subnormal exponent baseline).

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.

### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e + 1`
- Mantissa product: `product = a_m * b_m * 4`
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)

**Stage 6 implementation and verification requirement**

Stage 6 is a complete same-cycle calculation. The implementation must perform
the required underflow alignment, normalization, and RNE rounding while
`counter == 6`, using the result of each sub-step as the input to the next
sub-step.

Use local working variables with blocking assignments (`=`) for intermediate
Stage 6 calculations, and commit the final values to the registered state
signals with non-blocking assignments (`<=`) at the end of the Stage 6 branch.

Do not replace the required Stage 6 algorithm with a simplified normal-input
fast path.

Do not move any Stage 6 arithmetic into Stage 7. Stage 7 is only responsible
for packing the already-finalized Stage 6 result and generating `out_valid`.

Do not perform speculative rewrites merely because the code uses block-local
temporary variables. If the RTL compiles successfully, preserve the working
implementation structure. Change the implementation only when an actual
compiler, simulator, or test failure requires it.

If block-local temporary declarations cause a real compiler error, use
module-level temporary signals while preserving the same blocking-assignment
calculation order. Do not remove or postpone the Stage 6 algorithm merely to
avoid temporary variables.

The intended Stage 6 flow is:

1. Start from working copies of `z_m`, `z_e`, `guard_bit`, `round_bit`, and
   `sticky`.
2. Perform underflow alignment and update the working G/R/S information.
3. Perform required normalization using the updated working values.
4. Perform RNE using the updated working mantissa and G/R/S values.
5. Handle rounding carry by renormalizing the mantissa and incrementing the
   exponent when required.
6. Commit the final working values to the registered state signals.
7. Advance to `counter == 7`.

This stage performs:
1. **Underflow alignment** toward the minimum normal exponent:
   - If the intermediate exponent `z_e` is below `-126`, the significand
     must be shifted right enough to bring the exponent to `-126`.
   - Every bit discarded by this right shift must contribute to the sticky
     condition used for rounding.
   - The existing guard, round, and sticky information must not be silently
     discarded when performing this alignment; the implementation must
     preserve enough information to make the final RNE decision correctly.
   - After alignment, the exponent used for packing must represent the
     resulting value at the `-126` boundary.

2. **Normalize** when the significand is not in the required normalized
   position:
   - If the most-significant significand bit is missing, shift the
     significand left until the required leading bit is restored.
   - Decrement the exponent for every left shift.
   - Bits shifted through the rounding boundary must be incorporated into
     the rounding information rather than discarded.
3. **RNE rounding**:
   - Perform round-to-nearest-even using the values of `G`, `R`, `S`, and
  the current least-significant retained mantissa bit (`LSB`) after all
  normalization and underflow alignment are complete.
- The rounding increment condition is exactly:
  `round_up = G && (R || S || LSB)`.
- Do not make the rounding decision using G/R/S values from before a
  normalization or underflow shift.
- If rounding increments the retained mantissa and causes a carry into
  the next exponent position, renormalize the mantissa and increment the
  exponent by one.
   - Handles carry-out from rounding:
     - If rounding overflows mantissa, set mantissa to 0x800000 and increment exponent.

### Stage 7 — Pack
- For normal path:
  - Pack sign, biased exponent, fraction.
  - If exponent indicates overflow -> output INF.
  - If exponent indicates exact denorm boundary -> force exponent field to 0 (denormal/zero representation).
- Asserts `out_valid` for one cycle and clears `busy`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands,

---


