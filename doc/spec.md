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
### Sequential implementation requirements

- The asynchronous reset must initialize all state-holding registers to deterministic values, including `busy`, `counter`, operand registers, sign/exponent/mantissa registers, product registers, rounding registers, and output registers.
- The first operation accepted after reset must therefore produce a deterministic result.
- Each clocked `case(counter)` branch may contain multiple logical operations that depend on one another.
- Do not assume that a non-blocking assignment (`<=`) updates a register immediately within the same clocked branch. If a later calculation in the same stage depends on an earlier calculation, use local temporary variables with blocking assignments (`=`) and perform the dependent operations in order.
- Commit the final values to the actual state registers with non-blocking assignments (`<=`) after the intermediate calculations are complete.
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

Stage 6 is one FSM state, but its operations are logically sequential within
that clock cycle. Use local working values for the mantissa, exponent, guard,
round, and sticky information so that each sub-step sees the result of the
previous sub-step.

The required order is:

1. Underflow alignment
2. Normalization
3. Round-to-Nearest-Even

#### Sub-step A — Underflow alignment

- If the intermediate exponent `z_e` is below `-126`, calculate:

  `sh = -126 - $signed(z_e)`

- Shift the retained significand right by `sh` so that the working exponent
  reaches `-126`.
- Every `1` bit discarded from the retained significand must contribute to
  the sticky condition.
- Existing guard and round information that is shifted out of the retained
  precision must also contribute to sticky rather than being discarded.
- If the shift removes all retained significand bits, the retained
  significand becomes zero, while sticky remains set whenever any discarded
  information was nonzero.
- After this alignment, the working exponent is `-126`.

#### Sub-step B — Normalization

- If underflow alignment was not required and the working significand does
  not have the required leading `1` at bit 23, shift it left to restore the
  required normalized position.
- Decrement the working exponent for each left shift.
- Preserve rounding information when bits cross the rounding boundary.
- **Do not perform a left shift if it would make the exponent less than
  `-126`.** In particular, when the working exponent is `-126` and bit 23 of
  the working mantissa is zero, stop at the `-126` boundary.

#### Sub-step C — Round-to-Nearest-Even

- Perform rounding only after the underflow and normalization operations above
  are complete.
- Let `LSB` be the least-significant bit of the final retained mantissa.
- The exact RNE increment condition is:

  `round_up = G && (R || S || LSB)`

- The G/R/S values used for rounding must be the updated values resulting from
  the Stage 6 operations, not values from before a shift or normalization.
- If `round_up` is true, increment the retained mantissa by one.
- If the increment carries out of the retained mantissa, set the mantissa to
  `24'h800000` and increment the exponent by one.

After these sub-steps, write the final working mantissa, exponent, guard,
round, and sticky values back to `z_m`, `z_e`, `guard_bit`, `round_bit`, and
`sticky`.

### Stage 7 — Pack

- For the normal path, pack the final sign, biased exponent, and fraction
  into the 32-bit FP32 result.
- If the final unbiased exponent is greater than `127`, output signed
  infinity: `{z_s, 8'hFF, 23'd0}`.
- If the final exponent is exactly `-126` and the retained mantissa is not
  normalized (`z_m[23] == 0`), force the packed exponent field to zero while
  retaining the mantissa produced by Stage 6.
- Do not perform another independent general-purpose denormal conversion in
  Stage 7.
- Update `z`, assert `out_valid` for exactly one clock cycle, and clear
  `busy`.
---

## Assumptions & Constraints

- Inputs used for grading are finite, normalized IEEE-754 binary32 numbers.
- Therefore, operand exponent fields are in `1..254`.
- Operand zero, subnormal, infinity, and NaN cases are outside the required
  grading domain.
- The implementation should prioritize correct bit-exact multiplication of
  normal FP32 operands, including normalization, overflow handling, and
  round-to-nearest-even behavior.
- The interface, handshake behavior, and 7-cycle latency remain mandatory.

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands,

---


