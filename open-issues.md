# Open issues — CPU32 / 68332 support

Tracking known gaps in the CPU32 work on the `cpu32` branch. The disassembler is
cross-checked against binutils (`cpu32` machine) in the Runner; execution has no
independent oracle because Musashi has no CPU32 core.

## TBL execution semantics are unvalidated

The TBL family (TBLS/TBLSN/TBLU/TBLUN) disassembles correctly and is verified against
binutils across the full opcode space. **Execution, however, has not been
cross-checked against any reference** — Musashi has no CPU32 core, so the Runner only
compares the CPU32 disassembler, not the executed result (`compareDasm` skips Musashi
for `M68332`; `skip()` bypasses the execution comparison entirely).

The interpolation core and round-half-away-from-zero behavior were validated only by
the hand-written checks in development (basic interpolation at fraction 0/128/255, the
×256 surplus form, signed endpoints). The following aspects follow my best reading of
the MC68332 / CPU32 Reference Manual and **need verification against the manual or real
hardware**:

- **Rounding tie-break.** The rounded forms use round-half-away-from-zero
  (`tblInterpolate` in `Moira/MoiraExec_cpp.h`). Confirm the exact tie-handling the
  CPU32 uses, including the sign convention for `TBLS`.
- **Surplus-form result width.** The not-rounded forms (`TBL*N`) currently write the
  full `En*256 + (En+1-En)*f` result to `Dx` as a longword via `writeD<Long>`. Confirm
  the documented destination width and whether the operand size affects it.
- **Condition codes.** Currently N/Z are set from the result, V and C are cleared, X is
  untouched. Confirm CCR.V behavior (overflow) and the exact N/Z definition for both
  the rounded and surplus forms.
- **Memory-form address arithmetic.** The memory form reads ENTRY[n] at the effective
  address and ENTRY[n+1] at `ea + entrySize` (mirroring `execChkCmp2`). This assumes
  the EA already points at ENTRY[n] (the integer part of the independent variable is
  applied by the addressing mode, not by the instruction). Confirm against the manual.
- **Invalid extension words at execution time.** The disassembler rejects reserved-bit
  ext words (matching binutils via `isValidExt`), but the *execution* handlers
  (`execTblReg` / `execTblEa`) do not raise an illegal-instruction exception for
  reserved (non-size) bits being set — they execute it. Decide whether exec should trap
  on those, mirroring the dasm behavior. (The size field `0b11` is already trapped as
  illegal in both exec and dasm.)

### How to validate once an oracle exists
- Build a CPU32 test corpus (e.g. assembled with `gas -mcpu32`) with known
  inputs/outputs, or run against real 68332 hardware / a trusted CPU32 simulator.
- Extend the Runner to compare executed CPU32 results against that reference, then
  re-enable the execution comparison for `M68332` (currently short-circuited in
  `skip()`).

## Other CPU32 gaps (lower priority)

- **Memory-indirect addressing modes** are not rejected for CPU32. The CPU32 supports
  scaled indexing and base displacement but not the full memory-indirect modes
  (`([bd,An],Xn,od)`). Enforcement would live in the EA decode path
  (`MoiraDataflow_cpp.h` / `MoiraDasm`), not the init table. Does not block running
  real CPU32 code.
- **CPU32 instruction timing** is not modeled. CPU32 cycle counts differ from the
  68020; Moira's C68020 core is non-cycle-exact, and the TBL handlers use nominal
  cycle values.
